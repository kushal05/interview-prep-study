# Dynamic Programming in Swift

> Swift implementation reference for dynamic programming: 1D, grid, knapsack-style, string DP, and memoized recursion.
> New to this topic? Learn it first in [20. Dynamic Programming 1](../learn/20-dynamic-programming-1.md).

## When to Use DP
1. Problem asks for **max/min/count/can-do**
2. **Optimal substructure** + **overlapping subproblems**

## Approach
1. Define state: `dp[i]` = ?
2. Recurrence: `dp[i] = f(dp[i-1], ...)`
3. Base case: `dp[0] = ?`
4. Bottom-up or top-down with `inout` memo dictionary
5. Space optimize when only previous states needed

> **Key Interview Point:** Swift uses `guard` for early returns, `stride(from:through:by:)` for reverse iteration, and `Array(repeating:count:)` for initialization.

> **Common Mistake:** `stride(from: target, through: num, by: -1)` includes both endpoints. For exclusive, use `stride(from:to:by:)`.

> **Swift Range Gotcha:** `a...b` traps at runtime when `b < a` (e.g. `1...0`). Guard empty inputs before loops like `for i in 1...m`, or use `stride`, which simply produces nothing.

---

## Pattern 1: 1D Sequence DP

**Idea:** The answer at position `i` depends only on a few earlier positions (`i-1`, `i-2`), so keep those in variables instead of a full array. House Robber: either skip house `i` (`prev1`) or rob it (`prev2 + nums[i]`).

```swift
// House Robber
func rob(_ nums: [Int]) -> Int {
    guard !nums.isEmpty else { return 0 }
    guard nums.count > 1 else { return nums[0] }

    var prev2 = 0
    var prev1 = nums[0]

    for i in 1..<nums.count {
        let current = max(prev1, prev2 + nums[i])
        prev2 = prev1
        prev1 = current
    }

    return prev1
}

// Climbing Stairs
func climbStairs(_ n: Int) -> Int {
    guard n > 2 else { return n }

    var prev2 = 1
    var prev1 = 2

    for _ in 3...n {
        let current = prev1 + prev2
        prev2 = prev1
        prev1 = current
    }

    return prev1
}

// Maximum Subarray
func maxSubarray(_ nums: [Int]) -> Int {
    var maxSum = nums[0]
    var currentSum = nums[0]

    for i in 1..<nums.count {
        currentSum = max(nums[i], currentSum + nums[i])
        maxSum = max(maxSum, currentSum)
    }

    return maxSum
}

// Jump Game
func canJump(_ nums: [Int]) -> Bool {
    var maxReach = 0

    for i in 0..<nums.count {
        if i > maxReach { return false }

        maxReach = max(maxReach, i + nums[i])
        if maxReach >= nums.count - 1 { return true }
    }

    return true
}
```

## 🧱🧱 DP on Grids (2D)

**Idea:** `dp[i][j]` is the answer for the cell, built from the cells you can arrive from (usually top and left). Fill the first row and column as base cases, then the rest row by row. Coin Change is grouped here but is 1D: `dp[a]` = fewest coins for amount `a`.

```swift
// Unique Paths
func uniquePaths(_ m: Int, _ n: Int) -> Int {
    var dp = Array(repeating: Array(repeating: 1, count: n), count: m)

    for i in 1..<m {
        for j in 1..<n {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1]
        }
    }

    return dp[m - 1][n - 1]
}

// Minimum Path Sum
func minPathSum(_ grid: [[Int]]) -> Int {
    guard !grid.isEmpty, !grid[0].isEmpty else { return 0 }

    let m = grid.count
    let n = grid[0].count
    var dp = Array(repeating: Array(repeating: 0, count: n), count: m)

    // Initialize first cell
    dp[0][0] = grid[0][0]

    // Initialize first row
    for j in 1..<n {
        dp[0][j] = dp[0][j - 1] + grid[0][j]
    }

    // Initialize first column
    for i in 1..<m {
        dp[i][0] = dp[i - 1][0] + grid[i][0]
    }

    // Fill the rest
    for i in 1..<m {
        for j in 1..<n {
            dp[i][j] = grid[i][j] + min(dp[i - 1][j], dp[i][j - 1])
        }
    }

    return dp[m - 1][n - 1]
}

// Coin Change (Minimum coins to make amount)
func coinChange(_ coins: [Int], _ amount: Int) -> Int {
    guard amount > 0 else { return 0 } // 1...0 would trap
    var dp = Array(repeating: amount + 1, count: amount + 1)
    dp[0] = 0

    for i in 1...amount {
        for coin in coins {
            if coin <= i {
                dp[i] = min(dp[i], dp[i - coin] + 1)
            }
        }
    }

    return dp[amount] == amount + 1 ? -1 : dp[amount]
}
```

## 🔁 DP with State Transitions

**Idea:** For each item, decide take or skip. In the 1D 0/1 knapsack form, iterate capacity from high to low so each item is used at most once (going low to high would let it be reused).

