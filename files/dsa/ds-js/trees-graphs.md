# Trees & Graphs in JavaScript

> JavaScript implementation reference for binary trees, BSTs and graphs: DFS/BFS, iterative traversals, serialization, topological sort, union-find and flood fill.
> New to this topic? Learn it first in [12. Trees and Traversals](../learn/12-trees-and-traversals.md) and [16. Graphs Fundamentals](../learn/16-graphs-fundamentals.md).

## Complexity Overview

| Tree Operation | BST Average | BST Worst | General Tree |
|---------------|-------------|-----------|-------------|
| Search | O(log n) | O(n) | O(n) |
| Insert | O(log n) | O(n) | O(n) |
| Delete | O(log n) | O(n) | O(n) |
| Traversal | O(n) | O(n) | O(n) |

| Graph Operation | Adjacency List | Adjacency Matrix |
|----------------|---------------|-----------------|
| Add Edge | O(1) | O(1) |
| BFS/DFS | O(V + E) | O(V^2) |
| Space | O(V + E) | O(V^2) |

> **Key Interview Point:** Most tree problems use DFS (recursive). Graph problems typically need a visited set. For shortest path in unweighted graphs, use BFS.

> **JS Queue Gotcha:** `array.shift()` is O(n) because it re-indexes every element, which turns a BFS into O(n^2). The BFS code below keeps a `head` index into the array instead (`queue[head++]`), which is O(1) per dequeue.

---

## Tree Node & BST

```javascript
// Binary Tree Node
class TreeNode {
    constructor(val) {
        this.val = val;
        this.left = null;
        this.right = null;
    }
}

// Binary Search Tree
class BST {
    constructor() {
        this.root = null;
    }

    insert(value) {
        this.root = this.insertRec(this.root, value);
    }

    insertRec(node, value) {
        if (node === null) return new TreeNode(value);

        if (value < node.val) {
            node.left = this.insertRec(node.left, value);
        } else {
            node.right = this.insertRec(node.right, value);
        }

        return node;
    }

    search(value) {
        return this.searchRec(this.root, value);
    }

    searchRec(node, value) {
        if (node === null) return false;
        if (node.val === value) return true;

        return value < node.val ?
            this.searchRec(node.left, value) :
            this.searchRec(node.right, value);
    }

    delete(value) {
        this.root = this.deleteRec(this.root, value);
    }

    deleteRec(node, value) {
        if (node === null) return null;

        if (value < node.val) {
            node.left = this.deleteRec(node.left, value);
        } else if (value > node.val) {
            node.right = this.deleteRec(node.right, value);
        } else {
            // Node with one or no child
            if (node.left === null) return node.right;
            if (node.right === null) return node.left;

            // Node with two children - get inorder successor
            const successor = this.minValueNode(node.right);
            node.val = successor.val;
            node.right = this.deleteRec(node.right, successor.val);
        }
        return node;
    }

    minValueNode(node) {
        let current = node;
        while (current.left !== null) {
            current = current.left;
        }
        return current;
    }
}
```

## 🧭 DFS on Trees (Recursive)

**Idea:** Solve the problem for the left and right subtrees, then combine at the current node (postorder), or pass information down from the parent (preorder, like the remaining sum in Path Sum). The base case is the empty tree.

```javascript
// Maximum depth of binary tree
function maxDepth(root) {
    if (root === null) return 0;

    const leftDepth = maxDepth(root.left);
    const rightDepth = maxDepth(root.right);

    return Math.max(leftDepth, rightDepth) + 1;
}

// Path sum
function hasPathSum(root, targetSum) {
    if (root === null) return false;
    if (root.left === null && root.right === null) {
        return root.val === targetSum;
    }

    const remainingSum = targetSum - root.val;
    return hasPathSum(root.left, remainingSum) || hasPathSum(root.right, remainingSum);
}

// Check if symmetric
function isSymmetric(root) {
    if (root === null) return true; // root?.left would give undefined, which isMirror does not handle
    return isMirror(root.left, root.right);
}

function isMirror(left, right) {
    if (left === null && right === null) return true;
    if (left === null || right === null) return false;

    return (left.val === right.val) &&
           isMirror(left.left, right.right) &&
           isMirror(left.right, right.left);
}
```

