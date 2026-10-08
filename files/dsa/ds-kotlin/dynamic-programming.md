# Dynamic Programming in Kotlin

> Kotlin implementation reference for dynamic programming: 1D, grid, knapsack-style, string DP, and top-down memoization.
> New to this topic? Learn it first in [20. Dynamic Programming 1](../learn/20-dynamic-programming-1.md).

## When to Use DP
1. Problem asks for **max/min/count/can-do**
2. **Optimal substructure** + **overlapping subproblems**

## Approach
1. Define state: `dp[i]` = ?
2. Recurrence: `dp[i] = f(dp[i-1], ...)`
3. Base case: `dp[0] = ?`
4. Bottom-up or top-down with memoization
5. Space optimize if only previous states needed

> **Key Interview Point:** Kotlin's `maxOf()`, `minOf()`, and `IntArray(n) { init }` initializer make DP code cleaner.

> **Common Mistake:** In 0/1 knapsack with 1D array, iterate `downTo` (backwards). Forward iteration reuses items.

---

## Pattern 1: 1D Sequence DP

**Idea:** The answer at position `i` depends only on a few earlier positions (`i-1`, `i-2`). Store those in variables instead of a full array to get O(1) space.

```kotlin
// House Robber
fun rob(nums: IntArray): Int {
    if (nums.isEmpty()) return 0
    if (nums.size == 1) return nums[0]

    var prev2 = 0
    var prev1 = nums[0]

    for (i in 1 until nums.size) {
        val current = maxOf(prev1, prev2 + nums[i])
        prev2 = prev1
        prev1 = current
    }

    return prev1
}

// Climbing Stairs
fun climbStairs(n: Int): Int {
    if (n <= 2) return n

    var prev2 = 1
    var prev1 = 2

    for (i in 3..n) {
        val current = prev1 + prev2
        prev2 = prev1
        prev1 = current
    }

    return prev1
}

// Maximum Subarray
fun maxSubarray(nums: IntArray): Int {
    var maxSum = nums[0]
    var currentSum = nums[0]

    for (i in 1 until nums.size) {
        currentSum = maxOf(nums[i], currentSum + nums[i])
        maxSum = maxOf(maxSum, currentSum)
    }

    return maxSum
}

// Jump Game
fun canJump(nums: IntArray): Boolean {
    var maxReach = 0

    for (i in nums.indices) {
        if (i > maxReach) return false

        maxReach = maxOf(maxReach, i + nums[i])
        if (maxReach >= nums.size - 1) return true
    }

    return true
}
```

## 🧱🧱 DP on Grids (2D)

**Idea:** `dp[i][j]` is the answer for the cell, built from the cell above and the cell to the left (the only ways to arrive). Coin Change is included here too: `dp[i]` = fewest coins for amount `i`, trying every coin as the last one.

```kotlin
// Unique Paths
fun uniquePaths(m: Int, n: Int): Int {
    val dp = Array(m) { IntArray(n) { 1 } }

    for (i in 1 until m) {
        for (j in 1 until n) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1]
        }
    }

    return dp[m - 1][n - 1]
}

// Minimum Path Sum
fun minPathSum(grid: Array<IntArray>): Int {
    if (grid.isEmpty() || grid[0].isEmpty()) return 0

    val m = grid.size
    val n = grid[0].size

    val dp = Array(m) { IntArray(n) }

    // Initialize first cell
    dp[0][0] = grid[0][0]

    // Initialize first row
    for (j in 1 until n) {
        dp[0][j] = dp[0][j - 1] + grid[0][j]
    }

    // Initialize first column
    for (i in 1 until m) {
        dp[i][0] = dp[i - 1][0] + grid[i][0]
    }

    // Fill the rest
    for (i in 1 until m) {
        for (j in 1 until n) {
            dp[i][j] = grid[i][j] + minOf(dp[i - 1][j], dp[i][j - 1])
        }
    }

    return dp[m - 1][n - 1]
}

// Coin Change (Minimum coins to make amount)
fun coinChange(coins: IntArray, amount: Int): Int {
    val dp = IntArray(amount + 1) { amount + 1 }
    dp[0] = 0

    for (i in 1..amount) {
        for (coin in coins) {
            if (coin <= i) {
                dp[i] = minOf(dp[i], dp[i - coin] + 1)
            }
        }
    }

    return if (dp[amount] == amount + 1) -1 else dp[amount]
}
```

## 🔁 DP with State Transitions

**Idea:** Each item is either taken or skipped, and `dp` tracks which sums/capacities are reachable. In the 1D version, loop capacity backwards so an item is used at most once.

