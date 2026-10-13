---
layout: post
title: "From shared reads to a distance matrix to clusters"
date: 2026-07-15 15:30:45 +0000
categories: python metagenomics clustering
---

INSaFLU's metagenomics module needs to decide which reference genomes are the same organism. The signal we
have is the reads: if two references share many reads after mapping, they are probably close. This post walks
through how a BAM file per reference becomes a distance matrix, a tree, and finally a set of clades.

### Step 1: one BAM per reference

Each accession has a BAM of mapped reads. `_symlink_bams_to_temp` (`overlap_manager.py:655-667`) links every
BAM into a temporary directory under the name `{accid}.bam`. The external binary is called as:

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

`max_reads` defaults to 500000. The binary writes `distance_matrix.tsv` and `presence_absence_matrix.tsv`.

### Step 2: shared-read counts

The presence matrix is read back as a pandas frame. Shared reads between two references are a matrix
product:

```python
self.overlap_matrix = pairwise_shared_count(presence_matrix)   # binary @ binary.T
```

There is also a Python version of the proportion distance (`pairwise_shared_reads_distance`,
`overlap_manager.py:210-229`). It takes the larger of the two directional proportions, then returns
`1 - proportion`. In the production path, though, the tree is built from the binary's `distance_matrix.tsv`.
The Python version is called from tests. That split is worth knowing before you change either one.

### Step 3: tree

`tree_from_distance_matrix` uses neighbour-joining when there is one row or fewer, and UPGMA otherwise.

### Step 4: clades

Traversal decides where to stop cutting the tree. The default `clustering_model_type` is `"Fixed"`
(`overlap_manager.py:384`). In that mode a node is accepted when its minimum pairwise shared proportion
reaches `shared_proportion_min`. Any other value sends the traversal to the ML service:

```python
result = client.predict_composition_stop_traversal(features, model=model_type)
return bool(result["stop_traversal"]), float(result.get("probability", 0.0))
```

The client (`ml_api_client.py`) is a small wrapper over `requests`. It has a base URL built from
`insaflu-ml-app` and port 8000, a 30 second timeout, and an `MLAPIError` class for non-OK responses.

### Problems I found while reading

1. **Fallback is only in the log message.** When the ML call fails, `_predict_stop_traversal` returns
   `None`. The log says "falling back to fixed traversal", but the traversal path then raises
   `RuntimeError("ML API unavailable and no fallback model provided.")`.
2. **Connection errors are not wrapped.** `ml_api_client` turns HTTP errors into `MLAPIError`, but
   connection failures and timeouts from `requests` propagate unchanged.
3. **Cache hit leaves attributes unset.** When `generate_distance_matrix` reads the cached TSV, it does not
   set `read_profile_matrix_filtered`, `overlap_matrix`, or `_accid_statistics_df`. Code that expects those
   after a cache hit will fail or need a recompute.
4. **Sampling is unseeded.** `filter_read_matrix` samples `max_reads` columns with no random seed, so two
   runs can produce different clusters.

### What I would change

Seed the sample, and make the cache path set the same attributes as the compute path (or move the attributes
into a single load function). Make the ML fallback real, or change the log message so it says what happens.

The threshold that decides when a set of references is "the same" is the same kind of question as the
stopping rule in [the stop-splitting post]({% post_url 2026-09-28-deciding-when-to-stop-splitting-a-tree %}).
