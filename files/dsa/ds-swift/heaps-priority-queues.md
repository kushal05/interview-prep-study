# Heaps & Priority Queues in Swift

> Swift implementation reference for heaps: a hand-written binary-heap priority queue, plus top-K, scheduling, and merge-K patterns.
> New to this topic? Learn it first in [14. Heaps and Priority Queues](../learn/14-heaps-and-priority-queues.md).

## Complexity Overview

| Operation | Min/Max Heap |
|-----------|-------------|
| Insert (enqueue) | O(log n) |
| Extract min/max (dequeue) | O(log n) |
| Peek | O(1) |

> **Key Interview Point:** The Swift standard library has **no built-in heap** (the separate `swift-collections` package has `Heap`, but it is usually not available on LeetCode or in interviews). You must implement your own PriorityQueue (shown below). In interviews, state "I would use a heap" and implement key operations if asked.

> **Common Mistake:** Tuples, `ListNode`, and most custom structs are **not** `Comparable`, so a `PriorityQueue<Element: Comparable>` will not even compile for them. Give the heap a comparator closure instead (as below): `{ $0 < $1 }` = min-heap, `{ $0 > $1 }` = max-heap.

---

## Priority Queue Implementation (Min Heap)

```swift
// Priority Queue backed by a binary heap stored in an array.
// `areSorted(a, b) == true` means `a` should come out before `b`.
struct PriorityQueue<Element> {
    private var heap: [Element] = []
    private let areSorted: (Element, Element) -> Bool

    init(sort: @escaping (Element, Element) -> Bool) {
        self.areSorted = sort
    }

    var isEmpty: Bool { heap.isEmpty }
    var count: Int { heap.count }

    mutating func enqueue(_ element: Element) {
        heap.append(element)
        siftUp(heap.count - 1)
    }

    mutating func dequeue() -> Element? {
        guard !heap.isEmpty else { return nil }

        if heap.count == 1 {
            return heap.removeLast()
        }

        let root = heap[0]
        heap[0] = heap.removeLast()
        siftDown(0)
        return root
    }

    func peek() -> Element? {
        return heap.first
    }

    private mutating func siftUp(_ index: Int) {
        var childIndex = index
        let child = heap[childIndex]
        var parentIndex = (childIndex - 1) / 2

        while childIndex > 0 && areSorted(child, heap[parentIndex]) {
            heap[childIndex] = heap[parentIndex]
            childIndex = parentIndex
            parentIndex = (childIndex - 1) / 2
        }

        heap[childIndex] = child
    }

    private mutating func siftDown(_ index: Int) {
        let leftChildIndex = 2 * index + 1
        let rightChildIndex = 2 * index + 2
        var minIndex = index

        if leftChildIndex < heap.count && areSorted(heap[leftChildIndex], heap[minIndex]) {
            minIndex = leftChildIndex
        }

        if rightChildIndex < heap.count && areSorted(heap[rightChildIndex], heap[minIndex]) {
            minIndex = rightChildIndex
        }

        if minIndex != index {
            heap.swapAt(index, minIndex)
            siftDown(minIndex)
        }
    }
}

// Convenience: plain min-heap for Comparable types
extension PriorityQueue where Element: Comparable {
    init() { self.init(sort: <) }
}

// Basic operations
func priorityQueueBasics() {
    // Min heap
    var minHeap = PriorityQueue<Int>()

    // Add elements
    minHeap.enqueue(3)
    minHeap.enqueue(1)
    minHeap.enqueue(2)

    // Peek min element
    print(minHeap.peek() ?? 0) // 1

    // Remove min element
    print(minHeap.dequeue() ?? 0) // 1

    // Max heap: just flip the comparator
    var maxHeap = PriorityQueue<Int>(sort: >)
    maxHeap.enqueue(1)
    maxHeap.enqueue(3)
    maxHeap.enqueue(2)
    print(maxHeap.dequeue() ?? 0) // 3
}

// Custom priority queue with objects
struct Task {
    let priority: Int
    let name: String
}

func customPriorityQueue() {
    // Max heap on priority: highest priority comes out first
    var tasks = PriorityQueue<Task>(sort: { $0.priority > $1.priority })

    tasks.enqueue(Task(priority: 1, name: "Low"))
    tasks.enqueue(Task(priority: 3, name: "High"))
    tasks.enqueue(Task(priority: 2, name: "Medium"))

    print(tasks.dequeue()?.name ?? "") // "High"
}
```

