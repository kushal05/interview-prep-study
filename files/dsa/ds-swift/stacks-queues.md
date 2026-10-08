# Stacks & Queues in Swift

> Swift implementation reference for stacks and queues (arrays): monotonic stack, parsing, expression evaluation, BFS, monotonic deque and stack-based designs.
> New to this topic? Learn it first in [11. Stacks and Queues](../learn/11-stacks-and-queues.md).

## Complexity Overview

| Operation | Array-based Stack | Array-based Queue |
|-----------|------------------|------------------|
| Push / Enqueue (append) | O(1) | O(1) |
| Pop (removeLast) | O(1) | - |
| Dequeue (removeFirst) | - | O(n)* |
| Peek | O(1) | O(1) |

*`Array.removeFirst()` is O(n) in Swift. For O(1) dequeue, use a head index (as in `Queue` below).

> **Key Interview Point:** Swift has no built-in deque (only the separate `swift-collections` package has one). `Array.removeFirst()` is O(n), which turns a BFS into O(n^2) on large inputs. The code below keeps a `head` index into the array instead, so each dequeue is O(1).

---

## Basic Implementation

```swift
// Stack using Array
struct Stack<Element> {
    private var elements: [Element] = []

    mutating func push(_ element: Element) {
        elements.append(element)
    }

    mutating func pop() -> Element? {
        return elements.popLast()
    }

    func peek() -> Element? {
        return elements.last
    }

    func isEmpty() -> Bool {
        return elements.isEmpty
    }

    func size() -> Int {
        return elements.count
    }
}

// Queue using Array + head index (O(1) dequeue; removeFirst() would be O(n))
struct Queue<Element> {
    private var elements: [Element] = []
    private var head = 0 // index of the current front

    mutating func enqueue(_ element: Element) {
        elements.append(element)
    }

    mutating func dequeue() -> Element? {
        guard head < elements.count else { return nil }
        let element = elements[head]
        head += 1
        // Occasionally drop the consumed prefix so memory does not grow forever
        if head > 32 && head * 2 > elements.count {
            elements.removeFirst(head)
            head = 0
        }
        return element
    }

    func peek() -> Element? {
        return head < elements.count ? elements[head] : nil
    }

    func isEmpty() -> Bool {
        return head >= elements.count
    }

    func size() -> Int {
        return elements.count - head
    }
}
```

## 🧱 Monotonic Stack Pattern

**Idea:** Keep the stack in sorted order. When the current element breaks that order, pop: each popped element has just found its "next greater" (or, in the histogram, its right boundary). Every element is pushed and popped once, so it is O(n) despite the inner `while`.

```swift
// Next greater element
func nextGreaterElement(_ nums1: [Int], _ nums2: [Int]) -> [Int] {
    var map = [Int: Int]()
    var stack = [Int]()

    for num in nums2 {
        while !stack.isEmpty && stack.last! < num {
            map[stack.removeLast()] = num
        }
        stack.append(num)
    }

    return nums1.map { map[$0] ?? -1 }
}

// Daily temperatures
func dailyTemperatures(_ temperatures: [Int]) -> [Int] {
    var result = [Int](repeating: 0, count: temperatures.count)
    var stack = [Int]() // indices

    for i in 0..<temperatures.count {
        while !stack.isEmpty && temperatures[stack.last!] < temperatures[i] {
            let prevIndex = stack.removeLast()
            result[prevIndex] = i - prevIndex
        }
        stack.append(i)
    }

    return result
}

// Largest rectangle in histogram
func largestRectangleArea(_ heights: [Int]) -> Int {
    var stack = [Int]() // indices
    var maxArea = 0

    for i in 0..<heights.count {
        while !stack.isEmpty && heights[stack.last!] >= heights[i] {
            let height = heights[stack.removeLast()]
            let width = stack.isEmpty ? i : i - stack.last! - 1
            maxArea = max(maxArea, height * width)
        }
        stack.append(i)
    }

    while !stack.isEmpty {
        let height = heights[stack.removeLast()]
        let width = stack.isEmpty ? heights.count : heights.count - stack.last! - 1
        maxArea = max(maxArea, height * width)
    }

    return maxArea
}
```

