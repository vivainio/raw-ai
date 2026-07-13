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

## Where naive RAG breaks

Everything above runs *before* generation starts, once, unconditionally,
and application code — not the model — decides the query, the k, and
whether to search at all. That's fine for a fixed corpus and a
well-worded query, and it falls apart the moment either isn't true:

- There's no checkpoint where the model evaluates what came back. It never
  gets a turn to notice "these three chunks don't actually answer this"
  and go fetch different ones — the search result is already baked into
  the prompt by the time the model produces any output at all, so its only
  option is to answer with whatever the top-k search returned, wrong or
  not.
- A single query embedding can't represent a question that actually needs
  two different lookups — "compare last quarter's revenue to this
  quarter's" is one question and two retrievals.
- Every answer pays for retrieval whether it's needed or not. "What's 2+2"
  gets the same chunk-stuffing treatment as a question that genuinely
  depends on the corpus.

The fix for all three is the same: stop running retrieval once, before
generation starts, with no way back. Make it a tool call the model can
invoke, inspect the result of, and invoke again — inside the loop, not in
front of it.

## Integrating with an agent: retrieval as a tool call

The [Agent Dialogue](../agent-dialogue/index.md) chapter covers the actual
mechanism a coding agent uses to run a shell command: the model predicts a
`tool_use` block, the harness executes it for real, the result comes back
as a `tool_result` in the next message. Retrieval fits that same shape
exactly — a search function with a name, a description, and a schema,
callable mid-conversation instead of pre-run before it:

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
That's the entire upgrade from "naive RAG" to what's usually called
agentic RAG: nothing new gets added to the protocol, retrieval just moves
from a preprocessing step outside the loop to a tool call inside it.

The practical differences follow directly from that move:

- **Cost is conditional.** A question the model can already answer, or one
  that doesn't touch the corpus at all, skips the tool call entirely
  instead of always paying for a retrieval it doesn't need.
- **Multi-hop questions work.** "Compare X and Y" can become two `search_docs`
  calls with two different queries in the same turn, each returning its own
  chunks, instead of one query embedding trying to represent both halves at
  once.
- **A bad first retrieval is recoverable.** If the returned chunks don't
  answer the question, the model can reformulate the query and call the
  tool again — something a pre-loop retrieval step has no mechanism to do,
  since by the time it would notice, generation has already started.
- **The tool implementation still does the real work.** Chunking, embedding,
  the similarity search, any reranking — none of that goes away or gets
  simpler. Making retrieval agentic changes *when* it's invoked and by
  *whom*, not what happens inside `search_docs` itself.

The corpus-search half of RAG is unglamorous information retrieval, done
the same way whether an agent is involved or not. The "agentic" half is
just an ordinary tool-use loop, the same one driving every shell command
in this book, pointed at a search function instead of a filesystem.