```swift
// Partition Equal Subset Sum
func canPartition(_ nums: [Int]) -> Bool {
    let sum = nums.reduce(0, +)
    guard sum % 2 == 0 else { return false }

    let target = sum / 2
    var dp = Array(repeating: false, count: target + 1)
    dp[0] = true

    for num in nums {
        for j in stride(from: target, through: num, by: -1) {
            dp[j] = dp[j] || dp[j - num]
        }
    }

    return dp[target]
}

// Target Sum (with + and - operators)
func findTargetSumWays(_ nums: [Int], _ target: Int) -> Int {
    let sum = nums.reduce(0, +)
    guard abs(target) <= sum else { return 0 }

    let offset = sum
    var dp = Array(repeating: 0, count: 2 * sum + 1)
    dp[offset] = 1

    for num in nums {
        var nextDp = Array(repeating: 0, count: 2 * sum + 1)

        for i in dp.indices {
            if dp[i] > 0 {
                nextDp[i + num] += dp[i]
                nextDp[i - num] += dp[i]
            }
        }

        dp = nextDp
    }

    return dp[offset + target]
}

// 0/1 Knapsack
func knapsack(_ weights: [Int], _ values: [Int], _ capacity: Int) -> Int {
    let n = weights.count
    guard n > 0 else { return 0 }
    var dp = Array(repeating: Array(repeating: 0, count: capacity + 1), count: n + 1)

    for i in 1...n {
        for w in 0...capacity {
            if weights[i - 1] <= w {
                dp[i][w] = max(
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

**Idea:** `dp[i][j]` describes prefixes `text1[0..<i]` and `text2[0..<j]` (or substring `s[i...j]` for palindromes). If the current characters match, extend the diagonal; otherwise take the best of dropping a character from one side.

```swift
// Longest Common Subsequence
func longestCommonSubsequence(_ text1: String, _ text2: String) -> Int {
    let m = text1.count
    let n = text2.count
    let chars1 = Array(text1)
    let chars2 = Array(text2)
    guard m > 0, n > 0 else { return 0 }

    var dp = Array(repeating: Array(repeating: 0, count: n + 1), count: m + 1)

    for i in 1...m {
        for j in 1...n {
            if chars1[i - 1] == chars2[j - 1] {
                dp[i][j] = dp[i - 1][j - 1] + 1
            } else {
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])
            }
        }
    }

    return dp[m][n]
}

// Edit Distance
func minDistance(_ word1: String, _ word2: String) -> Int {
    let m = word1.count
    let n = word2.count
    let chars1 = Array(word1)
    let chars2 = Array(word2)
    guard m > 0, n > 0 else { return max(m, n) } // empty word: insert/delete everything

    var dp = Array(repeating: Array(repeating: 0, count: n + 1), count: m + 1)

    // Initialize base cases
    for i in 0...m { dp[i][0] = i }
    for j in 0...n { dp[0][j] = j }

    for i in 1...m {
        for j in 1...n {
            if chars1[i - 1] == chars2[j - 1] {
                dp[i][j] = dp[i - 1][j - 1]
            } else {
                dp[i][j] = 1 + min(
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
func longestPalindromeSubseq(_ s: String) -> Int {
    let n = s.count
    let chars = Array(s)
    guard n > 1 else { return n } // 2...1 would trap
    var dp = Array(repeating: Array(repeating: 0, count: n), count: n)

    // Base case: single characters are palindromes
    for i in 0..<n {
        dp[i][i] = 1
    }

    for length in 2...n {
        for i in 0...(n - length) {
            let j = i + length - 1

            if chars[i] == chars[j] {
                dp[i][j] = dp[i + 1][j - 1] + 2
            } else {
                dp[i][j] = max(dp[i + 1][j], dp[i][j - 1])
            }
        }
    }

    return dp[0][n - 1]
}
```

## 🧠 Memoization (Top-Down DP)

**Idea:** Write the plain recursion first, then cache each subproblem's result (keyed by its parameters) so it is computed only once. Pass the memo as `inout` so all calls share it.

```swift
// Fibonacci with memoization
func fib(_ n: Int) -> Int {
    var memo = [Int: Int]()
    return fibHelper(n, &memo)
}

private func fibHelper(_ n: Int, _ memo: inout [Int: Int]) -> Int {
    if n <= 1 { return n }
    if let result = memo[n] { return result }

    memo[n] = fibHelper(n - 1, &memo) + fibHelper(n - 2, &memo)
    return memo[n]!
}

// Word Break
func wordBreak(_ s: String, _ wordDict: [String]) -> Bool {
    let wordSet = Set(wordDict)
    var memo = [Int: Bool]()
    return wordBreakHelper(Array(s), wordSet, 0, &memo) // convert once, not per call
}

private func wordBreakHelper(_ chars: [Character], _ wordSet: Set<String>, _ start: Int, _ memo: inout [Int: Bool]) -> Bool {
    if start == chars.count { return true }
    if let result = memo[start] { return result }

    for end in (start + 1)...chars.count {
        let word = String(chars[start..<end])
        if wordSet.contains(word) && wordBreakHelper(chars, wordSet, end, &memo) {
            memo[start] = true
            return true
        }
    }

    memo[start] = false
    return false
}

// Can Jump (with memoization)
func canJumpMemo(_ nums: [Int]) -> Bool {
    var memo = [Int: Bool]()
    return canJumpHelper(nums, 0, &memo)
}

private func canJumpHelper(_ nums: [Int], _ position: Int, _ memo: inout [Int: Bool]) -> Bool {
    if position == nums.count - 1 { return true }
    if let result = memo[position] { return result }

    let furthestJump = min(position + nums[position], nums.count - 1)
    guard furthestJump > position else { memo[position] = false; return false } // nums[position] == 0

    for nextPosition in (position + 1)...furthestJump {
        if canJumpHelper(nums, nextPosition, &memo) {
            memo[position] = true
            return true
        }
    }

    memo[position] = false
    return false
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