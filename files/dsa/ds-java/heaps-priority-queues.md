# Heaps & Priority Queues in Java

> Java implementation reference for `PriorityQueue`: min/max heaps, custom comparators, top-K and scheduling / merge-K patterns.
> New to this topic? Learn it first in [14. Heaps and Priority Queues](../learn/14-heaps-and-priority-queues.md).

## Complexity Overview

| Operation | Min/Max Heap |
|-----------|-------------|
| Insert (add) | O(log n) |
| Extract min/max (poll) | O(log n) |
| Peek min/max | O(1) |
| Build heap from array | O(n) |
| Search | O(n) |

> **Key Interview Point:** Java's `PriorityQueue` is a **min-heap** by default. For max-heap, use `Collections.reverseOrder()` or a custom comparator.

---

## Basic Usage

```java
import java.util.*;

// Min heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.add(3); minHeap.add(1); minHeap.add(2);
minHeap.peek();  // 1
minHeap.poll();  // 1

// Max heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator -- sort by frequency
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> Integer.compare(a[1], b[1])); // min by index 1
// or: new PriorityQueue<>(Comparator.comparingInt(a -> a[1]))
```

> **Common Mistake:** Writing comparators as `a - b`. The subtraction overflows for large or negative values (e.g. `Integer.MIN_VALUE - 1`) and silently breaks the ordering. Use `Integer.compare(a, b)` or `Comparator.comparingInt(...)`.

> **Common Mistake:** `PriorityQueue` does NOT guarantee sorted iteration order. Only `peek()`/`poll()` give the min/max.

---

## Pattern 1: Top-K Elements

**When to use:** "Kth largest", "top K frequent", "K closest"

**Strategy:** Use a min-heap of size K. The heap top is the Kth largest.

```java
// Kth largest element -- Time: O(n log k), Space: O(k)
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.add(num);
        if (minHeap.size() > k) minHeap.poll(); // remove smallest
    }
    return minHeap.peek(); // kth largest remains
}

// Top K frequent elements -- Time: O(n log k), Space: O(n)
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Min-heap by frequency -- keeps top k frequent
    PriorityQueue<Map.Entry<Integer,Integer>> pq =
        new PriorityQueue<>((a, b) -> Integer.compare(a.getValue(), b.getValue()));
    for (var entry : freq.entrySet()) {
        pq.add(entry);
        if (pq.size() > k) pq.poll();
    }
    return pq.stream().mapToInt(e -> e.getKey()).toArray();
}

// K closest points to origin -- Time: O(n log k), Space: O(k)
public int[][] kClosest(int[][] points, int k) {
    // Max-heap by squared distance -- keeps k closest
    PriorityQueue<int[]> pq = new PriorityQueue<>(
        (a, b) -> Integer.compare(b[0]*b[0] + b[1]*b[1], a[0]*a[0] + a[1]*a[1]));
    for (int[] p : points) {
        pq.add(p);
        if (pq.size() > k) pq.poll();
    }
    return pq.toArray(new int[k][]);
}
```

> **Key Interview Point:** For "K smallest", use a **max-heap** of size K. For "K largest", use a **min-heap** of size K. This is counterintuitive but correct -- you evict the least qualifying element.

---

## Pattern 2: Scheduling / Merge K Sorted

**Idea:** The heap always holds the "frontier": the end times of rooms in use, or the current head of each sorted list. Peeking the minimum tells you which room frees up first or which node comes next; each step costs O(log k).

```java
// Meeting rooms II -- minimum rooms needed -- Time: O(n log n)
public int minMeetingRooms(int[][] intervals) {
    if (intervals.length == 0) return 0;
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0])); // sort by start time
    PriorityQueue<Integer> endTimes = new PriorityQueue<>(); // min-heap of end times
    endTimes.add(intervals[0][1]);

    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] >= endTimes.peek())
            endTimes.poll(); // reuse room (earliest ending room is free)
        endTimes.add(intervals[i][1]);
    }
    return endTimes.size(); // number of rooms in use
}

// Merge K sorted lists -- Time: O(N log k), Space: O(k)
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));
    for (ListNode l : lists) if (l != null) pq.offer(l);

    ListNode dummy = new ListNode(0), curr = dummy;
    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        curr.next = node;
        curr = curr.next;
        if (node.next != null) pq.offer(node.next);
    }
    return dummy.next;
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 215 | Kth Largest Element | Top-K (min-heap) | Medium |
| 347 | Top K Frequent Elements | Top-K + Freq Map | Medium |
| 973 | K Closest Points to Origin | Top-K (max-heap) | Medium |
| 253 | Meeting Rooms II | Scheduling (min-heap) | Medium |
| 23 | Merge K Sorted Lists | Merge (min-heap) | Hard |
| 295 | Find Median from Data Stream | Two Heaps | Hard |
| 621 | Task Scheduler | Greedy + Heap | Medium |
