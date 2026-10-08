# Trees & Graphs in Kotlin

> Kotlin implementation reference for binary trees, BSTs and graphs: DFS/BFS, iterative traversals, serialization, topological sort, union-find and flood fill.
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

> **Key Interview Point:** Kotlin's `when` expression, `?.let{}`, and `Pair`/`Triple` make tree/graph code more concise than Java. Use `in 0 until n` for bounds checking.

---

## Tree Node & BST

```kotlin
// Binary Tree Node
class TreeNode(var `val`: Int) {
    var left: TreeNode? = null
    var right: TreeNode? = null
}

// Binary Search Tree
class BST() {
    var root: TreeNode? = null

    fun insert(value: Int) {
        root = insertRec(root, value)
    }

    private fun insertRec(node: TreeNode?, value: Int): TreeNode {
        if (node == null) return TreeNode(value)

        if (value < node.`val`) {
            node.left = insertRec(node.left, value)
        } else {
            node.right = insertRec(node.right, value)
        }

        return node
    }

    fun search(value: Int): Boolean {
        return searchRec(root, value)
    }

    private fun searchRec(node: TreeNode?, value: Int): Boolean {
        if (node == null) return false
        if (node.`val` == value) return true

        return if (value < node.`val`) {
            searchRec(node.left, value)
        } else {
            searchRec(node.right, value)
        }
    }

    fun delete(value: Int) {
        root = deleteRec(root, value)
    }

    private fun deleteRec(node: TreeNode?, value: Int): TreeNode? {
        if (node == null) return null

        when {
            value < node.`val` -> node.left = deleteRec(node.left, value)
            value > node.`val` -> node.right = deleteRec(node.right, value)
            else -> {
                // Node with one or no child.
                // Copy `right` into a local val: `node.right` is a mutable property, so
                // Kotlin cannot smart-cast it to non-null after a null check.
                val right = node.right ?: return node.left
                if (node.left == null) return right

                // Node with two children - get inorder successor
                val successor = minValueNode(right)
                node.`val` = successor.`val`
                node.right = deleteRec(right, successor.`val`)
            }
        }
        return node
    }

    private fun minValueNode(node: TreeNode): TreeNode {
        var current = node
        while (current.left != null) {
            current = current.left!!
        }
        return current
    }
}
```

## 🧭 DFS on Trees (Recursive)

**Idea:** Solve the problem for the left and right subtrees, then combine at the current node (postorder), or pass information down from the parent (preorder, like the remaining sum in Path Sum). The base case is the empty tree.

```kotlin
// Maximum depth of binary tree
fun maxDepth(root: TreeNode?): Int {
    if (root == null) return 0

    val leftDepth = maxDepth(root.left)
    val rightDepth = maxDepth(root.right)

    return maxOf(leftDepth, rightDepth) + 1
}

// Path sum
fun hasPathSum(root: TreeNode?, targetSum: Int): Boolean {
    if (root == null) return false
    if (root.left == null && root.right == null) {
        return root.`val` == targetSum
    }

    val remainingSum = targetSum - root.`val`
    return hasPathSum(root.left, remainingSum) || hasPathSum(root.right, remainingSum)
}

// Check if symmetric
fun isSymmetric(root: TreeNode?): Boolean {
    return isMirror(root?.left, root?.right)
}

private fun isMirror(left: TreeNode?, right: TreeNode?): Boolean {
    if (left == null && right == null) return true
    if (left == null || right == null) return false

    return (left.`val` == right.`val`) &&
           isMirror(left.left, right.right) &&
           isMirror(left.right, right.left)
}
```

## 🚪 BFS on Trees (Level Order)

**Idea:** Use a queue. At the start of each round, `queue.size` is exactly the number of nodes on the current level, so process that many, enqueueing their children for the next level. The first leaf BFS meets is at the minimum depth.

