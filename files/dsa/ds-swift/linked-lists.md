# Linked Lists in Swift

> Swift implementation reference for linked lists: node classes, fast/slow pointers, reversal (including k-groups), merging, the dummy-node trick and random-pointer copy.
> New to this topic? Learn it first in [10. Linked Lists](../learn/10-linked-lists.md).

## Complexity Overview

| Operation | Singly Linked | Doubly Linked |
|-----------|--------------|---------------|
| Access by index | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n)* | O(1) |
| Delete at head | O(1) | O(1) |
| Search | O(n) | O(n) |

> **Key Interview Point:** Swift uses reference types (`class`) for nodes. Use `===` (identity comparison) not `==` for node equality checks. Optionals (`?`) make linked list code safe.

> **Common Mistake:** Swift linked list nodes are classes (reference types). When comparing nodes for cycle detection, use `===` not `==`.

---

## Node Definition & Basic Operations

```swift
// Singly Linked List Node
class ListNode {
    var val: Int
    var next: ListNode?

    init(_ val: Int) {
        self.val = val
        self.next = nil
    }
}

// Doubly Linked List Node
class DoublyListNode {
    var val: Int
    var next: DoublyListNode?
    var prev: DoublyListNode?

    init(_ val: Int) {
        self.val = val
        self.next = nil
        self.prev = nil
    }
}

// Basic Singly Linked List Operations
class SinglyLinkedList {
    private var head: ListNode?

    func addFirst(_ value: Int) {
        let newNode = ListNode(value)
        newNode.next = head
        head = newNode
    }

    func addLast(_ value: Int) {
        if head == nil {
            head = ListNode(value)
            return
        }

        var current = head
        while current?.next != nil {
            current = current?.next
        }
        current?.next = ListNode(value)
    }

    func removeFirst() -> Int? {
        guard let currentHead = head else { return nil }

        let value = currentHead.val
        head = currentHead.next
        return value
    }

    func contains(_ value: Int) -> Bool {
        var current = head
        while current != nil {
            if current?.val == value {
                return true
            }
            current = current?.next
        }
        return false
    }

    func size() -> Int {
        var count = 0
        var current = head
        while current != nil {
            count += 1
            current = current?.next
        }
        return count
    }
}
```

## 🐢 Fast & Slow Pointers Pattern

**Idea:** Move `slow` one step and `fast` two steps. If there is a cycle, fast eventually laps slow and they meet (`===`); if not, fast hits the end, at which point slow is at the middle. To find the cycle start, reset one pointer to `head` after they meet and move both one step at a time.

```swift
// Detect cycle in linked list
func hasCycle(_ head: ListNode?) -> Bool {
    guard let head = head, head.next != nil else { return false }

    var slow: ListNode? = head
    var fast: ListNode? = head

    while fast?.next != nil {
        slow = slow?.next
        fast = fast?.next?.next

        if slow === fast {
            return true
        }
    }
    return false
}

// Find cycle start
func detectCycle(_ head: ListNode?) -> ListNode? {
    guard let head = head, head.next != nil else { return nil }

    var slow: ListNode? = head
    var fast: ListNode? = head

    while fast?.next != nil {
        slow = slow?.next
        fast = fast?.next?.next

        if slow === fast {
            slow = head
            while slow !== fast {
                slow = slow?.next
                fast = fast?.next
            }
            return slow
        }
    }
    return nil
}

// Find middle of linked list
func findMiddle(_ head: ListNode?) -> ListNode? {
    var slow: ListNode? = head
    var fast: ListNode? = head

    while fast?.next != nil {
        slow = slow?.next
        fast = fast?.next?.next
    }
    return slow
}
```

## 🔄 Reversing Linked List Pattern

**Idea:** Walk the list once, flipping each `next` pointer to point backwards. Keep three references (`prev`, `curr`, `next`) so you never lose the rest of the list. For k-groups, reverse one block of k at a time and reconnect it to the nodes before and after it.

