---
layout: post
title: Boundary Block Groups
date: 2021-07-23
permalink: /conjectures/boundary-block-groups/
categories: conjectures
---

This summer I've been running Mapper, a topological data analysis algorithm, on census block group data for Mecklenburg County, NC. The dataset has 550 block groups, each described by 13 proportional variables pulled from the 2018 ACS 5-year survey: things like the share of residents below 20% of the poverty line, the proportion of households renting, the share of housing units that are multi-unit. The goal is to find structure in how neighborhoods cluster along these dimensions without deciding in advance what that structure should look like.

Mapper works in three steps. First, a filter function projects the data into a lower-dimensional space—I'm using truncated SVD with two components. Then a cover partitions that projected space into overlapping intervals. Finally, a clustering algorithm groups data points within each interval. The result is a graph where nodes are clusters of block groups and edges connect nodes that share block groups between them.

I keep thinking about that last part: block groups shared between nodes. With 40% overlap on the cubical cover, some block groups land in two adjacent intervals simultaneously, which means they can end up flagged as members of two different nodes. In the code, this looks like:

```python
def add_flags(data_input, node_elements):
    data_output = data_input.copy(deep=True)
    for node in range(len(node_elements)):
        if len(node_elements[node]) >= MIN_NODE_SIZE:
            row_flags = [int(row_num in node_elements[node])
                         for row_num in range(len(data_input))]
            data_output = pd.concat(
                [data_output, pd.DataFrame(row_flags, columns=["node_" + str(node)])],
                axis=1
            )
    return data_output
```

A block group with a flag sum greater than one is simultaneously a member of two nodes. My working assumption was that these were just algorithmic artifacts, boundary cases created by the 40% overlap parameter. But when I look at where they fall in the data, I don't think that's right.

The SVD filter exposes two dominant axes in the data. The first is roughly a housing-type axis: block groups with high negative loading tend to have high average occupancy and few multi-unit homes, while those with high positive loading have the reverse. The second captures something closer to concentrated disadvantage—high proportions of residents without health insurance, below the poverty line, speaking English less than very well.

```python
u, s, v = svds(merged_data, k=2)

# v[0] — housing axis
#   average_occupancy:           -0.527
#   prop_multiunit_homes:        +0.482
#   prop_households_renting:     +0.378

# v[1] — disadvantage axis
#   prop_no_health_insurance:    +0.420
#   prop_below_20_income:        +0.393
#   prop_english_less_than_well: +0.324
```

The cover divides the projected space into five overlapping intervals along these axes. Block groups at the boundary of adjacent intervals have SVD projections that sit in the overlap zone between two distinct regions of this space. They are neighborhoods that score somewhere between two socioeconomic types, not firmly in either.

What I find harder to dismiss as coincidence is what happens inside individual cover elements. Two nodes, 3 and 4, both map entirely to the same cover element and have the same SVD projection neighborhood. The clustering algorithm still splits them into separate nodes because clustering happens in the full 13-dimensional space, not the projected one. When I run z-tests on their weighted variable means, they're significantly different on variables that barely load onto the two SVD components:

```python
node_3_node_4_pvalues.loc[node_3_node_4_pvalues[0] < 0.05]

# prop_english_less_than_well              0.003
# prop_65_and_over                         0.001
# prop_over_30_percent_income_to_mortgage  <0.001
# prop_in_popular_industry                 0.037
```

So the filter catches two directions of variation; the cover partitions along those; then the clustering finds sub-structure the filter missed. The block groups that appear in both nodes 3 and 4 aren't randomly at the boundary — they're the neighborhoods that are indistinguishable on the axes Mapper used to organize the space, but distinguishable once you look at everything else.

My argument is that this structure—SVD projection, cover overlap, within-cover clustering—is not just a pipeline but a decomposition. Each stage makes a different layer of socioeconomic variation visible, and the block groups that end up shared between nodes are the seams of that decomposition: neighborhoods that are genuinely in between types, not administratively miscategorized or statistically noisy. The 40% overlap parameter does influence which block groups end up at the boundary, but it doesn't create the fact of socioeconomic transition — it reveals it.

I don't know how to test this rigorously yet. The most direct approach would probably be to vary the overlap fraction and check whether the same block groups consistently appear at boundaries across runs, then look at whether their geographic locations correspond to places where neighborhood change is documented.
