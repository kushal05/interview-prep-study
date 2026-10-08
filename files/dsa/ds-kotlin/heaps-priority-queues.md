# Heaps & Priority Queues in Kotlin

> Kotlin implementation reference for heaps (`java.util.PriorityQueue`): basic usage, custom comparators, top-K, and scheduling / k-way merge.
> New to this topic? Learn it first in [14. Heaps and Priority Queues](../learn/14-heaps-and-priority-queues.md).

## Complexity Overview

| Operation | PriorityQueue |
|-----------|--------------|
| add | O(log n) |
| poll (extract min) | O(log n) |
| peek | O(1) |

> **Key Interview Point:** Kotlin uses Java's `PriorityQueue` (min-heap by default). For max-heap: `PriorityQueue(compareByDescending { it })`. For custom objects: `PriorityQueue(compareBy<Item> { it.freq })`. Avoid subtraction comparators like `{ a, b -> a.freq - b.freq }`: they overflow for large or negative values and silently break heap order.

---

## Basic Usage

```kotlin
// Priority Queue (Min Heap) in Kotlin
import java.util.PriorityQueue

// Basic operations
fun priorityQueueBasics() {
    // Min heap (natural order)
    val minHeap = PriorityQueue<Int>()

    // Add elements
    minHeap.add(3)
    minHeap.add(1)
    minHeap.add(2)

    // Peek min element
    println(minHeap.peek()) // 1

    // Remove min element
    println(minHeap.poll()) // 1

    // Max heap
    val maxHeap = PriorityQueue<Int>(compareByDescending { it })
    maxHeap.addAll(listOf(1, 3, 2))
    println(maxHeap.peek()) // 3
}

// Custom priority queue with objects
data class Task(val priority: Int, val name: String)

fun customPriorityQueue() {
    val pq = PriorityQueue<Task>(compareByDescending { it.priority }) // Max heap

    pq.add(Task(1, "Low"))
    pq.add(Task(3, "High"))
    pq.add(Task(2, "Medium"))

    println(pq.poll()?.name) // "High"
}
```

## ⬆️ Top-K Elements Pattern

**Idea:** To keep the k largest, use a min-heap of size k: push every element and pop whenever the size exceeds k, so the smallest of the top k sits at the root. Flip it (max-heap) to keep the k smallest/closest. O(n log k).

```kotlin
// Kth largest element in array
fun findKthLargest(nums: IntArray, k: Int): Int {
    val pq = PriorityQueue<Int>() // Min heap

    for (num in nums) {
        pq.add(num)
        if (pq.size > k) {
            pq.poll()
        }
    }

    return pq.peek()
}

// Top K frequent elements
fun topKFrequent(nums: IntArray, k: Int): IntArray {
    val frequencyMap = mutableMapOf<Int, Int>()
    for (num in nums) {
        frequencyMap[num] = frequencyMap.getOrDefault(num, 0) + 1
    }

    // Min heap based on frequency
    val pq = PriorityQueue<Pair<Int, Int>>(compareBy { it.second })

    for ((num, freq) in frequencyMap) {
        pq.add(Pair(num, freq))
        if (pq.size > k) {
            pq.poll()
        }
    }

    return pq.map { it.first }.toIntArray()
}

// K closest points to origin
fun kClosest(points: Array<IntArray>, k: Int): Array<IntArray> {
    // Max heap based on distance
    val pq = PriorityQueue<Pair<IntArray, Int>>(compareByDescending { it.second })

    for (point in points) {
        val distance = point[0] * point[0] + point[1] * point[1]
        pq.add(Pair(point, distance))

        if (pq.size > k) {
            pq.poll()
        }
    }

    return pq.map { it.first }.toTypedArray()
}
```

## ⏱ Min Heap for Scheduling Pattern

**Idea:** A min-heap always tells you what finishes or comes next. For meeting rooms, the heap holds end times of rooms in use. For merging k lists, it holds the current head of each list.

```kotlin
// Meeting rooms (find minimum number of rooms needed)
fun minMeetingRooms(intervals: Array<IntArray>): Int {
    if (intervals.isEmpty()) return 0

    intervals.sortBy { it[0] }

    val pq = PriorityQueue<Int>() // Min heap for end times
    pq.add(intervals[0][1])

    for (i in 1 until intervals.size) {
        if (intervals[i][0] >= pq.peek()) {
            pq.poll() // Reuse the room
        }
        pq.add(intervals[i][1])
    }

    return pq.size
}

// Merge K sorted lists (using priority queue)
fun mergeKLists(lists: Array<ListNode?>): ListNode? {
    if (lists.isEmpty()) return null

    val dummy = ListNode(0)
    var current = dummy

    // Min heap for nodes
    val pq = PriorityQueue<ListNode>(compareBy { it.`val` })

    // Add first node from each list
    for (list in lists) {
        if (list != null) {
            pq.add(list)
        }
    }

    while (pq.isNotEmpty()) {
        val node = pq.poll()
        current.next = node
        current = current.next!!

        node.next?.let { pq.add(it) } // smart cast doesn't work on a mutable `var next`
    }

    return dummy.next
}
```
