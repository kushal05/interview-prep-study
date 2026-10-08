# Trees & Graphs in Swift

> Swift implementation reference for binary trees, BSTs and graphs: DFS/BFS, iterative traversals, serialization, topological sort, union-find and flood fill.
> New to this topic? Learn it first in [12. Trees and Traversals](../learn/12-trees-and-traversals.md) and [16. Graphs Fundamentals](../learn/16-graphs-fundamentals.md).

## Complexity Overview

| Tree Operation | BST Average | BST Worst |
|---------------|-------------|-----------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Traversal | O(n) | O(n) |

| Graph Operation | Adjacency List |
|----------------|---------------|
| BFS/DFS | O(V + E) |
| Space | O(V + E) |

> **Key Interview Point:** Swift's `guard let` and optional chaining (`?.`) make tree code elegant. Use `if let` for unwrapping in BFS queue operations.

> **Swift Queue Gotcha:** `Array.removeFirst()` is O(n), which turns a BFS into O(n^2). The BFS code below dequeues with a `head` index (`queue[head]; head += 1`) instead, which is O(1).

---

## Tree Node & BST

```swift
// Binary Tree Node
class TreeNode {
    var val: Int
    var left: TreeNode?
    var right: TreeNode?

    init(_ val: Int) {
        self.val = val
        self.left = nil
        self.right = nil
    }
}

// Binary Search Tree
class BST {
    var root: TreeNode?

    func insert(_ value: Int) {
        root = insertRec(root, value)
    }

    private func insertRec(_ node: TreeNode?, _ value: Int) -> TreeNode {
        guard let node = node else { return TreeNode(value) }

        if value < node.val {
            node.left = insertRec(node.left, value)
        } else {
            node.right = insertRec(node.right, value)
        }

        return node
    }

    func search(_ value: Int) -> Bool {
        return searchRec(root, value)
    }

    private func searchRec(_ node: TreeNode?, _ value: Int) -> Bool {
        guard let node = node else { return false }
        if node.val == value { return true }

        return if value < node.val {
            searchRec(node.left, value)
        } else {
            searchRec(node.right, value)
        }
    }

    func delete(_ value: Int) {
        root = deleteRec(root, value)
    }

    private func deleteRec(_ node: TreeNode?, _ value: Int) -> TreeNode? {
        guard let node = node else { return nil }

        // (Swift has no subject-less `switch { case cond: }`, so use if/else)
        if value < node.val {
            node.left = deleteRec(node.left, value)
        } else if value > node.val {
            node.right = deleteRec(node.right, value)
        } else {
            // Node with one or no child
            guard let left = node.left else { return node.right }
            guard let right = node.right else { return left }

            // Node with two children - get inorder successor
            let successor = minValueNode(right)
            node.val = successor.val
            node.right = deleteRec(right, successor.val)
        }
        return node
    }

    private func minValueNode(_ node: TreeNode) -> TreeNode {
        var current = node
        while current.left != nil {
            current = current.left!
        }
        return current
    }
}
```

## 🧭 DFS on Trees (Recursive)

**Idea:** Solve the problem for the left and right subtrees, then combine at the current node (postorder), or pass information down from the parent (preorder, like the remaining sum in Path Sum). The base case is the empty tree.

```swift
// Maximum depth of binary tree
func maxDepth(_ root: TreeNode?) -> Int {
    guard let root = root else { return 0 }

    let leftDepth = maxDepth(root.left)
    let rightDepth = maxDepth(root.right)

    return max(leftDepth, rightDepth) + 1
}

// Path sum
func hasPathSum(_ root: TreeNode?, _ targetSum: Int) -> Bool {
    guard let root = root else { return false }
    if root.left == nil && root.right == nil {
        return root.val == targetSum
    }

    let remainingSum = targetSum - root.val
    return hasPathSum(root.left, remainingSum) || hasPathSum(root.right, remainingSum)
}

// Check if symmetric
func isSymmetric(_ root: TreeNode?) -> Bool {
    return isMirror(root?.left, root?.right)
}

private func isMirror(_ left: TreeNode?, _ right: TreeNode?) -> Bool {
    if left == nil && right == nil { return true }
    if left == nil || right == nil { return false }

    return (left!.val == right!.val) &&
           isMirror(left!.left, right!.right) &&
           isMirror(left!.right, right!.left)
}
```

