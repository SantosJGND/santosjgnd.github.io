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


### Counting shared occurrences

The naive approach is to loop through the sequence, and for every position in the sequence, check if the pattern appears in another. This is not efficient. For a sequence of length `n`, and a pattern of length `k`, this has a time complexity of `O(nk)`.

Lets try to solve this problem using matrix multiplication. Let's say we have constructed a list of profiles, 

{% highlight python %}

profiles = [
    [1, 0, 1, 1],
    [0, 1, 1, 0],
    [0, 1, 0, 1]
]

{% endhighlight %}

And we wish to create a matrix of shared occurrences. We can do this by multiplying the transpose of the profiles with the profiles themselves, with the resulting product:

{% highlight python %}

product= [
    [3, 1, 1],
    [1, 2, 1],
    [1, 1, 2]
]

{% endhighlight %}

where the diagonal will represent the sum of patterns in each profile, and the off-diagonal elements will represent the number of shared occurrences between profiles.

this is easy in python using the numpy package:

{% highlight python %}

profiles= np.array([
    [1, 0, 1, 1],
    [0, 1, 1, 0],
    [0, 1, 0, 1]
])


product = profiles @ profiles.T

{% endhighlight %}

This is much more efficient. The time complexity of matrix multiplication is `O(n^3)`, so this is much faster than the naive approach.


### very large matrices

While matrix multiplication is efficient, it is not practical to keep entire matrices in memory. The solution is to split these operations into chunks, e.g.:

{% highlight python %}


def chunk_matrix(matrix, chunk_size):

    for i in range(0, matrix.shape[0], chunk_size):
        yield matrix[i:i+chunk_size, :]


for chunk in chunk_matrix(profiles, 2):

    subset_product= chunk @ profiles.T

    ## do something with subset_product

{% endhighlight %}
    
    


# Conclusion

This is a simple idea with many applications. For example i have used it to extract the mutation spectrum in primates [1], more recently to study patterns of cross-mapping in metagenomics [2]. It occurs me that it could be useful to apply to VCF files given their binary nature. 

[1] https://doi.org/10.1093/gbe/evad019

[2] https://insaflu.insa.pt/



