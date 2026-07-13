---
icon: lucide/home
---

# Raw AI

Notes on AI as it's actually used, not as it's marketed. Educational
material, technical analysis, and opinions — no press-release language, no
"revolutionary paradigm shift" copy, no repeating a vendor's talking points
because everyone else repeats them. If something works, that's said once,
with the reason. If it doesn't, that's said too.

It grows one topic at a time, not from a fixed curriculum.

Text, not video. Most chapters here read in a few minutes — closer to a
sharp blog post than a training course. No 40-minute talk to sit through
for ten minutes of content, no podcast to have on in the background hoping
something useful lands. If a topic takes three paragraphs, it gets three
paragraphs and stops.

Most video and podcast content about AI ends up disappointing anyway —
not because the format is wrong, but because so much of it stalls at the
same entry-level material: what a prompt is, what a token is, the same
three demos everyone already did. That's fine as a first pass, but it's
the first ten minutes of every video, forever, rarely the ninety after it.
Text doesn't fix that by itself, but it makes padding much harder to hide.

Written with AI — which is very fitting for a book about AI. The human role
here is closer to curator or producer than author — deciding what's worth
covering, what's accurate, and what gets cut — not typing every sentence by
hand.

Most examples are drawn from Claude Code, since that's the tool this book
is written with and the one most readers will have actually touched. Where
it matters, the underlying mechanism — system prompts, tool calls, memory
files — is common across coding agents; the vendor-specific parts are the
directory names and file formats, not the shape of the thing.

## Chapters

- [Agent Skills](agent-skills/index.md) — what a skill actually is, keeping
  one skill set across tools, managing skill repos with
  [skillset](https://github.com/vivainio/skillset), and when a CLI plus a
  skill beats an MCP server.
- [Agent Dialogue](agent-dialogue/index.md) — what's actually in the system
  prompt, how a tool call is just a predicted block of JSON, and why asking
  you a question is the same mechanism as running a shell command.
- [Agent Memory](agent-memory/index.md) — how Claude Code saves notes about
  you and the project between sessions: where the files live, what's in
  them, and why they're never fully trusted on read.
- [Theory: Tokenization](tokenization/index.md) — the byte-pair-encoding
  algorithm that builds a token vocabulary, why Claude's own tokenizer
  changed between model generations, and why code and non-English text
  cost more tokens per character than English prose does.
- [Hosting: AWS Bedrock](aws-bedrock/index.md) — the model-hosting layer
  underneath AgentCore, Strands, and Pydantic AI: one API across
  providers, plus Guardrails, Knowledge Bases, and the classic Agents
  product AgentCore didn't replace.
- [Hosting: AWS AgentCore](aws-agentcore/index.md) — pulling apart AWS's
  bundle of a dozen billed services to see which parts are genuinely new
  and which are existing AWS primitives with an agent-shaped label.
- [Frameworks: Pydantic AI](pydantic-ai/index.md) — a Python-native agent
  library: typed dependencies and structured output, deferred tools for
  pausing on approval, and how much of AgentCore's surface stays manual
  no matter which framework calls it.
- [Frameworks: Strands Agents](strands-agents/index.md) — AWS's own
  open-source framework, the one running headless inside AgentCore's
  Harness, compared against Pydantic AI on session state, multi-agent
  orchestration, and pausing for human approval.
- [Frameworks: Microsoft Agent Framework](microsoft-agent-framework/index.md) —
  Semantic Kernel and AutoGen merged into one project, compared against
  Pydantic AI and Strands on swappable model clients, workflows, and
  hosting on Microsoft's own Foundry runtime.
- [The OpenAI API](openai-api/index.md) — how the Chat Completions shape
  became the thing every inference server clones, where that cloning stops
  at the Responses API's statefulness and hosted tools, and what OpenAI's
  own churn from Assistants to Responses says about the rest.
