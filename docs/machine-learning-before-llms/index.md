---
icon: lucide/history
---

# Theory: Machine Learning Before LLMs

The [Models: Lineage and Licenses](../model-lineage-licenses/index.md) chapter's
history starts with "before transformers, sequence models were recurrent" —
which itself starts mid-story. Recurrent networks weren't the first machine
learning, just the last stop before transformers. This chapter covers what
came before that: roughly sixty years of machine learning that solved real
problems without anything resembling a neural network doing the heavy
lifting, and the several intermediate waves of neural methods that
followed. None of it is obsolete trivia — most of it is still the right
tool for problems that aren't text, images, or audio.

## The core idea: a human decides what the numbers mean

Modern deep learning takes raw pixels or raw tokens as input and learns
its own internal representation of what matters. Classical machine
learning didn't have that option, and for a long time didn't want it —
compute was too limited to learn representations from scratch. Instead, a
person did that work by hand: **feature engineering**. Given a task —
say, is this email spam — someone decided which measurable properties of
the email might predict the answer (word counts, presence of certain
phrases, number of links, sender domain) and turned each email into a
fixed-length row of numbers built from those decisions. The learning
algorithm never saw the raw email; it saw the spreadsheet a human derived
from it.

This split — feature engineering as a separate, manual step upstream of
the model — is the single biggest difference between old and new machine
learning, and it explains most of the other differences in this chapter.
A classical model is only as good as the features it's fed, which means
the modeler's domain knowledge was doing real work, not just the
algorithm. It also means classical models tended to be small and fast:
once the hard part (deciding what to measure) was done by a human, the
algorithm's job was comparatively simple arithmetic over a short row of
numbers, not a hierarchy of learned transformations over raw data.

## The classical toolbox

None of these are historical curiosities that got fully replaced. Several
still win on the kind of data they were built for — structured rows in a
database, not raw pixels or free text.

| Model | Core idea | Still the default for |
|---|---|---|
| Linear / logistic regression | Fit a weighted sum of the input features to the target, directly or through a sigmoid for classification | Cases needing an interpretable, auditable coefficient per feature |
| Decision trees | Learn a sequence of if/else splits on feature values | Rules a human can read straight off the model |
| k-nearest neighbors | Classify a point by majority vote among its closest labeled neighbors | Small datasets where "similar past cases" is the whole model |
| Naive Bayes | Apply Bayes' theorem assuming features are independent given the class | Text classification with bag-of-words features, cheap and hard to beat as a baseline |
| Support vector machines | Find the boundary that maximizes margin between classes, optionally in a higher-dimensional space via a kernel | Small-to-medium datasets with a clear margin between classes |
| Random forests / gradient-boosted trees (XGBoost, LightGBM) | Combine many decision trees, either averaged or trained sequentially to correct prior errors | Tabular prediction — still the standard choice on Kaggle-style structured data, frequently outperforming deep learning there |

That last row matters for a reason worth stating directly: tabular data —
rows of independent, already-structured columns like age, income, and zip
code — doesn't have the kind of raw perceptual structure (pixels adjacent
in space, tokens adjacent in a sequence) that convolutional and
transformer architectures are built to exploit. A gradient-boosted tree
ensemble doesn't need to learn what a "pixel" or a "token" is because
there's no such intermediate structure to discover — the columns already
are the features. The LLM era displaced classical ML in text, vision, and
speech; it mostly didn't displace it on a spreadsheet.

## Turning text into numbers, before embeddings existed

Before anything neural touched language, text still had to become a row
of numbers, and the classical answer was to count words.

**Bag-of-words** represents a document as a vector as long as the entire
vocabulary, with each position holding how many times that word appeared
— word order is discarded entirely, hence "bag." **TF-IDF** (term
frequency–inverse document frequency) refines the raw count: it
downweights words that appear in nearly every document (common words
carry little information about what makes *this* document distinctive)
and upweights words that are frequent locally but rare across the
corpus. **N-grams** partially restore order by counting short runs of
adjacent words instead of single words, at the cost of a much larger and
sparser vocabulary.

The structural weakness all three share: the resulting vector has one
dimension per vocabulary word, almost all of them zero for any given
document, and — critically — no notion of similarity between words at
all. "Car" and "automobile" are two unrelated dimensions to a bag-of-words
model; nothing in the representation says they mean roughly the same
thing. Two synonymous documents that happen to use different words for
the same concepts can end up looking unrelated. Fixing that required
representing a word by where it tends to appear, not by which exact
string it is.

