# Trees & Graphs in Java

> Java implementation reference for binary trees, BSTs and graphs: DFS/BFS, iterative traversals, serialization, topological sort, union-find and flood fill.
> New to this topic? Learn it first in [12. Trees and Traversals](../learn/12-trees-and-traversals.md) and [16. Graphs Fundamentals](../learn/16-graphs-fundamentals.md).

## Complexity Overview

### Binary Tree / BST Operations
| Operation | Average (BST) | Worst (BST) | Binary Tree |
|-----------|--------------|-------------|-------------|
| Search | O(log n) | O(n) | O(n) |
| Insert | O(log n) | O(n) | O(n) |
| Delete | O(log n) | O(n) | O(n) |
| Traversal | O(n) | O(n) | O(n) |

### Graph Operations
| Operation | Adjacency List | Adjacency Matrix |
|-----------|---------------|-----------------|
| Add Edge | O(1) | O(1) |
| Remove Edge | O(E) | O(1) |
| Check Edge | O(degree) | O(1) |
| Space | O(V + E) | O(V^2) |
| BFS/DFS | O(V + E) | O(V^2) |

> **Key Interview Point:** Use adjacency list for sparse graphs (most interview problems). Use adjacency matrix only when you need O(1) edge lookups.

---

## Tree Node & BST

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}

class BST {
    TreeNode root;

    // Insert -- O(log n) average, O(n) worst
    TreeNode insert(TreeNode node, int val) {
        if (node == null) return new TreeNode(val);
        if (val < node.val) node.left = insert(node.left, val);
        else node.right = insert(node.right, val);
        return node;
    }

    // Search -- O(log n) average
    boolean search(TreeNode node, int val) {
        if (node == null) return false;
        if (node.val == val) return true;
        return val < node.val ? search(node.left, val) : search(node.right, val);
    }

    // Delete -- O(log n) average
    TreeNode delete(TreeNode node, int val) {
        if (node == null) return null;
        if (val < node.val) node.left = delete(node.left, val);
        else if (val > node.val) node.right = delete(node.right, val);
        else {
            if (node.left == null) return node.right;
            if (node.right == null) return node.left;
            TreeNode successor = node.right;
            while (successor.left != null) successor = successor.left;
            node.val = successor.val;
            node.right = delete(node.right, successor.val);
        }
        return node;
    }
}
```

---

## Tree Pattern 1: DFS (Recursive)

> **Key Interview Point:** Most tree problems are DFS. Ask yourself: do I need info from children (postorder) or do I pass info downward (preorder)?

```java
// Max depth -- Time: O(n), Space: O(h)
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;
}

// Path sum -- does any root-to-leaf path sum to target?
public boolean hasPathSum(TreeNode root, int target) {
    if (root == null) return false;
    if (root.left == null && root.right == null) return root.val == target;
    return hasPathSum(root.left, target - root.val) || hasPathSum(root.right, target - root.val);
}

// Symmetric tree
public boolean isSymmetric(TreeNode root) {
    if (root == null) return true; // empty tree is symmetric (avoids NPE)
    return isMirror(root.left, root.right);
}
private boolean isMirror(TreeNode a, TreeNode b) {
    if (a == null && b == null) return true;
    if (a == null || b == null) return false;
    return a.val == b.val && isMirror(a.left, b.right) && isMirror(a.right, b.left);
}
```

---

## Tree Pattern 2: BFS (Level Order)

**Idea:** Use a queue. At the start of each round, `queue.size()` is exactly the number of nodes on the current level, so process that many nodes, enqueueing their children for the next level.

```java
// Level order traversal -- Time: O(n), Space: O(n)
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}

// Right side view
public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (i == size - 1) result.add(node.val); // last node in level
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }
    return result;
}
```

---

## Tree Pattern 3: Iterative Traversals

**Idea:** An explicit stack replaces the call stack. Inorder: push the whole left spine, pop a node, visit it, then move to its right child. Preorder: pop, visit, push right then left (so left comes out first).

```java
// Inorder (Left, Node, Right) -- gives sorted order for BST
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>(); // prefer ArrayDeque over legacy Stack
    TreeNode curr = root;
    while (curr != null || !stack.isEmpty()) {
        while (curr != null) { stack.push(curr); curr = curr.left; }
        curr = stack.pop();
        result.add(curr.val);
        curr = curr.right;
    }
    return result;
}

// Preorder (Node, Left, Right)
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;
    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);
    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);
        if (node.right != null) stack.push(node.right); // right first so left processed first
        if (node.left != null) stack.push(node.left);
    }
    return result;
}
```

---

## Tree Pattern 4: BST Invariants

**Idea:** Every node must lie strictly inside the `(min, max)` range inherited from its ancestors, not just compare with its parent. Inorder traversal of a BST visits values in sorted order, so the kth visited node is the kth smallest.

```java
// Validate BST -- Time: O(n)
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) && validate(node.right, node.val, max);
}

