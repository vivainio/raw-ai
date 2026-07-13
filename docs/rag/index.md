---
icon: lucide/search
---

# RAG

RAG — retrieval-augmented generation — means putting text into the prompt
that wasn't there when the model was trained, fetched from somewhere at the
moment of answering rather than baked into the weights. The name makes it
sound like a technique. It's really just two ordinary things stapled
together: a search step, and a generation step that gets to read the search
results before it writes anything.

The reason it exists at all: a model's parametric knowledge is frozen at
training time, can't be updated without retraining, and never contained
your private documents, your company's internal wiki, or last week's
support tickets in the first place. Retrieval sidesteps all three problems
by not asking the model to *know* the answer — only to *read* it, the same
way you'd hand a colleague the right page instead of expecting them to have
memorized the manual.

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

Everything above runs *before* generation starts, once, unconditionally.
That's fine for a fixed corpus and a well-worded query, and it falls apart
the moment either isn't true:

- The retrieval happens before the model has said anything, so there's no
  way to notice "these three chunks don't actually answer this" and go
  fetch different ones — the model gets one shot at whatever the top-k
  search returned, wrong or not.
- A single query embedding can't represent a question that actually needs
  two different lookups — "compare last quarter's revenue to this
  quarter's" is one question and two retrievals.
- Every answer pays for retrieval whether it's needed or not. "What's 2+2"
  gets the same chunk-stuffing treatment as a question that genuinely
  depends on the corpus.

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
