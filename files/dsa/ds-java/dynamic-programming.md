# Dynamic Programming in Java

> Java implementation reference for dynamic programming: 1D sequence, 2D grid, knapsack, string DP and top-down memoization.
> New to this topic? Learn it first in [20. Dynamic Programming 1](../learn/20-dynamic-programming-1.md).

## When to Use DP

1. Problem asks for **max/min/count/can-do**
2. Problem has **optimal substructure** (optimal solution uses optimal sub-solutions)
3. Problem has **overlapping subproblems** (same subproblems computed multiple times)

## Approach

1. **Define state:** What does `dp[i]` represent?
2. **Recurrence:** How does `dp[i]` depend on previous states?
3. **Base case:** What is `dp[0]`?
4. **Order:** Fill bottom-up (tabulation) or top-down (memoization)
5. **Optimize space** if only previous state(s) needed

> **Key Interview Point:** Start with brute-force recursion. Identify repeated work. Add memoization. Then convert to bottom-up if the interviewer asks for space optimization.

---

## Pattern 1: 1D Sequence DP

**State:** `dp[i]` = optimal value considering first i elements

```java
// House Robber -- Time: O(n), Space: O(1)
// dp[i] = max(dp[i-1], dp[i-2] + nums[i])
public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    if (nums.length == 1) return nums[0];
    int prev2 = 0, prev1 = nums[0];
    for (int i = 1; i < nums.length; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

// Climbing Stairs -- Time: O(n), Space: O(1)
// dp[i] = dp[i-1] + dp[i-2]  (Fibonacci-like)
public int climbStairs(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2;
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

// Jump Game -- greedy approach, O(n) time O(1) space
public boolean canJump(int[] nums) {
    int maxReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;
        maxReach = Math.max(maxReach, i + nums[i]);
        if (maxReach >= nums.length - 1) return true;
    }
    return true;
}
```

---

## Pattern 2: 2D Grid DP

**State:** `dp[i][j]` = optimal value to reach cell (i,j)

```java
// Unique Paths -- Time: O(m*n), Space: O(m*n)
// dp[i][j] = dp[i-1][j] + dp[i][j-1]
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
    return dp[m-1][n-1];
}

// Minimum Path Sum -- Time: O(m*n), Space: O(m*n)
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];
    for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];
    for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = grid[i][j] + Math.min(dp[i-1][j], dp[i][j-1]);
    return dp[m-1][n-1];
}

// Coin Change (1D, grouped here for comparison) -- Time: O(amount * coins), Space: O(amount)
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // sentinel for "impossible"
    dp[0] = 0;
    for (int i = 1; i <= amount; i++)
        for (int coin : coins)
            if (coin <= i) dp[i] = Math.min(dp[i], dp[i - coin] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}
```

---

## Pattern 3: Knapsack / State Transitions

**Idea:** For each item, decide take or skip. `dp[j]` answers the question for capacity/sum `j`. In the 1D 0/1 form, iterate `j` from high to low so each item is used at most once.

```java
// Partition Equal Subset Sum (0/1 Knapsack variant)
// Time: O(n * sum), Space: O(sum)
public boolean canPartition(int[] nums) {
    int sum = 0;
    for (int n : nums) sum += n;
    if (sum % 2 != 0) return false;
    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int num : nums)
        for (int j = target; j >= num; j--) // iterate backwards!
            dp[j] = dp[j] || dp[j - num];
    return dp[target];
}

// 0/1 Knapsack -- Time: O(n * capacity), Space: O(n * capacity)
public int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];
    for (int i = 1; i <= n; i++)
        for (int w = 0; w <= capacity; w++)
            dp[i][w] = weights[i-1] <= w
                ? Math.max(dp[i-1][w], dp[i-1][w - weights[i-1]] + values[i-1])
                : dp[i-1][w];
    return dp[n][capacity];
}

// Target Sum -- Time: O(n * sum), Space: O(sum)
public int findTargetSumWays(int[] nums, int target) {
    int sum = 0;
    for (int n : nums) sum += n;
    if (Math.abs(target) > sum) return 0;
    int offset = sum;
    int[] dp = new int[2 * sum + 1];
    dp[offset] = 1;
    for (int num : nums) {
        int[] next = new int[2 * sum + 1];
        for (int i = 0; i < dp.length; i++)
            if (dp[i] > 0) { next[i + num] += dp[i]; next[i - num] += dp[i]; }
        dp = next;
    }
    return dp[offset + target];
}
```

