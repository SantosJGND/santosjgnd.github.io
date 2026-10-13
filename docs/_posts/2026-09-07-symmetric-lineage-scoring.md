---
layout: post
title: "The symmetric fix that only fixed half of compare_lineages"
date: 2026-09-07 01:01:16 +0100
categories: python debugging bioinformatics
---

Our changelog says we fixed lineage scoring to be symmetric. When I went back to check that entry I found that the fix is in a function the pipeline does not call in the common case. This post is
about how that happened, and how a test on the right function let the wrong one through.

### The problem with the old score

When we match a reference taxon to a leaf, we compare their lineages from the root down. The old score divided
the shared prefix by the length of the first lineage only. That means the score depended on which taxon was
passed first. A leaf with extra sub-species ranks scored badly against its own species, and with our threshold
of `min_taxonomic_score=0.7`, correct matches were being dropped.

The fix was to divide by the shorter of the two lineages instead:

```
shared_prefix / min(len(lineage1), len(lineage2))
```

### The function I fixed

The symmetric version lives in `compare_lineages()` in `metagenomics_utils/ncbi_tools.py`. It splits both
lineages, takes the shorter length, and counts matching ranks until the first mismatch:

```python
levels1 = lineage1.split("; ")
levels2 = lineage2.split("; ")
min_length = min(len(levels1), len(levels2))
score = 0
for i in range(min_length):
    if levels1[i] == levels2[i]:
        score += 1
    else:
        break
...
return score / min_length, level
```

There is a test file for it, `tests/deployment/model_evaluation/test_recall_fixes.py`, and the test passes.
I was pleased with that at the time.

### The function that actually runs

The matching code in `node_stats.py` never calls `compare_lineages()` directly. It calls
`compare_lineages_relative()` on the wrapper class. Reading it again, I found that it only uses the symmetric
version in one case, when one of the taxa has a rank of level zero:

```python
if min_levels == 0:
    ...  # symmetric compare_lineages(), as above
    return tax, level

lineage1 = sorted(self.lineages[taxid1].items(), key=lambda x: x[1]["level"])
for tax, level in lineage1:
    if self.get_level(taxid2, tax) != level["name"]:
        return (level.get("level", 0) - 1) / len(lineage1), tax   # still asymmetric
return 1.0, lineage1[-1][0] if lineage1 else None
```

For every other pair, the score is still divided by `len(lineage1)`. So the asymmetry I fixed in one function
is still there in the path that matters.

### Why the test missed it

The test only calls `compare_lineages`. Nothing calls `compare_lineages_relative` with a leaf that has extra
sub-species ranks, so the asymmetric branch is never exercised. The test is correct for what it checks. It just
checks the wrong function.

### What the numbers do and do not show

The changelog reports per-taxid assembly recall going from 0.305 to 0.564, with 3,012 gains. That entry also
lists other fixes made in the same run, and it does not separate out how much of the gain came from the
symmetric score. So I cannot say how much this change contributed, and I am not going to guess.

### What I would do next

Add a test for `compare_lineages_relative` with a leaf that has extra ranks. Log how often the `min_levels == 0`
branch runs on a real dataset. If it is rare, the symmetric fix is mostly inactive and should be applied to the
main branch too. Then run the recall analysis once with each version, so we can see the difference on its own.

This is the same investigation as [the recall metric post]({% post_url 2026-09-07-why-your-classifier-beats-your-recall-metric %}).
The lineage score was one of the three bugs fixed there.
