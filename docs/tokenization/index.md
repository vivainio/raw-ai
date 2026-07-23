---
icon: lucide/split
---

# Theory: Tokenization

This book's own intro admits that most AI content stalls at "what a token
is" and never gets past it. Fair — here's the part that usually gets
skipped: the actual merge algorithm that builds a token vocabulary, why two
Claude models released eight months apart can tokenize the same sentence
into a different number of pieces, and why that number can be higher for
code and non-English text than for plain English prose. The worked example uses
byte-pair encoding (BPE), one common tokenizer family. Current models also
use related schemes such as WordPiece, Unigram, and vendor-specific
variants, so the exact training algorithm and boundaries differ even when
the practical consequences are similar.

## A token is not a word, and not fixed in advance

The model is not given a raw string or one input position per character.
Every request is converted to a sequence of integers, each one an index
into a fixed vocabulary — tens of thousands
to a few hundred thousand entries, depending on the model family. Those
entries aren't linguistic units chosen by a linguist. They're byte
sequences chosen by an algorithm that ran once, offline, over a training
corpus, looking for whatever chunks appeared together often enough to be
worth a dedicated slot.

In the original subword version of BPE, the algorithm starts with every word
broken into individual characters, finds the adjacent pair that occurs most
often across the corpus, merges it into one symbol, and repeats until the
vocabulary hits its target size. Many modern tokenizers instead start from
bytes, which guarantees that any Unicode text can be represented without an
unknown-token fallback. Here is the character-level algorithm on the classic
four-word toy corpus from the original BPE-for-subwords paper:

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
part-of-speech tagging or linguistic analysis — only word frequencies and
pair frequencies, over and over, until the budget runs out. Byte-level
implementations do not require the pre-split words or explicit `</w>` marker
used by this teaching example.

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

Merges get created when a pair shows up often enough in the tokenizer's
training corpus to be worth a slot. If English prose is overrepresented,
common English words and fragments receive more of those slots and compress
more efficiently. The size and composition of the corpus matter here; the
imbalance is not identical across model families.

Code may not compress the same way. Indentation, identifier boundaries,
punctuation, and project-specific names create a different frequency
distribution from prose. Tokenizers trained with plenty of code can add
merges for those patterns, so code's token cost is model- and language-
dependent rather than inherently high.

The same qualification applies across human languages. Scripts or languages
that occupy less of the tokenizer-training mix tend to receive fewer useful
merges, and byte-level tokenizers may split one visible character into
multiple tokens. A sentence that looks the same length to a person can
therefore cost noticeably more in one language than another. That is a
property of the tokenizer and its training corpus, not of the language
itself.

## What this explains, not just what it costs

Token boundaries also help explain some famously "dumb" model mistakes.
If "strawberry" arrives as a few multi-character tokens, the model is not
given a ready-made sequence with one position per letter. It can still
learn spelling from text and can often reconstruct the characters, but
counting them requires an indirect transformation rather than reading one
letter from each input position. Tokenization makes the task awkward; it
does not make the characters unavailable in principle.

Tokenization does not, however, explain every text-formatting compatibility
change. Whether a tool-call argument preserves trailing whitespace is a
property of model behavior and the API's serialization and parsing layers.
Both `"foo"` and `"foo\n"` are representable as token sequences, and an
ordinary string comparison correctly treats them as different values. If
an integration wants to ignore surrounding whitespace, it should normalize
the input deliberately rather than attribute the difference to tokens.

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