```swift
// Reverse entire linked list (Iterative)
func reverseList(_ head: ListNode?) -> ListNode? {
    var prev: ListNode? = nil
    var curr = head

    while curr != nil {
        let next = curr?.next
        curr?.next = prev
        prev = curr
        curr = next
    }
    return prev
}

// Reverse entire linked list (Recursive)
func reverseListRecursive(_ head: ListNode?) -> ListNode? {
    guard let head = head, head.next != nil else { return head }

    let reversedHead = reverseListRecursive(head.next)
    head.next?.next = head
    head.next = nil
    return reversedHead
}

// Reverse nodes in k-groups
func reverseKGroup(_ head: ListNode?, _ k: Int) -> ListNode? {
    guard let head = head, k > 1 else { return head }

    let dummy = ListNode(0)
    dummy.next = head
    var prev = dummy

    while true {
        guard let kth = getKth(prev, k) else { break } // fewer than k nodes left
        let nextGroup = kth.next
        let groupStart = prev.next!

        // Reverse the group; the first node ends up pointing at nextGroup
        var p: ListNode? = nextGroup
        var curr: ListNode? = groupStart
        while curr !== nextGroup {
            let next = curr?.next
            curr?.next = p
            p = curr
            curr = next
        }

        prev.next = kth      // kth is the new head of this group
        prev = groupStart    // old head is now the group's tail
    }

    return dummy.next
}

private func getKth(_ curr: ListNode?, _ k: Int) -> ListNode? {
    var count = 0
    var node = curr
    while node != nil && count < k {
        node = node?.next
        count += 1
    }
    return node
}
```

## 🧵 Merging Linked Lists Pattern

**Idea:** Like the merge step of merge sort: repeatedly attach the smaller of the two current heads to the result, then append whatever remains. For K lists, a min-heap of the K current heads picks the smallest in O(log k), giving O(N log k) overall. Swift has no built-in heap, so a small `PriorityQueue` is defined at the end of this file.

```swift
// Merge two sorted lists
func mergeTwoLists(_ list1: ListNode?, _ list2: ListNode?) -> ListNode? {
    let dummy = ListNode(0)
    var current = dummy

    var l1 = list1
    var l2 = list2

    while l1 != nil && l2 != nil {
        if l1!.val <= l2!.val {
            current.next = l1
            l1 = l1?.next
        } else {
            current.next = l2
            l2 = l2?.next
        }
        current = current.next!
    }

    current.next = l1 ?? l2
    return dummy.next
}

// Merge K sorted lists (using priority queue)
struct ListNodeWrapper: Comparable {
    let node: ListNode

    static func < (lhs: ListNodeWrapper, rhs: ListNodeWrapper) -> Bool {
        return lhs.node.val < rhs.node.val
    }

    static func == (lhs: ListNodeWrapper, rhs: ListNodeWrapper) -> Bool {
        return lhs.node.val == rhs.node.val
    }
}

func mergeKLists(_ lists: [ListNode?]) -> ListNode? {
    guard !lists.isEmpty else { return nil }

    let dummy = ListNode(0)
    var current = dummy

    var pq = PriorityQueue<ListNodeWrapper>()

    for list in lists {
        if let list = list {
            pq.enqueue(ListNodeWrapper(node: list))
        }
    }

    while let wrapper = pq.dequeue() {
        current.next = wrapper.node
        current = current.next!

        if let nextNode = wrapper.node.next {
            pq.enqueue(ListNodeWrapper(node: nextNode))
        }
    }

    return dummy.next
}
```

## 🧩 Dummy Node Technique

**Idea:** Put a fake node before the head so the real head can be removed or replaced with the same code as any other node; return `dummy.next` at the end. For "nth from end", move one pointer n+1 steps ahead first, then move both until it falls off.

```swift
// Remove Nth node from end
func removeNthFromEnd(_ head: ListNode?, _ n: Int) -> ListNode? {
    let dummy = ListNode(0)
    dummy.next = head

    var first: ListNode? = dummy
    var second: ListNode? = dummy

    // Advance first pointer by n+1 steps
    for _ in 0...n {
        first = first?.next
    }

    // Move both pointers until first reaches end
    while first != nil {
        first = first?.next
        second = second?.next
    }

    // Remove the node
    second?.next = second?.next?.next

    return dummy.next
}

// Add two numbers (represented as linked lists)
func addTwoNumbers(_ l1: ListNode?, _ l2: ListNode?) -> ListNode? {
    let dummy = ListNode(0)
    var current = dummy

    var p1 = l1
    var p2 = l2
    var carry = 0

    while p1 != nil || p2 != nil || carry > 0 {
        let sum = (p1?.val ?? 0) + (p2?.val ?? 0) + carry
        carry = sum / 10

        current.next = ListNode(sum % 10)
        current = current.next!

        p1 = p1?.next
        p2 = p2?.next
    }

    return dummy.next
}
```

