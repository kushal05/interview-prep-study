# Linked Lists in Java

> Java implementation reference for linked lists: node definitions, fast/slow pointers, reversal, merging, dummy nodes and pointer-rewiring tricks.
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

> **Key Interview Point:** Linked lists excel when you need O(1) insertions/deletions at known positions. Use them over arrays when you do not need random access.

---

## Node Definition

```java
// Singly Linked List
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}

// Doubly Linked List
class DoublyListNode {
    int val;
    DoublyListNode next, prev;
    DoublyListNode(int val) { this.val = val; }
}
```

## Basic Operations

```java
class SinglyLinkedList {
    private ListNode head;

    // O(1) - insert at head
    public void addFirst(int value) {
        ListNode node = new ListNode(value);
        node.next = head;
        head = node;
    }

    // O(n) - insert at tail
    public void addLast(int value) {
        if (head == null) { head = new ListNode(value); return; }
        ListNode curr = head;
        while (curr.next != null) curr = curr.next;
        curr.next = new ListNode(value);
    }

    // O(1) - remove from head
    public Integer removeFirst() {
        if (head == null) return null;
        int val = head.val;
        head = head.next;
        return val;
    }

    // O(n) - search
    public boolean contains(int value) {
        for (ListNode curr = head; curr != null; curr = curr.next)
            if (curr.val == value) return true;
        return false;
    }
}
```

---

## Pattern 1: Fast & Slow Pointers

**When to use:** Cycle detection, finding middle node, intersection.

```
Slow:  1 -> 2 -> 3 -> 4 -> 5
Fast:  1 ------> 3 ------> 5
                 ^ middle
```

```java
// Detect cycle -- Time: O(n), Space: O(1)
public static boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}

// Find cycle start -- Time: O(n), Space: O(1)
public static ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            slow = head; // reset slow to head
            while (slow != fast) { slow = slow.next; fast = fast.next; }
            return slow; // cycle start
        }
    }
    return null;
}

// Find middle -- Time: O(n), Space: O(1)
public static ListNode findMiddle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // for even-length list, returns second middle
}
```

> **Key Interview Point:** To find cycle start: after fast/slow meet, reset one pointer to head and advance both by 1. They meet at the cycle entrance. This works because of the mathematical relationship between distances.

---

## Pattern 2: Reversing a Linked List

**Idea:** Walk the list once, flipping each `next` pointer to point backwards. Keep three references (`prev`, `curr`, `next`) so you never lose the rest of the list.

```
Before: 1 -> 2 -> 3 -> null
After:  null <- 1 <- 2 <- 3
```

```java
// Iterative reverse -- Time: O(n), Space: O(1)
public static ListNode reverseList(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next; // save
        curr.next = prev;          // reverse
        prev = curr;               // advance
        curr = next;
    }
    return prev;
}

// Recursive reverse -- Time: O(n), Space: O(n) call stack
public static ListNode reverseListRecursive(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverseListRecursive(head.next);
    head.next.next = head;
    head.next = null;
    return newHead;
}
```

> **Common Mistake:** Forgetting to save `curr.next` before overwriting `curr.next = prev`. This loses the rest of the list.

---

## Pattern 3: Merging Linked Lists

**Idea:** Like the merge step of merge sort: repeatedly attach the smaller of the two current heads to the result, then append whatever remains. For K lists, a min-heap of the K current heads picks the smallest in O(log k).

```java
// Merge two sorted lists -- Time: O(n+m), Space: O(1)
public static ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), curr = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { curr.next = l1; l1 = l1.next; }
        else { curr.next = l2; l2 = l2.next; }
        curr = curr.next;
    }
    curr.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}

// Merge K sorted lists using min-heap -- Time: O(N log k), Space: O(k)
public static ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));
    for (ListNode list : lists) if (list != null) pq.offer(list);

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

## Pattern 4: Dummy Node Technique

**When to use:** Any time the head might change (deletions, insertions).

```java
// Remove Nth from end -- Time: O(n), Space: O(1)
public static ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;

    for (int i = 0; i <= n; i++) fast = fast.next; // advance fast by n+1
    while (fast != null) { fast = fast.next; slow = slow.next; }
    slow.next = slow.next.next; // skip the Nth node
    return dummy.next;
}

