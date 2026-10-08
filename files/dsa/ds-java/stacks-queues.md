# Stacks & Queues in Java

> Java implementation reference for stacks and queues (`ArrayDeque`): monotonic stack, parsing, expression evaluation, BFS, monotonic deque and stack-based designs.
> New to this topic? Learn it first in [11. Stacks and Queues](../learn/11-stacks-and-queues.md).

## Complexity Overview

| Operation | Stack (ArrayDeque) | Queue (ArrayDeque) |
|-----------|-------------------|-------------------|
| Push / Enqueue | O(1) | O(1) |
| Pop / Dequeue | O(1) | O(1) |
| Peek | O(1) | O(1) |
| Search | O(n) | O(n) |

> **Key Interview Point:** In Java, prefer `ArrayDeque` over `Stack` (legacy, synchronized) and `LinkedList` (more overhead). `ArrayDeque` is faster for both stack and queue operations.

---

## Basic Implementation

```java
import java.util.*;

// Stack: use ArrayDeque, call addLast/removeLast/getLast
Deque<Integer> stack = new ArrayDeque<>();
stack.addLast(1);          // push
stack.addLast(2);          // push
int top = stack.removeLast(); // pop -> 2
int peek = stack.getLast();   // peek -> 1

// Queue: use ArrayDeque, call addLast/removeFirst/getFirst
Deque<Integer> queue = new ArrayDeque<>();
queue.addLast(1);             // enqueue
queue.addLast(2);             // enqueue
int front = queue.removeFirst(); // dequeue -> 1
int peek2 = queue.getFirst();    // peek -> 2
```

---

## Pattern 1: Monotonic Stack

**When to use:** Next/previous greater or smaller element.

**How it works:** Maintain stack in sorted order. Pop when current element breaks the order.

```java
// Next greater element -- Time: O(n), Space: O(n)
public static int[] nextGreaterElement(int[] nums1, int[] nums2) {
    Map<Integer, Integer> map = new HashMap<>();
    Deque<Integer> stack = new ArrayDeque<>();

    for (int num : nums2) {
        while (!stack.isEmpty() && stack.getLast() < num)
            map.put(stack.removeLast(), num); // num is next greater for popped element
        stack.addLast(num);
    }
    return Arrays.stream(nums1).map(n -> map.getOrDefault(n, -1)).toArray();
}

// Daily temperatures -- Time: O(n), Space: O(n)
public static int[] dailyTemperatures(int[] temperatures) {
    int[] result = new int[temperatures.length];
    Deque<Integer> stack = new ArrayDeque<>(); // stores indices

    for (int i = 0; i < temperatures.length; i++) {
        while (!stack.isEmpty() && temperatures[stack.getLast()] < temperatures[i]) {
            int prevIdx = stack.removeLast();
            result[prevIdx] = i - prevIdx;
        }
        stack.addLast(i);
    }
    return result;
}

// Largest rectangle in histogram -- Time: O(n), Space: O(n)
public static int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int maxArea = 0;

    for (int i = 0; i < heights.length; i++) {
        while (!stack.isEmpty() && heights[stack.getLast()] >= heights[i]) {
            int h = heights[stack.removeLast()];
            int w = stack.isEmpty() ? i : i - stack.getLast() - 1;
            maxArea = Math.max(maxArea, h * w);
        }
        stack.addLast(i);
    }
    while (!stack.isEmpty()) {
        int h = heights[stack.removeLast()];
        int w = stack.isEmpty() ? heights.length : heights.length - stack.getLast() - 1;
        maxArea = Math.max(maxArea, h * w);
    }
    return maxArea;
}
```

> **Key Interview Point:** Monotonic stack processes each element at most twice (push + pop), so it is O(n) despite the nested while loop.

---

## Pattern 2: Stack for Parsing

**Idea:** Push openers (or the context you are leaving, like the partial string and repeat count) and pop when the matching closer arrives. The most recently opened thing is always closed first, which is exactly LIFO.

```java
// Valid parentheses -- Time: O(n), Space: O(n)
public static boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', '}', '{', ']', '[');

    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) stack.addLast(c);
        else if (pairs.containsKey(c)) {
            // .equals, not != : both sides are boxed Character objects
            if (stack.isEmpty() || !stack.removeLast().equals(pairs.get(c))) return false;
        }
    }
    return stack.isEmpty();
}

// Decode string "3[a2[c]]" -> "accaccacc"
public static String decodeString(String s) {
    Deque<Integer> numStack = new ArrayDeque<>();
    Deque<StringBuilder> strStack = new ArrayDeque<>();
    StringBuilder curr = new StringBuilder();
    int num = 0;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            num = num * 10 + (c - '0');
        } else if (c == '[') {
            numStack.addLast(num);
            strStack.addLast(curr);
            curr = new StringBuilder();
            num = 0;
        } else if (c == ']') {
            StringBuilder prev = strStack.removeLast();
            prev.append(curr.toString().repeat(numStack.removeLast()));
            curr = prev;
        } else {
            curr.append(c);
        }
    }
    return curr.toString();
}
```

---

## Pattern 3: Expression Evaluation

**Idea:** Push numbers; on an operator, pop the top two values (the second pop is the left operand), apply it, and push the result. The final stack top is the answer.

