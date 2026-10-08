# Linked Lists in JavaScript

> JavaScript implementation reference for linked lists: node classes, fast/slow pointers, reversal, merging, the dummy-node trick and random-pointer copy.
> New to this topic? Learn it first in [10. Linked Lists](../learn/10-linked-lists.md).

## Complexity Overview

| Operation | Singly Linked | Doubly Linked |
|-----------|--------------|---------------|
| Access by index | O(n) | O(n) |
| Insert at head | O(1) | O(1) |
| Insert at tail | O(n)* | O(1) |
| Delete at head | O(1) | O(1) |
| Search | O(n) | O(n) |

*O(1) if tail pointer is maintained

> **Key Interview Point:** Use a **dummy node** whenever the head might change. Use **fast/slow pointers** for cycle and middle-node problems. Always draw diagrams.

> **Common Mistake:** Losing reference to the next node during reversal. Always save `curr.next` before overwriting.

---

## Node Definition & Basic Operations

```javascript
// Singly Linked List Node
class ListNode {
    constructor(val) {
        this.val = val;
        this.next = null;
    }
}

// Doubly Linked List Node
class DoublyListNode {
    constructor(val) {
        this.val = val;
        this.next = null;
        this.prev = null;
    }
}

// Basic Linked List Operations
class SinglyLinkedList {
    constructor() {
        this.head = null;
    }

    addFirst(value) {
        const newNode = new ListNode(value);
        newNode.next = this.head;
        this.head = newNode;
    }

    addLast(value) {
        if (this.head === null) {
            this.head = new ListNode(value);
            return;
        }

        let current = this.head;
        while (current.next !== null) {
            current = current.next;
        }
        current.next = new ListNode(value);
    }

    removeFirst() {
        if (this.head === null) return null;

        const value = this.head.val;
        this.head = this.head.next;
        return value;
    }

    contains(value) {
        let current = this.head;
        while (current !== null) {
            if (current.val === value) return true;
            current = current.next;
        }
        return false;
    }

    size() {
        let count = 0;
        let current = this.head;
        while (current !== null) {
            count++;
            current = current.next;
        }
        return count;
    }
}
```

## 🐢 Fast & Slow Pointers Pattern

**Idea:** Move `slow` one step and `fast` two steps. If there is a cycle, fast eventually laps slow and they meet; if not, fast hits the end, at which point slow is at the middle.

```javascript
// Detect cycle in linked list
function hasCycle(head) {
    if (!head || !head.next) return false;

    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow === fast) return true;
    }
    return false;
}

// Find cycle start
function detectCycle(head) {
    if (!head || !head.next) return null;

    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow === fast) {
            slow = head;
            while (slow !== fast) {
                slow = slow.next;
                fast = fast.next;
            }
            return slow;
        }
    }
    return null;
}

// Find middle of linked list
function findMiddle(head) {
    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```

## 🔄 Reversing Linked List Pattern

**Idea:** Walk the list once, flipping each `next` pointer to point backwards. Keep three references (`prev`, `curr`, `next`) so you never lose the rest of the list.

```javascript
// Reverse entire linked list (Iterative)
function reverseList(head) {
    let prev = null;
    let curr = head;

    while (curr) {
        const next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Reverse entire linked list (Recursive)
function reverseListRecursive(head) {
    if (!head || !head.next) return head;

    const reversedHead = reverseListRecursive(head.next);
    head.next.next = head;
    head.next = null;
    return reversedHead;
}

// Reverse nodes in k-groups
function reverseKGroup(head, k) {
    if (!head || k === 1) return head;

    const dummy = new ListNode(0);
    dummy.next = head;
    let prev = dummy;

    while (true) {
        const kth = getKth(prev, k);
        if (!kth) break;

        const groupStart = prev.next;
        const nextGroup = kth.next;

        // Reverse the group; the first node ends up pointing at nextGroup
        let p = nextGroup;
        let curr = groupStart;
        while (curr !== nextGroup) {
            const next = curr.next;
            curr.next = p;
            p = curr;
            curr = next;
        }

        prev.next = kth;      // kth is the new head of this group
        prev = groupStart;    // old head is now the group's tail
    }

    return dummy.next;
}

function getKth(curr, k) {
    let count = 0;
    let node = curr;
    while (node && count < k) {
        node = node.next;
        count++;
    }
    return node;
}
```

## 🧵 Merging Linked Lists Pattern

**Idea:** Like the merge step of merge sort: repeatedly attach the smaller of the two current heads to the result, then append whatever remains.