```kotlin
// Partition Equal Subset Sum
fun canPartition(nums: IntArray): Boolean {
    val sum = nums.sum()
    if (sum % 2 != 0) return false

    val target = sum / 2
    val dp = BooleanArray(target + 1)
    dp[0] = true

    for (num in nums) {
        for (j in target downTo num) {
            dp[j] = dp[j] || dp[j - num]
        }
    }

    return dp[target]
}

// Target Sum (with + and - operators)
fun findTargetSumWays(nums: IntArray, target: Int): Int {
    val sum = nums.sum()
    if (Math.abs(target) > sum) return 0

    val offset = sum
    var dp = IntArray(2 * sum + 1) { 0 }
    dp[offset] = 1

    for (num in nums) {
        val nextDp = IntArray(2 * sum + 1) { 0 }

        for (i in dp.indices) {
            if (dp[i] > 0) {
                nextDp[i + num] += dp[i]
                nextDp[i - num] += dp[i]
            }
        }

        dp = nextDp
    }

    return dp[offset + target]
}

// 0/1 Knapsack
fun knapsack(weights: IntArray, values: IntArray, capacity: Int): Int {
    val n = weights.size
    val dp = Array(n + 1) { IntArray(capacity + 1) }

    for (i in 1..n) {
        for (w in 0..capacity) {
            if (weights[i - 1] <= w) {
                dp[i][w] = maxOf(
                    dp[i - 1][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                )
            } else {
                dp[i][w] = dp[i - 1][w]
            }
        }
    }

    return dp[n][capacity]
}
```

## 🧵 DP on Strings

**Idea:** `dp[i][j]` describes the answer for prefixes `text1[0 until i]` and `text2[0 until j]` (or the substring `s[i..j]`). Matching characters extend the diagonal; otherwise take the best of dropping one character.

```kotlin
// Longest Common Subsequence
fun longestCommonSubsequence(text1: String, text2: String): Int {
    val m = text1.length
    val n = text2.length
    val dp = Array(m + 1) { IntArray(n + 1) }

    for (i in 1..m) {
        for (j in 1..n) {
            if (text1[i - 1] == text2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1] + 1
            } else {
                dp[i][j] = maxOf(dp[i - 1][j], dp[i][j - 1])
            }
        }
    }

    return dp[m][n]
}

// Edit Distance
fun minDistance(word1: String, word2: String): Int {
    val m = word1.length
    val n = word2.length
    val dp = Array(m + 1) { IntArray(n + 1) }

    // Initialize base cases
    for (i in 0..m) dp[i][0] = i
    for (j in 0..n) dp[0][j] = j

    for (i in 1..m) {
        for (j in 1..n) {
            if (word1[i - 1] == word2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1]
            } else {
                dp[i][j] = 1 + minOf(
                    dp[i - 1][j],     // Delete
                    dp[i][j - 1],     // Insert
                    dp[i - 1][j - 1]  // Replace
                )
            }
        }
    }

    return dp[m][n]
}

// Longest Palindromic Subsequence
fun longestPalindromeSubseq(s: String): Int {
    val n = s.length
    val dp = Array(n) { IntArray(n) }

    // Base case: single characters are palindromes
    for (i in 0 until n) {
        dp[i][i] = 1
    }

    for (length in 2..n) {
        for (i in 0 until n - length + 1) {
            val j = i + length - 1

            if (s[i] == s[j]) {
                dp[i][j] = dp[i + 1][j - 1] + 2
            } else {
                dp[i][j] = maxOf(dp[i + 1][j], dp[i][j - 1])
            }
        }
    }

    return dp[0][n - 1]
}
```

## 🧠 Memoization (Top-Down DP)

**Idea:** Write the plain recursion first, then cache each subproblem's result in a map so it is computed only once. This turns exponential recursion into polynomial time.

```kotlin
// Fibonacci with memoization
fun fib(n: Int): Int {
    val memo = mutableMapOf<Int, Int>()
    return fibHelper(n, memo)
}

private fun fibHelper(n: Int, memo: MutableMap<Int, Int>): Int {
    if (n <= 1) return n
    if (n in memo) return memo[n]!!

    memo[n] = fibHelper(n - 1, memo) + fibHelper(n - 2, memo)
    return memo[n]!!
}

// Word Break
fun wordBreak(s: String, wordDict: List<String>): Boolean {
    val wordSet = wordDict.toSet()
    val memo = mutableMapOf<Int, Boolean>()
    return wordBreakHelper(s, wordSet, 0, memo)
}

private fun wordBreakHelper(s: String, wordSet: Set<String>, start: Int, memo: MutableMap<Int, Boolean>): Boolean {
    if (start == s.length) return true
    if (start in memo) return memo[start]!!

    for (end in start + 1..s.length) {
        val word = s.substring(start, end)
        if (word in wordSet && wordBreakHelper(s, wordSet, end, memo)) {
            memo[start] = true
            return true
        }
    }

    memo[start] = false
    return false
}

// Can Jump (with memoization)
fun canJump(nums: IntArray): Boolean {
    return canJumpHelper(nums, 0, mutableMapOf())
}

private fun canJumpHelper(nums: IntArray, position: Int, memo: MutableMap<Int, Boolean>): Boolean {
    if (position == nums.size - 1) return true
    if (position in memo) return memo[position]!!

    val furthestJump = minOf(position + nums[position], nums.size - 1)

    for (nextPosition in position + 1..furthestJump) {
        if (canJumpHelper(nums, nextPosition, memo)) {
            memo[position] = true
            return true
        }
    }

    memo[position] = false
    return false
}
```
