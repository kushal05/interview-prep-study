# Data Structures & Algorithms - Index

Everything DSA in this repo: a beginner-to-interview **course**, an **interactive study app** built from it, pattern cheat sheets, and **reference implementations** in Java, JavaScript, Kotlin and Swift.

---

## Start Here

New to DSA, or coming back after a while? Start with **[00. Start Here](./learn/00-start-here.md)** and work through the course in order. Each chapter explains the idea from scratch, walks through code in JavaScript and Java, and ends with practice problems and self-check questions.

### Interactive study app

Open [`study/dsa-study.html`](./study/dsa-study.html) in a browser. It is a single file and works offline. It has every chapter in reading order (plus the revision sheets below), JS/Java code tabs, a practice-problem tracker, flashcards and search.

After editing any chapter in `learn/`, rebuild the app with:

```bash
node dsa/study/build.mjs
```

### The course (26 chapters)

| Phase | Chapters |
|-------|----------|
| **Foundations** | [00. Start Here](./learn/00-start-here.md) · [01. Complexity and Big-O](./learn/01-complexity-big-o.md) · [02. Arrays and Strings](./learn/02-arrays-and-strings.md) · [03. Hashing, Maps and Sets](./learn/03-hashing-maps-sets.md) · [04. Recursion](./learn/04-recursion.md) |
| **Core Patterns** | [05. Two Pointers](./learn/05-two-pointers.md) · [06. Sliding Window](./learn/06-sliding-window.md) · [07. Prefix Sums](./learn/07-prefix-sums.md) · [08. Binary Search](./learn/08-binary-search.md) · [09. Sorting](./learn/09-sorting.md) |
| **Linear Structures** | [10. Linked Lists](./learn/10-linked-lists.md) · [11. Stacks and Queues](./learn/11-stacks-and-queues.md) |
| **Trees & Graphs** | [12. Trees and Traversals](./learn/12-trees-and-traversals.md) · [13. Binary Search Trees](./learn/13-binary-search-trees.md) · [14. Heaps and Priority Queues](./learn/14-heaps-and-priority-queues.md) · [15. Tries](./learn/15-tries.md) · [16. Graphs Fundamentals](./learn/16-graphs-fundamentals.md) · [17. Advanced Graphs](./learn/17-advanced-graphs.md) |
| **Algorithm Paradigms** | [18. Backtracking](./learn/18-backtracking.md) · [19. Greedy Algorithms](./learn/19-greedy.md) · [20. Dynamic Programming 1](./learn/20-dynamic-programming-1.md) · [21. Dynamic Programming 2](./learn/21-dynamic-programming-2.md) |
| **Advanced & Interview** | [22. Bit Manipulation and Math](./learn/22-bit-manipulation-and-math.md) · [23. Advanced Data Structures](./learn/23-advanced-data-structures.md) · [24. String Algorithms](./learn/24-string-algorithms.md) · [25. Interview Playbook](./learn/25-interview-playbook.md) |

### Study plans

Use the plans in [25. Interview Playbook → Study plans](./learn/25-interview-playbook.md#study-plans): a **4-week plan** if your interview is soon, an **8-week plan** if you are learning from scratch, and a checklist for **the last 48 hours**.

---

## Pattern Guides (Revision)

Language-agnostic summaries for quick review once you have done the course.

| File | Purpose |
|------|---------|
| [CRAM.md](./CRAM.md) | Last-minute revision sheet |
| [patterns-cheatsheet.md](./patterns-cheatsheet.md) | Pattern -> trigger words -> template |
| [patterns.md](./patterns.md) | Detailed pattern explanations with examples |
| [patterns-basic.md](./patterns-basic.md) | One-glance pattern overview by data structure |

---

## Reference Implementations

The language folders are **reference implementations**, not lessons: compact, interview-style code for each data structure and its common patterns. Each file links back to the course chapter that teaches the topic. Use them to drill syntax in your interview language or to compare the same solution across languages.

| Topic | Java | JavaScript | Kotlin | Swift | Learn it |
|-------|------|------------|--------|-------|----------|
| Arrays & Strings | [Java](./ds-java/arrays-strings.md) | [JS](./ds-js/arrays-strings.md) | [Kotlin](./ds-kotlin/arrays-strings.md) | [Swift](./ds-swift/arrays-strings.md) | [02](./learn/02-arrays-and-strings.md), [05](./learn/05-two-pointers.md), [06](./learn/06-sliding-window.md) |
| Hash Tables | [Java](./ds-java/hash-tables.md) | [JS](./ds-js/hash-tables.md) | [Kotlin](./ds-kotlin/hash-tables.md) | [Swift](./ds-swift/hash-tables.md) | [03](./learn/03-hashing-maps-sets.md) |
| Linked Lists | [Java](./ds-java/linked-lists.md) | [JS](./ds-js/linked-lists.md) | [Kotlin](./ds-kotlin/linked-lists.md) | [Swift](./ds-swift/linked-lists.md) | [10](./learn/10-linked-lists.md) |
| Stacks & Queues | [Java](./ds-java/stacks-queues.md) | [JS](./ds-js/stacks-queues.md) | [Kotlin](./ds-kotlin/stacks-queues.md) | [Swift](./ds-swift/stacks-queues.md) | [11](./learn/11-stacks-and-queues.md) |
| Trees & Graphs | [Java](./ds-java/trees-graphs.md) | [JS](./ds-js/trees-graphs.md) | [Kotlin](./ds-kotlin/trees-graphs.md) | [Swift](./ds-swift/trees-graphs.md) | [12](./learn/12-trees-and-traversals.md), [16](./learn/16-graphs-fundamentals.md) |
| Heaps & Priority Queues | [Java](./ds-java/heaps-priority-queues.md) | [JS](./ds-js/heaps-priority-queues.md) | [Kotlin](./ds-kotlin/heaps-priority-queues.md) | [Swift](./ds-swift/heaps-priority-queues.md) | [14](./learn/14-heaps-and-priority-queues.md) |
| Recursion & Backtracking | [Java](./ds-java/recursion-backtracking.md) | [JS](./ds-js/recursion-backtracking.md) | [Kotlin](./ds-kotlin/recursion-backtracking.md) | [Swift](./ds-swift/recursion-backtracking.md) | [04](./learn/04-recursion.md), [18](./learn/18-backtracking.md) |
| Dynamic Programming | [Java](./ds-java/dynamic-programming.md) | [JS](./ds-js/dynamic-programming.md) | [Kotlin](./ds-kotlin/dynamic-programming.md) | [Swift](./ds-swift/dynamic-programming.md) | [20](./learn/20-dynamic-programming-1.md), [21](./learn/21-dynamic-programming-2.md) |

### Choosing your language

| Target | Recommended Language |
|--------|---------------------|
| Android interviews | Kotlin or Java |
| iOS interviews | Swift |
| Web / general SWE | JavaScript |
| Any company | Whichever of the four you are fastest in |
