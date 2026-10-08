# Linked Lists in Kotlin

> Kotlin implementation reference for linked lists: node classes, fast/slow pointers, reversal (including k-groups), merging, the dummy-node trick and random-pointer copy.
> New to this topic? Learn it first in [10. Linked Lists](../learn/10-linked-lists.md).

## Complexity Overview

| Operation | Singly Linked | Doubly Linked |
|-----------|--------------|---------------|
| Access by index | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n)* | O(1) |
| Delete at head | O(1) | O(1) |
| Search | O(n) | O(n) |

> **Key Interview Point:** Kotlin's null safety (`?.`, `?:`, `!!`) makes linked list code cleaner than Java. Use safe calls extensively.

> **Common Mistake:** Using `!!` (force unwrap) can cause NPE. Prefer `?.` chains for linked list traversal.

> **Kotlin Smart-Cast Gotcha:** `if (node.next != null) use(node.next)` does not compile when `use` needs a non-null value, because `next` is a mutable `var` property and could change between the check and the use. Copy it into a local `val` first, or write `node.next?.let { use(it) }`.

---

## Node Definition & Basic Operations

```kotlin
// Singly Linked List Node
class ListNode(var `val`: Int) {
    var next: ListNode? = null
}

// Doubly Linked List Node
class DoublyListNode(var `val`: Int) {
    var next: DoublyListNode? = null
    var prev: DoublyListNode? = null
}

// Basic Linked List Operations
class SinglyLinkedList {
    private var head: ListNode? = null

    fun addFirst(value: Int) {
        val newNode = ListNode(value)
        newNode.next = head
        head = newNode
    }

    fun addLast(value: Int) {
        if (head == null) {
            head = ListNode(value)
            return
        }

        var current = head
        while (current?.next != null) {
            current = current.next
        }
        current?.next = ListNode(value)
    }

    fun removeFirst(): Int? {
        if (head == null) return null

        val value = head?.`val`
        head = head?.next
        return value
    }

    fun contains(value: Int): Boolean {
        var current = head
        while (current != null) {
            if (current.`val` == value) return true
            current = current.next
        }
        return false
    }

    fun size(): Int {
        var count = 0
        var current = head
        while (current != null) {
            count++
            current = current.next
        }
        return count
    }
}
```

## 🐢 Fast & Slow Pointers Pattern

**Idea:** Move `slow` one step and `fast` two steps. If there is a cycle, fast eventually laps slow and they meet; if not, fast hits the end, at which point slow is at the middle. To find the cycle start, reset one pointer to `head` after they meet and move both one step at a time.

```kotlin
// Detect cycle in linked list
fun hasCycle(head: ListNode?): Boolean {
    if (head?.next == null) return false

    var slow: ListNode? = head
    var fast: ListNode? = head

    while (fast?.next != null) {
        slow = slow?.next
        fast = fast?.next?.next

        if (slow == fast) return true
    }
    return false
}

// Find cycle start
fun detectCycle(head: ListNode?): ListNode? {
    if (head?.next == null) return null

    var slow: ListNode? = head
    var fast: ListNode? = head

    while (fast?.next != null) {
        slow = slow?.next
        fast = fast?.next?.next

        if (slow == fast) {
            slow = head
            while (slow != fast) {
                slow = slow?.next
                fast = fast?.next
            }
            return slow
        }
    }
    return null
}

// Find middle of linked list
fun findMiddle(head: ListNode?): ListNode? {
    var slow: ListNode? = head
    var fast: ListNode? = head

    while (fast?.next != null) {
        slow = slow?.next
        fast = fast?.next?.next
    }
    return slow
}
```

## 🔄 Reversing Linked List Pattern

**Idea:** Walk the list once, flipping each `next` pointer to point backwards. Keep three references (`prev`, `curr`, `next`) so you never lose the rest of the list. For k-groups, reverse one block of k at a time and reconnect it to the nodes before and after it.

```kotlin
// Reverse entire linked list (Iterative)
fun reverseList(head: ListNode?): ListNode? {
    var prev: ListNode? = null
    var curr = head

    while (curr != null) {
        val next = curr.next
        curr.next = prev
        prev = curr
        curr = next
    }
    return prev
}

// Reverse entire linked list (Recursive)
fun reverseListRecursive(head: ListNode?): ListNode? {
    if (head?.next == null) return head

    val reversedHead = reverseListRecursive(head.next)
    head.next?.next = head
    head.next = null
    return reversedHead
}

// Reverse nodes in k-groups
fun reverseKGroup(head: ListNode?, k: Int): ListNode? {
    if (head == null || k == 1) return head

    val dummy = ListNode(0)
    dummy.next = head
    var prev = dummy

    while (true) {
        val kth = getKth(prev, k) ?: break // fewer than k nodes left: stop

        val nextGroup = kth.next
        val groupStart = prev.next!!

        // Reverse the group; the first node ends up pointing at nextGroup
        var p: ListNode? = nextGroup
        var curr: ListNode? = groupStart
        while (curr !== nextGroup) {
            val next = curr!!.next
            curr.next = p
            p = curr
            curr = next
        }

        prev.next = kth      // kth is the new head of this group
        prev = groupStart    // old head is now the group's tail
    }

    return dummy.next
}

private fun getKth(curr: ListNode?, k: Int): ListNode? {
    var count = 0
    var node = curr
    while (node != null && count < k) {
        node = node.next
        count++
    }
    return node
}
```

## 🧵 Merging Linked Lists Pattern