## 🚪 BFS on Trees (Level Order)

**Idea:** Use a queue. At the start of each round, the number of queued nodes is exactly the size of the current level, so process that many, enqueueing their children for the next level. The first leaf BFS meets is at the minimum depth.

```swift
// Level order traversal
func levelOrder(_ root: TreeNode?) -> [[Int]] {
    var result = [[Int]]()
    guard let root = root else { return result }

    var queue = [TreeNode]()
    var head = 0 // index of the next node to dequeue
    queue.append(root)

    while head < queue.count {
        let levelSize = queue.count - head
        var level = [Int]()

        for _ in 0..<levelSize {
            let node = queue[head]
            head += 1
            level.append(node.val)

            if let left = node.left { queue.append(left) }
            if let right = node.right { queue.append(right) }
        }

        result.append(level)
    }

    return result
}

// Minimum depth of binary tree
func minDepth(_ root: TreeNode?) -> Int {
    guard let root = root else { return 0 }

    var queue = [(TreeNode, Int)]()
    var head = 0
    queue.append((root, 1))

    while head < queue.count {
        let (node, depth) = queue[head]
        head += 1

        if node.left == nil && node.right == nil {
            return depth
        }

        if let left = node.left { queue.append((left, depth + 1)) }
        if let right = node.right { queue.append((right, depth + 1)) }
    }

    return 0
}

// Right side view
func rightSideView(_ root: TreeNode?) -> [Int] {
    var result = [Int]()
    guard let root = root else { return result }

    var queue = [TreeNode]()
    var head = 0
    queue.append(root)

    while head < queue.count {
        let levelSize = queue.count - head

        for i in 0..<levelSize {
            let node = queue[head]
            head += 1

            // Add the last node of each level
            if i == levelSize - 1 {
                result.append(node.val)
            }

            if let left = node.left { queue.append(left) }
            if let right = node.right { queue.append(right) }
        }
    }

    return result
}
```

## 🔁 Iterative Tree Traversals

**Idea:** An explicit stack replaces the call stack. Inorder: push the whole left spine, pop and visit, then go right. Preorder: pop, visit, push right then left. Postorder: do a "node, right, left" preorder and reverse the output.

```swift
// Inorder traversal (iterative)
func inorderTraversal(_ root: TreeNode?) -> [Int] {
    var result = [Int]()
    var stack = [TreeNode]()
    var current = root

    while current != nil || !stack.isEmpty {
        while current != nil {
            stack.append(current!)
            current = current?.left
        }

        current = stack.removeLast()
        result.append(current!.val)
        current = current?.right
    }

    return result
}

// Preorder traversal (iterative)
func preorderTraversal(_ root: TreeNode?) -> [Int] {
    var result = [Int]()
    guard let root = root else { return result }

    var stack = [TreeNode]()
    stack.append(root)

    while !stack.isEmpty {
        let node = stack.removeLast()
        result.append(node.val)

        // Push right first, then left (so left is processed first)
        if let right = node.right { stack.append(right) }
        if let left = node.left { stack.append(left) }
    }

    return result
}

// Postorder traversal (iterative)
func postorderTraversal(_ root: TreeNode?) -> [Int] {
    var result = [Int]()
    guard let root = root else { return result }

    var stack = [TreeNode]()
    var output = [TreeNode]()

    stack.append(root)

    while !stack.isEmpty {
        let node = stack.removeLast()
        output.append(node)

        // Push left first, then right
        if let left = node.left { stack.append(left) }
        if let right = node.right { stack.append(right) }
    }

    while !output.isEmpty {
        result.append(output.removeLast().val)
    }

    return result
}
```

## 🔄 BST Invariants