## 🧠 Linked List + Stack Pattern

**Idea:** A stack gives you the list in reverse order, which helps compare from both ends (palindrome) at the cost of O(n) space. The O(1)-space alternative (used in `reorderList`) is: find the middle, reverse the second half, then walk both halves together.

```swift
// Check if palindrome
func isPalindrome(_ head: ListNode?) -> Bool {
    var stack = [ListNode]()
    var current = head

    // Push all nodes to stack
    while current != nil {
        stack.append(current!)
        current = current?.next
    }

    // Compare with original list
    current = head
    while current != nil {
        if current?.val != stack.popLast()?.val {
            return false
        }
        current = current?.next
    }

    return true
}

// Reorder list
func reorderList(_ head: ListNode?) {
    guard let head = head, head.next != nil else { return }

    // Find middle
    var slow: ListNode? = head
    var fast: ListNode? = head

    while fast?.next != nil {
        slow = slow?.next
        fast = fast?.next?.next
    }

    // Reverse second half
    var prev: ListNode? = nil
    var curr = slow?.next
    slow?.next = nil

    while curr != nil {
        let next = curr?.next
        curr?.next = prev
        prev = curr
        curr = next
    }

    // Merge two halves
    var first = head
    var second = prev

    while second != nil {
        let temp1 = first.next
        let temp2 = second?.next

        first.next = second
        second?.next = temp1

        first = temp1!
        second = temp2
    }
}
```

## 🔀 Copy List with Random Pointer

**Idea:** Insert each copy right after its original (A -> A' -> B -> B'), so `original.random.next` is the copy's random target. Then unweave the two lists. O(n) time, O(1) extra space (a dictionary from `ObjectIdentifier(original)` to copy is the simpler O(n)-space version).

```swift
class RandomListNode {
    var val: Int
    var next: RandomListNode?
    var random: RandomListNode?

    init(_ val: Int) {
        self.val = val
        self.next = nil
        self.random = nil
    }
}

func copyRandomList(_ head: RandomListNode?) -> RandomListNode? {
    guard let head = head else { return nil }

    // `curr` must be Optional: `copy.next!` / `curr.next!` would crash at the end of the list
    // Step 1: Create copy nodes interleaved with original nodes
    var curr: RandomListNode? = head
    while let node = curr {
        let copy = RandomListNode(node.val)
        copy.next = node.next
        node.next = copy
        curr = copy.next
    }

    // Step 2: Set random pointers for copy nodes
    curr = head
    while let node = curr {
        node.next?.random = node.random?.next
        curr = node.next?.next
    }

    // Step 3: Separate original and copy lists
    curr = head
    let copyHead = head.next
    while let node = curr {
        let copy = node.next!          // every original is followed by its copy
        node.next = copy.next          // restore original list
        copy.next = copy.next?.next    // link copy to the next copy
        curr = node.next
    }

    return copyHead
}

// Priority Queue implementation for Swift
struct PriorityQueue<Element: Comparable> {
    private var heap: [Element] = []

    var isEmpty: Bool { heap.isEmpty }

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

    private mutating func siftUp(_ index: Int) {
        var childIndex = index
        let child = heap[childIndex]
        var parentIndex = (childIndex - 1) / 2

        while childIndex > 0 && child < heap[parentIndex] {
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

        if leftChildIndex < heap.count && heap[leftChildIndex] < heap[minIndex] {
            minIndex = leftChildIndex
        }

        if rightChildIndex < heap.count && heap[rightChildIndex] < heap[minIndex] {
            minIndex = rightChildIndex
        }

        if minIndex != index {
            heap.swapAt(index, minIndex)
            siftDown(minIndex)
        }
    }
}
```

---

## Summary: Linked List Pattern Map

| Pattern               | Keywords / When to Use                |
|----------------------|-------------------------------------|
| Fast & Slow Pointers  | middle node, cycle, loop              |
| Reversing             | reverse list, palindrome              |
| Merging               | merge sorted lists, merge sort        |
| Cycle Detection       | loop detection, find cycle start      |
| Dummy Node            | insert/delete at head, simplify logic |
| Stack-based Traversal | palindrome, reorder                   |
| Random Pointer Copy   | deep copy with next + random          |

---