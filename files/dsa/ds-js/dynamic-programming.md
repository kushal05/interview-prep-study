# Dynamic Programming in JavaScript

> JavaScript implementation reference for dynamic programming: 1D sequence, grid, knapsack-style state transitions, string DP and top-down memoization.
> New to this topic? Learn it first in [20. Dynamic Programming 1](../learn/20-dynamic-programming-1.md) (then [21. Dynamic Programming 2](../learn/21-dynamic-programming-2.md)).

## When to Use DP

1. Problem asks for **max/min/count/can-do**
2. **Optimal substructure:** optimal solution uses optimal sub-solutions
3. **Overlapping subproblems:** same subproblems recomputed

## Approach

1. **Define state:** What does `dp[i]` represent?
2. **Recurrence:** How does `dp[i]` depend on previous states?
3. **Base case:** What is `dp[0]`?
4. **Order:** Bottom-up (tabulation) or top-down (memoization)
5. **Optimize space** when only previous state(s) are needed

> **Key Interview Point:** Start with brute-force recursion. Add memoization (`Map`). Convert to bottom-up for space optimization.

> **Common Mistake:** In 0/1 knapsack with 1D array, iterate capacity **backwards**. Forward iteration allows reusing the same item.

---

## Pattern 1: 1D Sequence DP

**Idea:** The answer at position `i` depends only on a few earlier positions (`i-1`, `i-2`), so keep those in variables instead of a full array. House Robber: either skip house `i` (`prev1`) or rob it (`prev2 + nums[i]`).

```javascript
// House Robber
function rob(nums) {
    if (nums.length === 0) return 0;
    if (nums.length === 1) return nums[0];

    let prev2 = 0;
    let prev1 = nums[0];

    for (let i = 1; i < nums.length; i++) {
        const current = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = current;
    }

    return prev1;
}

// Climbing Stairs
function climbStairs(n) {
    if (n <= 2) return n;

    let prev2 = 1;
    let prev1 = 2;

    for (let i = 3; i <= n; i++) {
        const current = prev1 + prev2;
        prev2 = prev1;
        prev1 = current;
    }

    return prev1;
}

// Maximum Subarray
function maxSubarray(nums) {
    let maxSum = nums[0];
    let currentSum = nums[0];

    for (let i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }

    return maxSum;
}

// Jump Game
function canJump(nums) {
    let maxReach = 0;

    for (let i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;

        maxReach = Math.max(maxReach, i + nums[i]);
        if (maxReach >= nums.length - 1) return true;
    }

    return true;
}
```

## 🧱🧱 DP on Grids (2D)

**Idea:** `dp[i][j]` is the answer for the cell, built from the cells you can arrive from (usually top and left). Fill the first row and column as base cases, then the rest row by row. Coin Change is grouped here but is 1D: `dp[a]` = fewest coins for amount `a`.

```javascript
// Unique Paths
function uniquePaths(m, n) {
    const dp = Array.from({ length: m }, () => Array(n).fill(1));

    for (let i = 1; i < m; i++) {
        for (let j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }

    return dp[m - 1][n - 1];
}

// Minimum Path Sum
function minPathSum(grid) {
    if (!grid || grid.length === 0 || grid[0].length === 0) return 0;

    const m = grid.length;
    const n = grid[0].length;

    const dp = Array.from({ length: m }, () => Array(n).fill(0));

    // Initialize first cell
    dp[0][0] = grid[0][0];

    // Initialize first row
    for (let j = 1; j < n; j++) {
        dp[0][j] = dp[0][j - 1] + grid[0][j];
    }

    // Initialize first column
    for (let i = 1; i < m; i++) {
        dp[i][0] = dp[i - 1][0] + grid[i][0];
    }

    // Fill the rest
    for (let i = 1; i < m; i++) {
        for (let j = 1; j < n; j++) {
            dp[i][j] = grid[i][j] + Math.min(dp[i - 1][j], dp[i][j - 1]);
        }
    }

    return dp[m - 1][n - 1];
}

// Coin Change (Minimum coins to make amount)
function coinChange(coins, amount) {
    const dp = new Array(amount + 1).fill(amount + 1);
    dp[0] = 0;

    for (let i = 1; i <= amount; i++) {
        for (const coin of coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }

    return dp[amount] === amount + 1 ? -1 : dp[amount];
}
```

## 🔁 DP with State Transitions

**Idea:** For each item, decide take or skip. In the 1D 0/1 knapsack form, iterate capacity from high to low so each item is used at most once (going low to high would let it be reused). Target Sum shifts indices by `sum` so negative totals fit in the array.

