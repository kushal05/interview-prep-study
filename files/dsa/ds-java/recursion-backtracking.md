# Recursion & Backtracking in Java

> Java implementation reference for backtracking: subsets, combinations, permutations, N-Queens, grid DFS and duplicate pruning.
> New to this topic? Learn it first in [04. Recursion](../learn/04-recursion.md) and [18. Backtracking](../learn/18-backtracking.md).

## When to Use

- Problem says "return **all** combinations/permutations/subsets/solutions"
- You need to explore every valid configuration
- Solution space is exponential but pruning can help

## Backtracking Template

```
backtrack(path, choices):
    if isComplete(path):
        result.add(copy of path)    // MUST copy
        return
    for choice in choices:
        if isValid(choice):
            path.add(choice)         // choose
            backtrack(updated)       // explore
            path.removeLast()        // un-choose (backtrack)
```

> **Key Interview Point:** Backtracking = DFS + undo. The three steps are: choose, explore, un-choose. Always make a copy when adding to results.

---

## Pattern 1: Subsets / Combinations

**Idea:** Each call records the current path, then tries adding every element from `start` onward. Passing `i + 1` means each element is used at most once and order is ignored; passing `i` (Combination Sum) allows reuse of the same element.

```java
// All subsets -- Time: O(n * 2^n) (2^n subsets, O(n) to copy each), Space: O(n) recursion depth
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}
private void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path)); // add copy of current subset
    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, i + 1, path, result); // i+1 avoids reusing same element
        path.remove(path.size() - 1);
    }
}

// Combination Sum -- reuse allowed -- Time: O(2^target)
public List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    combBacktrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}
private void combBacktrack(int[] cands, int target, int start, List<Integer> path, List<List<Integer>> result) {
    if (target == 0) { result.add(new ArrayList<>(path)); return; }
    for (int i = start; i < cands.length; i++) {
        if (cands[i] > target) break; // prune: sorted, so all after are too large
        path.add(cands[i]);
        combBacktrack(cands, target - cands[i], i, path, result); // i, not i+1 (reuse allowed)
        path.remove(path.size() - 1);
    }
}
```

---

## Pattern 2: Permutations

**Idea:** Order matters, so every position may take any element not yet used. A `used[]` array marks which indices are already in the path; un-mark on the way back.

```java
// All permutations -- Time: O(n * n!) (n! permutations, O(n) to copy each), Space: O(n)
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    permuteBacktrack(nums, new boolean[nums.length], new ArrayList<>(), result);
    return result;
}
private void permuteBacktrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) { result.add(new ArrayList<>(path)); return; }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue; // skip already used
        used[i] = true;
        path.add(nums[i]);
        permuteBacktrack(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

> **Common Mistake:** Checking `path.contains(num)` to skip used elements. It is O(n) per check and breaks when the input has duplicate values. Use a `boolean[] used` array indexed by position.

---

## Pattern 3: N-Queens

**Idea:** Place one queen per row. For each column in the current row, check the column and both diagonals above; if safe, place it, recurse to the next row, then remove it.

```java
// Time: O(n!), Space: O(n^2)
public List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    char[][] board = new char[n][n];
    for (char[] row : board) Arrays.fill(row, '.');
    solveNQ(board, 0, result);
    return result;
}
private void solveNQ(char[][] board, int row, List<List<String>> result) {
    if (row == board.length) {
        List<String> snapshot = new ArrayList<>();
        for (char[] r : board) snapshot.add(new String(r));
        result.add(snapshot);
        return;
    }
    for (int col = 0; col < board.length; col++) {
        if (isValid(board, row, col)) {
            board[row][col] = 'Q';
            solveNQ(board, row + 1, result);
            board[row][col] = '.'; // backtrack
        }
    }
}
private boolean isValid(char[][] board, int row, int col) {
    for (int i = 0; i < row; i++) if (board[i][col] == 'Q') return false;
    for (int i = row-1, j = col-1; i >= 0 && j >= 0; i--, j--) if (board[i][j] == 'Q') return false;
    for (int i = row-1, j = col+1; i >= 0 && j < board.length; i--, j++) if (board[i][j] == 'Q') return false;
    return true;
}
```

---

## Pattern 4: Recursive DFS (Word Search, Phone Combos)

**Idea:** Grid backtracking: temporarily overwrite the current cell (`'#'`) so the path cannot reuse it, explore the four neighbours, then restore it. Phone combos: one recursion level per digit, one branch per letter.

```java
// Word search -- Time: O(m*n*4^L), Space: O(L) where L=word length
public boolean exist(char[][] board, String word) {
    for (int i = 0; i < board.length; i++)
        for (int j = 0; j < board[0].length; j++)
            if (board[i][j] == word.charAt(0) && dfs(board, word, i, j, 0)) return true;
    return false;
}
private boolean dfs(char[][] board, String word, int i, int j, int idx) {
    if (idx == word.length()) return true;
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] != word.charAt(idx)) return false;
    char tmp = board[i][j];
    board[i][j] = '#'; // mark visited
    boolean found = dfs(board, word, i-1, j, idx+1) || dfs(board, word, i+1, j, idx+1)
                 || dfs(board, word, i, j-1, idx+1) || dfs(board, word, i, j+1, idx+1);
    board[i][j] = tmp; // restore (backtrack)
    return found;
}