// Kth smallest in BST -- inorder traversal stops at kth
public int kthSmallest(TreeNode root, int k) {
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;
    int count = 0;
    while (curr != null || !stack.isEmpty()) {
        while (curr != null) { stack.push(curr); curr = curr.left; }
        curr = stack.pop();
        if (++count == k) return curr.val;
        curr = curr.right;
    }
    throw new IllegalArgumentException();
}
```

---

## Tree Pattern 5: Serialization

**Idea:** Write the tree level by level, including `null` markers for missing children, so the shape is recoverable. To rebuild, read values in the same BFS order: each dequeued parent takes the next two tokens as its left and right child.

```java
public String serialize(TreeNode root) {
    if (root == null) return "null";
    List<String> result = new ArrayList<>();
    Queue<TreeNode> queue = new LinkedList<>(); // LinkedList, not ArrayDeque: we enqueue nulls
    queue.offer(root);
    while (!queue.isEmpty()) {
        TreeNode node = queue.poll();
        if (node != null) {
            result.add(String.valueOf(node.val));
            queue.offer(node.left);
            queue.offer(node.right);
        } else result.add("null");
    }
    return String.join(",", result);
}

public TreeNode deserialize(String data) {
    if ("null".equals(data)) return null;
    String[] vals = data.split(",");
    TreeNode root = new TreeNode(Integer.parseInt(vals[0]));
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    for (int i = 1; i < vals.length; ) {
        TreeNode node = queue.poll();
        if (!"null".equals(vals[i])) { node.left = new TreeNode(Integer.parseInt(vals[i])); queue.offer(node.left); }
        i++;
        if (i < vals.length && !"null".equals(vals[i])) { node.right = new TreeNode(Integer.parseInt(vals[i])); queue.offer(node.right); }
        i++;
    }
    return root;
}
```

---

## Graph Implementation

```java
class Graph {
    private List<List<Integer>> adj;
    public Graph(int v) {
        adj = new ArrayList<>();
        for (int i = 0; i < v; i++) adj.add(new ArrayList<>());
    }
    public void addEdge(int u, int v) { adj.get(u).add(v); adj.get(v).add(u); }
    public List<Integer> neighbors(int v) { return adj.get(v); }
}
```

---

## Graph Pattern 1: DFS

**Idea:** Go as deep as possible from a node, marking it visited so it is never processed twice. On a grid, "visited" can be done by overwriting the cell. For directed cycles, also track which nodes are on the current path (`inStack`).

```java
// Number of islands (DFS) -- Time: O(m*n)
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++)
        for (int j = 0; j < grid[0].length; j++)
            if (grid[i][j] == '1') { count++; dfs(grid, i, j); }
    return count;
}
private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] == '0') return;
    grid[i][j] = '0';
    dfs(grid, i-1, j); dfs(grid, i+1, j); dfs(grid, i, j-1); dfs(grid, i, j+1);
}

// Detect cycle in directed graph -- Time: O(V+E)
public boolean hasCycle(List<List<Integer>> graph) {
    boolean[] visited = new boolean[graph.size()], inStack = new boolean[graph.size()];
    for (int i = 0; i < graph.size(); i++)
        if (dfsCycle(graph, i, visited, inStack)) return true;
    return false;
}
private boolean dfsCycle(List<List<Integer>> g, int v, boolean[] visited, boolean[] inStack) {
    if (inStack[v]) return true;
    if (visited[v]) return false;
    visited[v] = inStack[v] = true;
    for (int n : g.get(v)) if (dfsCycle(g, n, visited, inStack)) return true;
    inStack[v] = false;
    return false;
}
```

> **Key Interview Point:** For cycle detection in directed graphs, you need BOTH a visited array AND a recursion stack array. An edge to a node in the current recursion stack means a cycle.

---

## Graph Pattern 2: BFS (Shortest Path)

**Idea:** In an unweighted graph, BFS explores nodes in order of distance, so the level at which you first reach the target is the shortest path length. Multi-source BFS starts with every source in the queue at distance 0.

```java
// Word ladder -- Time: O(M^2 * N), M=word length, N=word count
public int ladderLength(String begin, String end, List<String> wordList) {
    Set<String> words = new HashSet<>(wordList);
    if (!words.contains(end)) return 0;
    Queue<String> queue = new LinkedList<>();
    queue.offer(begin);
    int level = 1;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int s = 0; s < size; s++) {
            char[] word = queue.poll().toCharArray();
            for (int i = 0; i < word.length; i++) {
                char orig = word[i];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == orig) continue;
                    word[i] = c;
                    String next = new String(word);
                    if (next.equals(end)) return level + 1;
                    if (words.remove(next)) queue.offer(next);
                }
                word[i] = orig;
            }
        }
        level++;
    }
    return 0;
}