## ⬆️ Top-K Elements Pattern

**Idea:** To keep the K *largest* items, use a *min*-heap of size K: push each item and pop when size exceeds K, so the smallest of the current top K is evicted. The root is then the Kth largest. O(n log k).

```swift
// Kth largest element in array
func findKthLargest(_ nums: [Int], _ k: Int) -> Int {
    var pq = PriorityQueue<Int>() // Min heap

    for num in nums {
        pq.enqueue(num)
        if pq.count > k {
            _ = pq.dequeue()
        }
    }

    return pq.peek() ?? 0
}

// Top K frequent elements
func topKFrequent(_ nums: [Int], _ k: Int) -> [Int] {
    var frequencyMap = [Int: Int]()
    for num in nums {
        frequencyMap[num, default: 0] += 1
    }

    // Min heap based on frequency (tuples aren't Comparable, so pass a comparator)
    var pq = PriorityQueue<(freq: Int, num: Int)>(sort: { $0.freq < $1.freq })

    for (num, freq) in frequencyMap {
        pq.enqueue((freq, num))
        if pq.count > k {
            _ = pq.dequeue()
        }
    }

    var result = [Int]()
    while !pq.isEmpty {
        let (_, num) = pq.dequeue()!
        result.append(num)
    }

    return result
}

// K closest points to origin
func kClosest(_ points: [[Int]], _ k: Int) -> [[Int]] {
    // Max heap based on distance: evict the farthest point when size > k
    var pq = PriorityQueue<(dist: Int, point: [Int])>(sort: { $0.dist > $1.dist })

    for point in points {
        let distance = point[0] * point[0] + point[1] * point[1]
        pq.enqueue((distance, point))

        if pq.count > k {
            _ = pq.dequeue()
        }
    }

    var result = [[Int]]()
    while !pq.isEmpty {
        let (_, point) = pq.dequeue()!
        result.append(point)
    }

    return result
}
```

## ⏱ Min Heap for Scheduling Pattern

**Idea:** Sort meetings by start time and keep a min-heap of end times of rooms in use. If the earliest-ending room is free by the time the next meeting starts, reuse it (pop); always push the new end time. The heap size at the end is the number of rooms. For Merge K Lists, the heap holds the current head of each list, so the smallest node is always on top.

```swift
// Meeting rooms (find minimum number of rooms needed)
func minMeetingRooms(_ intervals: [[Int]]) -> Int {
    guard !intervals.isEmpty else { return 0 }

    let sortedIntervals = intervals.sorted { $0[0] < $1[0] }

    var pq = PriorityQueue<Int>() // Min heap for end times
    pq.enqueue(sortedIntervals[0][1])

    for i in 1..<sortedIntervals.count {
        if sortedIntervals[i][0] >= (pq.peek() ?? 0) {
            _ = pq.dequeue() // Reuse the room
        }
        pq.enqueue(sortedIntervals[i][1])
    }

    return pq.count
}

// Merge K sorted lists (using priority queue)
// ListNode is defined in linked-lists.md
func mergeKLists(_ lists: [ListNode?]) -> ListNode? {
    guard !lists.isEmpty else { return nil }

    let dummy = ListNode(0)
    var current = dummy

    // Min heap for nodes, ordered by value (ListNode isn't Comparable)
    var pq = PriorityQueue<ListNode>(sort: { $0.val < $1.val })

    // Add first node from each list
    for list in lists {
        if let list = list {
            pq.enqueue(list)
        }
    }

    while !pq.isEmpty {
        let node = pq.dequeue()!
        current.next = node
        current = current.next!

        if let nextNode = node.next {
            pq.enqueue(nextNode)
        }
    }

    return dummy.next
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