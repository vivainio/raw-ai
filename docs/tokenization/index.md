---
icon: lucide/split
---

# Theory: Tokenization

This book's own intro admits that most AI content stalls at "what a token
is" and never gets past it. Fair — here's the part that usually gets
skipped: the actual merge algorithm that builds a token vocabulary, why two
Claude models released eight months apart can tokenize the same sentence
into a different number of pieces, and why that number is worse for code
and non-English text than for plain English prose. None of this is
Claude-specific mechanics dressed up as theory — it's the same algorithm
class (byte-pair encoding, or a close variant) behind every current
frontier model's tokenizer, Claude's included.

## A token is not a word, and not fixed in advance

The model never reads characters. Every request is converted to a sequence
of integers, each one an index into a fixed vocabulary — tens of thousands
to a few hundred thousand entries, depending on the model family. Those
entries aren't linguistic units chosen by a linguist. They're byte
sequences chosen by an algorithm that ran once, offline, over a training
corpus, looking for whatever chunks appeared together often enough to be
worth a dedicated slot.

The algorithm is byte-pair encoding (BPE): start with every word broken
into individual characters, find the adjacent pair that occurs most often
across the whole corpus, merge it into one symbol, and repeat until the
vocabulary hits its target size. Here it is on the classic four-word toy
corpus from the original BPE paper — real output, not a hand-typed example:

```python
from collections import Counter

corpus = {
    ("l","o","w","</w>"): 5,
    ("l","o","w","e","r","</w>"): 2,
    ("n","e","w","e","s","t","</w>"): 6,
    ("w","i","d","e","s","t","</w>"): 3,
}

def get_pair_counts(corpus):
    pairs = Counter()
    for word, freq in corpus.items():
        for a, b in zip(word, word[1:]):
            pairs[(a, b)] += freq
    return pairs

def merge_pair(pair, corpus):
    a, b = pair
    merged = a + b
    new_corpus = {}
    for word, freq in corpus.items():
        new_word, i = [], 0
        while i < len(word):
            if i < len(word) - 1 and word[i] == a and word[i + 1] == b:
                new_word.append(merged); i += 2
            else:
                new_word.append(word[i]); i += 1
        new_corpus[tuple(new_word)] = freq
    return new_corpus

for step in range(6):
    pairs = get_pair_counts(corpus)
    best = max(pairs, key=pairs.get)
    print(f"merge {step+1}: {best[0]!r}+{best[1]!r} -> {best[0]+best[1]!r}  (count={pairs[best]})")
    corpus = merge_pair(best, corpus)
```

```
merge 1: 'e'+'s' -> 'es'  (count=9)
merge 2: 'es'+'t' -> 'est'  (count=9)
merge 3: 'est'+'</w>' -> 'est</w>'  (count=9)
merge 4: 'l'+'o' -> 'lo'  (count=7)
merge 5: 'lo'+'w' -> 'low'  (count=7)
merge 6: 'n'+'e' -> 'ne'  (count=6)

resulting word segmentations:
  low </w>        (freq=5)
  low e r </w>     (freq=2)
  ne w est</w>     (freq=6)
  w i d est</w>    (freq=3)
```

Six merges in, `est</w>` — the "-est" superlative suffix plus a word
boundary — is already its own symbol, because "newest" and "widest" pushed
`e`+`s` to the top of the frequency count. "low" earned a merged symbol
too, from `lower` plus its own occurrences. Run this over billions of real
words instead of four toy ones, stop at tens of thousands of merges
instead of six, and this is the whole algorithm. There's no dictionary, no
part-of-speech tagging, no notion of a "word" at all going in — only pair
frequency, over and over, until the budget runs out.

The consequence that matters: a token is not a property of a string. It's
a property of a (string, trained vocabulary) pair. Change the vocabulary
and the same sentence tokenizes into a different sequence of a different
length.

## Different Claude models, different tokenizers

