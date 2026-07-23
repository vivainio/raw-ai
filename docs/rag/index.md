---
icon: lucide/search
---

# RAG

Start from something already familiar: full-text search. Grep the corpus,
or run it through Postgres's `tsvector`, and paste the matching lines into
the prompt — that's already retrieval-augmented generation, in the literal
sense of "text fetched at answer time instead of baked into the weights."
What's usually meant by "RAG" often adds vector search: instead of requiring
shared words, the search step compares learned embeddings, so a query and a
chunk that share no vocabulary can still rank near each other. Calling this
"matching meaning" is useful shorthand, but embeddings capture statistical
similarity, not a guaranteed semantic judgment.

That gap is a major part of the value proposition. Many real queries don't
share the important words with the answer — "how do I get my money back"
doesn't contain
"refund," and a support ticket describing a bug rarely uses the phrasing
of the changelog entry that fixed it. `grep "money back"` returns nothing;
a full-text index gets partial credit if stemming happens to line up; an
embedding of the query lands close to the embedding of the refund-policy
chunk regardless of phrasing, because it's comparing meaning instead of
tokens.

The limitation appears in reverse on exact identifiers. An error code, a
function name, or a SKU is a natural lexical-search query: lexical search
finds it directly and cheaply, while a similarity search can bury an exact
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
3. **Embed the query** with a compatible model at answer time, search the
   stored vectors using cosine similarity or another configured distance,
   and keep the top-k candidates. An index usually approximates this search
   rather than comparing against every vector.
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

Handed that tool, the model can decide *whether* to call it at all, decide
*what query* to send — which doesn't have to be the user's literal
wording, it can be a reformulation aimed straight at the corpus — and can
call it again with a different query if the first batch of chunks didn't
answer the question. A `tool_result` full of retrieved passages isn't
structurally different from a `tool_result` full of `ls` output; the model
reads it, decides if it's enough, and either answers or asks for more.

That gets you:

- **Retrieval cost is conditional.** A question the model can already answer, or one
  that doesn't touch the corpus at all, skips the tool call entirely
  instead of always paying for a retrieval it doesn't need.
- **Multi-hop questions work.** "Compare X and Y" can become two
  `search_docs` calls with two different queries in the same turn, each
  returning its own chunks, instead of relying on one query embedding to
  represent both halves at once.
- **A bad retrieval is recoverable.** If the returned chunks don't answer
  the question, the model can reformulate the query and call the tool
  again.

The tool can expose the same retrieval pipeline as the code in the last
section — query embedding, search, and any reranking. Chunking and document
embedding normally happened earlier at index time. What changes is
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
- A naive one-shot implementation sends one query even when the question
  needs several lookups. Application code can decompose the question or run
  multiple searches, but then it is taking on orchestration that an agent
  loop could otherwise perform.
- Every call pays for retrieval whether the question needs it or not —
  there's no cheap "skip it" path when nothing's deciding whether to
  search in the first place.

That's a real tradeoff, not a mistake — worth taking when the corpus is
narrow enough that a wrong top-k is unlikely, or when there's no agent
loop to hand the decision to in the first place. Outside those cases, the
tool call is the one worth reaching for by default.

## RAG over code

Everything above assumed an English corpus. Code raises a real question:
does a function need an explicit English explanation sitting next to it
before embedding search can find it at all?

Not strictly. Code-capable embedding models can learn from several signals:
paired code and natural language, identifiers and comments, and structural
patterns in the code itself. Names such as `apply_discount` provide a strong
bridge to an English query about discounts, while `f(a, b)` removes that
signal and is therefore harder to retrieve. It is too strong to say syntax
contributes nothing—the arithmetic and control-flow patterns still affect
the representation—but terse names and missing context make natural-language
retrieval less reliable.

Two fixes exist for code too terse to carry that signal on its own:
generate a synthetic one-line summary per function at index time — an
LLM writes it, not a human — and embed the summary instead of the raw
code; or use an embedding model trained specifically for code↔NL
alignment rather than a general text embedding model, since it learned
that correspondence more thoroughly to begin with.

In practice, most of that apparatus gets skipped for code specifically,
and the reason is the agentic-search point from earlier applied at full
strength: if the model does the searching itself, it can try several
keyword phrasings — a function name it suspects, then a related error
string, then an import — the same way a person would grep by hand,
without needing an embedding space to already know those terms are
related. Code queries usually already share a literal token with the
target, which favors lexical search outright. This isn't theoretical for
this book: Claude Code, the tool writing it, searches codebases with a
`Grep` tool, not an embedding index. Asked where refunds are handled, the
model doesn't run a similarity search over a vector store — it tries
`grep -ri refund`, and if that comes up empty, tries `reimburs`, or
`credit.*back`, reformulating by retry the same way the tool-call section
above described, just pointed at source instead of a document corpus.
For many repositories the corpus is small enough and the queries lexical
enough that the tool-call loop works well without a vector index. Whatever
docstring a human already wrote for their own benefit does double duty as
the retrieval signal—no separate embedding pipeline required.

## Older than tool calling

Everything above treats retrieval as one tool among others, callable
mid-conversation the same way `Bash` or `Grep` is. That's not how the term
started. "RAG" comes from a 2020 paper — "Retrieval-Augmented Generation
for Knowledge-Intensive NLP Tasks" — written well before commercial LLM
APIs offered reliable function calling at all, which didn't become
standard until around 2023. In 2020 there was no loop to put a search step
inside. The only shape available was retrieve first, generate second,
done — exactly the one-shot pipeline this chapter calls the alternative.

That history is most of the reason RAG gets treated as a discipline of its
own — chunking strategies, reranking models, vector database vendors,
entire job titles — rather than what it structurally is: a search step,
wired up either as fixed code that runs before the model or as a tool the
model calls when it decides to. The vocabulary and tooling built up around
it during the years before agents could call tools has stuck around even
though the constraint that produced it — no loop to put search inside —
mostly stopped applying once function calling shipped. Meeting the term
today, after tool-calling agents were already normal, gives a reader every
reason to assume "RAG" names something as fundamental as embeddings or
attention. It names a search step from an era when there was nowhere else
to put one, plus the infrastructure built to make that one, fixed search
step as good as possible — chunking, embedding, ranking — which is still
exactly what's running inside the tool call. It's just no longer stuck
running once, up front, with no way back.