**Idea:** Every node must lie strictly inside the `(low, high)` range inherited from its ancestors, not just compare with its parent. Optional bounds (`nil` = unbounded) avoid sentinel values like `Int.min` that a real node could equal. Inorder traversal of a BST visits values in sorted order, so the kth visited node is the kth smallest.

```swift
// Validate BST
func isValidBST(_ root: TreeNode?) -> Bool {
    return isValidBSTHelper(root, nil, nil)
}

private func isValidBSTHelper(_ node: TreeNode?, _ low: Int?, _ high: Int?) -> Bool {
    guard let node = node else { return true }

    if let low = low, node.val <= low { return false }
    if let high = high, node.val >= high { return false }

    return isValidBSTHelper(node.left, low, node.val) &&
           isValidBSTHelper(node.right, node.val, high)
}

// Kth smallest element in BST
func kthSmallest(_ root: TreeNode?, _ k: Int) -> Int {
    var stack = [TreeNode]()
    var current = root
    var count = 0

    while current != nil || !stack.isEmpty {
        while current != nil {
            stack.append(current!)
            current = current?.left
        }

        current = stack.removeLast()
        count += 1

        if count == k { return current!.val }

        current = current?.right
    }

    fatalError("Invalid k value")
}
```

## 🧩 Tree Serialization / Deserialization

**Idea:** Write the tree level by level, including `"null"` markers for missing children, so the shape is recoverable. To rebuild, read values in the same BFS order: each dequeued parent takes the next two tokens as its left and right child.

```swift
// Serialize binary tree
func serialize(_ root: TreeNode?) -> String {
    guard let root = root else { return "null" }

    var result = [String]()
    var queue = [TreeNode?]()
    var head = 0
    queue.append(root)

    while head < queue.count {
        let node = queue[head]
        head += 1

        if let node = node {
            result.append(String(node.val))
            queue.append(node.left)
            queue.append(node.right)
        } else {
            result.append("null")
        }
    }

    return result.joined(separator: ",")
}

// Deserialize binary tree
func deserialize(_ data: String) -> TreeNode? {
    if data == "null" { return nil }

    let values = data.split(separator: ",").map { String($0) }
    var queue = [TreeNode]()
    var head = 0
    let root = TreeNode(Int(values[0])!)
    queue.append(root)

    var i = 1
    while head < queue.count && i < values.count {
        let node = queue[head]
        head += 1

        // Left child
        if values[i] != "null" {
            node.left = TreeNode(Int(values[i])!)
            queue.append(node.left!)
        }
        i += 1

        // Right child
        if i < values.count && values[i] != "null" {
            node.right = TreeNode(Int(values[i])!)
            queue.append(node.right!)
        }
        i += 1
    }

    return root
}
```

## Basic Graph Implementation

```swift
// Graph using adjacency list
class Graph {
    private var adjList: [[Int]]

    init(vertices: Int) {
        adjList = Array(repeating: [Int](), count: vertices)
    }

    func addEdge(_ src: Int, _ dest: Int) {
        adjList[src].append(dest)
        adjList[dest].append(src) // For undirected graph
    }

    func getNeighbors(_ vertex: Int) -> [Int] {
        return adjList[vertex]
    }
}
```

## 🔁 DFS on Graphs

**Idea:** Go as deep as possible from a node, marking it visited so it is never processed twice (on a grid, overwrite the cell; when cloning, the original-to-clone dictionary doubles as the visited set). For directed cycles, also track which nodes are on the current path (`recStack`).

