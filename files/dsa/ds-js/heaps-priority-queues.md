# Heaps & Priority Queues in JavaScript

> JavaScript implementation reference for heaps: a reusable array-backed `PriorityQueue` class, top-K selection and heap-based scheduling/merging.
> New to this topic? Learn it first in [14. Heaps and Priority Queues](../learn/14-heaps-and-priority-queues.md).

## Complexity Overview

| Operation | Min/Max Heap |
|-----------|-------------|
| Insert (push) | O(log n) |
| Extract min/max (pop) | O(log n) |
| Peek | O(1) |

> **Key Interview Point:** JavaScript has **no built-in heap**. You must implement your own PriorityQueue class (shown below). In interviews, tell the interviewer you would use a heap and implement the key operations.

> **Common Mistake:** Forgetting that JS sorts are lexicographic by default. `[10,2,1].sort()` gives `[1,10,2]`. Always provide a comparator: `.sort((a,b) => a-b)`.

---

## Priority Queue Implementation (Min Heap)

```javascript
// Priority Queue (Min Heap) implementation
class PriorityQueue {
    constructor(compare = (a, b) => a - b) {
        this.heap = [];
        this.compare = compare;
    }

    push(element) {
        this.heap.push(element);
        this._bubbleUp(this.heap.length - 1);
    }

    pop() {
        if (this.isEmpty()) return null;

        const root = this.heap[0];
        const last = this.heap.pop();

        if (this.heap.length > 0) {
            this.heap[0] = last;
            this._sinkDown(0);
        }

        return root;
    }

    peek() {
        return this.isEmpty() ? null : this.heap[0];
    }

    isEmpty() {
        return this.heap.length === 0;
    }

    size() {
        return this.heap.length;
    }

    _bubbleUp(index) {
        while (index > 0) {
            const parentIndex = Math.floor((index - 1) / 2);
            if (this.compare(this.heap[index], this.heap[parentIndex]) >= 0) break;

            [this.heap[index], this.heap[parentIndex]] = [this.heap[parentIndex], this.heap[index]];
            index = parentIndex;
        }
    }

    _sinkDown(index) {
        const length = this.heap.length;
        while (true) {
            let leftIndex = 2 * index + 1;
            let rightIndex = 2 * index + 2;
            let smallest = index;

            if (leftIndex < length && this.compare(this.heap[leftIndex], this.heap[smallest]) < 0) {
                smallest = leftIndex;
            }
            if (rightIndex < length && this.compare(this.heap[rightIndex], this.heap[smallest]) < 0) {
                smallest = rightIndex;
            }

            if (smallest === index) break;

            [this.heap[index], this.heap[smallest]] = [this.heap[smallest], this.heap[index]];
            index = smallest;
        }
    }
}

// Basic operations
function priorityQueueBasics() {
    // Min heap
    const minHeap = new PriorityQueue();

    // Add elements
    minHeap.push(3);
    minHeap.push(1);
    minHeap.push(2);

    // Peek min element
    console.log(minHeap.peek()); // 1

    // Remove min element
    console.log(minHeap.pop()); // 1

    // Max heap using custom comparator
    const maxHeap = new PriorityQueue((a, b) => b - a);
    maxHeap.push(1);
    maxHeap.push(3);
    maxHeap.push(2);
    console.log(maxHeap.peek()); // 3
}

// Custom priority queue with objects
class Task {
    constructor(priority, name) {
        this.priority = priority;
        this.name = name;
    }
}

function customPriorityQueue() {
    // Max heap based on priority
    const pq = new PriorityQueue((a, b) => b.priority - a.priority);

    pq.push(new Task(1, "Low"));
    pq.push(new Task(3, "High"));
    pq.push(new Task(2, "Medium"));

    console.log(pq.pop().name); // "High"
}
```

## ⬆️ Top-K Elements Pattern

**Idea:** To keep the K *largest* items, use a *min*-heap capped at size K: whenever it grows past K, pop the smallest. The heap root is then the Kth largest. O(n log k) time, O(k) space.

```javascript
// Kth largest element in array
function findKthLargest(nums, k) {
    const pq = new PriorityQueue(); // Min heap

    for (const num of nums) {
        pq.push(num);
        if (pq.size() > k) {
            pq.pop();
        }
    }

    return pq.peek();
}

// Top K frequent elements
function topKFrequent(nums, k) {
    const frequencyMap = new Map();
    for (const num of nums) {
        frequencyMap.set(num, (frequencyMap.get(num) || 0) + 1);
    }

    // Min heap based on frequency
    const pq = new PriorityQueue((a, b) => a[1] - b[1]);

    for (const [num, freq] of frequencyMap) {
        pq.push([num, freq]);
        if (pq.size() > k) {
            pq.pop();
        }
    }

    const result = [];
    while (!pq.isEmpty()) {
        result.push(pq.pop()[0]);
    }

    return result;
}

// K closest points to origin
function kClosest(points, k) {
    // Max heap based on distance
    const pq = new PriorityQueue((a, b) => b.distance - a.distance);

    for (const point of points) {
        const distance = point[0] * point[0] + point[1] * point[1];
        pq.push({ point, distance });

        if (pq.size() > k) {
            pq.pop();
        }
    }

    const result = [];
    while (!pq.isEmpty()) {
        result.push(pq.pop().point);
    }

    return result;
}
```

## ⏱ Min Heap for Scheduling Pattern

**Idea:** A min-heap always hands you the earliest-finishing (or smallest) item next. For meeting rooms, the heap holds end times of rooms in use: if the next meeting starts after the earliest end, reuse that room; the heap size is the room count.

```javascript
// Meeting rooms (find minimum number of rooms needed)
function minMeetingRooms(intervals) {
    if (intervals.length === 0) return 0;

    intervals.sort((a, b) => a[0] - b[0]);

    const pq = new PriorityQueue(); // Min heap for end times
    pq.push(intervals[0][1]);

    for (let i = 1; i < intervals.length; i++) {
        if (intervals[i][0] >= pq.peek()) {
            pq.pop(); // Reuse the room
        }
        pq.push(intervals[i][1]);
    }

    return pq.size();
}

// Merge K sorted lists (using priority queue)
class ListNode {
    constructor(val) {
        this.val = val;
        this.next = null;
    }
}

function mergeKLists(lists) {
    if (lists.length === 0) return null;

    const dummy = new ListNode(0);
    let current = dummy;

    // Min heap for nodes
    const pq = new PriorityQueue((a, b) => a.val - b.val);

    // Add first node from each list
    for (const list of lists) {
        if (list !== null) {
            pq.push(list);
        }
    }

    while (!pq.isEmpty()) {
        const node = pq.pop();
        current.next = node;
        current = current.next;

        if (node.next !== null) {
            pq.push(node.next);
        }
    }

    return dummy.next;
}
```

---

## Summary: Heap Pattern Map

| Pattern                   | Keywords / Use Case                    |
|--------------------------|---------------------------------------|
| Top-K Elements           | Kth largest/smallest, frequent items  |
| Min Heap for Scheduling  | meeting rooms, merge intervals        |
| Merge K Sorted           | multiple sorted lists, priority merge |
| Priority Queue Basics    | min/max heap, custom comparators      |

---