> **Common Mistake:** In 0/1 knapsack with 1D array, you must iterate capacity **backwards** (right to left). Iterating forwards would allow using the same item multiple times.

---

## Pattern 4: String DP

**State:** `dp[i][j]` = result comparing `s1[0..i-1]` and `s2[0..j-1]`

```java
// Longest Common Subsequence -- Time: O(m*n), Space: O(m*n)
public int longestCommonSubsequence(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = s1.charAt(i-1) == s2.charAt(j-1)
                ? dp[i-1][j-1] + 1
                : Math.max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}

// Edit Distance -- Time: O(m*n), Space: O(m*n)
public int minDistance(String w1, String w2) {
    int m = w1.length(), n = w2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = w1.charAt(i-1) == w2.charAt(j-1)
                ? dp[i-1][j-1]
                : 1 + Math.min(dp[i-1][j-1], Math.min(dp[i-1][j], dp[i][j-1]));
    return dp[m][n];
}

// Longest Palindromic Subsequence -- Time: O(n^2), Space: O(n^2)
public int longestPalindromeSubseq(String s) {
    int n = s.length();
    int[][] dp = new int[n][n];
    for (int i = 0; i < n; i++) dp[i][i] = 1;
    for (int len = 2; len <= n; len++)
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = s.charAt(i) == s.charAt(j)
                ? dp[i+1][j-1] + 2
                : Math.max(dp[i+1][j], dp[i][j-1]);
        }
    return dp[0][n-1];
}
```

---

## Pattern 5: Memoization (Top-Down)

**Idea:** Write the plain recursion first, then cache each subproblem's result (keyed by its parameters) so it is computed only once. Same complexity as bottom-up, often easier to derive.

```java
// Fibonacci -- Time: O(n), Space: O(n)
public int fib(int n) {
    return fibHelper(n, new HashMap<>());
}
private int fibHelper(int n, Map<Integer, Integer> memo) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    int result = fibHelper(n-1, memo) + fibHelper(n-2, memo);
    memo.put(n, result);
    return result;
}

// Word Break -- Time: O(n^3) (n^2 start/end pairs, each substring O(n)), Space: O(n)
public boolean wordBreak(String s, List<String> wordDict) {
    return wb(s, new HashSet<>(wordDict), 0, new HashMap<>());
}
private boolean wb(String s, Set<String> dict, int start, Map<Integer, Boolean> memo) {
    if (start == s.length()) return true;
    if (memo.containsKey(start)) return memo.get(start);
    for (int end = start + 1; end <= s.length(); end++) {
        if (dict.contains(s.substring(start, end)) && wb(s, dict, end, memo)) {
            memo.put(start, true);
            return true;
        }
    }
    memo.put(start, false);
    return false;
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 70 | Climbing Stairs | 1D DP | Easy |
| 198 | House Robber | 1D DP | Medium |
| 55 | Jump Game | Greedy/DP | Medium |
| 62 | Unique Paths | 2D Grid DP | Medium |
| 64 | Minimum Path Sum | 2D Grid DP | Medium |
| 322 | Coin Change | Unbounded Knapsack | Medium |
| 416 | Partition Equal Subset Sum | 0/1 Knapsack | Medium |
| 494 | Target Sum | Knapsack | Medium |
| 1143 | Longest Common Subsequence | String DP | Medium |
| 72 | Edit Distance | String DP | Medium |
| 139 | Word Break | Memoization | Medium |
| 516 | Longest Palindromic Subsequence | String DP | Medium |
| 300 | Longest Increasing Subsequence | 1D DP | Medium |