```kotlin
// Level order traversal
fun levelOrder(root: TreeNode?): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    if (root == null) return result

    val queue = ArrayDeque<TreeNode>()
    queue.addLast(root)

    while (queue.isNotEmpty()) {
        val levelSize = queue.size
        val level = mutableListOf<Int>()

        for (i in 0 until levelSize) {
            val node = queue.removeFirst()
            level.add(node.`val`)

            node.left?.let { queue.addLast(it) }
            node.right?.let { queue.addLast(it) }
        }

        result.add(level)
    }

    return result
}

// Minimum depth of binary tree
fun minDepth(root: TreeNode?): Int {
    if (root == null) return 0

    val queue = ArrayDeque<Pair<TreeNode, Int>>()
    queue.addLast(Pair(root, 1))

    while (queue.isNotEmpty()) {
        val (node, depth) = queue.removeFirst()

        if (node.left == null && node.right == null) {
            return depth
        }

        node.left?.let { queue.addLast(Pair(it, depth + 1)) }
        node.right?.let { queue.addLast(Pair(it, depth + 1)) }
    }

    return 0
}

// Right side view
fun rightSideView(root: TreeNode?): List<Int> {
    val result = mutableListOf<Int>()
    if (root == null) return result

    val queue = ArrayDeque<TreeNode>()
    queue.addLast(root)

    while (queue.isNotEmpty()) {
        val levelSize = queue.size

        for (i in 0 until levelSize) {
            val node = queue.removeFirst()

            // Add the last node of each level
            if (i == levelSize - 1) {
                result.add(node.`val`)
            }

            node.left?.let { queue.addLast(it) }
            node.right?.let { queue.addLast(it) }
        }
    }

    return result
}
```

## 🔁 Iterative Tree Traversals

**Idea:** An explicit stack replaces the call stack. Inorder: push the whole left spine, pop and visit, then go right. Preorder: pop, visit, push right then left. Postorder: do a "node, right, left" preorder and reverse the output.

```kotlin
// Inorder traversal (iterative)
fun inorderTraversal(root: TreeNode?): List<Int> {
    val result = mutableListOf<Int>()
    val stack = ArrayDeque<TreeNode>()
    var current = root

    while (current != null || stack.isNotEmpty()) {
        while (current != null) {
            stack.addLast(current)
            current = current.left
        }

        current = stack.removeLast()
        result.add(current.`val`)
        current = current.right
    }

    return result
}

// Preorder traversal (iterative)
fun preorderTraversal(root: TreeNode?): List<Int> {
    val result = mutableListOf<Int>()
    if (root == null) return result

    val stack = ArrayDeque<TreeNode>()
    stack.addLast(root)

    while (stack.isNotEmpty()) {
        val node = stack.removeLast()
        result.add(node.`val`)

        // Push right first, then left (so left is processed first)
        node.right?.let { stack.addLast(it) }
        node.left?.let { stack.addLast(it) }
    }

    return result
}

// Postorder traversal (iterative)
fun postorderTraversal(root: TreeNode?): List<Int> {
    val result = mutableListOf<Int>()
    if (root == null) return result

    val stack = ArrayDeque<TreeNode>()
    val output = ArrayDeque<TreeNode>()

    stack.addLast(root)

    while (stack.isNotEmpty()) {
        val node = stack.removeLast()
        output.addLast(node)

        // Push left first, then right
        node.left?.let { stack.addLast(it) }
        node.right?.let { stack.addLast(it) }
    }

    while (output.isNotEmpty()) {
        result.add(output.removeLast().`val`)
    }

    return result
}
```

## 🔄 BST Invariants

**Idea:** Every node must lie strictly inside the `(min, max)` range inherited from its ancestors, not just compare with its parent (`Long` bounds so `Int.MIN_VALUE`/`Int.MAX_VALUE` nodes still pass). Inorder traversal of a BST visits values in sorted order, so the kth visited node is the kth smallest.

```kotlin
// Validate BST
fun isValidBST(root: TreeNode?): Boolean {
    return isValidBSTHelper(root, Long.MIN_VALUE, Long.MAX_VALUE)
}

private fun isValidBSTHelper(node: TreeNode?, min: Long, max: Long): Boolean {
    if (node == null) return true

    if (node.`val` <= min || node.`val` >= max) return false

    return isValidBSTHelper(node.left, min, node.`val`.toLong()) &&
           isValidBSTHelper(node.right, node.`val`.toLong(), max)
}

// Kth smallest element in BST
fun kthSmallest(root: TreeNode?, k: Int): Int {
    val stack = ArrayDeque<TreeNode>()
    var current = root
    var count = 0

    while (current != null || stack.isNotEmpty()) {
        while (current != null) {
            stack.addLast(current)
            current = current.left
        }

        current = stack.removeLast()
        count++

        if (count == k) return current.`val`

        current = current.right
    }

    throw IllegalArgumentException("Invalid k value")
}
```

## 🧩 Tree Serialization / Deserialization

**Idea:** Write the tree level by level, including `"null"` markers for missing children, so the shape is recoverable. To rebuild, read values in the same BFS order: each dequeued parent takes the next two tokens as its left and right child.

