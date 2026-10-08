# Stacks & Queues in Kotlin

> Kotlin implementation reference for stacks and queues (`ArrayDeque`): monotonic stack, parsing, expression evaluation, BFS, monotonic deque and stack-based designs.
> New to this topic? Learn it first in [11. Stacks and Queues](../learn/11-stacks-and-queues.md).

## Complexity Overview

| Operation | ArrayDeque |
|-----------|-----------|
| addLast / removeLast (stack) | O(1) |
| addLast / removeFirst (queue) | O(1) |
| Peek (first/last) | O(1) |

> **Key Interview Point:** Kotlin's `ArrayDeque` works efficiently as both stack and queue. Use `addLast()/removeLast()` for stack, `addLast()/removeFirst()` for queue.

---

## Basic Implementation

```kotlin
// Stack using ArrayDeque
class Stack<T> {
    private val deque = ArrayDeque<T>()

    fun push(item: T) {
        deque.addLast(item)
    }

    fun pop(): T? {
        return deque.removeLastOrNull()
    }

    fun peek(): T? {
        return deque.lastOrNull()
    }

    fun isEmpty(): Boolean {
        return deque.isEmpty()
    }

    fun size(): Int {
        return deque.size
    }
}

// Queue using ArrayDeque
class Queue<T> {
    private val deque = ArrayDeque<T>()

    fun enqueue(item: T) {
        deque.addLast(item)
    }

    fun dequeue(): T? {
        return deque.removeFirstOrNull()
    }

    fun peek(): T? {
        return deque.firstOrNull()
    }

    fun isEmpty(): Boolean {
        return deque.isEmpty()
    }

    fun size(): Int {
        return deque.size
    }
}
```

## 🧱 Monotonic Stack Pattern

**Idea:** Keep the stack in sorted order. When the current element breaks that order, pop: each popped element has just found its "next greater" (or, in the histogram, its right boundary). Every element is pushed and popped once, so it is O(n) despite the inner `while`.

```kotlin
// Next greater element
fun nextGreaterElement(nums1: IntArray, nums2: IntArray): IntArray {
    val map = mutableMapOf<Int, Int>()
    val stack = ArrayDeque<Int>()

    for (num in nums2) {
        while (stack.isNotEmpty() && stack.last() < num) {
            map[stack.removeLast()] = num
        }
        stack.addLast(num)
    }

    return nums1.map { map.getOrDefault(it, -1) }.toIntArray()
}

// Daily temperatures
fun dailyTemperatures(temperatures: IntArray): IntArray {
    val result = IntArray(temperatures.size)
    val stack = ArrayDeque<Int>() // indices

    for (i in temperatures.indices) {
        while (stack.isNotEmpty() && temperatures[stack.last()] < temperatures[i]) {
            val prevIndex = stack.removeLast()
            result[prevIndex] = i - prevIndex
        }
        stack.addLast(i)
    }

    return result
}

// Largest rectangle in histogram
fun largestRectangleArea(heights: IntArray): Int {
    val stack = ArrayDeque<Int>() // indices
    var maxArea = 0

    for (i in heights.indices) {
        while (stack.isNotEmpty() && heights[stack.last()] >= heights[i]) {
            val height = heights[stack.removeLast()]
            val width = if (stack.isEmpty()) i else i - stack.last() - 1
            maxArea = maxOf(maxArea, height * width)
        }
        stack.addLast(i)
    }

    while (stack.isNotEmpty()) {
        val height = heights[stack.removeLast()]
        val width = if (stack.isEmpty()) heights.size else heights.size - stack.last() - 1
        maxArea = maxOf(maxArea, height * width)
    }

    return maxArea
}
```

## 🔁 Stack for Backtracking / Parsing

**Idea:** Push openers (or the context you are leaving, like the partial string and repeat count) and pop when the matching closer arrives. The most recently opened thing is always closed first, which is exactly LIFO.

```kotlin
// Valid parentheses
fun isValid(s: String): Boolean {
    val stack = ArrayDeque<Char>()
    val brackets = mapOf(')' to '(', '}' to '{', ']' to '[')

    for (char in s) {
        when (char) {
            '(', '{', '[' -> stack.addLast(char)
            ')', '}', ']' -> {
                if (stack.isEmpty() || stack.removeLast() != brackets[char]) {
                    return false
                }
            }
        }
    }

    return stack.isEmpty()
}

// Decode string
fun decodeString(s: String): String {
    val numStack = ArrayDeque<Int>()
    val strStack = ArrayDeque<StringBuilder>()
    var currentStr = StringBuilder()
    var currentNum = 0

    for (char in s) {
        when {
            char.isDigit() -> {
                currentNum = currentNum * 10 + (char - '0')
            }
            char == '[' -> {
                numStack.addLast(currentNum)
                strStack.addLast(currentStr)
                currentStr = StringBuilder()
                currentNum = 0
            }
            char == ']' -> {
                val repeatCount = numStack.removeLast()
                val prevStr = strStack.removeLast()
                val repeated = currentStr.toString().repeat(repeatCount)
                prevStr.append(repeated)
                currentStr = prevStr
            }
            else -> {
                currentStr.append(char)
            }
        }
    }

    return currentStr.toString()
}
```

## 🧠 Expression Evaluation with Stack

**Idea:** Push numbers; on an operator, pop the top two values (the second pop is the left operand), apply it, and push the result. The final stack top is the answer.