```swift
// Number of islands (DFS version)
func numIslandsDFS(_ grid: [[Character]]) -> Int {
    guard !grid.isEmpty, !grid[0].isEmpty else { return 0 }

    let rows = grid.count
    let cols = grid[0].count
    var grid = grid
    var count = 0

    for i in 0..<rows {
        for j in 0..<cols {
            if grid[i][j] == "1" {
                count += 1
                dfs(&grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private func dfs(_ grid: inout [[Character]], _ i: Int, _ j: Int, _ rows: Int, _ cols: Int) {
    guard i >= 0, i < rows, j >= 0, j < cols, grid[i][j] == "1" else { return }

    grid[i][j] = "0" // Mark as visited

    dfs(&grid, i - 1, j, rows, cols)
    dfs(&grid, i + 1, j, rows, cols)
    dfs(&grid, i, j - 1, rows, cols)
    dfs(&grid, i, j + 1, rows, cols)
}

// Clone graph (DFS)
class GraphNode {
    var val: Int
    var neighbors: [GraphNode?]

    init(_ val: Int) {
        self.val = val
        self.neighbors = []
    }
}

// GraphNode is a class that is not Hashable, so key the dictionary by ObjectIdentifier
func cloneGraph(_ node: GraphNode?) -> GraphNode? {
    guard let node = node else { return nil }

    var visited = [ObjectIdentifier: GraphNode]()
    return dfsClone(node, &visited)
}

private func dfsClone(_ node: GraphNode, _ visited: inout [ObjectIdentifier: GraphNode]) -> GraphNode {
    if let clone = visited[ObjectIdentifier(node)] { return clone }

    let clone = GraphNode(node.val)
    visited[ObjectIdentifier(node)] = clone

    for neighbor in node.neighbors {
        if let neighbor = neighbor {
            clone.neighbors.append(dfsClone(neighbor, &visited))
        }
    }

    return clone
}

// Detect cycle in directed graph
func hasCycleDirected(_ graph: [[Int]]) -> Bool {
    let n = graph.count
    var visited = [Bool](repeating: false, count: n)
    var recStack = [Bool](repeating: false, count: n)

    for i in 0..<n {
        if hasCycleDFS(graph, i, &visited, &recStack) {
            return true
        }
    }

    return false
}

private func hasCycleDFS(_ graph: [[Int]], _ vertex: Int, _ visited: inout [Bool], _ recStack: inout [Bool]) -> Bool {
    if recStack[vertex] { return true }
    if visited[vertex] { return false }

    visited[vertex] = true
    recStack[vertex] = true

    for neighbor in graph[vertex] {
        if hasCycleDFS(graph, neighbor, &visited, &recStack) {
            return true
        }
    }

    recStack[vertex] = false
    return false
}
```

## 🚪 BFS on Graphs

**Idea:** In an unweighted graph, BFS explores nodes in order of distance, so the first time you reach the target is via a shortest path. Multi-source BFS (rotting oranges) starts with every source in the queue at distance 0.

```swift
// Word ladder (shortest transformation sequence)
func ladderLength(_ beginWord: String, _ endWord: String, _ wordList: [String]) -> Int {
    var wordSet = Set(wordList)
    guard wordSet.contains(endWord) else { return 0 }

    var queue = [(String, Int)]()
    var head = 0
    queue.append((beginWord, 1))

    while head < queue.count {
        let (word, level) = queue[head]
        head += 1

        if word == endWord { return level }

        var wordChars = Array(word)
        for i in 0..<wordChars.count {
            let original = wordChars[i]

            for c in "abcdefghijklmnopqrstuvwxyz" {
                if c == original { continue }

                wordChars[i] = c
                let newWord = String(wordChars)

                if wordSet.contains(newWord) {
                    wordSet.remove(newWord)
                    queue.append((newWord, level + 1))
                }
            }

            wordChars[i] = original
        }
    }

    return 0
}

// Rotten oranges (multi-source BFS)
func orangesRotting(_ grid: [[Int]]) -> Int {
    let rows = grid.count
    let cols = grid[0].count
    var grid = grid

    var queue = [(Int, Int, Int)]() // x, y, minutes
    var freshCount = 0

    // Add all rotten oranges to queue and count fresh ones
    for i in 0..<rows {
        for j in 0..<cols {
            switch grid[i][j] {
            case 1:
                freshCount += 1
            case 2:
                queue.append((i, j, 0))
            default:
                break
            }
        }
    }

    let directions = [(-1, 0), (1, 0), (0, -1), (0, 1)]

    var minutes = 0
    var head = 0

    while head < queue.count {
        let (x, y, mins) = queue[head]
        head += 1
        minutes = max(minutes, mins)

        for dir in directions {
            let nx = x + dir.0
            let ny = y + dir.1

            if nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] == 1 {
                grid[nx][ny] = 2 // Mark as rotten
                freshCount -= 1
                queue.append((nx, ny, mins + 1))
            }
        }
    }

    return freshCount == 0 ? minutes : -1
}
```

