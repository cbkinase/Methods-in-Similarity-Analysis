# Table of Contents

- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
  - [Applications](#applications)
  - [Three Essential Techniques for Similar Documents](#three-essential-techniques-for-similar-documents)
  - [The Big Picture](#the-big-picture)
- [Shingling](#shingling)
- [MinHashing](#minhashing)
  - [From Sets to Boolean Matrices](#from-sets-to-boolean-matrices)
  - [Min-Wise Independent Permutations](#min-wise-independent-permutations)
  - [Being Practical About MinHashing](#being-practical-about-minhashing)
- [References](#references)


# Introduction

We will now consider a class of problems where we are given a collection of sets and asked to determine which amongst them are similar.

We will use the concept of <b>Jaccard similarity</b> to formalize our notion of what similarity means &mdash; we will worry about the definition later, but the intuition is that the more elements in common two sets have, the more similar they are.

Solving this problem at scale is not trivial. If we have a collection of only size 1 million (not particularly large in some contexts), we would need to examine half a trillion pairs:

$$ \begin{aligned}
  \binom{n}{k} &= \frac{n!}{k! (n - k)!} \\
  \\
  &= \frac{1,000,000!}{2! (1,000,000 - 2)!} \\
  \\
  &= 5 \times 10^{11} \text{ possible pairs}
\end{aligned} $$

It is easy to see that in general, the number of possible pairs grows quadratically with respect to the number of elements:

$$ \binom{n}{2} = \frac{n!}{2!(n-2)!} = \frac{n \cdot (n - 1)}{2} \text{ possible pairs} $$

The infeasibility of this exhaustive enumeration method is self-evident. Even for a relatively small set, this is impractical. We need to focus our attention primarily on those sets that are likely to be similar, and never examine the vast majority of pairs.


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

## The Big Picture

![TODO: alt text](https://i.gyazo.com/8866764a78b5e26e93d44fdd914ac574.png)

The benefits of this three-step procedure are twofold. First, by shingling and MinHashing, we convert documents into comparatively much shorter signatures, allowing us to work more in main memory. However, this alone wouldn't be enough, as we'd still be considering a quantity of pairs that is quadratic with respect to the number of sets. Locality-sensitive hashing massively reduces the number of candidate pairs we must consider.

All of this introduces the risks of false positives (i.e., we claim two sets are similar when they are not) and false negatives (erroneously claiming that two sets are not similar), but we can control the error rates by tuning various parameters to our liking.


# Shingling

A <b><i>k</i>-shingle</b> (or <b><i>k</i>-gram</b>) for a document is a sequence of $k$ characters that appear in the document. We can represent a document by its set of <i>k</i>-shingles.

For example, if our document is "abcab" and we set $k=2$, then our set of 2-shingles is

$$ \lbrace \text{ab}, \text{bc}, \text{ca} \rbrace$$

It's worth noting two things here. The first is that we don't necessarily need to partition by characters. Shingles could in principle be letters, words, lines, or, from a mathematical point of view, anything countable. Secondly, it is also possible to represent documents with <i>labelled k-shingling</i>, which also stores the occurrence number for each element, though this is less efficient.

$k$ should be selected to be large enough that the probability of finding a random shingle in a random document is quite low. The exact number might depend on the nature of the document &mdash; e.g., $k=5$ might suffice for a tweet, but for a news article $k=10$ might be more appropriate.

We often compress long shingles to save space, hashing them into <b>tokens</b>. This can save upwards of 60% on space by converting a string of length 10 into a 4-byte token &mdash; the number of bytes needed to store the number of buckets for the entire set of <i>k</i>-shingles.

Ultimately, we can represent a document by its tokens: the set of hash values of its <i>k</i>-shingles. Of course, there is the rare possibility that two documents could appear to have shingles in common, when in fact only hash values were shared.


# MinHashing

Before talking about MinHashing, let's more precisely define our notion of similarity.

For two sets $A$ and $B$, we define their Jaccard similarity $J(A, \space B)$ as

$$ J(A, \space B) = \frac{| A \space \cap \space B |}{| A \space \cup \space B |}$$

From shingling, each document $D$ gets an associated set $S_D$. Thus, we can define the similarity of two documents $A$ and $B$ to be

$$ J(A, \space B) = \frac{| S_A \space \cap \space S_B |}{| S_A \space \cup \space S_B |} $$


## From Sets to Boolean Matrices

Instead of thinking of our data as a collection of sets, it's helpful to conceptualize it instead as a Boolean matrix, even if it is unlikely to be stored that way, or ever be materialized in the first place.

1) Let the rows of the matrix be the elements of the universal set: for example, the set of all <i>k</i>-shingles.
2) Let the columns of the matrix represent each set.
   - The entry at row $e$ and column $S$ will be 1 if and only if $e$ is a member of $S$.
3) The column similarity is the Jaccard similarity of the sets of their rows with 1.

The typical matrix is quite sparse, although the examples we use will generally not be for the sake of illustration.

![TODO: alt text](https://i.gyazo.com/945737d3154627677a9779345243bb29.png)

Here we see a matrix for which the size of the union of the sets is 5 (at least one of them has a 1 for the indices 0, 1, 2, 4, and 5) and the size of their intersection is 2 (indices 2 and 4), so their Jaccard similarity is 40%.

Notice how given two columns $C_1$ and $C_2$, rows can be classified into one of four types:

- $a$: $(1, 1)$
- $b$: $(1, 0)$
- $c$: $(0, 1)$
- $d$: $(0, 0)$

Letting $a$, $b$, $c$, and $d$ represent the number of rows of each type

$$ J(A, \space B) = \frac{a}{a + b + c} = \frac{| A \space \cap \space B |}{| A \space \cup \space B |} $$


## Min-Wise Independent Permutations

MinHashing is predicated on the notion of <b>min-wise independent families of permutations</b>. We say that $\mathscr{F} \subseteq S_n$ (the symmetric group) is <i>exactly</i> min-wise independent if for any set $X \subseteq [n]$ and any $x \in X$, when $\pi$ is chosen at random in $\mathscr{F}$ we have

$$  \textbf{Pr} \left( \min \lbrace \pi (X) \rbrace = \pi(X) \right) = \frac{1}{|X|}  $$

This family of permutations has the remarkable property that for two documents $A$ and $B$

$$  \textbf{Pr} \left( \min \lbrace \pi (S_A) \rbrace = \min \lbrace \pi (S_B) \rbrace \right) = \frac{| S_A \space \cap \space S_B |}{| S_A \space \cup \space S_B |} = J(A, \space B)  $$

For a given document we can choose, say, 100 independent random permutations $\pi_1, \space \dots \space, \pi_{100}$ and store a comparatively small, fixed-size <i>sketch</i> $\overline{S}_D$, which is the list

$$ \overline{S}_D =
(
  \min \lbrace \pi_1 (S_D) \rbrace , \space
  \min \lbrace \pi_2 (S_D) \rbrace , \space
  \dots \space , \space
  \min \lbrace \pi _{100} (S_D) \rbrace
) $$

and so the Jaccard similarity of two sets $A$ and $B$ is readily estimated by comparing how many corresponding elements in $\overline{S}_A$ and $\overline{S}_B$ are common. Since it is impossible to choose $\pi$ uniformly at random in $S_n$, for practical purposes we relax this constraint slightly and allow $\mathscr{F}$ to be <i>approximately</i> min-wise independent, where

$$  \left| \space \textbf{Pr} \left( \text{min} \lbrace \pi (X) \rbrace = \pi(X) \right) - \frac{1}{|X|} \space \right| \le \frac{\epsilon}{|X|} $$

The practical set of permutations outlined by Broder <i>et al.</i> (1998) are linear transformations. We focus on the case where the universe of elements is

$$\lbrace 0, \space \dots \space, \space p - 1 \rbrace $$

for some prime $p$, and the family of permutations is given by all permutations of the form

$$ \pi(x) = ax + b \space \text{mod} \space p \space\space\space (\text{where } a \ne 0) $$

which, despite not being exactly min-wise independent, offers a reasonable and performant approximation to the Jaccard similarity.

## Being Practical About MinHashing

In progress...

# References

A. Z. Broder.
<b>On the Resemblance and Containment of Documents</b>.
Proc. Compression and Complexity of Sequences, pp. 21--29.
IEEE Computer Society, 1997.

A. Z. Broder, M. Charikar, A. Frieze, and M. Mitzenmacher.
<b>Min-Wise Independent Permutations</b>.
Proc. 30th Symposium on Theory of Computing, pp. 327--336, 1998.

J. Leskovec, A. Rajaraman, and J. Ullman.
<b>Mining of Massive Datasets</b>.
(Cambridge University Press, 2022).