## 🚪 BFS on Trees (Level Order)

**Idea:** Use a queue. At the start of each round, the number of queued nodes is exactly the size of the current level, so process that many, enqueueing their children for the next level. The first leaf BFS meets is at the minimum depth.

```javascript
// Level order traversal
function levelOrder(root) {
    const result = [];
    if (root === null) return result;

    const queue = [root];
    let head = 0; // index of the next node to dequeue

    while (head < queue.length) {
        const levelSize = queue.length - head;
        const level = [];

        for (let i = 0; i < levelSize; i++) {
            const node = queue[head++];
            level.push(node.val);

            if (node.left !== null) queue.push(node.left);
            if (node.right !== null) queue.push(node.right);
        }

        result.push(level);
    }

    return result;
}

// Minimum depth of binary tree
function minDepth(root) {
    if (root === null) return 0;

    const queue = [[root, 1]];
    let head = 0;

    while (head < queue.length) {
        const [node, depth] = queue[head++];

        if (node.left === null && node.right === null) {
            return depth;
        }

        if (node.left !== null) queue.push([node.left, depth + 1]);
        if (node.right !== null) queue.push([node.right, depth + 1]);
    }

    return 0;
}

// Right side view
function rightSideView(root) {
    const result = [];
    if (root === null) return result;

    const queue = [root];
    let head = 0;

    while (head < queue.length) {
        const levelSize = queue.length - head;

        for (let i = 0; i < levelSize; i++) {
            const node = queue[head++];

            // Add the last node of each level
            if (i === levelSize - 1) {
                result.push(node.val);
            }

            if (node.left !== null) queue.push(node.left);
            if (node.right !== null) queue.push(node.right);
        }
    }

    return result;
}
```

## 🔁 Iterative Tree Traversals

**Idea:** An explicit stack replaces the call stack. Inorder: push the whole left spine, pop and visit, then go right. Preorder: pop, visit, push right then left. Postorder: do a "node, right, left" preorder and reverse the output.

```javascript
// Inorder traversal (iterative)
function inorderTraversal(root) {
    const result = [];
    const stack = [];
    let current = root;

    while (current !== null || stack.length > 0) {
        while (current !== null) {
            stack.push(current);
            current = current.left;
        }

        current = stack.pop();
        result.push(current.val);
        current = current.right;
    }

    return result;
}

// Preorder traversal (iterative)
function preorderTraversal(root) {
    const result = [];
    if (root === null) return result;

    const stack = [root];

    while (stack.length > 0) {
        const node = stack.pop();
        result.push(node.val);

        // Push right first, then left (so left is processed first)
        if (node.right !== null) stack.push(node.right);
        if (node.left !== null) stack.push(node.left);
    }

    return result;
}

// Postorder traversal (iterative)
function postorderTraversal(root) {
    const result = [];
    if (root === null) return result;

    const stack = [root];
    const output = [];

    while (stack.length > 0) {
        const node = stack.pop();
        output.push(node);

        // Push left first, then right
        if (node.left !== null) stack.push(node.left);
        if (node.right !== null) stack.push(node.right);
    }

    while (output.length > 0) {
        result.push(output.pop().val);
    }

    return result;
}
```

## 🔄 BST Invariants

**Idea:** Every node must lie strictly inside the `(min, max)` range inherited from its ancestors, not just compare with its parent. Inorder traversal of a BST visits values in sorted order, so the kth visited node is the kth smallest.

```javascript
// Validate BST
function isValidBST(root) {
    // +/-Infinity, not MIN/MAX_SAFE_INTEGER: a node equal to a finite sentinel would be wrongly rejected
    return isValidBSTHelper(root, -Infinity, Infinity);
}

function isValidBSTHelper(node, min, max) {
    if (node === null) return true;

    if (node.val <= min || node.val >= max) return false;

    return isValidBSTHelper(node.left, min, node.val) &&
           isValidBSTHelper(node.right, node.val, max);
}

// Kth smallest element in BST
function kthSmallest(root, k) {
    const stack = [];
    let current = root;
    let count = 0;

    while (current !== null || stack.length > 0) {
        while (current !== null) {
            stack.push(current);
            current = current.left;
        }

        current = stack.pop();
        count++;

        if (count === k) return current.val;

        current = current.right;
    }

    throw new Error("Invalid k value");
}
```

