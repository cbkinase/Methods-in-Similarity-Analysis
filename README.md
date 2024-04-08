# Introduction

We will now consider a class of problems where we are given a collection of sets and asked to determine which amongst them are similar.

We will use the concept of <b>Jaccard similarity</b> to formalize our notion of what similarity means &mdash; we will worry about the definition later, but the intuition is that the more elements in common two sets have, the more similar they are.

Solving this problem at scale is not trivial: if we have a collection of only size 1 million (not particularly large in some contexts), we would need to examine half a trillion pairs:

$$ \begin{aligned}
  \binom{n}{k} &= \frac{n!}{k! (n - k)!} \\
  \\
  &= \frac{1,000,000!}{2! (1,000,000 - 2)!} \\
  \\
  &= 5 \times 10^{11} \text{ possible pairs}
\end{aligned} $$

The infeasibility of this exhaustive enumeration method is self-evident. Even for a relatively small set, this is already impractical. We need to focus our attention primarily on those sets that are likely to be similar, and never examine the vast majority of pairs.


## Applications

Many data mining problems can be expressed as finding similar sets.

- Finding Netflix users with similar tastes in movies &mdash; or the dual of this: finding movies with similar sets of fans. We can use such insight to build recommendation systems.
- Entity resolution
- Given a body of documents, find pairs with a lot in common
  - Clustering or classification by topic, content, etc
  - Plagiarism detection
  - Excluding mirror sites from searches


## Three Essential Techniques for Similar Documents

We will apply three key techniques in our analysis. At a high level, here is how each of them factors in.

1) <b>Shingling</b>: convert documents into sets.

2) <b>MinHashing</b>: convert large sets to short signatures while preserving similarity.

3) <b>Locality-sensitive hashing</b>: focus only on pairs of signatures likely to be similar.

### The Big Picture

![TODO: alt text](https://i.gyazo.com/8866764a78b5e26e93d44fdd914ac574.png)

The benefits of this are twofold. First, by shingling and MinHashing, we convert documents into the comparatively much shorter signatures, allowing us to work more in main memory. However, this alone wouldn't be enough, as we'd still be considering a number of pairs that is quadratic with respect to the number of sets. Locality-sensitive hashing massively reduces the number of candidate pairs we must consider.

All of this introduces the risks of false positives (i.e., we claim two sets are similar when they are in fact not) and false negatives (erroneously claiming that two sets are not similar), but we are able to control this by tuning various parameters to our liking.


## Shingling

A <b><i>k</i>-shingle</b> (or <b><i>k</i>-gram</b>) for a document is a sequence of $k$ characters that appears in the document. We can represent a document by its set of <i>k</i>-shingles.

For example, if our document is "abcab" and we set $k=2$, then our set of 2-shingles is

$$ \lbrace \text{ab}, \text{bc}, \text{ca} \rbrace$$

It is important that we convince ourselves that shingling preserves similarity between documents as we would think about it intuitively. To what extent does having shingles in common imply similarity?

In the case that we...
- Change a single word in a document: we only affect <i>k</i>-shingles within distance $k$ of the word.
- Reorder a paragraph: only affects the $2k$ shingles that cross paragraph boundaries.

It is common to use a value for $k$ around 10 to ensure that most shingles do not occur in a given document. We often compress shingles to save space, hashing them into <b>tokens</b>. This can save upwards of 60% on space by converting a string of length 10 into a 4 byte token &mdash; the amount of bytes needed to store the amount of buckets for the entire set of <i>k</i>-shingles.

Ultimately, then, we represent a document by its tokens: the set of hash values of its <i>k</i>-shingles. Of course, there is the rare possibility that two documents could appear to have shingles in common, when in fact only hash values were shared.
