# Algorithms and Problem Solving: Deep Analysis

> **Audience:** L4 Engineers | **Focus:** Mathematical paradigms, complexity lower bounds, algorithm design

## 🎯 Conceptual Overview

An algorithm is a **sequence of discrete steps that solves a computational problem**. Unlike data structures (which organize data), algorithms organize the **solving process**.

**Fundamental question:** For a given problem, what is the **minimum computational resource** (time, memory) required?

At L4 level, you must:
- Understand **lower bounds** (can we do better than O(n log n)?), not just know Big-O notation
- Recognize **problem structure** (dynamic programming vs greedy vs divide-and-conquer)
- Analyze **recurrence relations** via Master Theorem
- Choose algorithms based on **real constants**, not just asymptotics

---

## 📍 1. Sorting: Information-Theoretic Bounds

### 1.1 Lower Bound for Comparison-Based Sorting

**Theorem:** Any sorting algorithm based on comparisons requires **Ω(n log n)** comparisons in the worst case.

**Proof via information theory:**

Sorting n elements means producing one of n! possible orderings.

Each comparison gives **2 possible results** (a < b or a ≥ b), so the decision tree has:
- Depth ≥ log₂(n!)
- By Stirling's formula: log₂(n!) ≈ n log₂(n) - n/ln(2)

Therefore: **Lower bound = Ω(n log n)**

This is an **absolute limit**—no comparison-based algorithm can be faster.

### 1.2 Algorithms Achieving the Lower Bound

**MergeSort:** O(n log n) in all cases
- Stable: preserves relative order of equal elements
- Requires O(n) additional memory
- Comparison count: ≈ n log₂(n) - n + 1

**HeapSort:** O(n log n) in all cases
- In-place: O(1) additional memory
- Not stable
- Worse cache performance than MergeSort

**QuickSort:** O(n log n) on average
- Average case: ≈ 2n ln(n) comparisons
- Worst case: O(n²) with bad pivot selection
- In practice often fastest due to cache locality

### 1.3 Linear Sorting: Breaking the Comparison Barrier

**Idea:** If we avoid comparisons, we can sort faster!

**CountingSort:** O(n + k), where k = range of values
- Assumption: integers in [0, k)
- Algorithm: count occurrences of each value, reconstruct array
- Impractical when k >> n

**RadixSort:** O(n × d), where d = number of digits
- Sorts by digits from least to most significant
- Base algorithm (subroutine): CountingSort
- Effective when number of digits is small (d = O(log n))

**Key idea:** Via **non-comparison** operations, we circumvent the Ω(n log n) lower bound.

---

## 📍 2. Dynamic Programming: Structure and Patterns

### 2.1 Definition via Optimal Substructure

**Dynamic Programming (DP)** applies to problems with **optimal substructure**:

> Optimal solution to a problem contains optimal solutions to subproblems.

**Formalization:**

Let OPT(n) = optimal value for problem of size n.

OPT(n) is defined via: OPT(n) = f(OPT(n₁), OPT(n₂), ...) + cost(n)

where n₁, n₂, ... < n are subproblem sizes.

**Example: Fibonacci**

F(n) = F(n-1) + F(n-2), with base cases F(0)=0, F(1)=1

Optimal substructure: F(n) explicitly depends on F(n-1) and F(n-2)

### 2.2 Naive Recursion vs Memoization

**Naive recursion for Fibonacci:**

```
T(n) = T(n-1) + T(n-2) + O(1)
```

Recursion tree:
- At level i: 2ⁱ nodes
- Height: n
- Total nodes: Σ(2ⁱ) = 2ⁿ⁺¹ - 1 = **O(2ⁿ)**

Exponential complexity! F(50) requires ~2⁵⁰ ≈ 10¹⁵ operations.

**Memoization (Tabulation):**

Store results F(0), F(1), ..., F(n) in an array.

Each value computed **exactly once**:
- Time complexity: O(n)
- Space complexity: O(n)

From exponential to linear!

**Key idea:** DP trades memory for time by storing intermediate results.

### 2.3 DP Types: Top-Down vs Bottom-Up

**Top-Down (Memoization):**
- Recursive function with cache
- Computes only needed subproblems
- Natural order: from large to small problem

**Bottom-Up (Tabulation):**
- Iterative table filling
- Computes all subproblems, even unnecessary ones
- Better cache locality, less recursion overhead