**Idea:** Like the merge step of merge sort: repeatedly attach the smaller of the two current heads to the result, then append whatever remains. For K lists, a min-heap of the K current heads picks the smallest in O(log k), giving O(N log k) overall.

```kotlin
import java.util.PriorityQueue

// Merge two sorted lists
fun mergeTwoLists(list1: ListNode?, list2: ListNode?): ListNode? {
    val dummy = ListNode(0)
    var current = dummy

    var l1 = list1
    var l2 = list2

    while (l1 != null && l2 != null) {
        if (l1.`val` <= l2.`val`) {
            current.next = l1
            l1 = l1.next
        } else {
            current.next = l2
            l2 = l2.next
        }
        current = current.next!!
    }

    current.next = l1 ?: l2
    return dummy.next
}

// Merge K sorted lists (using priority queue)
fun mergeKLists(lists: Array<ListNode?>): ListNode? {
    if (lists.isEmpty()) return null

    val dummy = ListNode(0)
    var current = dummy

    val pq = PriorityQueue<ListNode>(compareBy { it.`val` }) // not a.val - b.val (can overflow)

    for (list in lists) {
        if (list != null) {
            pq.offer(list)
        }
    }

    while (pq.isNotEmpty()) {
        val node = pq.poll()
        current.next = node
        current = current.next!!

        node.next?.let { pq.offer(it) } // `next` is a var: no smart cast after a null check
    }

    return dummy.next
}
```

## 🧩 Dummy Node Technique

**Idea:** Put a fake node before the head so the real head can be removed or replaced with the same code as any other node; return `dummy.next` at the end. For "nth from end", move one pointer n+1 steps ahead first, then move both until it falls off.

```kotlin
// Remove Nth node from end
fun removeNthFromEnd(head: ListNode?, n: Int): ListNode? {
    val dummy = ListNode(0)
    dummy.next = head

    var first: ListNode? = dummy
    var second: ListNode? = dummy

    // Advance first pointer by n+1 steps
    for (i in 0..n) {
        first = first?.next
    }

    // Move both pointers until first reaches end
    while (first != null) {
        first = first.next
        second = second?.next
    }

    // Remove the node
    second?.next = second?.next?.next

    return dummy.next
}

// Add two numbers (represented as linked lists)
fun addTwoNumbers(l1: ListNode?, l2: ListNode?): ListNode? {
    val dummy = ListNode(0)
    var current = dummy

    var p1 = l1
    var p2 = l2
    var carry = 0

    while (p1 != null || p2 != null || carry > 0) {
        val sum = (p1?.`val` ?: 0) + (p2?.`val` ?: 0) + carry
        carry = sum / 10

        current.next = ListNode(sum % 10)
        current = current.next!!

        p1 = p1?.next
        p2 = p2?.next
    }

    return dummy.next
}
```

## 🧠 Linked List + Stack Pattern

**Idea:** A stack gives you the list in reverse order, which helps compare from both ends (palindrome) at the cost of O(n) space. The O(1)-space alternative (used in `reorderList`) is: find the middle, reverse the second half, then walk both halves together.

```kotlin
// Check if palindrome
fun isPalindrome(head: ListNode?): Boolean {
    // kotlin.collections.ArrayDeque has no push/pop: use addLast/removeLast as a stack
    val stack = ArrayDeque<ListNode>()
    var current = head

    // Push all nodes to stack
    while (current != null) {
        stack.addLast(current)
        current = current.next
    }

    // Compare with original list
    current = head
    while (current != null) {
        if (current.`val` != stack.removeLast().`val`) {
            return false
        }
        current = current.next
    }

    return true
}

// Reorder list
fun reorderList(head: ListNode?): Unit {
    if (head?.next == null) return

    // Find middle
    var slow: ListNode? = head
    var fast: ListNode? = head

    while (fast?.next != null) {
        slow = slow?.next
        fast = fast?.next?.next
    }

    // Reverse second half
    var prev: ListNode? = null
    var curr = slow?.next
    slow?.next = null

    while (curr != null) {
        val next = curr.next
        curr.next = prev
        prev = curr
        curr = next
    }

    // Merge two halves
    var first = head
    var second = prev

    while (second != null) {
        val temp1 = first?.next
        val temp2 = second.next

        first?.next = second
        second.next = temp1

        first = temp1
        second = temp2
    }
}
```

## 🔀 Copy List with Random Pointer

**Idea:** Insert each copy right after its original (A -> A' -> B -> B'), so `original.random.next` is the copy's random target. Then unweave the two lists. O(n) time, O(1) extra space (a `HashMap` from original to copy is the simpler O(n)-space version).

```kotlin
class RandomListNode(var `val`: Int) {
    var next: RandomListNode? = null
    var random: RandomListNode? = null
}

fun copyRandomList(head: RandomListNode?): RandomListNode? {
    if (head == null) return null

    // Step 1: Create copy nodes interleaved with original nodes
    var curr: RandomListNode? = head // explicit nullable type: head is smart-cast to non-null here
    while (curr != null) {
        val copy = RandomListNode(curr.`val`)
        copy.next = curr.next
        curr.next = copy
        curr = copy.next
    }

    // Step 2: Set random pointers for copy nodes
    curr = head
    while (curr != null) {
        if (curr.random != null) {
            curr.next?.random = curr.random?.next
        }
        curr = curr.next?.next
    }

    // Step 3: Separate original and copy lists
    curr = head
    val copyHead = head.next
    var copyCurr = copyHead

    while (curr != null) {
        curr.next = curr.next?.next
        copyCurr?.next = copyCurr?.next?.next

        curr = curr.next
        copyCurr = copyCurr?.next
    }

    return copyHead
}
```