## 🧩 Tree Serialization / Deserialization

**Idea:** Write the tree level by level, including `"null"` markers for missing children, so the shape is recoverable. To rebuild, read values in the same BFS order: each dequeued parent takes the next two tokens as its left and right child.

```javascript
// Serialize binary tree
function serialize(root) {
    if (root === null) return "null";

    const result = [];
    const queue = [root];
    let head = 0;

    while (head < queue.length) {
        const node = queue[head++];

        if (node !== null) {
            result.push(node.val.toString());
            queue.push(node.left);
            queue.push(node.right);
        } else {
            result.push("null");
        }
    }

    return result.join(",");
}

// Deserialize binary tree
function deserialize(data) {
    if (data === "null") return null;

    const values = data.split(",");
    const queue = [];
    let head = 0;
    const root = new TreeNode(parseInt(values[0]));
    queue.push(root);

    let i = 1;
    while (head < queue.length && i < values.length) {
        const node = queue[head++];

        // Left child
        if (values[i] !== "null") {
            node.left = new TreeNode(parseInt(values[i]));
            queue.push(node.left);
        }
        i++;

        // Right child
        if (i < values.length && values[i] !== "null") {
            node.right = new TreeNode(parseInt(values[i]));
            queue.push(node.right);
        }
        i++;
    }

    return root;
}
```

## Basic Graph Implementation

```javascript
// Graph using adjacency list
class Graph {
    constructor(vertices) {
        this.adjList = Array.from({ length: vertices }, () => []);
    }

    addEdge(src, dest) {
        this.adjList[src].push(dest);
        this.adjList[dest].push(src); // For undirected graph
    }

    getNeighbors(vertex) {
        return this.adjList[vertex];
    }
}
```

## 🔁 DFS on Graphs

**Idea:** Go as deep as possible from a node, marking it visited so it is never processed twice (on a grid, overwrite the cell; when cloning, the original-to-clone `Map` doubles as the visited set). For directed cycles, also track which nodes are on the current path (`recStack`).

```javascript
// Number of islands (DFS version)
function numIslandsDFS(grid) {
    if (!grid || grid.length === 0 || grid[0].length === 0) return 0;

    const rows = grid.length;
    const cols = grid[0].length;
    let count = 0;

    function dfs(i, j) {
        if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] === '0') return;

        grid[i][j] = '0'; // Mark as visited

        dfs(i - 1, j);
        dfs(i + 1, j);
        dfs(i, j - 1);
        dfs(i, j + 1);
    }

    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (grid[i][j] === '1') {
                count++;
                dfs(i, j);
            }
        }
    }

    return count;
}

// Clone graph (DFS)
class GraphNode {
    constructor(val) {
        this.val = val;
        this.neighbors = [];
    }
}

function cloneGraph(node) {
    if (node === null) return null;

    const visited = new Map();
    return dfsClone(node, visited);
}

function dfsClone(node, visited) {
    if (visited.has(node)) return visited.get(node);

    const clone = new GraphNode(node.val);
    visited.set(node, clone);

    for (const neighbor of node.neighbors) {
        if (neighbor !== null) {
            clone.neighbors.push(dfsClone(neighbor, visited));
        }
    }

    return clone;
}

// Detect cycle in directed graph
function hasCycleDirected(graph) {
    const n = graph.length;
    const visited = new Array(n).fill(false);
    const recStack = new Array(n).fill(false);

    function dfs(vertex) {
        if (recStack[vertex]) return true;
        if (visited[vertex]) return false;

        visited[vertex] = true;
        recStack[vertex] = true;

        for (const neighbor of graph[vertex]) {
            if (dfs(neighbor)) {
                return true;
            }
        }

        recStack[vertex] = false;
        return false;
    }

    for (let i = 0; i < n; i++) {
        if (dfs(i)) {
            return true;
        }
    }

    return false;
}
```

## 🚪 BFS on Graphs

**Idea:** In an unweighted graph, BFS explores nodes in order of distance, so the first time you reach the target is via a shortest path. Multi-source BFS (rotting oranges) starts with every source in the queue at distance 0.