**Which to choose?**
- Top-Down: When subproblems are sparse (not all needed)
- Bottom-Up: When most subproblems needed, large fraction of problem

### 2.4 Classic Problems and Their Structure

**0/1 Knapsack:**

OPT(i, w) = maximum value using first i items and capacity w

Recurrence:
```
OPT(i, w) = max(
  OPT(i-1, w),                    // don't take item i
  OPT(i-1, w - weight[i]) + value[i]  // take item i
)
```

Base case: OPT(0, w) = 0

Complexity: O(n × W), where W = capacity

**Longest Common Subsequence (LCS):**

OPT(i, j) = length of LCS of first i chars of string1 and first j of string2

Recurrence:
```
OPT(i, j) = 
  - OPT(i-1, j-1) + 1,  if s1[i] == s2[j]
  - max(OPT(i-1, j), OPT(i, j-1)),  otherwise
```

Complexity: O(m × n)

**Matrix Chain Multiplication:**

For sequence of matrices A₁, A₂, ..., Aₙ, find optimal multiplication order.

OPT(i, j) = minimum scalar operations to multiply matrices i through j

```
OPT(i, j) = min(
  OPT(i, k) + OPT(k+1, j) + cost(i, k, j)
) for all i ≤ k < j
```

Complexity: O(n³)

### 2.5 Recurrence Master Function for DP

**General form of DP complexity:**

If subproblems of sizes {n₁, n₂, ..., nₖ}, then:

$$T(n) = c + \sum_{i=1}^{k} T(n_i)$$

where c = time to combine results.

**Example:**
- Fibonacci: T(n) = 2T(n-1) + O(1), with memoization becomes T(n) = O(n)
- Knapsack: T(n, W) = O(nW) (2D table)

---

## 📍 3. Divide and Conquer: Recurrence Relations

### 3.1 Structure of Divide-and-Conquer

Algorithm solves problem of size n by:

1. **Divide:** Split into a subproblems of size n/b
2. **Conquer:** Recursively solve each subproblem
3. **Combine:** Merge solutions in time D(n)

**Recurrence relation:**

$$T(n) = aT(n/b) + D(n)$$

### 3.2 Master Theorem

For T(n) = aT(n/b) + f(n), where a ≥ 1, b > 1:

**Theorem (Master):** Let $f(n) = O(n^d)$, where $d \geq 0$.

1. If $a > b^d$: **T(n) = Θ(n^{log_b a})** — recursion dominates
2. If $a = b^d$: **T(n) = Θ(n^d log n)** — recursion and work balanced
3. If $a < b^d$: **T(n) = Θ(n^d)** — work at upper level dominates

**Examples:**

**MergeSort:** T(n) = 2T(n/2) + O(n)
- a=2, b=2, d=1
- Check: a = 2 = 2¹ = b^d (case 2)
- Result: **T(n) = Θ(n log n)** ✓

**BinarySearch:** T(n) = 1·T(n/2) + O(1)
- a=1, b=2, d=0
- Check: a = 1 = 2⁰ = b^d (case 2)
- Result: **T(n) = Θ(log n)** ✓

**StrassenMatrixMult:** T(n) = 7T(n/2) + O(n²)
- a=7, b=2, d=2
- Check: a = 7 > 2² = 4 (case 1)
- Result: **T(n) = Θ(n^{log₂ 7}) ≈ Θ(n^{2.81})** (better than O(n³))

### 3.3 Recursion Tree: Intuitive Understanding

For T(n) = aT(n/b) + f(n):

- **Level 0:** 1 problem of size n, time f(n)
- **Level 1:** a problems of size n/b, time a·f(n/b)
- **Level 2:** a² problems of size n/b², time a²·f(n/b²)
- ...
- **Level log_b(n):** a^{log_b(n)} = n^{log_b(a)} problems of size 1

Total time:
$$T(n) = \sum_{i=0}^{\log_b(n)} a^i f(n/b^i)$$

Depending which sum dominates, either recursion or upper-level work dominates.

---

## 📍 4. Greedy Algorithms: Optimality via Local Choice

### 4.1 Structure of Greedy Algorithm

Greedy algorithm makes locally optimal choice at each step, hoping for global optimality.

**Does not always work!** Requires proof of optimality.

### 4.2 Huffman Coding: Greedy is Optimal

**Problem:** Encode text with minimum length using binary codes.