## 🔄 Topological Sort

**Idea:** Count incoming edges for each node. Start with all nodes of in-degree 0; each time you output one, decrement its neighbours' in-degrees and enqueue any that drop to 0. If not every node gets output, there is a cycle.

```swift
// Course Schedule I & II (Topological Sort)
func findOrder(_ numCourses: Int, _ prerequisites: [[Int]]) -> [Int] {
    var graph = [[Int]](repeating: [], count: numCourses)
    var inDegree = [Int](repeating: 0, count: numCourses)

    // Build graph and calculate in-degrees
    for prereq in prerequisites {
        graph[prereq[1]].append(prereq[0])
        inDegree[prereq[0]] += 1
    }

    // BFS with queue (Kahn's algorithm)
    var queue = [Int]()
    for i in 0..<numCourses {
        if inDegree[i] == 0 {
            queue.append(i)
        }
    }

    var result = [Int]()
    var head = 0

    while head < queue.count {
        let course = queue[head]
        head += 1
        result.append(course)

        for neighbor in graph[course] {
            inDegree[neighbor] -= 1
            if inDegree[neighbor] == 0 {
                queue.append(neighbor)
            }
        }
    }

    return result.count == numCourses ? result : []
}

// Alien Dictionary
func alienOrder(_ words: [String]) -> String {
    guard !words.isEmpty else { return "" }

    // Build graph
    var graph = [Character: Set<Character>]()
    var inDegree = [Character: Int]()

    // Initialize all characters
    for word in words {
        for char in word {
            if graph[char] == nil {
                graph[char] = Set<Character>()
                inDegree[char] = 0
            }
        }
    }

    // Build the graph by comparing adjacent words
    for i in 0..<words.count - 1 {
        // Arrays give O(1) indexing; String.index(offsetBy:) is O(j) per lookup
        let word1 = Array(words[i])
        let word2 = Array(words[i + 1])

        // Find the first different character
        let minLen = min(word1.count, word2.count)
        var j = 0

        while j < minLen && word1[j] == word2[j] {
            j += 1
        }

        if j == minLen && word1.count > word2.count {
            return "" // e.g. ["abc", "ab"]: a longer word cannot come before its own prefix
        }

        if j < minLen {
            let char1 = word1[j]
            let char2 = word2[j]

            if !graph[char1]!.contains(char2) {
                graph[char1]!.insert(char2)
                inDegree[char2] = (inDegree[char2] ?? 0) + 1
            }
        }
    }

    // Topological sort
    var queue = [Character]()
    for (char, degree) in inDegree {
        if degree == 0 {
            queue.append(char)
        }
    }

    var result = [Character]()
    var head = 0

    while head < queue.count {
        let char = queue[head]
        head += 1
        result.append(char)

        for neighbor in graph[char]! {
            inDegree[neighbor] = (inDegree[neighbor] ?? 0) - 1
            if inDegree[neighbor] == 0 {
                queue.append(neighbor)
            }
        }
    }

    return result.count == graph.count ? String(result) : ""
}
```

## 🪢 Union-Find (Disjoint Set)

**Idea:** Each set is a tree identified by its root. `find` walks to the root (flattening the path as it goes); `union` links two roots. With path compression and union by rank, both are nearly O(1) amortized. A `union` that returns false means the edge closes a cycle.