```kotlin
// Evaluate reverse polish notation
fun evalRPN(tokens: Array<String>): Int {
    val stack = ArrayDeque<Int>()

    for (token in tokens) {
        when (token) {
            "+", "-", "*", "/" -> {
                val b = stack.removeLast()
                val a = stack.removeLast()
                val result = when (token) {
                    "+" -> a + b
                    "-" -> a - b
                    "*" -> a * b
                    "/" -> a / b
                    else -> 0
                }
                stack.addLast(result)
            }
            else -> {
                stack.addLast(token.toInt())
            }
        }
    }

    return stack.last()
}
```

## 🚪 BFS with Queue Pattern

**Idea:** A FIFO queue visits cells in order of distance from the start, so the first time BFS reaches the target is via a shortest path. Mark cells visited when you enqueue them, not when you dequeue, to avoid adding duplicates.

```kotlin
// Number of islands
fun numIslands(grid: Array<CharArray>): Int {
    if (grid.isEmpty() || grid[0].isEmpty()) return 0

    val rows = grid.size
    val cols = grid[0].size
    var count = 0

    for (i in 0 until rows) {
        for (j in 0 until cols) {
            if (grid[i][j] == '1') {
                count++
                bfs(grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private fun bfs(grid: Array<CharArray>, i: Int, j: Int, rows: Int, cols: Int) {
    val queue = ArrayDeque<Pair<Int, Int>>()
    queue.addLast(Pair(i, j))
    grid[i][j] = '0'

    val directions = arrayOf(
        intArrayOf(-1, 0), intArrayOf(1, 0),
        intArrayOf(0, -1), intArrayOf(0, 1)
    )

    while (queue.isNotEmpty()) {
        val (x, y) = queue.removeFirst()

        for (dir in directions) {
            val nx = x + dir[0]
            val ny = y + dir[1]

            if (nx in 0 until rows && ny in 0 until cols && grid[nx][ny] == '1') {
                grid[nx][ny] = '0'
                queue.addLast(Pair(nx, ny))
            }
        }
    }
}

// Shortest path in binary matrix
fun shortestPathBinaryMatrix(grid: Array<IntArray>): Int {
    val n = grid.size
    if (grid[0][0] == 1 || grid[n-1][n-1] == 1) return -1

    val queue = ArrayDeque<Triple<Int, Int, Int>>() // x, y, distance
    queue.addLast(Triple(0, 0, 1))
    grid[0][0] = 1 // mark as visited

    val directions = arrayOf(
        intArrayOf(-1, -1), intArrayOf(-1, 0), intArrayOf(-1, 1),
        intArrayOf(0, -1), intArrayOf(0, 1),
        intArrayOf(1, -1), intArrayOf(1, 0), intArrayOf(1, 1)
    )

    while (queue.isNotEmpty()) {
        val (x, y, dist) = queue.removeFirst()

        if (x == n-1 && y == n-1) return dist

        for (dir in directions) {
            val nx = x + dir[0]
            val ny = y + dir[1]

            if (nx in 0 until n && ny in 0 until n && grid[nx][ny] == 0) {
                grid[nx][ny] = 1 // mark as visited
                queue.addLast(Triple(nx, ny, dist + 1))
            }
        }
    }

    return -1
}
```

## 🧮 Sliding Window with Deque

**Idea:** Keep indices in the deque with their values decreasing from front to back. Drop the front when it leaves the window and drop smaller values from the back (they can never be a future max). The front is always the window maximum.

```kotlin
// Sliding window maximum
fun maxSlidingWindow(nums: IntArray, k: Int): IntArray {
    if (nums.isEmpty()) return intArrayOf()

    val result = mutableListOf<Int>()
    val deque = ArrayDeque<Int>() // indices

    for (i in nums.indices) {
        // Remove elements outside the window
        while (deque.isNotEmpty() && deque.first() < i - k + 1) {
            deque.removeFirst()
        }

        // Remove elements smaller than current
        while (deque.isNotEmpty() && nums[deque.last()] < nums[i]) {
            deque.removeLast()
        }

        deque.addLast(i)

        // Add to result when window is complete
        if (i >= k - 1) {
            result.add(nums[deque.first()])
        }
    }

    return result.toIntArray()
}
```

## 🌊 Multi-Stack Simulations

**Idea:** A second stack stores extra per-level information (the minimum so far) or reverses order (two stacks make a queue: pour `input` into `output` only when `output` is empty, so each element moves once, amortized O(1)).

```kotlin
// Min Stack
class MinStack() {
    private val stack = ArrayDeque<Int>()
    private val minStack = ArrayDeque<Int>()

    fun push(`val`: Int) {
        stack.addLast(`val`)
        val min = if (minStack.isEmpty()) `val` else minOf(minStack.last(), `val`)
        minStack.addLast(min)
    }

    fun pop() {
        stack.removeLast()
        minStack.removeLast()
    }

    fun top(): Int {
        return stack.last()
    }

    fun getMin(): Int {
        return minStack.last()
    }
}

// Implement queue using stacks
class MyQueue() {
    private val input = ArrayDeque<Int>()
    private val output = ArrayDeque<Int>()

    fun push(x: Int) {
        input.addLast(x)
    }

    fun pop(): Int {
        if (output.isEmpty()) {
            while (input.isNotEmpty()) {
                output.addLast(input.removeLast())
            }
        }
        return output.removeLast()
    }

    fun peek(): Int {
        if (output.isEmpty()) {
            while (input.isNotEmpty()) {
                output.addLast(input.removeLast())
            }
        }
        return output.last()
    }

    fun empty(): Boolean {
        return input.isEmpty() && output.isEmpty()
    }
}
```
