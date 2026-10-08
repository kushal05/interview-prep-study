# Stacks & Queues in JavaScript

> JavaScript implementation reference for stacks and queues: array-backed stack, O(1) queue, monotonic stack, parsing, BFS, monotonic deque and stack-based designs.
> New to this topic? Learn it first in [11. Stacks and Queues](../learn/11-stacks-and-queues.md).

## Complexity Overview

| Operation | Stack (Array) | Queue (Array) |
|-----------|--------------|---------------|
| Push / Enqueue (end) | O(1) | O(1) |
| Pop (end) | O(1) | - |
| Dequeue (front/shift) | - | O(n)* |
| Peek | O(1) | O(1) |

*JS `Array.shift()` is O(n). For O(1) dequeue, use index pointer or linked list.

> **Key Interview Point:** JS has no built-in deque. For BFS, `Array.shift()` works but is O(n). In performance-critical code, use an index pointer instead of shifting.

> **Common Mistake:** Using `Array.shift()` in a tight BFS loop creates O(n^2) behavior. Track a front pointer `let i = 0` and access `queue[i++]` instead.

---

## Basic Implementation

```javascript
// Stack using Array
class Stack {
    constructor() {
        this.items = [];
    }

    push(item) {
        this.items.push(item);
    }

    pop() {
        return this.items.pop();
    }

    peek() {
        return this.items[this.items.length - 1];
    }

    isEmpty() {
        return this.items.length === 0;
    }

    size() {
        return this.items.length;
    }
}

// Queue using Array + head pointer (O(1) dequeue; shift() would be O(n))
class Queue {
    constructor() {
        this.items = [];
        this.head = 0;
    }

    enqueue(item) {
        this.items.push(item);
    }

    dequeue() {
        if (this.isEmpty()) return undefined;
        const item = this.items[this.head++];
        // Compact occasionally so consumed slots don't leak memory
        if (this.head > 1024 && this.head * 2 > this.items.length) {
            this.items = this.items.slice(this.head);
            this.head = 0;
        }
        return item;
    }

    peek() {
        return this.items[this.head];
    }

    isEmpty() {
        return this.size() === 0;
    }

    size() {
        return this.items.length - this.head;
    }
}
```

## 🧱 Monotonic Stack Pattern

**Idea:** Keep the stack sorted (e.g. decreasing). When a new element breaks the order, pop everything it beats: the new element is the "next greater" answer for each popped item. Every index is pushed and popped once, so O(n).

```javascript
// Next greater element
function nextGreaterElement(nums1, nums2) {
    const map = new Map();
    const stack = [];

    for (const num of nums2) {
        while (stack.length > 0 && stack[stack.length - 1] < num) {
            map.set(stack.pop(), num);
        }
        stack.push(num);
    }

    return nums1.map(num => map.get(num) ?? -1);
}

// Daily temperatures
function dailyTemperatures(temperatures) {
    const result = new Array(temperatures.length).fill(0);
    const stack = []; // indices

    for (let i = 0; i < temperatures.length; i++) {
        while (stack.length > 0 && temperatures[stack[stack.length - 1]] < temperatures[i]) {
            const prevIndex = stack.pop();
            result[prevIndex] = i - prevIndex;
        }
        stack.push(i);
    }

    return result;
}

// Largest rectangle in histogram
function largestRectangleArea(heights) {
    const stack = []; // indices
    let maxArea = 0;

    for (let i = 0; i < heights.length; i++) {
        while (stack.length > 0 && heights[stack[stack.length - 1]] >= heights[i]) {
            const height = heights[stack.pop()];
            const width = stack.length === 0 ? i : i - stack[stack.length - 1] - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        stack.push(i);
    }

    while (stack.length > 0) {
        const height = heights[stack.pop()];
        const width = stack.length === 0 ? heights.length : heights.length - stack[stack.length - 1] - 1;
        maxArea = Math.max(maxArea, height * width);
    }

    return maxArea;
}
```

## 🔁 Stack for Backtracking / Parsing

**Idea:** Nested structures close in reverse order of opening (last opened, first closed), which is exactly LIFO. Push on an opener, pop and check/combine on a closer.

```javascript
// Valid parentheses
function isValid(s) {
    const stack = [];
    const brackets = {
        ')': '(',
        '}': '{',
        ']': '['
    };

    for (const char of s) {
        if (['(', '{', '['].includes(char)) {
            stack.push(char);
        } else if ([')', '}', ']'].includes(char)) {
            if (stack.length === 0 || stack.pop() !== brackets[char]) {
                return false;
            }
        }
    }

    return stack.length === 0;
}

// Decode string
function decodeString(s) {
    const numStack = [];
    const strStack = [];
    let currentStr = '';
    let currentNum = 0;

    for (const char of s) {
        if (!isNaN(char)) {
            currentNum = currentNum * 10 + parseInt(char);
        } else if (char === '[') {
            numStack.push(currentNum);
            strStack.push(currentStr);
            currentStr = '';
            currentNum = 0;
        } else if (char === ']') {
            const repeatCount = numStack.pop();
            const prevStr = strStack.pop();
            const repeated = currentStr.repeat(repeatCount);
            currentStr = prevStr + repeated;
        } else {
            currentStr += char;
        }
    }

    return currentStr;
}
```

## 🧠 Expression Evaluation with Stack