```swift
class UnionFind {
    private var parent: [Int]
    private var rank: [Int]

    init(_ size: Int) {
        parent = Array(0..<size)
        rank = Array(repeating: 1, count: size)
    }

    func find(_ x: Int) -> Int {
        if parent[x] != x {
            parent[x] = find(parent[x]) // Path compression
        }
        return parent[x]
    }

    func union(_ x: Int, _ y: Int) -> Bool {
        let rootX = find(x)
        let rootY = find(y)

        if rootX == rootY { return false }

        // Union by rank
        if rank[rootX] > rank[rootY] {
            parent[rootY] = rootX
        } else if rank[rootX] < rank[rootY] {
            parent[rootX] = rootY
        } else {
            parent[rootY] = rootX
            rank[rootX] += 1
        }

        return true
    }
}

// Number of connected components
func countComponents(_ n: Int, _ edges: [[Int]]) -> Int {
    let uf = UnionFind(n)
    var components = n

    for edge in edges {
        if uf.union(edge[0], edge[1]) {
            components -= 1
        }
    }

    return components
}

// Redundant connection
func findRedundantConnection(_ edges: [[Int]]) -> [Int] {
    let uf = UnionFind(edges.count + 1)

    for edge in edges {
        if !uf.union(edge[0], edge[1]) {
            return edge
        }
    }

    return []
}
```

## 🌊 Flood Fill / Connected Components

**Idea:** Each DFS from an unvisited land cell sinks one whole island, so the number of DFS starts is the island count. For Surrounded Regions, start from the border instead: anything reachable from a border `"O"` is safe, everything else is captured.

```swift
// Number of islands (DFS version)
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
                dfsFloodFill(&grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private func dfsFloodFill(_ grid: inout [[Character]], _ i: Int, _ j: Int, _ rows: Int, _ cols: Int) {
    guard i >= 0, i < rows, j >= 0, j < cols, grid[i][j] == "1" else { return }

    grid[i][j] = "0" // Mark as visited

    dfsFloodFill(&grid, i - 1, j, rows, cols)
    dfsFloodFill(&grid, i + 1, j, rows, cols)
    dfsFloodFill(&grid, i, j - 1, rows, cols)
    dfsFloodFill(&grid, i, j + 1, rows, cols)
}

// Surrounded regions (capture regions)
func solve(_ board: inout [[Character]]) {
    guard !board.isEmpty, !board[0].isEmpty else { return }

    let rows = board.count
    let cols = board[0].count

    // Mark border 'O's and their connected 'O's as safe
    for i in 0..<rows {
        for j in 0..<cols {
            if (i == 0 || i == rows - 1 || j == 0 || j == cols - 1) && board[i][j] == "O" {
                dfsSurrounded(&board, i, j, rows, cols)
            }
        }
    }

    // Flip remaining 'O's to 'X' and marked 'O's back to 'O'
    for i in 0..<rows {
        for j in 0..<cols {
            switch board[i][j] {
            case "O":
                board[i][j] = "X"
            case "M":
                board[i][j] = "O" // 'M' was our marker
            default:
                break
            }
        }
    }
}

private func dfsSurrounded(_ board: inout [[Character]], _ i: Int, _ j: Int, _ rows: Int, _ cols: Int) {
    guard i >= 0, i < rows, j >= 0, j < cols, board[i][j] == "O" else { return }

    board[i][j] = "M" // Mark as safe

    dfsSurrounded(&board, i - 1, j, rows, cols)
    dfsSurrounded(&board, i + 1, j, rows, cols)
    dfsSurrounded(&board, i, j - 1, rows, cols)
    dfsSurrounded(&board, i, j + 1, rows, cols)
}
```

---

## Summary: Tree & Graph Pattern Map

| Pattern              | Keywords / Use Case                        |
|---------------------|------------------------------------------|
| DFS (Tree)          | recursive, compute from subtrees          |
| BFS (Tree)          | level order, minimum depth                |
| Iterative Traversals| inorder, preorder, postorder              |
| BST Invariants      | kth smallest, validate BST                |
| Serialize Tree      | encode/decode tree, design                |
| DFS (Graph)         | explore components, clone, cycle detection|
| BFS (Graph)         | shortest path, level traversal            |
| Topological Sort    | dependencies, course scheduling           |
| Union-Find          | connected components, undirected graphs   |
| Flood Fill          | 2D boards, clusters, fill                 |

---