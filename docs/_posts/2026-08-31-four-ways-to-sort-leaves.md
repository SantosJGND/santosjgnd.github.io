---
layout: post
title: "Four ways to sort leaves before a recall curve, and a baseline that ignores its own argument"
date: 2026-08-31 12:47:45 +0100
categories: python ml bioinformatics
---

A recall curve only means something once you fix an order for the leaves. The pipeline keeps the first N leaves
in that order and asks how many of the true taxa survived. That means the sort order is part of the metric, not
just a presentation detail. I wanted to compare a few orders, and while setting up the comparison I found a bug in
the simplest baseline.

### Four sort orders

The sort strategies are defined as a literal type at the top of the feature transformer:

```python
SORT_STRATEGIES = Literal["reads", "taxid_roundrobin", "rarity_boost", "tax_level_stratified"]
```

The default is `reads`, which sorts by unique reads, highest first. That is the simple choice, and the one we had
been using. The other three were my attempts to do better:

- **Round-robin by taxid** ranks leaves within each taxid and then sorts by that rank first. The idea is to get
  as many distinct taxids as possible early in the list:

  ```python
  df["_recall_sort_rank"] = df.groupby("taxid", sort=False).cumcount()
  result = df.sort_values(["_recall_sort_rank", "total_uniq_reads"], ascending=[True, False])
  ```

- **Rarity boost** scores each leaf as its read count times one minus the share of its taxon group. Leaves in
  crowded groups drop down the list.
- **Stratified by tax level** is the round-robin idea applied to the taxonomic level instead of the taxid.

### Why the order changes the target too

The recall targets are computed on the sorted list. `index_recall_d` is the fraction of best-match leaves that
fall inside the first `d / data_set_divide` of the sorted order. So changing the sort also changes what the model
is trained to predict. Two strategies are only comparable if they are trained and tested the same way, and the
comparison script does that: it trains on the training datasets and cuts the held-out test list at the predicted
percentile. It records the recall actually reached and divides it by the maximum possible recall.

### The results are not in the repo

I looked for the output of this comparison: the changelog, both READMEs, the analysis docs. There is none.
The script writes its results as TSV files when you run it, and those files are not committed. So I cannot tell
you which strategy wins, and I am not going to write one in. This post describes what the code does, and the
numbers are still to come.

### A baseline that ignores its argument

The baseline is supposed to keep leaves until it has covered N distinct best-match taxids. The class takes `n_taxids`
as an argument:

```python
class FixedCountModeller:
    def __init__(self, n_taxids: int = 12, feature_transformer=None):
        self.n_taxids = n_taxids
```

The cutoff method, though, has the number hard-coded:

```python
if sorted_stats.shape[0] <= self.n_taxids:
    return 1.0
else:
    return 12 / sorted_stats.shape[0]
```

So `fixed_20` would still compute `12 / N`. The documentation says any `fixed_N` is accepted, so the documentation
and the code disagree. The baseline is only correct for N equal to 12. The fix is to use `self.n_taxids` in the
division, but any `fixed_N` result we have already produced for N other than 12 should be thrown out.

### What I want to do next

Fix the baseline, run the sweep, and commit the TSV it produces, so the comparison can be checked by someone else.
Only then can we say which sort order helps. The cutoff model that consumes these sorted lists is in
[the cutoff post]({% post_url 2026-08-31-predicting-the-recall-cutoff %}).
