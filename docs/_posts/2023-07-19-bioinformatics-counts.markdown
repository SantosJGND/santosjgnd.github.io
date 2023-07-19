---
layout: post
title: "Bionformatics, Matrix Multiplication and Hash Tables"
date: 2023-07-18 23:34:26 +0000
categories: bioinformatics matrix-multiplication hash-tables counting
---

A common problem in bioinformatics is to count the number of times a sequence appears in a genome. For example, you might want to know how many times the sequence `ATG` appears in the genome of a given organism. This is a simple problem, but it can be computationally expensive. The genome of a bacteria can be millions of base pairs long, and the sequence you are looking for can be of any length. So how do you solve this problem efficiently?

### Constructing profiles

Lets begin by how we record profiles - think of a record of the presence / absence of set of patterns in a given sequence. For example, for the sorted set `[A, T, C, G]`, we can represent the letter pattern `A` as `[1, 0, 0, 0]`, the letter pattern `T` as `[0, 1, 0, 0]`, and so on. We can then represent the sequence `ATG` as `[[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 0, 1]]`.

You may be tempted to use generate these profiles using membership tests, as follows:

{% highlight python %}

pattern= ["A"]
sorted_set = ["A", "T", "C", "G"]

def count_patterns(pattern, sorted_set):
    
    return [1 if letter in pattern else 0 for letter in sorted_set]
    

{% endhighlight %}

But this is not efficient. The `in` operator will loop through the entire sequence, and for every element in the sequence it will loop through the entire pattern.

Alternatively, we can use hash-tables (dictionaries in python):

{% highlight python %}

pattern= {"A": 1}
sorted_set = ["A", "T", "C", "G"]

def count_patterns(pattern, sorted_set):
    
    return [
        pattern.get(letter, 0) for letter in sorted_set
    ]
    

{% endhighlight %}

This is more efficient. The `get` method of dictionaries has a time complexity of `O(1)`, so this will be faster than the membership test. The difference is not noticeable for small sequences, but it becomes noticeable for large sequences. Counting occurrences could be with little modifications. 


### Counting occurrences

This is a more difficult problem. The naive approach is to loop through the sequence, and for every position in the sequence, check if the pattern appears in that position. This is not efficient. For a sequence of length `n`, and a pattern of length `k`, this has a time complexity of `O(nk)`.

A more efficient approach is to use matrix multiplication. This is a technique that is used in many areas of bioinformatics, and it is worth learning.

Lets begin by representing the sequence as a matrix. For example, the sequence `ATG` can be represented as:

{% highlight python %}
sequence = [
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 0, 1]
]
{% endhighlight %}

and the identity matrix of our reference set of letters can be represented as:

{% highlight python %}

pattern = [
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
]

{% endhighlight %}


to establish the number of occurences of each letter in our sequence we can multiply the sequence matrix by the identity matrix. This is equivalent to counting the number of times each letter appears in the sequence. For example, the product of the two matrices above is:

{% highlight python %}

product = np.array(
    [
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 0, 1]
]
)

{% endhighlight %}

The first row of the product matrix tells us that the pattern appears once in the first position of the sequence. The second row tells us that the pattern appears once in the second position of the sequence. The third row tells us that the pattern does not appear in the third position of the sequence.

We get the counts by simply summing the rows of the product matrix:

{% highlight python %}

counts = np.sum(product, axis=0)

{% endhighlight %}

This has the added advantage that matrix multiplication is very easy to perform in python using the numpy package. For the case above, the product mattrix would be calculated as follows:

{% highlight python %}
import numpy as np

sequence = np.array([
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 0, 1]
])


pattern = np.array([
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
])

product = sequence @ pattern.T

{% endhighlight %}

It is also an example of how knowledge from different areas can be combined to solve a problem. In this case, the idea for using matrix multiplication came to me because i was studying coalescent theory, which relies heavily on markov chains, and markov chains can be represented as matrices. 

I first came across this technique in the book [Bioinformatics Algorithms](https://www.bioinformaticsalgorithms.org/), by Phillip Compeau and Pavel Pevzner.