```javascript
// Partition Equal Subset Sum
function canPartition(nums) {
    const sum = nums.reduce((acc, num) => acc + num, 0);
    if (sum % 2 !== 0) return false;

    const target = sum / 2;
    const dp = new Array(target + 1).fill(false);
    dp[0] = true;

    for (const num of nums) {
        for (let j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }

    return dp[target];
}

// Target Sum (with + and - operators)
function findTargetSumWays(nums, target) {
    const sum = nums.reduce((acc, num) => acc + num, 0);
    if (Math.abs(target) > sum) return 0;

    const offset = sum;
    let dp = new Array(2 * sum + 1).fill(0);
    dp[offset] = 1;

    for (const num of nums) {
        const nextDp = new Array(2 * sum + 1).fill(0);

        for (let i = 0; i < dp.length; i++) {
            if (dp[i] > 0) {
                nextDp[i + num] += dp[i];
                nextDp[i - num] += dp[i];
            }
        }

        dp = nextDp;
    }

    return dp[offset + target];
}

// 0/1 Knapsack
function knapsack(weights, values, capacity) {
    const n = weights.length;
    const dp = Array.from({ length: n + 1 }, () => Array(capacity + 1).fill(0));

    for (let i = 1; i <= n; i++) {
        for (let w = 0; w <= capacity; w++) {
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(
                    dp[i - 1][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                );
            } else {
                dp[i][w] = dp[i - 1][w];
            }
        }
    }

    return dp[n][capacity];
}
```

## 🧵 DP on Strings

**Idea:** `dp[i][j]` describes prefixes `text1[0..i)` and `text2[0..j)` (or the substring `s[i..j]` for palindromes). If the current characters match, extend the diagonal; otherwise take the best of dropping a character from one side.

```javascript
// Longest Common Subsequence
function longestCommonSubsequence(text1, text2) {
    const m = text1.length;
    const n = text2.length;
    const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

    for (let i = 1; i <= m; i++) {
        for (let j = 1; j <= n; j++) {
            if (text1[i - 1] === text2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }

    return dp[m][n];
}

// Edit Distance
function minDistance(word1, word2) {
    const m = word1.length;
    const n = word2.length;
    const dp = Array.from({ length: m + 1 }, () => Array(n + 1).fill(0));

    // Initialize base cases
    for (let i = 0; i <= m; i++) dp[i][0] = i;
    for (let j = 0; j <= n; j++) dp[0][j] = j;

    for (let i = 1; i <= m; i++) {
        for (let j = 1; j <= n; j++) {
            if (word1[i - 1] === word2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j],     // Delete
                    dp[i][j - 1],     // Insert
                    dp[i - 1][j - 1]  // Replace
                );
            }
        }
    }

    return dp[m][n];
}

// Longest Palindromic Subsequence
function longestPalindromeSubseq(s) {
    const n = s.length;
    if (n === 0) return 0; // dp[0] would be undefined below
    const dp = Array.from({ length: n }, () => Array(n).fill(0));

    // Base case: single characters are palindromes
    for (let i = 0; i < n; i++) {
        dp[i][i] = 1;
    }

    for (let length = 2; length <= n; length++) {
        for (let i = 0; i <= n - length; i++) {
            const j = i + length - 1;

            if (s[i] === s[j]) {
                dp[i][j] = dp[i + 1][j - 1] + 2;
            } else {
                dp[i][j] = Math.max(dp[i + 1][j], dp[i][j - 1]);
            }
        }
    }

    return dp[0][n - 1];
}
```

## 🧠 Memoization (Top-Down DP)

**Idea:** Write the plain recursion first, then cache each subproblem's result (keyed by its parameters, here a `Map`) so it is computed only once.

```javascript
// Fibonacci with memoization
function fib(n) {
    const memo = new Map();
    return fibHelper(n, memo);
}

function fibHelper(n, memo) {
    if (n <= 1) return n;
    if (memo.has(n)) return memo.get(n);

    const result = fibHelper(n - 1, memo) + fibHelper(n - 2, memo);
    memo.set(n, result);
    return result;
}

// Word Break
function wordBreak(s, wordDict) {
    const wordSet = new Set(wordDict);
    const memo = new Map();
    return wordBreakHelper(s, wordSet, 0, memo);
}

function wordBreakHelper(s, wordSet, start, memo) {
    if (start === s.length) return true;
    if (memo.has(start)) return memo.get(start);

    for (let end = start + 1; end <= s.length; end++) {
        const word = s.substring(start, end);
        if (wordSet.has(word) && wordBreakHelper(s, wordSet, end, memo)) {
            memo.set(start, true);
            return true;
        }
    }

    memo.set(start, false);
    return false;
}

// Can Jump (with memoization) -- O(n^2) time; the greedy canJump above is O(n).
// Named canJumpMemo: a second `function canJump` would silently replace the first.
function canJumpMemo(nums) {
    return canJumpHelper(nums, 0, new Map());
}

function canJumpHelper(nums, position, memo) {
    if (position === nums.length - 1) return true;
    if (memo.has(position)) return memo.get(position);

    const furthestJump = Math.min(position + nums[position], nums.length - 1);

    for (let nextPosition = position + 1; nextPosition <= furthestJump; nextPosition++) {
        if (canJumpHelper(nums, nextPosition, memo)) {
            memo.set(position, true);
            return true;
        }
    }

    memo.set(position, false);
    return false;
}
```

---

## Summary: DP Pattern Map

| Pattern                   | Keywords / Use Case                       |
|--------------------------|------------------------------------------|
| DP on Sequences (1D)     | max/min/ways, linear problems             |
| DP on Grids (2D)         | path problems on a board                  |
| DP with State Transitions| decisions + constraints (e.g. used items) |
| DP on Strings            | LCS, palindrome, edit distance            |
| Memoization (Top-Down)   | recursive + caching                       |

---