## 🔁 Stack for Backtracking / Parsing

**Idea:** Push openers (or the context you are leaving, like the partial string and repeat count) and pop when the matching closer arrives. The most recently opened thing is always closed first, which is exactly LIFO.

```swift
// Valid parentheses
func isValid(_ s: String) -> Bool {
    var stack = [Character]()
    let brackets: [Character: Character] = [")": "(", "}": "{", "]": "["]

    for char in s {
        switch char {
        case "(", "{", "[":
            stack.append(char)
        case ")", "}", "]":
            if stack.isEmpty || stack.removeLast() != brackets[char] {
                return false
            }
        default:
            break
        }
    }

    return stack.isEmpty
}

// Decode string
func decodeString(_ s: String) -> String {
    var numStack = [Int]()
    var strStack = [String]()
    var currentStr = ""
    var currentNum = 0

    for char in s {
        if let digit = char.wholeNumberValue { // isNumber + Int(...)! would crash on e.g. "½"
            currentNum = currentNum * 10 + digit
        } else if char == "[" {
            numStack.append(currentNum)
            strStack.append(currentStr)
            currentStr = ""
            currentNum = 0
        } else if char == "]" {
            let repeatCount = numStack.removeLast()
            let prevStr = strStack.removeLast()
            let repeated = String(repeating: currentStr, count: repeatCount)
            currentStr = prevStr + repeated
        } else {
            currentStr.append(char)
        }
    }

    return currentStr
}
```

## 🧠 Expression Evaluation with Stack

**Idea:** Push numbers; on an operator, pop the top two values (the second pop is the left operand), apply it, and push the result. The final stack top is the answer.

```swift
// Evaluate reverse polish notation
func evalRPN(_ tokens: [String]) -> Int {
    var stack = [Int]()

    for token in tokens {
        switch token {
        case "+", "-", "*", "/":
            let b = stack.removeLast()
            let a = stack.removeLast()
            var result: Int

            switch token {
            case "+":
                result = a + b
            case "-":
                result = a - b
            case "*":
                result = a * b
            case "/":
                result = a / b
            default:
                result = 0
            }
            stack.append(result)
        default:
            if let num = Int(token) {
                stack.append(num)
            }
        }
    }

    return stack.last ?? 0
}
```

## 🚪 BFS with Queue Pattern

**Idea:** A FIFO queue visits cells in order of distance from the start, so the first time BFS reaches the target is via a shortest path. Mark cells visited when you enqueue them, not when you dequeue, to avoid adding duplicates.

```swift
// Number of islands
func numIslands(_ grid: [[Character]]) -> Int {
    guard !grid.isEmpty, !grid[0].isEmpty else { return 0 }

    let rows = grid.count
    let cols = grid[0].count
    var grid = grid
    var count = 0

    for i in 0..<rows {
        for j in 0..<cols {
            if grid[i][j] == "1" {
                count += 1
                bfs(&grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private func bfs(_ grid: inout [[Character]], _ i: Int, _ j: Int, _ rows: Int, _ cols: Int) {
    var queue = [(Int, Int)]()
    var head = 0 // dequeue by index: removeFirst() is O(n)
    queue.append((i, j))
    grid[i][j] = "0"

    let directions = [(-1, 0), (1, 0), (0, -1), (0, 1)]

    while head < queue.count {
        let (x, y) = queue[head]
        head += 1

        for dir in directions {
            let nx = x + dir.0
            let ny = y + dir.1

            if nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] == "1" {
                grid[nx][ny] = "0"
                queue.append((nx, ny))
            }
        }
    }
}

// Shortest path in binary matrix
func shortestPathBinaryMatrix(_ grid: [[Int]]) -> Int {
    let n = grid.count
    guard n > 0, grid[0][0] == 0, grid[n-1][n-1] == 0 else { return -1 }

    var grid = grid
    var queue = [(Int, Int, Int)]() // x, y, distance
    var head = 0
    queue.append((0, 0, 1))
    grid[0][0] = 1 // mark as visited

    let directions = [
        (-1, -1), (-1, 0), (-1, 1),
        (0, -1), (0, 1),
        (1, -1), (1, 0), (1, 1)
    ]

    while head < queue.count {
        let (x, y, dist) = queue[head]
        head += 1

        if x == n-1 && y == n-1 {
            return dist
        }

        for dir in directions {
            let nx = x + dir.0
            let ny = y + dir.1

            if nx >= 0 && nx < n && ny >= 0 && ny < n && grid[nx][ny] == 0 {
                grid[nx][ny] = 1 // mark as visited
                queue.append((nx, ny, dist + 1))
            }
        }
    }

    return -1
}
```