## Word embeddings: the first dense, learned representation of meaning

**Word2vec** and **GloVe** (both 2013–2014) were the shift from sparse,
hand-derived text features to dense vectors learned from data —
specifically, from the company a word keeps. The training signal is
simple: predict a word from its neighbors (or vice versa) across a huge
corpus, with no labels required beyond the raw text itself. A word ends
up with a vector of a few hundred numbers, and words that appear in
similar contexts converge toward similar vectors, purely as a
side-effect of the prediction task — nobody hand-specified that "car" and
"automobile" should be close.

The famous demonstration was vector arithmetic: `king - man + woman`
lands close to `queen` in the learned space, showing the vectors captured
something like relational structure, not just topical clustering. This is
the direct ancestor of the token embeddings covered in the
[Transformer Architecture](../transformer-architecture/index.md) chapter
— but with one large limitation that chapter's mechanism specifically
fixes: a word2vec vector is one fixed vector per word, the same
regardless of context. "Bank" gets a single vector that has to average
over both the river and the financial-institution senses, because the
representation is learned and stored per word, not computed fresh from
the surrounding sentence the way a transformer's attention mechanism
does.

## Neural networks arrive early, and in stages

Neural networks aren't a recent idea grafted onto old machine learning —
they ran alongside it for decades, in a series of steps each solving one
specific limitation of the last:

- **The perceptron** (1958) was the first trainable artificial neuron: a
  weighted sum of inputs through a threshold, learning its weights from
  labeled examples. It could only separate data with a straight line,
  which an influential 1969 critique highlighted as a hard limit — a
  single perceptron can't even learn XOR — and that critique cooled
  funding for neural network research for over a decade.
- **Multi-layer networks trained by backpropagation** (popularized 1986)
  got past that limit by stacking layers of neurons with a nonlinearity
  between them, and by having an efficient algorithm — backprop — for
  computing how to adjust every weight in every layer from a single
  error signal at the output. Stacked layers with a nonlinearity can
  approximate the boundary a single perceptron couldn't.
- **Convolutional networks** (LeCun's LeNet, 1989 and 1998) specialized
  that architecture for images: instead of connecting every input pixel
  to every neuron, a small learned filter slides across the image,
  reusing the same weights at every position. That reuse encodes an
  assumption specific to images — a pattern worth detecting in one
  corner is just as worth detecting anywhere else — and cut the parameter
  count enough to make handwritten digit recognition practical.
- **Recurrent networks and LSTM** (1997) did the equivalent for
  sequences: a network that processes one element at a time, carrying a
  hidden state forward as a running summary of everything seen so far.
  This is exactly where the
  [Models: Lineage and Licenses](../model-lineage-licenses/index.md#a-short-history)
  chapter's own history picks up — and where the recurrence itself becomes
  the bottleneck that the transformer was built to remove.

## Where the old toolbox hit its ceiling

Two separate limits pushed the field toward what came next, and they're
worth keeping distinct because they were fixed by different
inventions.

Classical ML's ceiling was feature engineering itself: it doesn't scale
to raw, unstructured data. Nobody can hand-design a feature set that
captures everything relevant in a photograph or a paragraph — pixels and
tokens don't have that structure until something learns to find it, which
is precisely what convolutional and transformer architectures do that a
decision tree or an SVM cannot.

Early sequence models' ceiling was the recurrence itself, as the
[Lineage and Licenses](../model-lineage-licenses/index.md#a-short-history)
chapter covers: a hidden state that has to compress everything seen so
far degrades over long inputs, and processing one token at a time means
training can't be parallelized across a sequence. Removing the recurrence
entirely — letting every token attend to every other token directly — is
what "Attention Is All You Need" did, and it's the point where this
chapter's story and that one's meet.

The takeaway isn't that eighty years of machine learning got obsoleted by
one paper. It's that each wave solved one specific limitation of the
wave before it, and the parts that never hit their ceiling — logistic
regression, gradient-boosted trees, k-nearest neighbors on tabular data —
are still exactly the right tool, running quietly underneath plenty of
production systems that have nothing to do with an LLM.
