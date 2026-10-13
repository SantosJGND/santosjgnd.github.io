---
layout: post
title: "Predicting where to stop: a recall cutoff model and a penalty written the wrong way round"
date: 2026-08-31 11:54:21 +0100
categories: python ml scikit-learn
---

Once the pipeline has ranked the leaves, someone has to decide how many of them to keep. Keep too few and we lose
recall. Keep too many and the analysis gets slow and noisy. I spent a while on a model that predicts this cutoff,
and while reading its loss function I found a penalty that points in opposite directions in two places.

### What we are trying to predict

For each dataset, the true cutoff is the first point in the sorted leaf list where recall reaches a threshold
`tau`. `compute_ground_truth_division` in `om_models.py` finds that point by interpolating between the two
fractions either side of the threshold:

```python
def compute_ground_truth_division(recall_at_divs, tau, n_divisions, fractions):
    for i in range(n_divisions):
        if recall_at_divs[i] >= tau:
            ...
            frac = f_low + (tau - r_low) * (f_high - f_low) / (r_high - r_low)
            return float(np.clip(frac, f_low, f_high))
    return 1.0
```

If recall never reaches `tau`, the target is 1.0, meaning keep everything.

### The features

The feature transformer builds one row per dataset. The inputs are the total and maximum unique read counts,
the number of hits, the skewness and kurtosis of the counts, the Shannon diversity of taxa, and the proportion of
reads in each taxon at a chosen level, which defaults to order. The targets are the recall reached within each
fraction of the sorted list.

### The loss I got wrong, first

The direct regression model weights its errors with a function that makes mistakes on small true fractions
count more. Its docstring says it penalises underestimation three times more than overestimation, and the code
passes `under_cost=3`, so that part matches:

```python
def _asymmetric_weights(self, gt, over_cost=1.0, under_cost=3.0, floor=0.05):
    base = 1.0 / np.clip(gt, floor, 1.0)
    return base * under_cost + (1.0 - base / base.max()) * over_cost
```

The grid search for the classifier model is different. It picks a probability threshold and a decision fraction
by minimising a loss, and it calls that loss with `over_cost=3, under_cost=1`. The docstring for that code says
the opposite: "Over-prediction (too late) penalised 3x heavier than under-prediction (too early), incentivising
slightly conservative cutoffs that preserve recall."

Those two cannot both be right. I read the second one carefully. If predicting too late costs three times more,
the optimiser will prefer lower predictions, which means earlier cutoffs. Earlier cutoffs keep less recall, not
more. So the docstring's claim that it preserves recall does not follow from the code. I have not run an
experiment to confirm this, so it is my reading, not a result.

The last-TP prediction script has the same penalty constants as the direct model, and a comment there that says
the goal is to fight overestimation. Three places, two directions, and a comment that disagrees with its own
numbers.

### What I think should happen

First, decide which mistake we care about. Predicting too early loses recall, which is the thing we measure.
Predicting too late costs time, which we care less about. Then name the arguments after what they penalise,
use the same convention everywhere, and check the true recall at the predicted cutoff on the test set against
`tau`. The model output file already reports that number, so the check is cheap.

The stopping rule for each split is in [the stop-splitting post]({% post_url 2026-09-28-deciding-when-to-stop-splitting-a-tree %}),
and the recall metric that these cutoffs are trained against is in
[the recall metric post]({% post_url 2026-09-07-why-your-classifier-beats-your-recall-metric %}).
