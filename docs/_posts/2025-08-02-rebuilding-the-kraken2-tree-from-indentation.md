---
layout: post
title: "Rebuilding the Kraken2 tree from indentation"
date: 2025-08-02 22:02:07 +0000
categories: python metagenomics parsing
---

Kraken2 is a popular taxonomic classifier for metagenomic reads. You give it reads, it tells you which
taxon each read was assigned to, and it writes a report summarising the calls. The report is human-readable
and extremely convenient — until you want to do anything programmatic with it.

Here is what it looks like:

```
  3.42  1250   1250  S   9606  Homo sapiens
 45.10 16500   4200  S   562   Escherichia coli
 40.20 14700    900  G   561   Escherichia
  1.10    400    400  S   1280  Staphylococcus aureus
```

The problem is the structure. Those leading spaces encode the taxonomy tree — more spaces, deeper rank —
but the file itself is flat. There are no parent IDs, no nesting, just rows ordered by a depth-first walk
with indentation. If I want to attribute reads to species, or roll counts up the tree, I need an actual
hierarchy.

### The format, precisely

Each line has six tab-separated fields: percentage of reads, reads at this clade, reads directly at this
rank, a rank code, a taxid, and an indented name. The rank code is one letter: `D` domain, `K` kingdom,
`P` phylum, `C` class, `O` order, `F` family, `G` genus, `S` species. A `2` in the last column is the
special "no rank" node.

The depth of a row is the number of leading spaces in the name field. I keep a tiny helper for it:

```python
def count_prefix_spaces(s: str) -> int:
    """Count leading spaces in a string."""
    return len(s) - len(s.lstrip())
```

### Rebuilding the DAG

The key observation is that the report is a pre-order (depth-first) traversal. So when I reach a row at
depth $d$, its parent is the most recent previous row whose depth is strictly less than $d$. I keep the
nodes in the order I read them and walk backwards:

```python
@staticmethod
def kraken_report_to_tree(report):
    nodes_list = []
    edges = []

    for _, row in report.iterrows():
        name = row["name"].strip()
        tax_id = int(row["taxID"])
        if tax_id == 0:
            continue
        prefix_spaces = count_prefix_spaces(row["name"])
        node = (tax_id, name, prefix_spaces,
                float(row["PercReads"]), int(row["Nreads"]), str(row["RankCode"]))
        nodes_list.append(node)

        parent_node = None
        if prefix_spaces > 0:
            i = len(nodes_list) - 1
            while i > 0 and nodes_list[i][2] >= prefix_spaces:
                i -= 1
            if i >= 0:
                parent_node = nodes_list[i]

        if parent_node:
            edges.append((parent_node[0], tax_id))

    return nodes_list, edges
```

Two details matter. First, the backwards walk stops at a node with *strictly fewer* spaces, because a node
at the same depth is a sibling, not a parent. Second, skipping taxid `0` avoids a phantom root that would
otherwise attach to the first real node. The result is a list of nodes and a list of `(parent, child)`
edges — exactly what `networkx` wants.

### From a graph to the leaves that matter

With the DAG built, the leaves are the nodes with no outgoing edges:

```python
import networkx as nx

def get_leaves(nodes, edges):
    G = nx.DiGraph()
    G.add_edges_from(edges)
    return [node[0] for node in nodes if G.out_degree(node[0]) == 0]
```

The complication is that Kraken2 rarely stops at a clean species. A read cluster might terminate at a
strain, or at a "no rank" node below a species. For attribution I want to collapse everything up to the
nearest ancestor that is actually a species, keeping the node itself if its parent is already species-rank:

```python
def get_simplified_leaves(self, nodes, edges):
    G = nx.DiGraph()
    G.add_edges_from(edges)
    simplified_leaves = {}

    for node in nodes:
        if G.out_degree(node[0]) == 0:          # a leaf of the DAG
            leaf = node[0]
            parent = list(G.predecessors(leaf))
            parent_detail = self.get_node_info(parent[0]) if parent else None
            while parent and "S" in (parent_detail[5] if parent_detail else ""):
                leaf = parent[0]                 # climb through species ancestors
                parent = list(G.predecessors(leaf))
                parent_detail = self.get_node_info(parent[0]) if parent else None
            if parent:
                parent = parent[0]
                simplified_leaves.setdefault(parent, []).append(leaf)
    return simplified_leaves
```

The `while "S" in rank` loop walks up while the parent is a species, which folds strain and no-rank nodes
into their species. The `setdefault(parent, []).append(leaf)` groups the resulting leaves by the first
non-species ancestor.

### Summarising

Finally, for each parent, I emit one representative row. If the parent is itself a species, use it;
otherwise fall back to the leaf with the most reads:

```python
def summarize_leaves(leaves_simple) -> pd.DataFrame:
    summary = []
    for parent, leaves in leaves_simple.items():
        if "S" in parent[5]:
            summary.append(parent)
        else:
            best_leaf = max(leaves, key=lambda x: x[3])   # highest read count
            summary.append(best_leaf)
    return pd.DataFrame(summary, columns=[
        "taxID", "description", "prefix_spaces", "perc_reads", "Nreads", "rank_code",
    ])
```

### The payoff

The result is a flat table with one row per taxon, ready to filter (`Nreads > min_uniq_reads`) and join
against assemblies, mutation spectra, or anything else downstream. The same three-stage recipe —
indentation to DAG, DAG to simplified leaves, leaves to summary — turns any indented taxonomy report into
a queryable structure, and it is short enough to keep in your head.

If you are parsing Kraken2 reports and reaching for a regex, this is the alternative: read the indentation
as structure.