// Rotten oranges (multi-source BFS)
public int orangesRotting(int[][] grid) {
    int rows = grid.length, cols = grid[0].length, fresh = 0;
    Queue<int[]> queue = new LinkedList<>();
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            if (grid[i][j] == 2) queue.offer(new int[]{i, j});
            else if (grid[i][j] == 1) fresh++;

    int[][] dirs = {{-1,0},{1,0},{0,-1},{0,1}};
    int mins = 0;
    while (!queue.isEmpty() && fresh > 0) {
        int size = queue.size();
        for (int s = 0; s < size; s++) {
            int[] p = queue.poll();
            for (int[] d : dirs) {
                int ni = p[0]+d[0], nj = p[1]+d[1];
                if (ni >= 0 && ni < rows && nj >= 0 && nj < cols && grid[ni][nj] == 1) {
                    grid[ni][nj] = 2; fresh--;
                    queue.offer(new int[]{ni, nj});
                }
            }
        }
        mins++;
    }
    return fresh == 0 ? mins : -1;
}
```

---

## Graph Pattern 3: Topological Sort (Kahn's Algorithm)

**Idea:** Count incoming edges for each node. Start with all nodes of in-degree 0; each time you output one, decrement its neighbours' in-degrees and enqueue any that drop to 0.

```java
// Course Schedule II -- Time: O(V+E)
public int[] findOrder(int numCourses, int[][] prereqs) {
    List<List<Integer>> graph = new ArrayList<>();
    int[] inDegree = new int[numCourses];
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());
    for (int[] p : prereqs) { graph.get(p[1]).add(p[0]); inDegree[p[0]]++; }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) if (inDegree[i] == 0) queue.offer(i);

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int course = queue.poll();
        order.add(course);
        for (int next : graph.get(course))
            if (--inDegree[next] == 0) queue.offer(next);
    }
    return order.size() == numCourses ? order.stream().mapToInt(i->i).toArray() : new int[0];
}
```

> **Key Interview Point:** If topological sort result has fewer nodes than total, there is a cycle -- return empty/false.

---

## Graph Pattern 4: Union-Find

**Idea:** Each set is a tree identified by its root. `find` walks to the root (flattening the path as it goes); `union` links two roots. With path compression and union by rank, both are nearly O(1) amortized. A `union` that returns false means the edge closes a cycle.

```java
class UnionFind {
    private int[] parent, rank;
    public UnionFind(int n) {
        parent = new int[n]; rank = new int[n];
        for (int i = 0; i < n; i++) { parent[i] = i; rank[i] = 1; }
    }
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }
    public boolean union(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;
        if (rank[rx] > rank[ry]) parent[ry] = rx;      // union by rank
        else if (rank[rx] < rank[ry]) parent[rx] = ry;
        else { parent[ry] = rx; rank[rx]++; }
        return true;
    }
}

// Connected components
public int countComponents(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    int components = n;
    for (int[] e : edges) if (uf.union(e[0], e[1])) components--;
    return components;
}

// Redundant connection -- find the edge that creates a cycle
public int[] findRedundantConnection(int[][] edges) {
    UnionFind uf = new UnionFind(edges.length + 1);
    for (int[] e : edges) if (!uf.union(e[0], e[1])) return e;
    return new int[0];
}
```

---

## Graph Pattern 5: Flood Fill

**Idea:** Instead of asking which regions are surrounded, start from the border: DFS from every border `'O'` marks the cells that can escape. Everything still `'O'` afterwards is captured.

```java
// Surrounded regions -- capture all 'O' not connected to border
public void solve(char[][] board) {
    int R = board.length, C = board[0].length;
    // Mark border-connected 'O's as safe
    for (int i = 0; i < R; i++) for (int j = 0; j < C; j++)
        if ((i == 0 || i == R-1 || j == 0 || j == C-1) && board[i][j] == 'O')
            markSafe(board, i, j, R, C);
    // Flip: remaining O->X, safe M->O
    for (int i = 0; i < R; i++) for (int j = 0; j < C; j++)
        if (board[i][j] == 'O') board[i][j] = 'X';
        else if (board[i][j] == 'M') board[i][j] = 'O';
}
private void markSafe(char[][] b, int i, int j, int R, int C) {
    if (i < 0 || i >= R || j < 0 || j >= C || b[i][j] != 'O') return;
    b[i][j] = 'M';
    markSafe(b,i-1,j,R,C); markSafe(b,i+1,j,R,C); markSafe(b,i,j-1,R,C); markSafe(b,i,j+1,R,C);
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 104 | Maximum Depth of Binary Tree | DFS | Easy |
| 101 | Symmetric Tree | DFS | Easy |
| 112 | Path Sum | DFS | Easy |
| 102 | Level Order Traversal | BFS | Medium |
| 98 | Validate BST | BST Invariant | Medium |
| 230 | Kth Smallest in BST | Inorder | Medium |
| 200 | Number of Islands | DFS/BFS | Medium |
| 207 | Course Schedule | Topological Sort | Medium |
| 210 | Course Schedule II | Topological Sort | Medium |
| 994 | Rotting Oranges | Multi-source BFS | Medium |
| 127 | Word Ladder | BFS | Hard |
| 297 | Serialize/Deserialize Binary Tree | BFS/DFS | Hard |
| 323 | Connected Components | Union-Find | Medium |
| 684 | Redundant Connection | Union-Find | Medium |
