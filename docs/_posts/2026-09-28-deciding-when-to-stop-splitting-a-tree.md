---
layout: post
title: "Deciding when to stop splitting a tree"
date: 2026-09-28 12:23:23 +0100
categories: python classification clustering decision-trees
use_math: true
---

In an [earlier post]({% post_url 2025-11-24-decision-tree-classification %}) I wrote about framing the
decision of *where to cut a tree* as a binary classification problem: at each node, predict whether
splitting it improves precision, and recurse only when it does. That post was about the prototype. This one
is about the rule that ended up in production, and about the one comparison that took me an embarrassingly
long time to get right.

### The setting

The trees come from metagenomic data. Reads are mapped against a set of reference assemblies; references
that share reads form an overlap graph, and a distance matrix over that overlap gives us a tree. Each leaf
corresponds to a reference with a known taxonomic identity, so for any internal node I can ask: *how many
distinct true taxa live in my subtree?*

Precision at a node follows directly: if a node covers $n$ leaves but its leaves belong to $t$ distinct
taxa, then

$$
\text{precision} = \frac{1}{\max(t, 1)}
$$

Perfect precision (1.0) means every leaf under the node is the same taxon — a clean clade. The more
taxonomically mixed a node is, the lower its precision. The traversal is then a single question: does
splitting this node into its children give me better precision than keeping it whole?

### The traversal

Here is the core recursion. At each node we compute the node's precision, the precision we would get by
splitting into children, and descend only if the split is not worse.

```python
def traversal_with_precision(overlap_manager, node, m_stats_stats_matrix,
                             input_taxa, tax_level="order",
                             results=None, force_stop=False):
    if results is None:
        results = []

    node_true_leaves = node_total_true_leaves(overlap_manager, node, m_stats_stats_matrix)
    node_precision = 1 / len(set(node_true_leaves)) if len(node_true_leaves) > 0 else 0.0
    node_precision = 1 / node_precision if node_precision > 1 else node_precision

    node_children = list(overlap_manager.tree.successors(node))
    new_precision = len(node_children) / len(set(node_true_leaves)) if len(node_true_leaves) > 0 else 0.0
    new_precision = 1 / new_precision if new_precision > 1 else new_precision

    precision_increased = new_precision >= node_precision
    stop_traversal = not precision_increased

    # ... record composition, diversity, Min_Dist, Min_Shared for this node ...

    for child in node_children:
        if overlap_manager.tree.out_degree(child) > 0:  # internal node
            traversal_with_precision(
                overlap_manager, child, m_stats_stats_matrix,
                input_taxa, tax_level=tax_level, results=results,
            )
    return results
```

Notice the recursion has no explicit "else stop" branch: if the split does not help, we simply record the
node and do not recurse into its children. The nodes we keep are the ones we would cut the tree at, and
they form the final antichain.

### The subtle part: `>=` versus `>`

For a long time the stopping rule was the obvious-looking `new_precision > node_precision`. That is, stop
unless the split is *strictly* better. The bug hides in ties.

Consider a node that is already pure — all its leaves are one taxon, precision 1.0. Splitting it into two
children that are each still pure also gives 1.0. Under `>`, $1.0 > 1.0$ is false, so we stop. Good, that
is what we want.

Now consider the opposite case: a node whose children have the *same* precision as the node itself but
which are genuinely separable clades — say the node covers two taxa and so does each child, but the child
groupings are finer between different branches. Under `>`, the tie stops the traversal early and we miss
resolvable structure below.

The fix was to change the comparison to `>=`:

```python
precision_increased = new_precision >= node_precision
stop_traversal = not precision_increased
```

The rule is now: continue descending whenever splitting does not make things *worse*. Ties are treated as
"keep going", which lets the traversal push through plateaus to reach finer clades. In a commit I described
this as "stop traversal only if precision drops — continue if same", which is exactly the `>=` semantics.

### What each node records

The traversal is also where we featurise the tree for the downstream models. Every visited node appends a
row:

```python
local_results = pd.DataFrame({
    "node": [node],
    "n_leaves": [len(overlap_manager.get_node_leaves(node))],
    "tax_diversity": [node_total_leaf_taxa_div],
    "n_true_leaves": [len(set(node_true_leaves))],
    "precision": [node_precision],
    "new_precision": [new_precision],
    "precision_increased": [precision_increased],
    "Min_Dist": [min_dist],
    "Min_Shared": [min_shared],
    "stop_traversal": [stop_traversal],
})
```

`tax_diversity` (Shannon diversity of the taxa under the node), `Min_Dist` and `Min_Shared` (the closest
pairwise distance and the minimum shared-read proportion in the subtree) are the features a learned model
uses in production to make the same call without ground-truth taxon counts. That is the bridge back to the
prototype: the rule-based traversal generates labelled examples, and the classifier learns to reproduce
the decision from features that do not require knowing the answer.

### What I took away

- A stopping rule expressed as "improves" is ambiguous at ties, and ties are where tree traversals go
  wrong. Decide explicitly whether equal is a stop or a continue.
- Recursion without an `else` branch is a clean way to express "keep the frontier of nodes that survive the
  rule".
- Instrumenting the traversal (recording features at every node) turns an algorithm into training data for
  free.