## 🧮 Sliding Window with Deque

**Idea:** Keep indices with their values decreasing from front to back. Drop the front when it leaves the window and drop smaller values from the back (they can never be a future max). The front is always the window maximum. A `head` index stands in for `removeFirst()` so the whole thing stays O(n).

```swift
// Sliding window maximum -- O(n)
func maxSlidingWindow(_ nums: [Int], _ k: Int) -> [Int] {
    guard !nums.isEmpty else { return [] }

    var result = [Int]()
    var deque = [Int]() // indices; the live deque is deque[head...]
    var head = 0

    for i in 0..<nums.count {
        // Remove elements outside the window (from the front)
        while head < deque.count && deque[head] < i - k + 1 {
            head += 1
        }

        // Remove elements smaller than current (from the back)
        while deque.count > head && nums[deque.last!] < nums[i] {
            deque.removeLast()
        }

        deque.append(i)

        // Add to result when window is complete
        if i >= k - 1 {
            result.append(nums[deque[head]])
        }
    }

    return result
}
```

## 🌊 Multi-Stack Simulations

**Idea:** A second stack stores extra per-level information (the minimum so far) or reverses order (two stacks make a queue: pour `input` into `output` only when `output` is empty, so each element moves once, amortized O(1)).

```swift
// Min Stack
class MinStack {
    private var stack: [Int]
    private var minStack: [Int]

    init() {
        stack = []
        minStack = []
    }

    func push(_ val: Int) {
        stack.append(val)
        let min = minStack.isEmpty ? val : Swift.min(minStack.last!, val)
        minStack.append(min)
    }

    func pop() {
        stack.removeLast()
        minStack.removeLast()
    }

    func top() -> Int {
        return stack.last ?? 0
    }

    func getMin() -> Int {
        return minStack.last ?? 0
    }
}

// Implement queue using stacks
class MyQueue {
    private var input: [Int]
    private var output: [Int]

    init() {
        input = []
        output = []
    }

    func push(_ x: Int) {
        input.append(x)
    }

    func pop() -> Int {
        if output.isEmpty {
            while !input.isEmpty {
                output.append(input.removeLast())
            }
        }
        return output.removeLast()
    }

    func peek() -> Int {
        if output.isEmpty {
            while !input.isEmpty {
                output.append(input.removeLast())
            }
        }
        return output.last ?? 0
    }

    func empty() -> Bool {
        return input.isEmpty && output.isEmpty
    }
}
```

---

## Summary: Stack & Queue Pattern Map

| Pattern                  | Keywords / Use Case                          |
|------------------------|--------------------------------------------|
| Monotonic Stack          | next greater/smaller, spans, temperatures    |
| Stack Backtracking       | parentheses, nested strings, undo            |
| Stack Expression Eval    | calculator, infix/postfix, evaluate          |
| BFS with Queue           | shortest path, level-order, graph traversal  |
| Queue for Sliding Window | max/min in window, sliding window efficiency |
| Multi-Stack Simulation   | min stack, queue via stack, custom design    |

---