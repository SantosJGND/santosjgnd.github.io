---
layout: post
title: "From shared reads to a distance matrix to clusters"
date: 2026-07-15 15:30:45 +0000
categories: python metagenomics clustering
---

In the metagenomics module we need to decide which reference genomes belong to the same organism. The signal we
have is the reads: if two references pull in many of the same reads after mapping, they are probably close
relatives. I wanted to understand how that signal becomes a tree and then a set of clades, so I followed the code
from the BAM files to the final clusters.

### One BAM per reference

Each accession has a BAM file of its mapped reads. The code links every BAM into a temporary directory under the
name of its accession, and then calls an external binary that does the heavy work:

```python
cmd = [
    binary,
    "--input-directory", output_dir,
    "--max-reads", str(self.max_reads),
    "--frequency-threshold", str(self.min_freq) if self.min_freq > 0 else "0.1",
    "--output-directory", output_dir,
    "--no-cluster-analysis",
]
```

The read limit defaults to 500,000. The binary writes two matrices: a distance matrix and a presence/absence
matrix of reads.

### From presence to shared reads

The presence matrix is read back into pandas. Shared reads between two references are then just a matrix product
of the binary matrix with its transpose:

```python
self.overlap_matrix = pairwise_shared_count(presence_matrix)
```

There is also a Python version of a proportion-based distance, `pairwise_shared_reads_distance`. It takes the larger
of the two directional proportions and returns one minus it. It is a nice idea, but when I checked, the production
path builds the tree from the binary's distance matrix. The Python version is only called from tests. That split
matters if you change one and expect the other to move.

### The tree and the clades

The tree is built with neighbour-joining when there is one row or fewer, and UPGMA otherwise. The clades come from
a traversal that decides where to stop cutting. By default, a node is accepted when its minimum pairwise shared
proportion reaches a threshold. Any other model setting sends the decision to our ML service:

```python
result = client.predict_composition_stop_traversal(features, model=model_type)
return bool(result["stop_traversal"]), float(result.get("probability", 0.0))
```

The client is a small wrapper around `requests`. It has a base URL built from the service name and port 8000, a
30 second timeout, and an error class for bad responses.

### Where it breaks

Reading this path, I found four things I would fix.

When the ML call fails, the code logs "falling back to fixed traversal". It does not fall back. The function
returns `None`, and the caller then raises an error saying no fallback model was provided. The log message describes
behaviour the code does not have.

The client only wraps HTTP errors. Connection failures and timeouts from `requests` pass straight through, so
callers see a different exception type depending on how the service failed.

When the distance matrix is loaded from cache, several attributes are never set: the filtered read matrix, the
overlap matrix, and the accession statistics. Any code that expects them after a cache hit will fail, or the
results will differ from a fresh run.

Finally, the read sampling is not seeded. If there are more reads than the limit, the script keeps a random subset,
so two runs on the same data can give different clusters. That makes the results hard to reproduce, and it is the
first thing I would change.

The cut-off question here is the same kind of question as the stopping rule in
[the stop-splitting post]({% post_url 2026-09-28-deciding-when-to-stop-splitting-a-tree %}): how far down the tree
should we go before we stop calling things the same organism?