// Add two numbers (digits in reverse order)
public static ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), curr = dummy;
    int carry = 0;
    while (l1 != null || l2 != null || carry > 0) {
        int sum = (l1 != null ? l1.val : 0) + (l2 != null ? l2.val : 0) + carry;
        carry = sum / 10;
        curr.next = new ListNode(sum % 10);
        curr = curr.next;
        if (l1 != null) l1 = l1.next;
        if (l2 != null) l2 = l2.next;
    }
    return dummy.next;
}
```

---

## Pattern 5: Linked List + Stack

**Idea:** A stack gives you the list in reverse order, which helps compare from both ends (palindrome) at the cost of O(n) space. The O(1)-space alternative (used in `reorderList`) is: find the middle, reverse the second half, then walk both halves together.

```java
// Check palindrome -- Time: O(n), Space: O(n)
public static boolean isPalindrome(ListNode head) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (ListNode curr = head; curr != null; curr = curr.next) stack.push(curr.val);
    for (ListNode curr = head; curr != null; curr = curr.next)
        if (curr.val != stack.pop()) return false;
    return true;
}

// Reorder list: 1->2->3->4->5 becomes 1->5->2->4->3
public static void reorderList(ListNode head) {
    if (head == null || head.next == null) return;

    // 1. Find middle
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) { slow = slow.next; fast = fast.next.next; }

    // 2. Reverse second half
    ListNode prev = null, curr = slow.next;
    slow.next = null;
    while (curr != null) { ListNode next = curr.next; curr.next = prev; prev = curr; curr = next; }

    // 3. Merge alternating
    ListNode first = head, second = prev;
    while (second != null) {
        ListNode t1 = first.next, t2 = second.next;
        first.next = second; second.next = t1;
        first = t1; second = t2;
    }
}
```

---

## Pattern 6: Copy List with Random Pointer

**Idea:** Insert each copy right after its original (A -> A' -> B -> B'), so `original.random.next` is the copy's random target. Then unweave the two lists. O(n) time, O(1) extra space (a `HashMap` from original to copy is the simpler O(n)-space version).

```java
class RandomListNode {
    int val;
    RandomListNode next, random;
    RandomListNode(int val) { this.val = val; }
}

// O(n) time, O(1) extra space (interleaving approach)
public static RandomListNode copyRandomList(RandomListNode head) {
    if (head == null) return null;

    // Step 1: Interleave copy nodes -- A->A'->B->B'->C->C'
    for (RandomListNode curr = head; curr != null; ) {
        RandomListNode copy = new RandomListNode(curr.val);
        copy.next = curr.next;
        curr.next = copy;
        curr = copy.next;
    }

    // Step 2: Set random pointers for copies
    for (RandomListNode curr = head; curr != null; curr = curr.next.next)
        if (curr.random != null) curr.next.random = curr.random.next;

    // Step 3: Separate the two lists
    RandomListNode copyHead = head.next, copyCurr = copyHead;
    for (RandomListNode curr = head; curr != null; curr = curr.next) {
        curr.next = curr.next.next;
        if (copyCurr.next != null) copyCurr.next = copyCurr.next.next;
        copyCurr = copyCurr.next;
    }
    return copyHead;
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 21 | Merge Two Sorted Lists | Merge + Dummy | Easy |
| 141 | Linked List Cycle | Fast/Slow | Easy |
| 206 | Reverse Linked List | Reverse | Easy |
| 2 | Add Two Numbers | Dummy Node | Medium |
| 19 | Remove Nth From End | Two Pointers + Dummy | Medium |
| 142 | Linked List Cycle II | Fast/Slow | Medium |
| 143 | Reorder List | Middle + Reverse + Merge | Medium |
| 148 | Sort List | Merge Sort | Medium |
| 234 | Palindrome Linked List | Stack or Reverse Half | Easy |
| 23 | Merge K Sorted Lists | Heap | Hard |
| 25 | Reverse Nodes in K-Group | Reverse + Counting | Hard |
| 138 | Copy List with Random Pointer | Interleave or HashMap | Medium |