**Greedy algorithm (Huffman):**
1. Create leaf for each character with its frequency
2. While one tree remains:
   - Select two leaves with smallest frequency
   - Merge into new internal node with frequency = sum
   - Add both to priority queue

**Proof of optimality:** Huffman tree minimizes average encoding length via **exchange argument**:

If optimal tree differs from Huffman, it can be transformed to Huffman tree without increasing cost.

**Complexity:** O(n log n) with heap

### 4.3 MST (Minimum Spanning Tree)

**Problem:** For graph with weighted edges, find tree (V-1 edges) connecting all vertices minimizing total weight.

**Kruskal (Greedy by edges):**
1. Sort edges by weight
2. For each edge (u, v) in increasing order:
   - If u and v in different components, add edge
3. Result: MST

**Proof via Cut Property:**

For any cut of graph, minimum edge crossing the cut belongs to some MST.

Kruskal makes this choice optimally.

**Complexity:** O(E log E) for sorting + O(E α(V)) for Union-Find = O(E log E)

### 4.4 Limitations of Greedy

**Example: Longest Path in DAG**

Greedy choice of longest edge at each step **does not guarantee optimality**.

Correct solution: dynamic programming via topological sort.

**Conclusion:** Greedy requires optimality proof for specific problem.

---

## 📍 5. Backtracking: Systematic Search

### 5.1 Definition and Structure

Backtracking is search technique that:
1. Builds solution incrementally
2. Abandons solution that cannot lead to optimality
3. Backtracks and tries another path

**Search space:** Tree of possible choices

Number of tree nodes: potentially exponential, but **pruning** drastically reduces search tree.

### 5.2 N-Queens: Classic Example

**Problem:** Place n queens on n×n board so no two attack each other.

**Backtracking approach:**
1. Place queen in row 0
2. For each column in row 1:
   - Check if position is safe
   - If yes, place and move to row 2
   - If no, try next column
3. If solution found, ask if more needed
4. If can't place in row k, backtrack to row k-1

**Complexity:**
- Search space: n! possible placements
- With effective pruning: ~O(n!) worst case, but much less in practice

### 5.3 Constraint Satisfaction Problems (CSP)

Backtracking + pruning strategies:

**Forward Checking:** When assigning variable value, remove incompatible values from other variables.

**Arc Consistency (AC-3):** Remove values that cannot satisfy other variables.

This dramatically shrinks search space.

---

## 📍 6. Graph Algorithms: Structure and Paradigms

### 6.1 DFS and Topological Sort

**DFS (Depth-First Search):**

Core operation: visit vertex, then recursively visit all unvisited adjacent vertices.

**Properties:**
- Explores deeply one path, then backtracks
- DFS tree encodes graph structure: tree edge, back edge, forward edge, cross edge

**Topological Sort (for DAG):**

Sequence of vertices such that for each edge (u, v), u appears before v.

Algorithm: Perform DFS, push vertex to stack after processing all neighbors. Then reverse stack.

**Complexity:** O(V + E)

### 6.2 BFS and Shortest Path in Unweighted Graphs

**BFS (Breadth-First Search):**

Level by level — first all at distance 1, then 2, etc.

Distance from s to v = shortest path length = number of edges.

**Properties:**
- Finds shortest path in unweighted graphs
- Order of visit = ordered by distance from s

**Complexity:** O(V + E)

### 6.3 Dijkstra: Shortest Path with Non-Negative Weights

**Algorithm:** Greedy + edge relaxation

At each step:
1. Select unvisited vertex with smallest distance from s
2. For each neighbor v of this vertex:
   - If distance[v] > distance[u] + weight(u, v), relax (update)

**Invariant:** At each step, we know shortest path to all selected vertices.

**Complexity:**
- With binary heap: O((V + E) log V)
- With Fibonacci heap: O(E + V log V) — theoretically optimal

**Why fails with negative weights?**

Dijkstra selects vertex as "final" and never revisits. But with negative edges, distance can change through other paths.

### 6.4 Bellman-Ford: Negative Weights and Cycle Detection

**Algorithm:**

Repeat V-1 times: for each edge (u, v), relax it.

On V-th iteration: if distance changed, negative cycle exists.

**Complexity:** O(V × E)

**Comparison:**
- Dijkstra: O((V+E) log V), but fails with negative weights
- Bellman-Ford: O(V×E), works with negative weights, detects cycles