// Letter combinations of phone number
public List<String> letterCombinations(String digits) {
    if (digits.isEmpty()) return new ArrayList<>();
    String[] map = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
    List<String> result = new ArrayList<>();
    phoneDFS(digits.toCharArray(), map, 0, new StringBuilder(), result);
    return result;
}
private void phoneDFS(char[] digits, String[] map, int idx, StringBuilder sb, List<String> result) {
    if (idx == digits.length) { result.add(sb.toString()); return; }
    for (char c : map[digits[idx] - '0'].toCharArray()) {
        sb.append(c);
        phoneDFS(digits, map, idx + 1, sb, result);
        sb.deleteCharAt(sb.length() - 1);
    }
}
```

---

## Pattern 5: Constraint Pruning (Skip Duplicates)

**Idea:** Sort first so equal values sit together and anything too large can `break` the loop. At one recursion level, skip a value equal to the previous one, since it would build the same combinations again.

```java
// Combination Sum II -- each candidate used at most once, skip duplicates
public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    comb2(candidates, target, 0, new ArrayList<>(), result);
    return result;
}
private void comb2(int[] c, int target, int start, List<Integer> path, List<List<Integer>> result) {
    if (target == 0) { result.add(new ArrayList<>(path)); return; }
    for (int i = start; i < c.length; i++) {
        if (i > start && c[i] == c[i-1]) continue; // skip duplicates at same level
        if (c[i] > target) break;
        path.add(c[i]);
        comb2(c, target - c[i], i + 1, path, result);
        path.remove(path.size() - 1);
    }
}

// Palindrome partitioning
public List<List<String>> partition(String s) {
    List<List<String>> result = new ArrayList<>();
    partBacktrack(s.toCharArray(), 0, new ArrayList<>(), result);
    return result;
}
private void partBacktrack(char[] chars, int start, List<String> path, List<List<String>> result) {
    if (start == chars.length) { result.add(new ArrayList<>(path)); return; }
    for (int end = start + 1; end <= chars.length; end++) {
        String sub = new String(chars, start, end - start);
        if (isPalin(sub)) {
            path.add(sub);
            partBacktrack(chars, end, path, result);
            path.remove(path.size() - 1);
        }
    }
}
private boolean isPalin(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) if (s.charAt(l++) != s.charAt(r--)) return false;
    return true;
}
```

> **Key Interview Point:** To skip duplicates in backtracking: sort first, then `if (i > start && arr[i] == arr[i-1]) continue`. The `i > start` check ensures we only skip at the same recursion level.

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 78 | Subsets | Backtracking | Medium |
| 46 | Permutations | Backtracking | Medium |
| 39 | Combination Sum | Backtracking | Medium |
| 40 | Combination Sum II | Pruning | Medium |
| 17 | Letter Combinations of Phone | DFS | Medium |
| 79 | Word Search | Grid DFS | Medium |
| 131 | Palindrome Partitioning | Backtracking | Medium |
| 51 | N-Queens | Constraint Backtracking | Hard |
| 37 | Sudoku Solver | Constraint Backtracking | Hard |
| 22 | Generate Parentheses | Backtracking | Medium |