**Idea:** In postfix (RPN) notation each operator applies to the two most recent values, so push numbers and, on an operator, pop two, compute, and push the result.

```javascript
// Evaluate reverse polish notation
function evalRPN(tokens) {
    const stack = [];

    for (const token of tokens) {
        if (token === '+' || token === '-' || token === '*' || token === '/') {
            const b = stack.pop();
            const a = stack.pop();
            let result;

            switch (token) {
                case '+':
                    result = a + b;
                    break;
                case '-':
                    result = a - b;
                    break;
                case '*':
                    result = a * b;
                    break;
                case '/':
                    result = Math.trunc(a / b);
                    break;
            }
            stack.push(result);
        } else {
            stack.push(parseInt(token));
        }
    }

    return stack[stack.length - 1];
}
```

## 🚪 BFS with Queue Pattern

**Idea:** A FIFO queue explores cells in rings of increasing distance from the start, so the first time BFS reaches a cell is via a shortest path (in unweighted grids). Mark cells visited when you enqueue them, not when you dequeue them.

```javascript
// Number of islands
function numIslands(grid) {
    if (grid.length === 0 || grid[0].length === 0) return 0;

    const rows = grid.length;
    const cols = grid[0].length;
    let count = 0;

    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (grid[i][j] === '1') {
                count++;
                bfs(grid, i, j, rows, cols);
            }
        }
    }

    return count;
}

function bfs(grid, i, j, rows, cols) {
    const queue = [[i, j]];
    let head = 0; // index pointer instead of shift() -> O(1) dequeue
    grid[i][j] = '0';

    const directions = [
        [-1, 0], [1, 0], [0, -1], [0, 1]
    ];

    while (head < queue.length) {
        const [x, y] = queue[head++];

        for (const [dx, dy] of directions) {
            const nx = x + dx;
            const ny = y + dy;

            if (nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] === '1') {
                grid[nx][ny] = '0';
                queue.push([nx, ny]);
            }
        }
    }
}

// Shortest path in binary matrix
function shortestPathBinaryMatrix(grid) {
    const n = grid.length;
    if (grid[0][0] === 1 || grid[n-1][n-1] === 1) return -1;

    const queue = [[0, 0, 1]]; // x, y, distance
    let head = 0;
    grid[0][0] = 1; // mark as visited

    const directions = [
        [-1, -1], [-1, 0], [-1, 1],
        [0, -1], [0, 1],
        [1, -1], [1, 0], [1, 1]
    ];

    while (head < queue.length) {
        const [x, y, dist] = queue[head++];

        if (x === n-1 && y === n-1) return dist;

        for (const [dx, dy] of directions) {
            const nx = x + dx;
            const ny = y + dy;

            if (nx >= 0 && nx < n && ny >= 0 && ny < n && grid[nx][ny] === 0) {
                grid[nx][ny] = 1; // mark as visited
                queue.push([nx, ny, dist + 1]);
            }
        }
    }

    return -1;
}
```

## 🧮 Sliding Window with Deque

**Idea:** Keep indices in a deque whose values are decreasing. The front is always the window max; drop it when it slides out, and pop smaller values from the back since they can never be a max again. O(n) total.

```javascript
// Sliding window maximum
function maxSlidingWindow(nums, k) {
    if (nums.length === 0) return [];

    const result = [];
    const deque = []; // indices, values decreasing; live part is deque[head..]
    let head = 0;     // front pointer so "pop front" is O(1) (no shift())

    for (let i = 0; i < nums.length; i++) {
        // Remove elements outside the window
        while (head < deque.length && deque[head] < i - k + 1) {
            head++;
        }

        // Remove elements smaller than current
        while (head < deque.length && nums[deque[deque.length - 1]] < nums[i]) {
            deque.pop();
        }

        deque.push(i);

        // Add to result when window is complete
        if (i >= k - 1) {
            result.push(nums[deque[head]]);
        }
    }

    return result;
}
```

## 🌊 Multi-Stack Simulations

**Idea:** A second stack can carry extra info alongside the first (the running minimum), or two stacks can simulate a queue: pour `input` into `output` only when `output` is empty, giving amortized O(1) per operation.

```javascript
// Min Stack
class MinStack {
    constructor() {
        this.stack = [];
        this.minStack = [];
    }

    push(val) {
        this.stack.push(val);
        const min = this.minStack.length === 0 ? val : Math.min(this.minStack[this.minStack.length - 1], val);
        this.minStack.push(min);
    }

    pop() {
        this.stack.pop();
        this.minStack.pop();
    }

    top() {
        return this.stack[this.stack.length - 1];
    }

    getMin() {
        return this.minStack[this.minStack.length - 1];
    }
}

// Implement queue using stacks
class MyQueue {
    constructor() {
        this.input = [];
        this.output = [];
    }

    push(x) {
        this.input.push(x);
    }

    pop() {
        if (this.output.length === 0) {
            while (this.input.length > 0) {
                this.output.push(this.input.pop());
            }
        }
        return this.output.pop();
    }

    peek() {
        if (this.output.length === 0) {
            while (this.input.length > 0) {
                this.output.push(this.input.pop());
            }
        }
        return this.output[this.output.length - 1];
    }

    empty() {
        return this.input.length === 0 && this.output.length === 0;
    }
}
```