```kotlin
// Serialize binary tree
fun serialize(root: TreeNode?): String {
    if (root == null) return "null"

    val result = mutableListOf<String>()
    val queue = ArrayDeque<TreeNode?>()
    queue.addLast(root)

    while (queue.isNotEmpty()) {
        val node = queue.removeFirst()

        if (node != null) {
            result.add(node.`val`.toString())
            queue.addLast(node.left)
            queue.addLast(node.right)
        } else {
            result.add("null")
        }
    }

    return result.joinToString(",")
}

// Deserialize binary tree
fun deserialize(data: String): TreeNode? {
    if (data == "null") return null

    val values = data.split(",")
    val queue = ArrayDeque<TreeNode>()
    val root = TreeNode(values[0].toInt())
    queue.addLast(root)

    var i = 1
    while (queue.isNotEmpty() && i < values.size) {
        val node = queue.removeFirst()

        // Left child
        if (values[i] != "null") {
            node.left = TreeNode(values[i].toInt())
            queue.addLast(node.left!!)
        }
        i++

        // Right child
        if (i < values.size && values[i] != "null") {
            node.right = TreeNode(values[i].toInt())
            queue.addLast(node.right!!)
        }
        i++
    }

    return root
}
```

## Basic Graph Implementation

```kotlin
// Graph using adjacency list
class Graph(val vertices: Int) {
    private val adjList = Array<MutableList<Int>>(vertices) { mutableListOf() }

    fun addEdge(src: Int, dest: Int) {
        adjList[src].add(dest)
        adjList[dest].add(src) // For undirected graph
    }

    fun getNeighbors(vertex: Int): List<Int> {
        return adjList[vertex]
    }
}
```

## 🔁 DFS on Graphs

**Idea:** Go as deep as possible from a node, marking it visited so it is never processed twice (on a grid, overwrite the cell; when cloning, the original-to-clone map doubles as the visited set). For directed cycles, also track which nodes are on the current path (`recStack`).

```kotlin
// Number of islands (DFS version)
fun numIslandsDFS(grid: Array<CharArray>): Int {
    if (grid.isEmpty() || grid[0].isEmpty()) return 0

    val rows = grid.size
    val cols = grid[0].size
    var count = 0

    for (i in 0 until rows) {
        for (j in 0 until cols) {
            if (grid[i][j] == '1') {
                count++
                dfs(grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private fun dfs(grid: Array<CharArray>, i: Int, j: Int, rows: Int, cols: Int) {
    if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] == '0') return

    grid[i][j] = '0' // Mark as visited

    dfs(grid, i - 1, j, rows, cols)
    dfs(grid, i + 1, j, rows, cols)
    dfs(grid, i, j - 1, rows, cols)
    dfs(grid, i, j + 1, rows, cols)
}

// Clone graph (DFS)
class GraphNode(var `val`: Int) {
    var neighbors: ArrayList<GraphNode?> = ArrayList()
}

fun cloneGraph(node: GraphNode?): GraphNode? {
    if (node == null) return null

    val visited = mutableMapOf<GraphNode, GraphNode>()
    return dfsClone(node, visited)
}

private fun dfsClone(node: GraphNode, visited: MutableMap<GraphNode, GraphNode>): GraphNode {
    if (node in visited) return visited[node]!!

    val clone = GraphNode(node.`val`)
    visited[node] = clone

    for (neighbor in node.neighbors) {
        if (neighbor != null) {
            clone.neighbors.add(dfsClone(neighbor, visited))
        }
    }

    return clone
}

// Detect cycle in directed graph
fun hasCycleDirected(graph: Array<MutableList<Int>>): Boolean {
    val visited = BooleanArray(graph.size)
    val recStack = BooleanArray(graph.size)

    for (i in graph.indices) {
        if (hasCycleDFS(graph, i, visited, recStack)) {
            return true
        }
    }

    return false
}

private fun hasCycleDFS(graph: Array<MutableList<Int>>, vertex: Int, visited: BooleanArray, recStack: BooleanArray): Boolean {
    if (recStack[vertex]) return true
    if (visited[vertex]) return false

    visited[vertex] = true
    recStack[vertex] = true

    for (neighbor in graph[vertex]) {
        if (hasCycleDFS(graph, neighbor, visited, recStack)) {
            return true
        }
    }

    recStack[vertex] = false
    return false
}
```

## 🚪 BFS on Graphs

**Idea:** In an unweighted graph, BFS explores nodes in order of distance, so the first time you reach the target is via a shortest path. Multi-source BFS (rotting oranges) starts with every source in the queue at distance 0.

