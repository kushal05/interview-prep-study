# DSA Patterns - Quick Overview

Organized by data structure. Each row gives the pattern, when to use it, and the words in a problem statement that hint at it.

> New to DSA? This page is a one-glance summary, not a course. Learn the patterns step by step in the [DSA course](./learn/00-start-here.md) (26 chapters, beginner to interview level), and use the "Learn it" links under each heading to jump to the right chapter.

---

## 1. Arrays & Strings

Learn it: [02. Arrays and Strings](./learn/02-arrays-and-strings.md), [05. Two Pointers](./learn/05-two-pointers.md), [06. Sliding Window](./learn/06-sliding-window.md), [07. Prefix Sums](./learn/07-prefix-sums.md), [08. Binary Search](./learn/08-binary-search.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Sliding Window | Contiguous subarrays/substrings with a constraint | "longest/shortest", "at most K", "subarray" |
| Two Pointers | Sorted array comparisons, in-place operations | "sorted", "in-place", "pairs" |
| Binary Search | Sorted data or monotonic condition | "sorted", "O(log n)", "find minimum" |
| Prefix Sum | Range sum queries, subarray sums | "sum from i to j", "subarray sum = K" |
| Cyclic Sort | Numbers in range 1..N, find missing/duplicate | "1 to N", "missing/duplicate", "O(1) space" |

---

## 2. Linked Lists

Learn it: [10. Linked Lists](./learn/10-linked-lists.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Fast & Slow Pointers | Cycle detection, finding middle | "cycle", "middle", "loop" |
| Reversal | Reverse all or part of list | "reverse", "palindrome" |
| Merge Two Lists | Combine sorted lists | "merge sorted", "combine" |
| Dummy Node | Simplify head insertion/deletion | "remove Nth", "delete node" |

---

## 3. Stacks & Queues

Learn it: [11. Stacks and Queues](./learn/11-stacks-and-queues.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Monotonic Stack | Next/previous greater or smaller element | "next greater", "temperatures", "histogram" |
| Stack for Parsing | Validate or decode nested structures | "parentheses", "decode", "nested" |
| BFS with Queue | Shortest path, level-order traversal | "shortest path", "minimum steps", "levels" |
| Deque for Window | Sliding window max/min | "max in each window of size K" |

---

## 4. Trees

Learn it: [12. Trees and Traversals](./learn/12-trees-and-traversals.md), [13. Binary Search Trees](./learn/13-binary-search-trees.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| DFS (Recursive) | Compute values from subtrees | "max depth", "path sum", "balanced" |
| BFS (Level Order) | Process nodes level by level | "level order", "min depth", "right view" |
| BST Invariants | Exploit sorted property of BSTs | "validate BST", "kth smallest" |

---

## 5. Graphs

Learn it: [16. Graphs Fundamentals](./learn/16-graphs-fundamentals.md), [17. Advanced Graphs](./learn/17-advanced-graphs.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| DFS | Explore components, detect cycles | "connected", "components", "cycle" |
| BFS | Shortest path in unweighted graph | "shortest", "minimum steps" |
| Topological Sort | Dependency ordering (DAG) | "prerequisites", "ordering", "schedule" |
| Union-Find | Group nodes, detect cycles in undirected graphs | "connected components", "redundant edge" |
| Flood Fill | Label/count clusters in 2D grid | "islands", "regions", "fill" |

---

## 6. Heaps / Priority Queues

Learn it: [14. Heaps and Priority Queues](./learn/14-heaps-and-priority-queues.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Top-K Elements | Find K largest/smallest/most frequent | "Kth largest", "top K", "most frequent" |
| Scheduling | Track earliest end time, merge sorted streams | "meeting rooms", "merge K lists" |

---

## 7. Hash Maps / Sets

Learn it: [03. Hashing, Maps and Sets](./learn/03-hashing-maps-sets.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Frequency Counting | Count occurrences, anagrams | "frequency", "majority", "anagram" |
| Hash + Window | Track characters in sliding window | "distinct characters", "window substring" |
| Set for Uniqueness | Check duplicates, detect cycles | "contains duplicate", "unique", "loop" |

---

## 8. Recursion & Backtracking

Learn it: [04. Recursion](./learn/04-recursion.md), [18. Backtracking](./learn/18-backtracking.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| Backtracking | Generate all valid configurations | "all combinations", "all permutations", "find all" |
| Constraint Pruning | Skip invalid/duplicate branches early | "no duplicates allowed", "constraints" |

---

## 9. Dynamic Programming

Learn it: [20. Dynamic Programming 1](./learn/20-dynamic-programming-1.md), [21. Dynamic Programming 2](./learn/21-dynamic-programming-2.md)

| Pattern | When to Use | How to Spot |
|---------|-------------|-------------|
| 1D DP | Linear sequence, each state depends on previous | "max/min", "count ways", "can reach" |
| 2D Grid DP | Path problems on a board | "unique paths", "min path sum" |
| String DP | Compare two strings character by character | "LCS", "edit distance", "palindrome" |
| Knapsack | Select items with weight/value constraints | "partition", "subset sum", "knapsack" |
| Memoization | Recursive solution has overlapping subproblems | "overlapping", "top-down", "cache" |
