---
layout: post
title: "Four ways to sort leaves before a recall curve, and a baseline that ignores its own argument"
date: 2026-08-31 12:47:45 +0100
categories: python ml bioinformatics
---

A recall curve only means something once you fix an order for the leaves. The pipeline keeps the first N
leaves in that order and measures how many true taxa survived. So the sort order is part of the metric.
This post covers the four sort strategies in the code, what they are for, and a bug in the simplest baseline.

### The four strategies

`feature_transformer.py:16` defines the options:

```python
SORT_STRATEGIES = Literal["reads", "taxid_roundrobin", "rarity_boost", "tax_level_stratified"]
```

- **`reads`**: sort by `total_uniq_reads`, descending. This is the default (`feature_transformer.py:49`).
- **`taxid_roundrobin`**: rank leaves within each `taxid` group, then sort by rank first. The docstring
  says it "maximises distinct taxids early".
- **`rarity_boost`**: score each leaf as reads times `(1 - fraction of its tax_level group)`. Leaves in
  crowded groups drop.
- **`tax_level_stratified`**: the same round-robin idea, grouped by `tax_level` instead of `taxid`.

The round-robin version is short:

```python
df["_recall_sort_rank"] = df.groupby("taxid", sort=False).cumcount()
result = df.sort_values(["_recall_sort_rank", "total_uniq_reads"], ascending=[True, False])
```

The rarity score:

```python
taxon_totals = df.groupby(self.tax_level)["total_uniq_reads"].transform("sum")
total_reads = df["total_uniq_reads"].sum()
fraction = taxon_totals / total_reads if total_reads > 0 else 0
df["_recall_sort_score"] = df["total_uniq_reads"] * (1.0 - fraction)
```

### What the target depends on

The recall targets are defined on the sorted list (`feature_transformer.py:136-146`). `index_recall_d` is the
fraction of best-match leaves that fall in the first `d / data_set_divide` of the sorted order. Changing the
sort changes the target itself, so two strategies are only comparable if they are trained and tested the
same way.

### The comparison script

`analysis_scripts/compare_sort_strategies.py` sweeps `sort_strategy` by `recall_model`:

```python
DEFAULT_SORT_STRATEGIES = ["reads", "taxid_roundrobin", "rarity_boost", "tax_level_stratified"]
DEFAULT_RECALL_MODELS = ["xgb", "morf", "direct_xgb", "gp_clf", "fixed_12_taxids"]
```

For each combination it trains on training datasets, cuts the held-out test list at the predicted
percentile, and records `actual_recall` and `standardized_recall = actual / max_possible`. The summary is
grouped by `(sort_strategy, recall_model)`.

### Results: none are in the repo

I searched the changelog, both READMEs, the output description, and the analysis docs for results of this
comparison. There are none. The script writes `sort_strategy_comparison_results.tsv` at runtime, and that
file is not committed. So I cannot tell you which strategy wins. Anything I wrote here about a winner would
be invented.

### A baseline that ignores its argument

The "fixed" baseline is supposed to keep leaves until N distinct best-match taxids are covered. The class
takes `n_taxids`:

```python
class FixedCountModeller:
    def __init__(self, n_taxids: int = 12, feature_transformer=None):
        self.n_taxids = n_taxids
```

But the cutoff is hard-coded:

```python
if sorted_stats.shape[0] <= self.n_taxids:
    return 1.0
else:
    return 12 / sorted_stats.shape[0]
```

`fixed_20` would still return `12 / N`. The documentation says any `fixed_N` pattern is accepted. The
baseline is therefore wrong for every N except 12. A `seen` set in the same method is also never used.

The fix is one line (`self.n_taxids / sorted_stats.shape[0]`), but the results from any `fixed_N` run
before that fix should be thrown away.

### Takeaway

Sort order changes the recall target and the metric together. Before comparing strategies, make sure the
baseline is correct, run the sweep, and commit the TSV so the comparison can be checked. The cutoff model
that consumes these sorted lists is in
[the cutoff post]({% post_url 2026-08-31-predicting-the-recall-cutoff %}).