```kotlin
// Word ladder (shortest transformation sequence)
fun ladderLength(beginWord: String, endWord: String, wordList: List<String>): Int {
    val wordSet = wordList.toMutableSet()
    if (endWord !in wordSet) return 0

    val queue = ArrayDeque<Pair<String, Int>>()
    queue.addLast(Pair(beginWord, 1))

    while (queue.isNotEmpty()) {
        val (word, level) = queue.removeFirst()

        if (word == endWord) return level

        val wordChars = word.toCharArray()
        for (i in wordChars.indices) {
            val original = wordChars[i]

            for (c in 'a'..'z') {
                if (c == original) continue

                wordChars[i] = c
                val newWord = String(wordChars)

                if (newWord in wordSet) {
                    wordSet.remove(newWord)
                    queue.addLast(Pair(newWord, level + 1))
                }
            }

            wordChars[i] = original
        }
    }

    return 0
}

// Rotten oranges (multi-source BFS)
fun orangesRotting(grid: Array<IntArray>): Int {
    val rows = grid.size
    val cols = grid[0].size

    val queue = ArrayDeque<Triple<Int, Int, Int>>() // x, y, minutes
    var freshCount = 0

    // Add all rotten oranges to queue and count fresh ones
    for (i in 0 until rows) {
        for (j in 0 until cols) {
            when (grid[i][j]) {
                1 -> freshCount++
                2 -> queue.addLast(Triple(i, j, 0))
            }
        }
    }

    val directions = arrayOf(
        intArrayOf(-1, 0), intArrayOf(1, 0),
        intArrayOf(0, -1), intArrayOf(0, 1)
    )

    var minutes = 0

    while (queue.isNotEmpty()) {
        val (x, y, mins) = queue.removeFirst()
        minutes = maxOf(minutes, mins)

        for (dir in directions) {
            val nx = x + dir[0]
            val ny = y + dir[1]

            if (nx in 0 until rows && ny in 0 until cols && grid[nx][ny] == 1) {
                grid[nx][ny] = 2 // Mark as rotten
                freshCount--
                queue.addLast(Triple(nx, ny, mins + 1))
            }
        }
    }

    return if (freshCount == 0) minutes else -1
}
```

## 🔄 Topological Sort

**Idea:** Count incoming edges for each node. Start with all nodes of in-degree 0; each time you output one, decrement its neighbours' in-degrees and enqueue any that drop to 0. If not every node gets output, there is a cycle.

```kotlin
// Course Schedule I & II (Topological Sort)
fun findOrder(numCourses: Int, prerequisites: Array<IntArray>): IntArray {
    val graph = Array<MutableList<Int>>(numCourses) { mutableListOf() }
    val inDegree = IntArray(numCourses)

    // Build graph and calculate in-degrees
    for (prereq in prerequisites) {
        graph[prereq[1]].add(prereq[0])
        inDegree[prereq[0]]++
    }

    // BFS with queue (Kahn's algorithm)
    val queue = ArrayDeque<Int>()
    for (i in 0 until numCourses) {
        if (inDegree[i] == 0) {
            queue.addLast(i)
        }
    }

    val result = mutableListOf<Int>()

    while (queue.isNotEmpty()) {
        val course = queue.removeFirst()
        result.add(course)

        for (neighbor in graph[course]) {
            inDegree[neighbor]--
            if (inDegree[neighbor] == 0) {
                queue.addLast(neighbor)
            }
        }
    }

    return if (result.size == numCourses) result.toIntArray() else intArrayOf()
}

// Alien Dictionary
fun alienOrder(words: Array<String>): String {
    if (words.isEmpty()) return ""

    // Build graph
    val graph = mutableMapOf<Char, MutableSet<Char>>()
    val inDegree = mutableMapOf<Char, Int>()

    // Initialize all characters
    for (word in words) {
        for (char in word) {
            if (char !in graph) {
                graph[char] = mutableSetOf()
                inDegree[char] = 0
            }
        }
    }

    // Build the graph by comparing adjacent words
    for (i in 0 until words.size - 1) {
        val word1 = words[i]
        val word2 = words[i + 1]

        // Find the first different character
        var j = 0
        val minLen = minOf(word1.length, word2.length)

        while (j < minLen && word1[j] == word2[j]) {
            j++
        }

        if (j == minLen && word1.length > word2.length) {
            return "" // e.g. ["abc", "ab"]: a longer word cannot come before its own prefix
        }

        if (j < minLen) {
            val char1 = word1[j]
            val char2 = word2[j]

            if (char2 !in graph[char1]!!) {
                graph[char1]!!.add(char2)
                inDegree[char2] = inDegree.getOrDefault(char2, 0) + 1
            }
        }
    }

    // Topological sort
    val queue = ArrayDeque<Char>()
    for ((char, degree) in inDegree) {
        if (degree == 0) {
            queue.addLast(char)
        }
    }

    val result = mutableListOf<Char>()

    while (queue.isNotEmpty()) {
        val char = queue.removeFirst()
        result.add(char)

        for (neighbor in graph[char]!!) {
            inDegree[neighbor] = inDegree[neighbor]!! - 1
            if (inDegree[neighbor] == 0) {
                queue.addLast(neighbor)
            }
        }
    }

    return if (result.size == graph.size) result.joinToString("") else ""
}
```

