---
layout: post
title: "Predicting where to stop: a recall cutoff model and a sign error in its loss"
date: 2026-08-31 11:54:21 +0100
categories: python ml scikit-learn
---

Once the pipeline ranks leaves, someone has to decide how many to keep. Keep too few and recall drops. Keep
too many and the analysis gets slow and noisy. This post is about the model that predicts that cutoff, and
about a penalty that is written in the opposite direction in two places.

### The target

For each dataset, the true cutoff is the first fraction of the sorted leaf list where recall reaches a
threshold `tau`. `compute_ground_truth_division` (`metagenomics_utils/overlap_manager/om_models.py:249-261`)
interpolates that fraction:

```python
def compute_ground_truth_division(recall_at_divs, tau, n_divisions, fractions):
    for i in range(n_divisions):
        if recall_at_divs[i] >= tau:
            if i == 0:
                return float(tau * fractions[0] / max(recall_at_divs[0], EPS))
            r_low, r_high = recall_at_divs[i - 1], recall_at_divs[i]
            f_low, f_high = fractions[i - 1], fractions[i]
            if r_high == r_low:
                return f_high
            frac = f_low + (tau - r_low) * (f_high - f_low) / (r_high - r_low)
            return float(np.clip(frac, f_low, f_high))
    return 1.0
```

If recall never reaches `tau`, the target is 1.0 (keep everything).

### The features

The feature transformer (`feature_transformer.py`) builds one row per dataset from:

- `total_uniq_reads`, `max_uniq_reads`, `number_hits`
- `counts_kurtosis`, `counts_skewness`
- `tax_diversity_shannon`
- per-taxon composition proportions at a chosen level (default `order`)

The targets are `index_recall_1` through `index_recall_N`: the recall reached within each fraction of the
sorted list.

### Two models, one loss

`DirectXGBRecallModeller` (`om_models.py:990-1084`) regresses the cutoff directly. Its sample weights make
errors on small true fractions count more:

```python
def _asymmetric_weights(self, gt, over_cost=1.0, under_cost=3.0, floor=0.05):
    base = 1.0 / np.clip(gt, floor, 1.0)
    return base * under_cost + (1.0 - base / base.max()) * over_cost
```

Its class docstring says it penalises underestimation 3:1. The code passes `under_cost=3`, so that matches.

`GPCLFRecallModeller` (`om_models.py:1492`) picks a probability threshold and a decision fraction by grid
search over a loss:

```python
tau_values = np.linspace(0.10, 0.975, 36)
X_values = np.linspace(0.025, 0.975, 39)
...
loss = asymmetric_loss(y_true_frac, y_pred, over_cost=3, under_cost=1)
```

Its docstring says: "Over-prediction (too late) penalised 3x heavier than under-prediction (too early),
incentivising slightly conservative cutoffs that preserve recall."

### The sign conflict

The "3:1" label means different things in different places:

| Location | Call | Heavier penalty, per the code |
|---|---|---|
| `om_models.py:1072` (DirectXGB weights) | `under_cost=3` | under-prediction |
| `om_models.py:1445`, `1697` (GP grid search) | `over_cost=3, under_cost=1` | over-prediction |
| `last_tp_division_prediction_second.py:918-919` | `TARGET_OVER=1`, `TARGET_UNDER=3` | under-prediction |

The DirectXGB weight formula is copied into the last-TP script, and its comment there says "fight
overestimation" while the constants weight underestimation 3:1. The two comments disagree with each other.

The GP grid search is the one that matters for the claim in its docstring. If too-late cutoffs cost 3x, the
optimiser prefers lower predicted fractions, meaning earlier cutoffs. Earlier cutoffs keep less recall. So
the docstring's "preserve recall" does not follow from the code as written. That is my reading, and I have
not run an experiment to confirm it.

### What to check before trusting the cutoff

1. Decide which error you want to penalise. Under-prediction loses recall; over-prediction costs time.
2. Make the three locations use the same convention and name the argument after what it penalises
   (`under_cost` and `over_cost` are not enough on their own).
3. Compare the true recall at the predicted cutoff on the test set against the target `tau`. The model
   output file (`om_models.py:1711-1723`) already reports this.

The decision rule itself is covered in [the stop-splitting post]({% post_url 2026-09-28-deciding-when-to-stop-splitting-a-tree %}).
The recall metric that the cutoff is trained against is in
[the recall metric post]({% post_url 2026-09-07-why-your-classifier-beats-your-recall-metric %}).
