# Recursion & Backtracking in JavaScript

> JavaScript implementation reference for recursion and backtracking: the choose/explore/un-choose template, subsets, permutations, combination sums, N-Queens, grid DFS and pruning.
> New to this topic? Learn it first in [04. Recursion](../learn/04-recursion.md) and [18. Backtracking](../learn/18-backtracking.md).

## When to Use

- "Return **all** combinations/permutations/subsets"
- Need to explore every valid configuration
- Solution space is exponential

## Backtracking Template
```
backtrack(path, choices):
    if isComplete(path):
        result.push([...path])  // SPREAD to copy!
        return
    for choice of choices:
        path.push(choice)        // choose
        backtrack(updated)       // explore
        path.pop()               // un-choose
```

> **Key Interview Point:** In JS, always use `[...path]` or `path.slice()` when adding to results. Arrays are reference types -- pushing `path` directly means all results point to the same (empty) array.

> **Common Mistake:** Forgetting to copy the path when adding to results. `result.push(path)` stores a reference, not a snapshot.

---

## Pattern 1: Subsets / Combinations

**Idea:** Build the answer one choice at a time. For subsets/combinations, pass a `start` index so each element is only considered after the previous one (no reordered duplicates); for permutations, any unused element may come next, so track a `used` array instead.

```javascript
// Subsets
function subsets(nums) {
    const result = [];
    backtrackSubsets(nums, 0, [], result);
    return result;
}

function backtrackSubsets(nums, start, current, result) {
    result.push([...current]);

    for (let i = start; i < nums.length; i++) {
        current.push(nums[i]);
        backtrackSubsets(nums, i + 1, current, result);
        current.pop();
    }
}

// Permutations
function permute(nums) {
    const result = [];
    const used = new Array(nums.length).fill(false);
    backtrackPermute(nums, used, [], result);
    return result;
}

function backtrackPermute(nums, used, current, result) {
    if (current.length === nums.length) {
        result.push([...current]);
        return;
    }

    for (let i = 0; i < nums.length; i++) {
        if (used[i]) continue;

        used[i] = true;
        current.push(nums[i]);
        backtrackPermute(nums, used, current, result);
        current.pop();
        used[i] = false;
    }
}

// Combination Sum
function combinationSum(candidates, target) {
    const result = [];
    candidates.sort((a, b) => a - b);
    backtrackCombinationSum(candidates, target, 0, [], result);
    return result;
}

function backtrackCombinationSum(candidates, target, start, current, result) {
    if (target === 0) {
        result.push([...current]);
        return;
    }

    for (let i = start; i < candidates.length; i++) {
        if (candidates[i] > target) break;

        current.push(candidates[i]);
        backtrackCombinationSum(candidates, target - candidates[i], i, current, result);
        current.pop();
    }
}

// N-Queens
function solveNQueens(n) {
    const result = [];
    const board = Array.from({ length: n }, () => Array(n).fill('.'));
    backtrackNQueens(board, 0, result);
    return result;
}

function backtrackNQueens(board, row, result) {
    if (row === board.length) {
        result.push(board.map(row => row.join('')));
        return;
    }

    for (let col = 0; col < board.length; col++) {
        if (isValidQueen(board, row, col)) {
            board[row][col] = 'Q';
            backtrackNQueens(board, row + 1, result);
            board[row][col] = '.';
        }
    }
}

function isValidQueen(board, row, col) {
    const n = board.length;

    // Check column
    for (let i = 0; i < row; i++) {
        if (board[i][col] === 'Q') return false;
    }

    // Check diagonal (top-left)
    for (let i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) {
        if (board[i][j] === 'Q') return false;
    }

    // Check diagonal (top-right)
    for (let i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++) {
        if (board[i][j] === 'Q') return false;
    }

    return true;
}
```

## 🧠 Recursive DFS (Stateful Exploration)

**Idea:** Recurse into each neighbour/next choice, temporarily marking the current cell or state as used, and restore it on the way back so other paths can reuse it.

```javascript
// Word Search
function exist(board, word) {
    if (!board || board.length === 0 || board[0].length === 0) return false;

    const rows = board.length;
    const cols = board[0].length;

    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (board[i][j] === word[0] && dfsWordSearch(board, word, i, j, 0)) {
                return true;
            }
        }
    }

    return false;
}

function dfsWordSearch(board, word, i, j, index) {
    if (index === word.length) return true;

    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] !== word[index]) {
        return false;
    }

    const temp = board[i][j];
    board[i][j] = '#'; // Mark as visited

    const found = dfsWordSearch(board, word, i - 1, j, index + 1) ||
                  dfsWordSearch(board, word, i + 1, j, index + 1) ||
                  dfsWordSearch(board, word, i, j - 1, index + 1) ||
                  dfsWordSearch(board, word, i, j + 1, index + 1);

    board[i][j] = temp; // Backtrack

    return found;
}

// Letter combinations of a phone number
function letterCombinations(digits) {
    if (digits.length === 0) return [];

    const phoneMap = {
        '2': 'abc', '3': 'def', '4': 'ghi',
        '5': 'jkl', '6': 'mno', '7': 'pqrs',
        '8': 'tuv', '9': 'wxyz'
    };

    const result = [];
    backtrackLetterCombinations(digits, phoneMap, 0, '', result);
    return result;
}

function backtrackLetterCombinations(digits, phoneMap, index, current, result) {
    if (index === digits.length) {
        result.push(current);
        return;
    }

    const letters = phoneMap[digits[index]];
    for (const letter of letters) {
        backtrackLetterCombinations(digits, phoneMap, index + 1, current + letter, result);
    }
}
```

## ⛔ Constraint-Based Backtracking

**Idea:** Prune branches that cannot succeed. Sorting first lets you `break` once a candidate exceeds the target and skip equal neighbours (`i > start && a[i] === a[i-1]`) to avoid duplicate results.

```javascript
// Combination Sum II (with duplicates in candidates)
function combinationSum2(candidates, target) {
    const result = [];
    candidates.sort((a, b) => a - b);
    backtrackCombinationSum2(candidates, target, 0, [], result);
    return result;
}

function backtrackCombinationSum2(candidates, target, start, current, result) {
    if (target === 0) {
        result.push([...current]);
        return;
    }

    for (let i = start; i < candidates.length; i++) {
        if (i > start && candidates[i] === candidates[i - 1]) continue;
        if (candidates[i] > target) break;

        current.push(candidates[i]);
        backtrackCombinationSum2(candidates, target - candidates[i], i + 1, current, result);
        current.pop();
    }
}

// Palindrome Partitioning
function partition(s) {
    const result = [];
    backtrackPartition(s, 0, [], result);
    return result;
}

function backtrackPartition(s, start, current, result) {
    if (start === s.length) {
        result.push([...current]);
        return;
    }

    for (let end = start + 1; end <= s.length; end++) {
        const substring = s.substring(start, end);
        if (isPalindrome(substring)) {
            current.push(substring);
            backtrackPartition(s, end, current, result);
            current.pop();
        }
    }
}

function isPalindrome(s) {
    let left = 0;
    let right = s.length - 1;

    while (left < right) {
        if (s[left] !== s[right]) return false;
        left++;
        right--;
    }

    return true;
}
```

---

## Summary: Recursion & Backtracking Pattern Map

| Pattern                     | Keywords / Use Case                        |
|---------------------------|------------------------------------------|
| Backtracking (Try & Undo)  | generate combinations, permutations       |
| Recursive DFS (Count Ways) | explore all options, word search          |
| Constraint Pruning         | skip duplicates, early termination        |
| N-Queens / Sudoku          | constraint satisfaction problems          |

---