## 🪢 Union-Find (Disjoint Set)

**Idea:** Each set is a tree identified by its root. `find` walks to the root (flattening the path as it goes); `union` links two roots. With path compression and union by rank, both are nearly O(1) amortized. A `union` that returns false means the edge closes a cycle.

```kotlin
class UnionFind(val size: Int) {
    private val parent = IntArray(size) { it }
    private val rank = IntArray(size) { 1 }

    fun find(x: Int): Int {
        if (parent[x] != x) {
            parent[x] = find(parent[x]) // Path compression
        }
        return parent[x]
    }

    fun union(x: Int, y: Int): Boolean {
        val rootX = find(x)
        val rootY = find(y)

        if (rootX == rootY) return false

        // Union by rank
        if (rank[rootX] > rank[rootY]) {
            parent[rootY] = rootX
        } else if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY
        } else {
            parent[rootY] = rootX
            rank[rootX]++
        }

        return true
    }
}

// Number of connected components
fun countComponents(n: Int, edges: Array<IntArray>): Int {
    val uf = UnionFind(n)
    var components = n

    for (edge in edges) {
        if (uf.union(edge[0], edge[1])) {
            components--
        }
    }

    return components
}

// Redundant connection
fun findRedundantConnection(edges: Array<IntArray>): IntArray {
    val uf = UnionFind(edges.size + 1)

    for (edge in edges) {
        if (!uf.union(edge[0], edge[1])) {
            return edge
        }
    }

    return intArrayOf()
}
```

## 🌊 Flood Fill / Connected Components

**Idea:** Each DFS from an unvisited land cell sinks one whole island, so the number of DFS starts is the island count. For Surrounded Regions, start from the border instead: anything reachable from a border `'O'` is safe, everything else is captured.

```kotlin
// Number of islands (DFS version)
fun numIslands(grid: Array<CharArray>): Int {
    if (grid.isEmpty() || grid[0].isEmpty()) return 0

    val rows = grid.size
    val cols = grid[0].size
    var count = 0

    for (i in 0 until rows) {
        for (j in 0 until cols) {
            if (grid[i][j] == '1') {
                count++
                dfsFloodFill(grid, i, j, rows, cols)
            }
        }
    }

    return count
}

private fun dfsFloodFill(grid: Array<CharArray>, i: Int, j: Int, rows: Int, cols: Int) {
    if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] == '0') return

    grid[i][j] = '0' // Mark as visited

    dfsFloodFill(grid, i - 1, j, rows, cols)
    dfsFloodFill(grid, i + 1, j, rows, cols)
    dfsFloodFill(grid, i, j - 1, rows, cols)
    dfsFloodFill(grid, i, j + 1, rows, cols)
}

// Surrounded regions (capture regions)
fun solve(board: Array<CharArray>): Unit {
    if (board.isEmpty() || board[0].isEmpty()) return

    val rows = board.size
    val cols = board[0].size

    // Mark border 'O's and their connected 'O's as safe
    for (i in 0 until rows) {
        for (j in 0 until cols) {
            if ((i == 0 || i == rows - 1 || j == 0 || j == cols - 1) && board[i][j] == 'O') {
                dfsSurrounded(board, i, j, rows, cols)
            }
        }
    }

    // Flip remaining 'O's to 'X' and marked 'O's back to 'O'
    for (i in 0 until rows) {
        for (j in 0 until cols) {
            when (board[i][j]) {
                'O' -> board[i][j] = 'X'
                'M' -> board[i][j] = 'O' // 'M' was our marker
            }
        }
    }
}

private fun dfsSurrounded(board: Array<CharArray>, i: Int, j: Int, rows: Int, cols: Int) {
    if (i < 0 || i >= rows || j < 0 || j >= cols || board[i][j] != 'O') return

    board[i][j] = 'M' // Mark as safe

    dfsSurrounded(board, i - 1, j, rows, cols)
    dfsSurrounded(board, i + 1, j, rows, cols)
    dfsSurrounded(board, i, j - 1, rows, cols)
    dfsSurrounded(board, i, j + 1, rows, cols)
}
```