Claude's tokenizer isn't published as a downloadable merge table the way
some open tokenizers are, and it isn't the same tokenizer OpenAI's models
use — `tiktoken` will silently give you the wrong count for a Claude
prompt, sometimes off by 15–20% on plain text and by much more on code.
Anthropic's own guidance is blunt about it: never estimate Claude token
counts with `tiktoken` — call the counting endpoint instead.

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": open("CLAUDE.md").read()}],
)
print(resp.input_tokens)
```

That call matters more than it looks, because the vocabulary isn't fixed
across a vendor's own model lineup either. Opus 4.7 shipped a new
tokenizer, and the same input text produces roughly 1×–1.35× as many
tokens under it as under the older Opus 4.6 tokenizer did — same words,
more pieces, because the merge table changed. Sonnet 5 went further: its
new tokenizer produces about 30% more tokens for identical text than
Sonnet 4.6's did. Fable 5 and Opus 4.8 reuse the tokenizer introduced with
Opus 4.7, so migrating between those specific models leaves counts roughly
where they were — but anyone jumping from Opus 4.6 or earlier straight to
Fable 5 inherits that same 1×–1.35× inflation.

None of this shows up as an error. A `max_tokens` value that comfortably
fit a response on the old model can quietly start truncating output on the
new one, and a context-window budget sized against the old token count can
run out sooner than expected — both because the text got longer in tokens
without changing a single character. The fix isn't a multiplier applied by
hand; it's re-running `count_tokens` against the new model on your own
prompts, because the ratio isn't constant across content types.

## Why code and other languages cost more tokens per character

Merges get created when a pair shows up often enough in the training
corpus to be worth a slot — and that corpus is overwhelmingly English
prose. The vocabulary's cheapest, most-compressed entries are common
English words and word fragments, because those are what the merge
algorithm saw over and over.

Code doesn't compress the same way. Indentation runs, camelCase and
snake_case boundaries, repeated punctuation, and identifiers that don't
look like English words don't line up with the merge patterns English
prose produced, so the same visible character count in a source file
typically breaks into more tokens than it would in a paragraph of prose.
Non-Latin scripts — Chinese, Japanese, Korean, Hindi, Arabic — see the
same effect for a more basic reason: the merges that would compress them
efficiently were never as frequent in the training mix, so encoding often
falls back toward one token per character, or even per byte, instead of
per common word-fragment. A sentence that reads as "the same length" to a
person can cost noticeably more tokens in one script than another. That's
a property of what the vocabulary was trained on, not of the language
itself — a tokenizer trained on a different corpus mix would compress
differently.

## What this explains, not just what it costs

The token boundary is also why some famously "dumb" model mistakes aren't
about reasoning at all. Asking a model to count the letters in
"strawberry" is hard for the same reason the toy corpus above merged
`e`+`s` before it ever considered a single letter: the model's input is
whatever token IDs `straw` + `berry` (or however the trained vocabulary
happens to split it) turned into — the individual letters were never a
unit it was shown. Recovering "how many r's" means reasoning about the
sub-token structure indirectly, not reading characters off a tape the way
a person would.

It's also why trailing whitespace inside a tool call's string parameters
became a real, documented compatibility break between model generations —
older Claude models stripped a trailing newline a tool's input string
carried; 4.5-and-later models started preserving it, because the boundary
between "text" and "token" isn't the same as the boundary between
"visible character" and "invisible character." Code that did exact string
matching against a tool's `input` value (`if name == "foo"`) had to add
`.rstrip()` to keep working, because a token-level model doesn't
distinguish `"foo"` from `"foo\n"` the way a naive string comparison does.

And it's why context windows, `max_tokens`, and pricing are all denominated
in tokens rather than characters or words in the first place: tokens are
the actual unit the model consumes and produces. A context window
described as "1M" doesn't hold a fixed number of English words, a fixed
number of lines of code, or a fixed number of Japanese characters — it
holds however many tokens that specific tokenizer turns that specific
content into, and that number moves depending on what the content is and
which model is reading it.

## The practical rule

Don't estimate token usage, cost, or context-window headroom from
character counts, word counts, or a number remembered from a different
model. Call `count_tokens` against the model you're actually going to run,
on the actual content you're going to send it, and re-run it after any
model migration — the tokenizer is part of what "migrating" changes,
usually silently.
