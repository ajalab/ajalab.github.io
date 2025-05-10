---
title: Full-Text Search on Multiple Texts Using the FM-Index
date: 2025-05-08T09:00:00+09:00
draft: false
katex: true
---

The _FM-index_ is a data structure (index) designed for efficient full-text search in strings. By constructing the FM-index of a text beforehand, queries for a given search pattern can be processed efficiently.

|Query|Description|
|---|---|
|$\mathrm{count}$|Counts the number of occurrences of the pattern in the text.
|$\mathrm{locate}$|Locates the positions of all occurrences of the pattern in the text.

Notably, the computation time of the $\mathrm{count}$ query depends on the length of the pattern, not the length of the text itself.

The FM-index is composed of elements such as [succinct data structures](https://en.wikipedia.org/wiki/Succinct_data_structure) and the [Burrows-Wheeler Transform](https://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform), allowing the index to be handled in a compressed form (compressed full-text index). Furthermore, since the index itself retains the information of the original text (self-index), it's possible to reconstruct text adjacent to the matched pattern in the search results.

While the FM-index provides search functionality for a single text, a natural application extends to **searching across a collection of multiple texts**. For instance, there are frequent scenarios where one might want to list files containing a specific keyword from a set of files. This article will introduce several approaches to achieve such multi-text searching using the FM-index.

## FM-index

Before delving into searching across multiple texts, let's first review the FM-index for a single text.

Let $T$ be a text string of length $n=∣T∣$ over an alphabet set of size $\sigma$. Assume that the smallest character in the alphabet set is $\text{\textdollar}$, and that this end-of-string marker is always appended to the end of the text ($T[n - 1]=\text\textdollar$).

>![NOTE]
>In this article, we will use 0-based indexing for string access.

### Construction of the FM-index

The FM-index is composed of the following elements:

- $T_\mathrm{BWT}$: The Burrows-Wheeler Transform of the text $T$. It can be constructed from the suffix array of the text.
- $C$: An array that, for each character $c$ in the alphabet, stores the number of occurrences of characters in $T$ that are lexicographically smaller than $c$.
- $SA$: The suffix array of the text $T$. This is needed to answer locate queries. To reduce its size, a sampled version of the suffix array with regular intervals can be employed.

#### Suffix Array ($SA$)

The suffix array of a text $T$ is an array that records the starting positions of all suffixes of $T$, sorted lexicographically. For example, if $T = \text{banana\textdollar}$, the suffix array is $[6,5,3,1,0,4,2]$, as shown in the table below:

| Suffix（in lexicographic order） | Starting position |
|---|---|
| $\text{\textdollar}$ | 6 |
| $\text{a\textdollar}$ | 5 |
| $\text{ana\textdollar}$ | 3 |
| $\text{anana\textdollar}$ | 1 |
| $\text{banana\textdollar}$ | 0 |
| $\text{na\textdollar}$ | 4 |
| $\text{nana\textdollar}$ | 2 |

The suffix array can be constructed in $O(n)$ time using algorithms such as SA-IS[^1].

#### Burrows-Wheeler Transform （$T_\mathrm{BWT}$）

The [Burrows-Wheeler transform](https://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform) is a transformation that obtains a string by lexicographically sorting all rotations of the text and extracting the last character of each rotation. For example, the rotations of $T = \text{banana\textdollar}$ are $\text{banana\textdollar}, \text{anana\textdollar b}, \text{nana\textdollar ba}, \ldots, \text{\textdollar banana}$. The result of sorting these lexicographically is shown below.

```
k  SA[k]  F     L
-----------------
0      6  $banana
1      5  a$banan
2      3  ana$ban
3      1  anana$b
4      0  banana$
5      4  na$bana
6      2  nana$ba
```

The Burrows-Wheeler transform $T_\mathrm{BWT}$ of the text $T$ is $\text{annb\textdollar aa}$, which is obtained by taking the last character of each rotation.

Here, the order of the rotations is the same as the order of the suffix array. Row $k$ of the table shows the suffix (and its rotation) corresponding to the value $SA[k]$ of the suffix array.

The first column of the table is called the F-column, and the last column is called the L-column. In the example, $F = \text{\textdollar aaabnn}$ and $L = \text{annb\textdollar aa}$. The Burrows-Wheeler transform $T_\mathrm{BWT}$ of the text $T$ is this L-column itself. These names will be used hereafter.

The Burrows-Wheeler transformed text $T_\mathrm{BWT}$​ is stored in a format that allows the following queries to be processed quickly, using data structures such as [wavelet trees](https://en.wikipedia.org/wiki/Wavelet_Tree) or [wavelet matrices](https://www.sciencedirect.com/science/article/abs/pii/S0306437914000945):

- $\mathrm{rank}_c(T\_\mathrm{BWT}, i)$: Counts the number of occurrences of the character $c$ in $T\_\mathrm{BWT}[0, i)$
- $\mathrm{select}_c(T\_\mathrm{BWT}, i)$: Calculates the position of the $i+1$-th occurrence of $c$ in ​$T\_\mathrm{BWT}$.
- $\mathrm{access}(T\_\mathrm{BWT}, i)$: Obtains $T_\mathrm{BWT}[i]$.

The data size and construction/query efficiency depend on the implementation of the wavelet tree used, but the computational complexity when using a wavelet matrix is as follows:

- Size: $O(n \log\sigma)$
- Construction time: $O(n \log\sigma)$
- Computation time of $\mathrm{rank}$ and $\mathrm{select}$: $O(\log\sigma)$

#### Character Counts ($C$)

The array $C$ stores, for each character $c$ in the alphabet, the total number of occurrences of characters in the text $T$ that are lexicographically smaller than $c$.
For example, if $T = \text{banana\textdollar}$, then $C[\text\textdollar] = 0, C[\text a] = 1, C[\text b] = 4, C[\text n] = 5$.

In practice, $C$ is typically stored as an array indexed by the numerical representation of the characters. In this case, the size of $C$ is $O(\sigma\log n)$.

### FM-index Queries

As an example, the rotation table for the text $T = \text{mississippi\textdollar}$ is shown below.

```
 k  SA[k]  F          L
-----------------------
 0     11  $mississippi
 1     10  i$mississipp
 2      7  ippi$mississ
 3      4  issippi$miss
 4      1  ississippi$m
 5      0  mississippi$
 6      9  pi$mississip
 7      8  ppi$mississi
 8      6  sippi$missis
 9      3  sissippi$mis
10      2  ssippi$missi
11      5  ssissippi$mi
```

The FM-index stores the Burrows-Wheeler transformed string $T_\mathrm{BWT} = \text{ipssm\textdollar pissii}$ (which is the L-column) and the array $C = \lbrace \text{\textdollar} \mapsto 0, \text{i} \mapsto 1, \text{m} \mapsto 5, \text{p} \mapsto 6, \text{s} \mapsto 8 \rbrace$.

An important property is that the suffixes in the text that correspond to occurrences of a pattern, i.e., suffixes that start with the pattern, appear in consecutive rows in this table.
For example, suffixes starting with the pattern $P = \text{s}$ are in rows 8 to 11, suffixes starting with $P=\text{is}$ are in rows 3 to 4, and suffixes starting with $P=\text{sis}$ are in row 9.
Therefore, to search for a pattern in the text, we need to identify the range of rows in the table, which corresponds to a range in the suffix array ($SA$), where the suffixes corresponding to that pattern appear.

Given a range $[s', e')$ in the $SA$ that corresponds to a pattern $P'$, we can compute the range $[s,e)$ in the $SA$ that corresponds to a new pattern $P = cP'$ (where $c$ is a character) using $T_\mathrm{BWT}$ and $C$.

For example, the range in the $SA$ for $P' = \text{s}$ is $[s', e') = [8, 12)$.
The range in the $SA$ for $P = \text{is}$ corresponds to the range in the F-column where the character $c = \text{i}$ appears, based on the occurrences of the character $\text{i}$ within the range $L[8,12)$ in the L-column (which are at $L[10]$ and $L[11]$).

In the F-column, the character $\text{i}$ appears consecutively starting from row $C[\text{i}] = 1$. Furthermore, the order of occurrences of the character $\text{i}$ in the F-column and the L-column is the same. Using these properties, we can determine the range $[s,e)$ in the $SA$ corresponding to $P$ as follows:

$$
\begin{align*}
s &= C[c] + \mathrm{rank}_c(T\_\mathrm{BWT}, s') \\\\
e &= C[c] + \mathrm{rank}_c(T\_\mathrm{BWT}, e').
\end{align*}
$$


```
      k  SA[k]  F          L
     -----------------------
      0     11  $mississippi
C[i]  1     10  i$mississipp
      2      7  ippi$mississ
  s   3      4 *issippi$miss <-+
  |   4      1 *ississippi$m <-|--+
  e   5      0  mississippi$   |  |
      6      9  pi$mississip   |  |
      7      8  ppi$mississi   |  |
  s'  8      6  sippi$missis   |  |
  |   9      3  sissippi$mis   |  |
  |  10      2  ssippi$missi*--+  |
  |  11      5  ssissippi$mi*-----+
  e'
```

Therefore, by initializing the range to $[s_0, e_0) = [0, n)$ and processing a given pattern $P$ character by character from its end using the method described above, we can identify the range $[s,e)$ in the $SA$ that corresponds to the entire pattern $P$. The time complexity for this process is $O(m\log \sigma)$, where $m$ is the length of the pattern.

#### Counting Pattern Occurrences ($\mathrm{count}$)

To calculate the number of occurrences ($\mathrm{locate}$) of a pattern, we first determine the range $[s,e)$ in the $SA$ corresponding to the pattern using the algorithm described above, and then return $e − s$.

#### Locating Pattern Occurrences ($\mathrm{locate}$)

To determine the occurrence positions ($\mathrm{locate}$) of a pattern, the information from the suffix array $SA$ itself is required. After finding the range $[s,e)$ in the $SA$ corresponding to the pattern, all the suffix array values within this range, $SA[s], SA[s+1], \ldots, SA[e-1]$, represent the occurrence positions of the pattern in the original text $T$.

Here, instead of storing all elements of $SA[0, n)$, we can reduce the data size by sampling and storing elements of $SA$ at regular intervals of $l$. The unsampled values of SA can be recovered as needed using $T_\mathrm{BWT}$ and $C$.

We use the technique called LF-mapping for this recovery. The LF-mapping ($LF$) indicates, for a character $L[k]$ at position $k$ in the L-column, its corresponding position in the F-column. It can be computed in the same way as the algorithm described in the FM-index Queries section, as follows:

$$
LF(k) = C[L[k]] + \mathrm{rank}_{L[k]}(L, k).
$$

Furthermore, LF-mapping can also be interpreted as the correspondence between positions defined using the suffix array $SA$ and its inverse $SA^{
−1}$, as follows:

$$
LF(k) = SA^{-1}[SA[k] -_n 1].
$$

Here, $a -_n b$ denotes subtraction modulo $n$. That is, $0-_n1 = n - 1$.

From $SA[LF(k)] = SA[SA^{-1}[SA[k] -_n 1]] = SA[k] -_n 1$, it follows that:

$$
SA[k] = SA[LF(k)] +_n 1 = \cdots = SA[LF^i(k)] + i.
$$

In other words, by repeatedly applying the LF-mapping starting from $k$, i.e., $LF(k), LF(LF(k)), \ldots$ we eventually reach a sampled element of $SA$. We can determine any $SA[k]$ from the value of the sampled $SA$ element and the number of LF-mapping steps taken.

## FM-index for Multiple Texts

Our goal is to efficiently process the following queries for a collection of $d$ text fragments $T_0, \ldots, T_{d-1}$ using the FM-index:

- Retrieve a list of identifiers of the text fragments that contain a given search pattern.
- Retrieve the occurrence positions of a search pattern within each text fragment.
- Retrieve a list of identifiers of the text fragments that contain a given search pattern as a prefix or a suffix.

To achieve full-text search across multiple texts, several approaches can be considered. These approaches can be categorized based on how the texts are concatenated into a single large text $T$.

| | Text Concatenation Method	 | $T$ |
|---|---|---|
|1|Concatenate with distinct end-of-text markers|$T = T_0\text\textdollar_0T_1\text\textdollar_1 \cdots T_{d-1}\text\textdollar_{d-1}$|
|2|Concatenate with the same end-of-text marker|$T = T_0\text\textdollar T_1\text\textdollar \cdots T_{d-1}\text\textdollar$|
|3|Concatenate without end-of-text markers (with a final $\text\textdollar$)|$T = T_0T_1 \cdots T_{d-1}\text\textdollar$|

### Concatenation with Distinct End-of-Text Markers

This method is employed in SXSI[^2], which aims to achieve fast XPath query processing on XML documents.
Each text fragment $T_x$ is appended with a unique end-of-text marker $\text\textdollar_x$, and these are then concatenated to form a single text $T = T_0\text\textdollar_0T_1\text\textdollar_1 \cdots T_{d-1}\text\textdollar_{d-1}$.
Importantly, these end-of-text markers $\text\textdollar_0, \ldots, \text\textdollar_{d-1}$ are assigned a lexicographical order that matches the order of appearance of the texts, i.e., $\text\textdollar_0<\text\textdollar_1<\cdots<\text\textdollar_{d-1}$.

> Since there are several $\text\textdollar$'s in $T$, we fix a special ordering such that the end-marker of the $i$-th text appears at $F[i]$ in M

#### Construction

To associate the searched positions with the identifiers of the text fragments, we introduce an additional data structure called $\mathrm{Doc}$.

Suppose $T[SA[k]]$ is the first character of a text fragment $T_x$.
In this case, $L[k]$ will always be the end-of-text marker $\text\textdollar$.
We then set $\mathrm{Doc}[\mathrm{rank}_\text\textdollar(L, k)] = x$.

As an example, the rotation table for the text $T=\text{foo}\text\textdollar_0\text{bar}\text\textdollar_1\text{baz}\text\textdollar_2$ is shown below.
For display purposes, the subscripts of the end-of-text markers are omitted, but they are actually distinct.

```
 k  SA[k]  F          L
-----------------------
 0      3  $bar$baz$foo
 1      7  $baz$foo$bar
 2     11  $foo$bar$baz
 3      5  ar$baz$foo$b
 4      9  az$foo$bar$b
 5      4  bar$baz$foo$ L[5] = $_0
 6      8  baz$foo$bar$ L[6] = $_1
 7      0  foo$bar$baz$ L[7] = $_2
 8      1  oo$bar$baz$f
 9      2  o$bar$baz$fo
10      6  r$baz$foo$ba
11     10  z$foo$bar$ba
```

Here is the $\mathrm{Doc}$ array for the example text.

|$k$|$T[SA[k]]$|$i$|$\mathrm{Doc[i]}$|
|---|---|---|---|
|5|$T_1[0]$|0|1|
|6|$T_2[0]$|1|2|
|7|$T_0[0]$|2|0|

To construct $\mathrm{Doc}$, we sequentially identify the positions $i$ of the end-of-text markers in the L-column ($T_\mathrm{BWT}$) using the $\mathrm{select}$ query.
Since $SA[i]$ is the starting position of the text fragment $T_{x-_l1}$, we determine the corresponding value of $\mathrm{Doc}[i]$ as $x$.

For query efficiency, we also record the corresponding text fragment identifier $x$ and the relative position within the text fragment as $\mathrm{Pos}$ for sampled positions $k$ in $SA$.

#### Queries

To identify the text fragments containing a given pattern $P$, we first find the range $[s, e)$ in the suffix array $SA$ corresponding to $P$.
Then, for each position $k$ within this range $s \leq k \lt e$, we repeat the following steps:

- Stop if $L[k] = \text\textdollar$ or if $k$ is a position sampled in $\mathrm{Pos}$.
- Set $k \gets LF(k)$.

From the position $k$ where the computation stops, the identifier $x$ of the text fragment $T_x$ ​containing the pattern $P$ is determined as follows:

- If $L[k] = \text\textdollar$, then $x = \mathrm{Doc}[\mathrm{rank}_\text\textdollar(L,k)]$.
- If $k$ is a position sampled in $\mathrm{Pos}$, then retrieve the text fragment identifier $x$ from $\mathrm{Pos}$.

Additionally, the relative position of $P$ within the text fragment $T_x$ ​can be calculated using the number of iterations and the relative position recorded in $\mathrm{Pos}$ (if applicable).

To list the positions where the pattern $P$ appears as a prefix of a text fragment, we first find the range $[s,e)$ in the $SA$ corresponding to the pattern $P$. For each $k$ within this range ($s \leq k \lt e$), if $L[k]$ is an end-of-text marker, then its occurrence position is the beginning of a text fragment.

To list the positions where the pattern $P$ appears as a suffix of a text fragment, we initialize the range search to $[0,d)$, which corresponds to the range of end-of-text markers in the F-column, and then identify the corresponding range in the $SA$ using a similar procedure.

#### Computational Complexity

Compared to the FM-index for a single text, this method requires the following additional data:

- The end-of-text marker for each text fragment stored in $T$.
- $\mathrm{Doc}$
- $\mathrm{Pos}$

The number of additional end-of-text markers is $O(d)$.

If $\mathrm{Doc}$ is represented as an array, its size is $O(d\log d)$ bits. If $\mathrm{Pos}$ is stored as an array with a sampling rate of $l$, its size is $O(\frac n l\log n)$ bits.

The time taken to compute the identifier and relative position of the corresponding text fragment for an occurrence of the pattern is $O(l\log \sigma)$.

### Concatenation with the Same End-of-Text Marker

In this method, each text fragment $T_x$ is appended with the same end-of-text marker $\text\textdollar$, and these are then concatenated to form a single text $T = T_0\text\textdollar T_1\text\textdollar \cdots T_{d-1}\text\textdollar$.

For the text $T=\text{foo}\text\textdollar_0\text{bar}\text\textdollar_1\text{baz}\text\textdollar_2$, the rotation table resulting from building the suffix array while treating all end-of-text markers as the same character (the smallest in the alphabet) is shown below.

```
 k  SA[k]  F          L
-----------------------
 0     11  $foo$bar$baz
 1      3  $bar$baz$foo
 2      7  $baz$foo$bar
 3      5  ar$baz$foo$b
 4      9  az$foo$bar$b
 5      4  bar$baz$foo$
 6      8  baz$foo$bar$
 7      0  foo$bar$baz$
 8      2  o$bar$baz$fo
 9      1  oo$bar$baz$f
10      6  r$baz$foo$ba
11     10  z$foo$bar$ba
```

Applying the standard LF-mapping $LF(k) = C[L[k]] + \mathrm{rank}_{L[k]}(L, k)$ directly to an FM-index constructed with this method causes issues when $L[k]=\text\textdollar$.

For example, $L[6]=\text\textdollar$ corresponds to the end of the text fragment $T_1=\text{bar}$. However, applying the standard LF-mapping formula yields:

$$
\begin{align*}
LF(6) &= C[\text\textdollar] + \mathrm{rank}_{\text\textdollar}(L, 6) \\\\
&= 0 + 1 \\\\
&= 1 \\\\
\end{align*}
$$

The second row of the F-column, $F[1]$, corresponds to the end of the text fragment $T_0=\text{foo}$, which is different from the end of $T_1$ where we started.
Since LF-mapping is used in $\mathrm{locate}$ queries, this discrepancy can hinder obtaining accurate search results.

On the other hand, it is not straightforward to construct a suffix array by assigning an order to the end-of-text markers.
While we could consider methods such as extending the alphabet to include pairs of (character, position) – where the position is the text index for end-of-text markers and 0 otherwise – or modifying the suffix array construction algorithm itself, these approaches increase implementation overhead and complexity.

Instead of introducing an order for the end-of-text markers, we can correctly compute the LF-mapping by storing the position $k_0$ in the $SA$ that corresponds to the beginning of the entire concatenated text $T$ (i.e., $SA[k_0] = 0$).

#### Construction

The additional data structures, $\mathrm{Doc}$ and $\mathrm{Pos}$, are handled similarly to the method using distinct end-of-text markers.

To find $k_0$, we can iterate through the positions $k$ of the end-of-text markers in $T_\mathrm{BWT}$ during the construction of $\mathrm{Doc}$.
When we encounter the end-of-text marker that corresponds to the end of the last text fragment, i.e., if the original distinct marker was $\text\textdollar_{d-1}$, then we set $k_0 = k$.

#### Queries

Let's consider how to compute the LF-mapping using $k_0$.
Below is the table showing the suffixes of the text $T=\text{foo}\text\textdollar_0\text{bar}\text\textdollar_1\text{baz}\text\textdollar_2$ sorted lexicographically.
Remember that during FM-index construction with the same end-of-text marker, $\text\textdollar_0$, $\text\textdollar_1$ and $\text\textdollar_2$ are treated as the same symbol $\text\textdollar$.

```
 k  SA[k]  F          L
-----------------------
 0     11  $          z F[0] = $_2
 1      3  $bar$baz$  o F[1] = $_0
 2      7  $baz$      r F[2] = $_1
 3      5  ar$baz$    b
 4      9  az$        b
 5      4  bar$baz$   $ L[5] = $_0
 6      8  baz$       $ L[6] = $_1
 7      0  foo$bar$baz$ L[7] = $_2
 8      2  o$bar$baz$ o
 9      1  oo$bar$baz$f
10      6  r$baz$     a
11     10  z$         a
```

In this example, $k_0 = 7$.

Let's compare the rows where the character in the L-column is $\text\textdollar$ (in the example, $k = 5, 6, 7$), and the rows where the character in the F-column is $\text\textdollar$ (in the example, $k = 0, 1, 2$) to observe how these end-of-text markers are related by the LF-mapping.

- If $k = k_0$, LF-mapping always maps $L[k_0] = \text\textdollar_{d-1}$ to $F[0]$.
This is because $\text\textdollar_{d-1}$ corresponds to the lexicographically smallest suffix.
In the example, $L[7] = \text\textdollar_2$ corresponds to $F[0]$.
- For other end-of-text markers, their order of appearance is consistent between the L-column and the F-column.
This is because for a row with $L[k] = \text\textdollar_x$, the corresponding suffix is $T[SA[k], n-1]$. The suffix corresponding to $LF(k)$ is $T[SA[LF(k)], n-1] = \text\textdollar_x T[SA[k], n-1]$, thus preserving the lexicographical order of the suffixes starting with end-of-text markers.
In this example, $L[5]=\text\textdollar_0$ and $L[6]=\text\textdollar_1$ appear in the same order as $F[1]$ and $F[2]$, respectively.

Therefore, the LF-mapping $LF(k)$ when $L[k]$ is an end-of-text marker can be computed as follows:

$$
LF(k) = \begin{cases}
\mathrm{rank}\_\text\textdollar(T\_\mathrm{BWT}, k) + 1 & \text{if } k < k_0 \\\\
0 & \text{if } k = k_0 \\\\
\mathrm{rank}\_\text\textdollar(T\_\mathrm{BWT}, k) & \text{if } k > k_0 \\\\
\end{cases}.
$$

For characters other than $\text\textdollar$, the standard LF-mapping $LF(k) = C[L[k]] + \mathrm{rank}_{L[k]}(L, k)$ is used.

### Concatenation Without End-of-Text Markers

In this method, all text fragments are simply concatenated without using explicit end-of-text markers to form a single text $T = T_0T_1 \cdots T_{d-1}\text\textdollar$.
In this case, it becomes necessary to record which text fragment each character in the concatenated text originates from.

To achieve this, we construct a bit vector $B$ of the same length as the concatenated text $T$, as follows:

- If the character $T[i]$ in the concatenated text is the last character of a text fragment, then $B[i]=1$.
- Otherwise,$B[i] = 0$.

We use a data structure for the bit vector $B$ that allows for efficient processing of queries like $\mathrm{rank}_1(B, i)$.

#### Construction

By representing the bit vector $B$ using succinct bit vectors[^3], we can achieve constant time for $\mathrm{rank_1}$ queries while keeping the size overhead small.
Furthermore, since the number of 1s in $B$ is typically small compared to the text length (i.e. $B$ is sparse), we can also store it more compactly using techniques like Elias-Fano encoding.

#### Queries

After obtaining the occurrence position $i$ of a pattern $P$ using the $\mathrm{locate}$ query, we can find the text fragment $T_x$ to which $P$ belongs by computing $x = \mathrm{rank}_1(B, i)$.
However, since the pattern might span across multiple text fragments, we need to separately verify if the end position of the pattern, $i + |P| - 1$, belongs to the same text fragment as the start position $i$.

The relative position of the pattern $P$ within the text fragment $T_x$ ​can be calculated as follows:

- If the occurrence is within the first text fragment $T_0$ ($x = 0$), then the occurrence position $i$ in the concatenated text is the relative position.
- If the occurrence is within a subsequent text fragment $T_x$ ($x > 0$), then the starting position of text fragment $T_x$ is $\mathrm{select_1}(B, x - 1) + 1$, and the relative position is $i - (\mathrm{select_1}(B, x - 1) + 1)$.

For example, for the text $T=\text{foobarbaz\textdollar}$, the bit vector recording the end positions of the text fragments would be $B = 0010010010$.

```
x   0  1  2
i   0123456789
--------------
T = foobarbaz$
B = 0010010010
--------------
       012
```

The occurrence position of the pattern $P = \text{ar}$ in the concatenated text is $i = 4$.
Since $\mathrm{rank}_1(B, 4) = 1$, we can determine that the pattern $P$ occurs in text fragment $T_1$.
Its relative position is $4 - (\mathrm{select_1}(B, 1 - 1) + 1) = 4 - (2 + 1) = 1$.

Furthermore, we can use $B$ to determine if a pattern $P$ occurs as a prefix or a suffix of a text fragment.

#### Computational Complexity

In addition to the FM-index for a single text, the bit vector $B$ is required.
If $B$ is implemented as a succinct bit vector, its size is $O(n)$ bits.
The identifier and relative position of the text fragment corresponding to an occurrence position can be computed in $O(1)$ time.

## Conclusion

We introduced how fast string searching across multiple texts can be achieved using the FM-index.

The content presented is based on the cited literature, but it also includes some ideas that the author conceived during the implementation of the FM-index.
There might exist more standard and efficient techniques that the author was not aware of[^4].

[^1]: Ge Nong, Sen Zhang, & Wai Hong Chan. (2011). Two Efficient Algorithms for Linear Time Suffix Array Construction. IEEE Transactions on Computers, 60(10), 1471–1484. https://doi.org/10.1109/TC.2010.188

[^2]: Arroyuelo, A., Claude, F., Maneth, S., Mäkinen, V., Navarro, G., Nguyen, K., Siren, J., & Välimäki, N. (2011). Fast In-Memory XPath Search over Compressed Text and Tree Indexes (No. arXiv:0907.2089). arXiv. https://doi.org/10.48550/arXiv.0907.2089

[^3]: Navarro, G., & Providel, E. (2012). Fast, small, simple rank/select on bitmaps. In International Symposium on Experimental Algorithms (pp. 295-306). Berlin, Heidelberg: Springer Berlin Heidelberg. https://doi.org/10.1007/978-3-642-30850-5_26

[^4]: In particular, [Compact Data Structures](https://www.cambridge.org/core/books/compact-data-structures/68A5983E6F1176181291E235D0B7EB44) is a book by Professor Navarro, a leading figure in the field, and it seems to cover a wide range of topics related to succinct data structures and indexing. It may also mention applications to multiple texts (not yet read).