```javascript
// Word ladder (shortest transformation sequence)
function ladderLength(beginWord, endWord, wordList) {
    const wordSet = new Set(wordList);
    if (!wordSet.has(endWord)) return 0;

    const queue = [[beginWord, 1]];
    let head = 0;

    while (head < queue.length) {
        const [word, level] = queue[head++];

        if (word === endWord) return level;

        const wordChars = word.split('');
        for (let i = 0; i < wordChars.length; i++) {
            const original = wordChars[i];

            for (let c = 97; c <= 122; c++) { // 'a' to 'z'
                const char = String.fromCharCode(c);
                if (char === original) continue;

                wordChars[i] = char;
                const newWord = wordChars.join('');

                if (wordSet.has(newWord)) {
                    wordSet.delete(newWord);
                    queue.push([newWord, level + 1]);
                }
            }

            wordChars[i] = original;
        }
    }

    return 0;
}

// Rotten oranges (multi-source BFS)
function orangesRotting(grid) {
    const rows = grid.length;
    const cols = grid[0].length;

    const queue = [];
    let freshCount = 0;

    // Add all rotten oranges to queue and count fresh ones
    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (grid[i][j] === 1) {
                freshCount++;
            } else if (grid[i][j] === 2) {
                queue.push([i, j, 0]);
            }
        }
    }

    const directions = [[-1, 0], [1, 0], [0, -1], [0, 1]];
    let minutes = 0;
    let head = 0;

    while (head < queue.length) {
        const [x, y, mins] = queue[head++];
        minutes = Math.max(minutes, mins);

        for (const [dx, dy] of directions) {
            const nx = x + dx;
            const ny = y + dy;

            if (nx >= 0 && nx < rows && ny >= 0 && ny < cols && grid[nx][ny] === 1) {
                grid[nx][ny] = 2; // Mark as rotten
                freshCount--;
                queue.push([nx, ny, mins + 1]);
            }
        }
    }

    return freshCount === 0 ? minutes : -1;
}
```

## 🔄 Topological Sort

**Idea:** Count incoming edges for each node. Start with all nodes of in-degree 0; each time you output one, decrement its neighbours' in-degrees and enqueue any that drop to 0. If not every node gets output, there is a cycle.

```javascript
// Course Schedule I & II (Topological Sort)
function findOrder(numCourses, prerequisites) {
    const graph = Array.from({ length: numCourses }, () => []);
    const inDegree = new Array(numCourses).fill(0);

    // Build graph and calculate in-degrees
    for (const [course, prereq] of prerequisites) {
        graph[prereq].push(course);
        inDegree[course]++;
    }

    // BFS with queue (Kahn's algorithm)
    const queue = [];
    for (let i = 0; i < numCourses; i++) {
        if (inDegree[i] === 0) {
            queue.push(i);
        }
    }

    const result = [];
    let head = 0;

    while (head < queue.length) {
        const course = queue[head++];
        result.push(course);

        for (const neighbor of graph[course]) {
            inDegree[neighbor]--;
            if (inDegree[neighbor] === 0) {
                queue.push(neighbor);
            }
        }
    }

    return result.length === numCourses ? result : [];
}

// Alien Dictionary
function alienOrder(words) {
    if (words.length === 0) return "";

    // Build graph
    const graph = new Map();
    const inDegree = new Map();

    // Initialize all characters
    for (const word of words) {
        for (const char of word) {
            if (!graph.has(char)) {
                graph.set(char, new Set());
                inDegree.set(char, 0);
            }
        }
    }

    // Build the graph by comparing adjacent words
    for (let i = 0; i < words.length - 1; i++) {
        const word1 = words[i];
        const word2 = words[i + 1];

        // Find the first different character
        const minLen = Math.min(word1.length, word2.length);
        let j = 0;

        while (j < minLen && word1[j] === word2[j]) {
            j++;
        }

        if (j === minLen && word1.length > word2.length) {
            return ""; // e.g. ["abc", "ab"]: a longer word cannot come before its own prefix
        }

        if (j < minLen) {
            const char1 = word1[j];
            const char2 = word2[j];

            if (!graph.get(char1).has(char2)) {
                graph.get(char1).add(char2);
                inDegree.set(char2, (inDegree.get(char2) || 0) + 1);
            }
        }
    }

    // Topological sort
    const queue = [];
    for (const [char, degree] of inDegree) {
        if (degree === 0) {
            queue.push(char);
        }
    }

    const result = [];
    let head = 0;

    while (head < queue.length) {
        const char = queue[head++];
        result.push(char);

        for (const neighbor of graph.get(char)) {
            inDegree.set(neighbor, inDegree.get(neighbor) - 1);
            if (inDegree.get(neighbor) === 0) {
                queue.push(neighbor);
            }
        }
    }

    return result.length === graph.size ? result.join("") : "";
}
```