```javascript
// Merge two sorted lists
function mergeTwoLists(list1, list2) {
    const dummy = new ListNode(0);
    let current = dummy;

    let l1 = list1;
    let l2 = list2;

    while (l1 && l2) {
        if (l1.val <= l2.val) {
            current.next = l1;
            l1 = l1.next;
        } else {
            current.next = l2;
            l2 = l2.next;
        }
        current = current.next;
    }

    current.next = l1 || l2;
    return dummy.next;
}

// Merge K sorted lists (divide & conquer: pair up lists and merge, log k rounds)
// O(N log k) time, where N = total nodes. JS has no built-in heap; with the PriorityQueue
// class from heaps-priority-queues.md the heap approach is also O(N log k).
function mergeKLists(lists) {
    if (lists.length === 0) return null;

    let interval = 1;
    while (interval < lists.length) {
        for (let i = 0; i + interval < lists.length; i += interval * 2) {
            lists[i] = mergeTwoLists(lists[i], lists[i + interval]);
        }
        interval *= 2;
    }
    return lists[0];
}
```

## 🧩 Dummy Node Technique

**Idea:** Put a fake node before the head so the real head can be removed or replaced with the same code as any other node; return `dummy.next` at the end.

```javascript
// Remove Nth node from end
function removeNthFromEnd(head, n) {
    const dummy = new ListNode(0);
    dummy.next = head;

    let first = dummy;
    let second = dummy;

    // Advance first pointer by n+1 steps
    for (let i = 0; i <= n; i++) {
        first = first.next;
    }

    // Move both pointers until first reaches end
    while (first) {
        first = first.next;
        second = second.next;
    }

    // Remove the node
    second.next = second.next?.next || null;

    return dummy.next;
}

// Add two numbers (represented as linked lists)
function addTwoNumbers(l1, l2) {
    const dummy = new ListNode(0);
    let current = dummy;

    let p1 = l1;
    let p2 = l2;
    let carry = 0;

    while (p1 || p2 || carry > 0) {
        const sum = (p1?.val || 0) + (p2?.val || 0) + carry;
        carry = Math.floor(sum / 10);

        current.next = new ListNode(sum % 10);
        current = current.next;

        p1 = p1?.next;
        p2 = p2?.next;
    }

    return dummy.next;
}
```

## 🧠 Linked List + Stack Pattern

**Idea:** A stack gives you the list in reverse order, which helps compare from both ends (palindrome) at the cost of O(n) space. The O(1)-space alternative (used in `reorderList`) is: find the middle, reverse the second half, then walk both halves together.

```javascript
// Check if palindrome
function isPalindrome(head) {
    const stack = [];
    let current = head;

    // Push all nodes to stack
    while (current) {
        stack.push(current);
        current = current.next;
    }

    // Compare with original list
    current = head;
    while (current) {
        if (current.val !== stack.pop().val) {
            return false;
        }
        current = current.next;
    }

    return true;
}

// Reorder list
function reorderList(head) {
    if (!head || !head.next) return;

    // Find middle
    let slow = head;
    let fast = head;

    while (fast && fast.next) {
        slow = slow.next;
        fast = fast.next.next;
    }

    // Reverse second half
    let prev = null;
    let curr = slow.next;
    slow.next = null;

    while (curr) {
        const next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }

    // Merge two halves
    let first = head;
    let second = prev;

    while (second) {
        const temp1 = first.next;
        const temp2 = second.next;

        first.next = second;
        second.next = temp1;

        first = temp1;
        second = temp2;
    }
}
```

## 🔀 Copy List with Random Pointer

**Idea:** Insert each copy right after its original (A -> A' -> B -> B'), so `original.random.next` is the copy's random target. Then unweave the two lists. O(n) time, O(1) extra space (a `Map` from original to copy is the simpler O(n)-space version).

```javascript
// Node with random pointer
class RandomListNode {
    constructor(val) {
        this.val = val;
        this.next = null;
        this.random = null;
    }
}

// Copy list with random pointer
function copyRandomList(head) {
    if (!head) return null;

    // Step 1: Create copy nodes interleaved with original nodes
    let curr = head;
    while (curr) {
        const copy = new RandomListNode(curr.val);
        copy.next = curr.next;
        curr.next = copy;
        curr = copy.next;
    }

    // Step 2: Set random pointers for copy nodes
    curr = head;
    while (curr) {
        if (curr.random) {
            curr.next.random = curr.random.next;
        }
        curr = curr.next.next;
    }

    // Step 3: Separate original and copy lists
    curr = head;
    const copyHead = head.next;
    let copyCurr = copyHead;

    while (curr) {
        curr.next = curr.next.next;
        if (copyCurr.next) {
            copyCurr.next = copyCurr.next.next;
        }

        curr = curr.next;
        copyCurr = copyCurr.next;
    }

    return copyHead;
}
```
