---
icon: lucide/search
---

# RAG

Start from something already familiar: full-text search. Grep the corpus,
or run it through Postgres's `tsvector`, and paste the matching lines into
the prompt — that's already retrieval-augmented generation, in the literal
sense of "text fetched at answer time instead of baked into the weights."
What's usually meant by "RAG" adds exactly one idea on top of that: instead
of matching shared *words*, the search step matches shared *meaning*, via
embeddings, so a query and a chunk that share no vocabulary at all can
still be found relevant.

That gap is the whole value proposition. Most real queries don't share
words with the answer — "how do I get my money back" doesn't contain
"refund," and a support ticket describing a bug rarely uses the phrasing
of the changelog entry that fixed it. `grep "money back"` returns nothing;
a full-text index gets partial credit if stemming happens to line up; an
embedding of the query lands close to the embedding of the refund-policy
chunk regardless of phrasing, because it's comparing meaning instead of
tokens.

It's also the whole limitation, in reverse: embeddings are worse than grep
at anything exact. An error code, a function name, a SKU — grep finds
those instantly and for free, and a similarity search can bury an exact
match under a pile of "semantically close" noise instead of surfacing it
first. That asymmetry is why systems that actually ship rarely pick one or
the other — lexical search (BM25, Postgres full-text, plain grep) and
vector search run side by side, results merged or reranked, so exact terms
don't lose to paraphrases and paraphrases don't lose to exact terms.

The rest of this chapter covers the embedding half — the part full-text
search doesn't already give you — and how to wire retrieval, of either
kind, into an agent's tool-call loop instead of running it once before
generation starts.

## How to write one

Strip away the vector-database marketing and the pipeline is four steps,
each doing one specific job:

1. **Chunk** the source documents into pieces small enough to be individually
   relevant — a paragraph, not a whole PDF, since retrieving a 40-page
   document because one sentence in it matched isn't retrieval, it's just
   moving the search problem into the prompt.
2. **Embed** each chunk: run it through an embedding model and keep the
   resulting vector alongside the chunk's text. This happens once, offline,
   before any question is asked.
3. **Embed the query** the same way, at answer time, and compare it against
   every stored vector — cosine similarity, ranked, top-k kept.
4. **Assemble**: the k highest-scoring chunks get concatenated into the
   prompt, and the model generates its answer with that text in front of it.

None of that requires a dedicated vector database — it requires a list of
vectors and a distance function. Here it is with no framework, using
Anthropic's embeddings-adjacent building blocks stood in for by plain numpy
math, to show what a "vector search" actually reduces to:

```python
import numpy as np

def embed(text: str) -> np.ndarray:
    # stand-in for a real call to an embedding model
    ...

def cosine(a: np.ndarray, b: np.ndarray) -> float:
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))

chunks = ["Refunds are processed within 5 business days.",
          "The API rate limit is 100 requests per minute.",
          "Support hours are 9am-5pm Eastern, Monday to Friday."]
chunk_vectors = [embed(c) for c in chunks]

def retrieve(query: str, k: int = 2) -> list[str]:
    q = embed(query)
    scored = [(cosine(q, v), c) for v, c in zip(chunk_vectors, chunks)]
    scored.sort(key=lambda x: x[0], reverse=True)
    return [c for _, c in scored[:k]]

context = "\n".join(retrieve("how long do refunds take?"))
prompt = f"Answer using only this context:\n{context}\n\nQuestion: how long do refunds take?"
```

A real system swaps `embed` for an actual API call and `chunk_vectors` for
an index (FAISS, pgvector, a hosted vector store) so the linear scan above
doesn't have to run against millions of chunks on every query. Nothing
else about the shape changes — it's still "embed, compare, keep the top
few, paste them into the prompt."

## As a tool call

The [Agent Dialogue](../agent-dialogue/index.md) chapter covers the actual
mechanism a coding agent uses to run a shell command: the model predicts a
`tool_use` block, the harness executes it for real, the result comes back
as a `tool_result` in the next message. Retrieval works the same way —
wrap `retrieve()` in a tool definition and the model calls it exactly like
it would call `Bash` or `Read`:

```python
search_docs_tool = {
    "name": "search_docs",
    "description": "Search the internal knowledge base for relevant passages.",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"],
    },
}

def search_docs(query: str) -> list[str]:
    return retrieve(query, k=3)
```

Handed that tool, the model decides *whether* to call it at all, decides
*what query* to send — which doesn't have to be the user's literal
wording, it can be a reformulation aimed straight at the corpus — and can
call it again with a different query if the first batch of chunks didn't
answer the question. A `tool_result` full of retrieved passages isn't
structurally different from a `tool_result` full of `ls` output; the model
reads it, decides if it's enough, and either answers or asks for more.

That gets you:

- **Cost is conditional.** A question the model can already answer, or one
  that doesn't touch the corpus at all, skips the tool call entirely
  instead of always paying for a retrieval it doesn't need.
- **Multi-hop questions work.** "Compare X and Y" can become two
  `search_docs` calls with two different queries in the same turn, each
  returning its own chunks, instead of one query embedding trying to
  represent both halves at once.
- **A bad retrieval is recoverable.** If the returned chunks don't answer
  the question, the model can reformulate the query and call the tool
  again.

The tool is doing the exact same work as the code in the last section —
chunking, embedding, the similarity search, any reranking. What changes is
*who* decides when it runs and with what query: the model, mid-conversation,
instead of your code, once, up front.

## The one-shot alternative

Sometimes there's no loop to put the tool call inside — a single-turn
summarization endpoint, a classification pipeline, anything that isn't
running an agent at all. Retrieval still has a place there: call
`retrieve()` yourself, once, before the model's only turn, and paste the
result straight into the prompt, the way the code example two sections
back did.

The cost of skipping the tool call is that the model never gets a chance
to react to what came back:

- Application code — not the model — decides the query, the k, and
  whether to search at all, and that decision is locked in before the
  model produces any output. If the search ranked the wrong chunks
  highest, there's no recovery; the model answers with whatever it got,
  right or wrong.
- A single query embedding can't represent a question that needs two
  different lookups — "compare last quarter's revenue to this quarter's"
  is one question and two retrievals, and a one-shot search only gets one.
- Every call pays for retrieval whether the question needs it or not —
  there's no cheap "skip it" path when nothing's deciding whether to
  search in the first place.

That's a real tradeoff, not a mistake — worth taking when the corpus is
narrow enough that a wrong top-k is unlikely, or when there's no agent
loop to hand the decision to in the first place. Outside those cases, the
tool call is the one worth reaching for by default.
