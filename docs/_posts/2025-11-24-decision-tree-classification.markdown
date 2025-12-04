---
layout: post
title: "Tree Branch Classification"
date: 2025-11-24 23:34:26 +0000
categories: python classification decision trees
---

This post does not concern Decision Trees in the traditional sense, although the idea is to make an informed decision while traversing a tree structure.

### Problem Statement

I am working with biological, structured data, for which I possess metadata (meaningful categories), and am equipped with a meaningful distance metric. I have:

- extracted pairwise distances between data points.
- constructed a tree structure from the resulting distance matrix using an agglomerative clustering algorithm (e.g. UPGMA or Neighbor-Joining).

Previously i had been using a similarity threshold to cut the tree into clusters. This works relatively well, but, depending on the data, it can lead to over- or under- splitting of clusters. My questions:

1. what is the optimal threshold?
2. We have metadata, it is likely that different branches of the tree, when corresponding to different categories, will require different thresholds.

### Proposed Solution

Using simulations, I have generated a series of data sets of known categories. with varying levels of noise, relative and absolute size etc. I will use similarity metrics below and between nodes and node leaf metadata to decide where to cut the tree.

### Intermediate problem

This problem should be resolved recursively: Calculating all possible combinations of nodes that cover every leaf with no overlaps (antichains) is computationally intensive even in training, and wouldn't be practical in application, especially for large trees.

- how does one relate node-specific features (composition and metrics) to an overall estimate of precision? for every possible node, in every possible antichain?

**Solution** : we don't. not exactly. We can however know whether a splitting a node will improve or worsen precision.

This has become a binary classification problem with a set of features:

- distance metrics (to parent, to children, average to leaves, variance etc)
- metadata composition (entropy, proportions of different categories etc)

###

Workflow: - Generate trees for all data sets. - from each tree, extract all nodes and calculate features.

Tranning Data set, traverse tree from root to leaves, recursively: - for each node, calculate features - calculate precision of current antichain (set of nodes covering all leaves with no overlaps) - calculate precision of antichain where current node is replaced by its children - if precision improves, label node as 1 (split), else 0 (do not split) - store features and label - continue recursively for children nodes - at leaves, stop

On test data sets, traverse tree from root to leaves, recursively: - for each node, calculate features - use trained classifier to predict whether to split or not - if split, continue recursively for children nodes - if not split, add node to final antichain - at leaves, stop

### Comments

The interesting thing here for me is the design of the application. The decision tree classifier is relatively straightforward, and can be implemented using any standard machine learning library (e.g. scikit-learn). But then the model (and any scaler implemented) needs to be handed to the tree traversal functions that will make decisions at each node. Like little spiders crawling the tree decinding where to cut.

Could easily be implemented for left-right decisions in binary trees.

Design of basic crawler, where node stats were pre-computed and model and scaler are stored in a CompositionModeller class.

```python
def traversal_with_prediction(overlap_manager: OverlapManager, node: str, modeller: CompositionModeller, m_stats_stats_matrix, tax_df: pd.DataFrame, tax_level: str = "order", results = []) -> List[pd.DataFrame]:
    """
    Recursive function.
    Traverse the tree, internal nodes only. At each node:
    - compute node precision.
    - compute node composition at tax_level
    - use model to predict if split increases precision.
    - store results.
    if model predicts precision is increased by splitting, traverse (internal nodes only) children. else stop.
    """

    composition = node_composition_level(overlap_manager, node, m_stats_stats_matrix)

    node_true_leaves = node_total_true_leaves(overlap_manager, node, m_stats_stats_matrix)
    node_precision = 1 / len(set(node_true_leaves)) if len(node_true_leaves) > 0 else 0.0

    input_features = pd.DataFrame({
        'n_leaves': [len(overlap_manager.get_node_leaves(node))],
        'stats': m_stats_stats_matrix.loc[node].values,
    })

    if modeller.scaler is not None:
        input_features = pd.DataFrame(modeller.scaler.transform(input_features), columns=input_features.columns)


    if stop_traversal_pred is False:
        print(f"Stopping at node {node} with {len(overlap_manager.get_node_leaves(node))} leaves.")


    if stop_traversal_pred == True or overlap_manager.tree.out_degree(node) == 0:  # stop condition or leaf
        node_leaves = overlap_manager.get_node_leaves(node)
        results.append({
            'node': node,
            'n_leaves': len(node_leaves),
            'leaves': node_leaves,
            'best_taxid_match': best_taxid_match,
            'node_precision': node_precision,
            'node_taxids': node_leaf_taxids,
        })

    # Traverse children if prediction is positive
    else:
        for child in overlap_manager.tree.successors(node):
            if overlap_manager.tree.out_degree(child) > 0:  # internal node
                traversal_with_prediction(overlap_manager, child, modeller, m_stats_stats_matrix, results=results)
            else:

                results.append(
                    {
                        'node': child,
                        'n_leaves': 1,
                        'leaves': [child],
                        'node_precision': 1.0,
                    }
                )


    return results
```