Bellman-Ford slower but more universal.

### 6.5 All-Pairs Shortest Paths

**Floyd-Warshall:**

Dynamic programming: OPT(i, j, k) = shortest path from i to j using vertices {1, ..., k} as intermediate.

Recurrence:
```
OPT(i, j, k) = min(
  OPT(i, j, k-1),            // don't use k
  OPT(i, k, k-1) + OPT(k, j, k-1)  // use k
)
```

**Complexity:** O(V³)

**When to use:**
- V small (< 500): Floyd-Warshall simpler
- V large, sparse graph: Run Dijkstra from each vertex = O(V(E + V) log V)

---

## 📍 7. String Algorithms: Search and Matching

### 7.1 Knuth-Morris-Pratt (KMP)

**Problem:** Find all occurrences of pattern in text.

**Naive algorithm:** O(n × m) worst case

**KMP idea:** Use information from search failures to avoid redundant comparisons.

**LPS array (Longest Proper Prefix that is Suffix):**

For each position k in pattern, find length of longest prefix that is suffix of pattern[0..k].

Example: pattern = "ABABAB"
```
LPS = [0, 0, 1, 2, 3, 4]
```

**Algorithm:**
1. Build LPS: O(m)
2. Search: O(n), using LPS for transitions

**Complexity:** O(n + m)

### 7.2 Rabin-Karp: Hashing for Search

**Idea:** Hash pattern and each window of text with size m. Compare hashes.

**Rolling hash:** For window [i, i+1, ..., i+m-1]:

hash = (text[i] × b^(m-1) + text[i+1] × b^(m-2) + ... + text[i+m-1]) mod p

Next window: hash_new = ((hash - text[i]×b^(m-1)) × b + text[i+m]) mod p

**Complexity:**
- Average case: O(n + m)
- Worst case: O(n × m) (with frequent hash collisions)

**Advantages over KMP:** Simpler code, easy to generalize to multiple patterns.

### 7.3 Z-Algorithm

Build Z array: Z[i] = length of longest substring starting at text[i] that matches prefix of text.

**Complexity:** O(n)

**Application:** Find all occurrences of pattern in text in O(n + m).

---

## 🎯 Table of Algorithm Paradigms

| Paradigm | Property | When to Use | Examples |
|---|---|---|---|
| Divide & Conquer | Split, solve subproblems, combine | Independent subproblems | MergeSort, BinarySearch, Strassen |
| Dynamic Programming | Optimal substructure + overlap | Dependent subproblems, overlap | Knapsack, LCS, Fibonacci |
| Greedy | Locally optimal choice | Problem has greedy-choice property | Huffman, Kruskal, Dijkstra |
| Backtracking | Systematic search with pruning | Enumerate all with constraints | N-Queens, CSP, Sudoku |
| BFS | Level-by-level search | Shortest path (unweighted) | Topology, connectivity |
| DFS | Deep search | Topology, cycles, strong connectivity | Topological Sort, SCC |

---

## 💡 Senior-Level Interview Questions

1. **Master Theorem doesn't cover T(n) = 2T(n/2) + n log n. How to analyze?**
   - Answer: Use recursion tree or extended Master Theorem

2. **Why is Quicksort better than MergeSort in practice, though both O(n log n)?**
   - Answer: Cache locality, in-place operations, smaller constants

3. **Prove Huffman coding is optimal.**
   - Answer: Exchange argument with two minimum-frequency leaves

4. **How to find longest simple path in graph in polynomial time?**
   - Answer: Impossible (NP-hard) unless special structure (DAG → O(V+E))

5. **Why doesn't Dijkstra work with negative weights?**
   - Answer: Selects vertex as "final" without revisiting. Negative edge can change distance later.

6. **What's the information-theoretic lower bound for comparison-based sorting?**
   - Answer: Ω(n log n); need log(n!) comparisons to distinguish n! permutations

---

## 🔑 Key Takeaways

1. **Lower bounds matter:** Ω(n log n) for comparison sorting is absolute limit
2. **DP vs Greedy:** DP for dependent subproblems, Greedy for independent (with proof)
3. **Master Theorem** quickly analyzes divide-and-conquer recurrences
4. **Graph algorithms specialized:** BFS for unweighted, Dijkstra for non-negative, Bellman-Ford for general
5. **String algorithms widespread:** KMP for exact search, Rabin-Karp for multiple patterns
