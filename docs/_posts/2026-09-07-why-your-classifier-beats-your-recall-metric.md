---
layout: post
title: "Why your classifier is better than your recall metric says"
date: 2026-09-07 01:01:16 +0100
categories: python metagenomics metrics
use_math: true
---

When we benchmark metagenomic classifiers, the headline number is usually recall: what fraction of the
organisms we put into a sample did the pipeline get back out? The trouble is that recall is a *pipeline*
result, not a *classifier* result. By the time we count it, the answer has passed through assembly
selection, reference matching, coverage filtering and clade cleanup. Any of those stages can quietly eat
a taxon the classifier actually found.

This post is about a disagreement I kept running into: on one viral study the classifier was clearly
detecting around 83% of the input taxa, but our recall metric reported roughly 31%. That is not a small
gap. It was a metric bug, not a classifier bug.

### How recall was being counted

The evaluation pipeline builds, for every dataset, a matrix of leaf statistics (`m_stats`). Each leaf is a
node in the overlap graph between reads and reference assemblies, annotated with its best taxonomic
match. Recall was computed by asking: for each input taxid, is there a leaf whose `best_match_taxid`
equals it?

```python
input_taxids = set(input_summary["taxid"].unique())
unique_taxids = len(input_taxids)

clean_unique = clean_m_stats.dropna(subset=["best_match_taxid"])
clean_unique = clean_unique[
    (clean_unique["is_trash"] == False) & (clean_unique["best_match_is_best"] == True)
]
output_taxids = set(clean_unique["best_match_taxid"].dropna().unique())

recall = len(output_taxids & input_taxids) / unique_taxids
```

Read that condition carefully: a taxid only counts if it survives as the *best match* of a non-trash leaf.
If the classifier names a species but no assembly got reads, or the taxid was resolved at a different
rank, the leaf is dropped and the taxon is scored as "not recalled" — even though the classifier found it.

### Three bugs stacked on each other

Once I started pulling the thread, the gap was not one bug but three, compounding.

**1. Recall was accidentally coverage-filtered.** The loop that selected best matches skipped leaves with
`coverage == 0`. So `recall_baseline` and `recall_baseline_cov_filtered` were byte-identical on every
dataset in the study — 439 out of 439. If two metrics that are supposed to measure different things never
differ, one of them is not measuring what you think.

**2. Lineages were only resolved for input taxids.** When scoring a leaf against an input reference, we
divided the length of the shared lineage prefix by `len(lineage1)` — the input taxon's lineage. A leaf
resolved one rank deeper than the input (extra sub-species nodes) could fall below the 0.7 match threshold
against *its own species*. The fix was to make the score symmetric, dividing by
`min(len(lineage1), len(lineage2))`, plus a zero-length guard and an exact-taxid shortcut.

**3. The metric was assembly-only by construction.** Nothing in the definition ever consulted the
classifier's own output.

### Making recall classification-aware

The fix for (3) is the interesting one. We still want the assembly-based number — it tells you whether you
recovered something you can actually map reads to — but we also want credit for what the classifier
detected. So `compute_recall` takes an optional set of taxids the classifier called, and unions it in:

```python
def compute_recall(clean_m_stats, input_summary, extra_detected_taxids=None):
    input_taxids = set(input_summary["taxid"].unique())
    unique_taxids = len(input_taxids)
    if unique_taxids == 0:
        return 0.0, 0.0, [], []

    clean_unique = clean_m_stats.dropna(subset=["best_match_taxid"])
    clean_unique = clean_unique[
        (clean_unique["is_trash"] == False) & (clean_unique["best_match_is_best"] == True)
    ]
    output_taxids = set(clean_unique["best_match_taxid"].dropna().unique())

    detected_taxids = set(output_taxids)
    if extra_detected_taxids:
        detected_taxids |= set(extra_detected_taxids)

    recall_raw = len(detected_taxids & input_taxids) / unique_taxids

    # strictly assembly-based, requires coverage > 0
    clean_cov = clean_unique[clean_unique["coverage"] > 0]
    output_cov = set(clean_cov["best_match_taxid"].dropna().astype(int).unique())
    recall_cov = len(output_cov & input_taxids) / unique_taxids

    return recall_raw, recall_cov, sorted(detected_taxids & input_taxids), sorted(output_cov & input_taxids)
```

Now we have two honest numbers instead of one dishonest one: `recall_raw` (did we *detect* it) and
`recall_cov_filtered` (did we *recover and map* it). The gap between them is itself a useful signal — it
separates classification failures from assembly failures.

### Decomposing the loss instead of guessing

A single recall number, even a correct one, does not tell you *where* a reference was lost. So alongside
recall we built a per-reference funnel. Each reference is assigned a `loss_stage` by a first-match cascade,
from "never classified" through "classified but had no assembly" down to "survived cleanup":

```python
LOSS_STAGES = (
    "absent_classification",
    "survived_classifier_only",
    "classified_no_assembly",
    "assembly_no_coverage",
    "assembly_zero_reads",
    "lost_recall_truncation",
    "lost_clade_prediction",
    "survived_post_cleanup",
)

# the only stages that count against classification-aware recall
LOSS_LOSSES = ("absent_classification", "classified_no_assembly")
```

The key trick is that the cascade is *verified*, not trusted. Each stage sets an independent boolean flag
(`lost_no_coverage`, `lost_zero_reads`, `recalled_classifier_only`, ...), and an invariant ties the funnel
back to the headline metric:

```
recalled_classification_aware == (loss_stage NOT IN LOSS_LOSSES)
```

If that assertion fails, the funnel and the recall number disagree and the run is invalid. When we first
enforced it, 12 of 14 references in the study had been mislabelled by the old, unverified decomposition.

### What I took away

- A metric computed at the end of a pipeline measures the whole pipeline. If you want to evaluate one
  component, you have to instrument that component directly.
- Two metrics that never disagree are not two metrics. The `recall_baseline ==
  recall_baseline_cov_filtered` identity on 439/439 datasets was the tell.
- Prefer a funnel you can *assert* against your headline number over a single scalar you have to trust.

The classifier really was finding those taxa. The metric just wasn't looking for them.

---

*This is part of a series on evaluating metagenomic classification. See also
[the tree-classification prototype]({% post_url 2025-11-24-decision-tree-classification %}) that motivated
this line of work.*
