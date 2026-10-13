---
layout: post
title: "The symmetric fix that only fixed half of compare_lineages"
date: 2026-09-07 01:01:16 +0100
categories: python debugging bioinformatics
---

Our changelog says we fixed lineage scoring to be symmetric. The function that does the fix is correct. The
function the pipeline actually calls is not. This post is about that gap, and about how a test on the
"right" function let the wrong one through.

### The asymmetric score

A lineage score asks: how much of taxon A's lineage matches taxon B's lineage, starting from the root? The
original code divided the shared prefix by the length of the first lineage only. So a leaf with extra
sub-species ranks scored badly against its own species. With our threshold `min_taxonomic_score=0.7`, that
was enough to drop correct matches.

The changelog entry (`CHANGELOG.md:55`) says the score became:

```
shared_prefix / min(len(lineage1), len(lineage2))
```

### The symmetric function

`compare_lineages()` in `metagenomics_utils/ncbi_tools.py:318-341` does this:

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

It has a unit test in `tests/deployment/model_evaluation/test_recall_fixes.py:26-56`. The test passes.

### The function the pipeline calls

The matching code in `node_stats.py` does not call `compare_lineages()`. It calls
`NCBITaxonomistWrapper.compare_lineages_relative()` (`ncbi_tools.py:248-270`). That function only falls back
to the symmetric version in one case:

```python
min_levels = min(
    min([x["level"] for x in self.lineages[taxid1].values()]),
    min([x["level"] for x in self.lineages[taxid2].values()]),
)

if min_levels == 0:
    ...  # symmetric compare_lineages(), as above
    return tax, level

lineage1 = sorted(self.lineages[taxid1].items(), key=lambda x: x[1]["level"])
for tax, level in lineage1:
    if self.get_level(taxid2, tax) != level["name"]:
        return (level.get("level", 0) - 1) / len(lineage1), tax   # still asymmetric
return 1.0, lineage1[-1][0] if lineage1 else None
```

In the common path, the score is still divided by `len(lineage1)`. The fix applies only when `min_levels == 0`.
I have not measured how often that branch runs on real data, so I cannot say how much of the fix is active.

### Why the test did not catch it

The test file only exercises `compare_lineages`. No test covers the asymmetric branch of
`compare_lineages_relative`. The test that exists is correct for what it tests, so it passes while the
production path still has the old formula.

### What the numbers say

The changelog reports per-taxid assembly recall moving from 0.305 to 0.564, with 3,012 gains and 3 tie
reassignments. The same entry lists other fixes (F3 to F5) in the same run. The changelog does not split
out how much of that came from the symmetric score, so I do not attribute the gain to it.

### What I would do next

1. Add a test for `compare_lineages_relative` with a leaf that has extra sub-species ranks.
2. Log how often `min_levels == 0` is true during a real run. If it is rare, the symmetric fix is mostly
   inactive, and the fix should be applied to the main branch too.
3. Run the recall analysis once with each variant and report the split.

This is the same recall investigation as [the recall metric post]({% post_url 2026-09-07-why-your-classifier-beats-your-recall-metric %}).
The lineage score is one of the three bugs fixed there.