```java
// Evaluate reverse polish notation -- Time: O(n), Space: O(n)
public static int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (String token : tokens) {
        switch (token) {
            case "+" -> { int b = stack.removeLast(), a = stack.removeLast(); stack.addLast(a + b); }
            case "-" -> { int b = stack.removeLast(), a = stack.removeLast(); stack.addLast(a - b); }
            case "*" -> { int b = stack.removeLast(), a = stack.removeLast(); stack.addLast(a * b); }
            case "/" -> { int b = stack.removeLast(), a = stack.removeLast(); stack.addLast(a / b); }
            default -> stack.addLast(Integer.parseInt(token));
        }
    }
    return stack.getLast();
}
```

---

## Pattern 4: BFS with Queue

**Idea:** A FIFO queue visits cells in order of distance from the start, so the first time BFS reaches the target is via a shortest path. Mark cells visited when you enqueue them, not when you dequeue, to avoid adding duplicates.

```java
// Number of islands -- Time: O(m*n), Space: O(min(m,n))
public static int numIslands(char[][] grid) {
    int rows = grid.length, cols = grid[0].length, count = 0;
    int[][] dirs = {{-1,0},{1,0},{0,-1},{0,1}};

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == '1') {
                count++;
                grid[i][j] = '0';
                Deque<int[]> queue = new ArrayDeque<>();
                queue.addLast(new int[]{i, j});
                while (!queue.isEmpty()) {
                    int[] pos = queue.removeFirst();
                    for (int[] d : dirs) {
                        int nx = pos[0]+d[0], ny = pos[1]+d[1];
                        if (nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] == '1') {
                            grid[nx][ny] = '0';
                            queue.addLast(new int[]{nx, ny});
                        }
                    }
                }
            }
        }
    }
    return count;
}

// Shortest path in binary matrix -- Time: O(n^2), Space: O(n^2)
public static int shortestPathBinaryMatrix(int[][] grid) {
    int n = grid.length;
    if (grid[0][0] == 1 || grid[n-1][n-1] == 1) return -1;

    int[][] dirs = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};
    Deque<int[]> queue = new ArrayDeque<>();
    queue.addLast(new int[]{0, 0, 1});
    grid[0][0] = 1;

    while (!queue.isEmpty()) {
        int[] pos = queue.removeFirst();
        if (pos[0] == n-1 && pos[1] == n-1) return pos[2];
        for (int[] d : dirs) {
            int nx = pos[0]+d[0], ny = pos[1]+d[1];
            if (nx >= 0 && nx < n && ny >= 0 && ny < n && grid[nx][ny] == 0) {
                grid[nx][ny] = 1;
                queue.addLast(new int[]{nx, ny, pos[2]+1});
            }
        }
    }
    return -1;
}
```

---

## Pattern 5: Sliding Window with Deque

**Idea:** Keep indices in the deque with their values decreasing from front to back. Drop the front when it leaves the window and drop smaller values from the back (they can never be a future max). The front is always the window maximum.

```java
// Sliding window maximum -- Time: O(n), Space: O(k)
public static int[] maxSlidingWindow(int[] nums, int k) {
    List<Integer> result = new ArrayList<>();
    Deque<Integer> deque = new ArrayDeque<>(); // stores indices, front = max

    for (int i = 0; i < nums.length; i++) {
        while (!deque.isEmpty() && deque.getFirst() < i - k + 1) deque.removeFirst(); // out of window
        while (!deque.isEmpty() && nums[deque.getLast()] < nums[i]) deque.removeLast(); // remove smaller
        deque.addLast(i);
        if (i >= k - 1) result.add(nums[deque.getFirst()]);
    }
    return result.stream().mapToInt(Integer::intValue).toArray();
}
```

---

## Pattern 6: Multi-Stack Simulations

**Idea:** A second stack stores extra per-level information (the minimum so far) or reverses order (two stacks make a queue: pour `in` into `out` only when `out` is empty, so each element moves once).

```java
// Min Stack -- all operations O(1)
class MinStack {
    private Deque<Integer> stack = new ArrayDeque<>();
    private Deque<Integer> minStack = new ArrayDeque<>();

    public void push(int val) {
        stack.addLast(val);
        minStack.addLast(minStack.isEmpty() ? val : Math.min(minStack.getLast(), val));
    }
    public void pop()       { stack.removeLast(); minStack.removeLast(); }
    public int top()        { return stack.getLast(); }
    public int getMin()     { return minStack.getLast(); }
}

// Queue using two stacks -- amortized O(1) per operation
class MyQueue {
    private Deque<Integer> in = new ArrayDeque<>(), out = new ArrayDeque<>();

    public void push(int x) { in.addLast(x); }
    public int pop()        { transfer(); return out.removeLast(); }
    public int peek()       { transfer(); return out.getLast(); }
    public boolean empty()  { return in.isEmpty() && out.isEmpty(); }
    private void transfer() { if (out.isEmpty()) while (!in.isEmpty()) out.addLast(in.removeLast()); }
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 20 | Valid Parentheses | Stack Parsing | Easy |
| 155 | Min Stack | Multi-Stack | Medium |
| 232 | Queue Using Stacks | Multi-Stack | Easy |
| 150 | Evaluate RPN | Expression Stack | Medium |
| 394 | Decode String | Stack Parsing | Medium |
| 496 | Next Greater Element I | Monotonic Stack | Easy |
| 739 | Daily Temperatures | Monotonic Stack | Medium |
| 239 | Sliding Window Maximum | Deque | Hard |
| 84 | Largest Rectangle in Histogram | Monotonic Stack | Hard |
| 200 | Number of Islands | BFS/DFS | Medium |