## 🪢 Union-Find (Disjoint Set)

**Idea:** Each set is a tree identified by its root. `find` walks to the root (flattening the path as it goes); `union` links two roots. With path compression and union by rank, both are nearly O(1) amortized. A `union` that returns false means the edge closes a cycle.

```javascript
class UnionFind {
    constructor(size) {
        this.parent = Array.from({ length: size }, (_, i) => i);
        this.rank = new Array(size).fill(1);
    }

    find(x) {
        if (this.parent[x] !== x) {
            this.parent[x] = this.find(this.parent[x]); // Path compression
        }
        return this.parent[x];
    }

    union(x, y) {
        const rootX = this.find(x);
        const rootY = this.find(y);

        if (rootX === rootY) return false;

        // Union by rank
        if (this.rank[rootX] > this.rank[rootY]) {
            this.parent[rootY] = rootX;
        } else if (this.rank[rootX] < this.rank[rootY]) {
            this.parent[rootX] = rootY;
        } else {
            this.parent[rootY] = rootX;
            this.rank[rootX]++;
        }

        return true;
    }
}

// Number of connected components
function countComponents(n, edges) {
    const uf = new UnionFind(n);
    let components = n;

    for (const [u, v] of edges) {
        if (uf.union(u, v)) {
            components--;
        }
    }

    return components;
}

// Redundant connection
function findRedundantConnection(edges) {
    const uf = new UnionFind(edges.length + 1);

    for (const [u, v] of edges) {
        if (!uf.union(u, v)) {
            return [u, v];
        }
    }

    return [];
}
```

## 🌊 Flood Fill / Connected Components

**Idea:** Each DFS from an unvisited land cell sinks one whole island, so the number of DFS starts is the island count. For Surrounded Regions, start from the border instead: anything reachable from a border `'O'` is safe, everything else is captured.

```javascript
// Number of islands (DFS version; same as numIslandsDFS above)
function numIslands(grid) {
    if (!grid || grid.length === 0 || grid[0].length === 0) return 0;

    const rows = grid.length;
    const cols = grid[0].length;
    let count = 0;

    function dfs(i, j) {
        if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] === '0') return;

        grid[i][j] = '0'; // Mark as visited

        dfs(i - 1, j);
        dfs(i + 1, j);
        dfs(i, j - 1);
        dfs(i, j + 1);
    }

    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (grid[i][j] === '1') {
                count++;
                dfs(i, j);
            }
        }
    }

    return count;
}

// Surrounded regions (capture regions)
function solve(board) {
    if (!board || board.length === 0 || board[0].length === 0) return;

    const rows = board.length;
    const cols = board[0].length;

    function dfs(i, j) {
        if (i < 0 || i >= rows || j < 0 || j >= cols || board[i][j] !== 'O') return;

        board[i][j] = 'M'; // Mark as safe

        dfs(i - 1, j);
        dfs(i + 1, j);
        dfs(i, j - 1);
        dfs(i, j + 1);
    }

    // Mark border 'O's and their connected 'O's as safe
    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if ((i === 0 || i === rows - 1 || j === 0 || j === cols - 1) && board[i][j] === 'O') {
                dfs(i, j);
            }
        }
    }

    // Flip remaining 'O's to 'X' and marked 'O's back to 'O'
    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (board[i][j] === 'O') {
                board[i][j] = 'X';
            } else if (board[i][j] === 'M') {
                board[i][j] = 'O'; // 'M' was our marker
            }
        }
    }
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