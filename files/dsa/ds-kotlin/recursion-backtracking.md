# Recursion & Backtracking in Kotlin

> Kotlin implementation reference for backtracking: subsets, permutations, combinations, N-Queens, grid DFS and duplicate pruning.
> New to this topic? Learn it first in [04. Recursion](../learn/04-recursion.md) and [18. Backtracking](../learn/18-backtracking.md).

## When to Use
- "Return **all** combinations/permutations/subsets"
- Explore every valid configuration
- Exponential solution space with pruning

## Template
```
fun backtrack(path, choices):
    if isComplete(path):
        result.add(ArrayList(path))  // COPY the list!
        return
    for choice in choices:
        path.add(choice)              // choose
        backtrack(updated)            // explore
        path.removeAt(path.size - 1)  // un-choose
```

> **Key Interview Point:** In Kotlin, use `ArrayList(current)` to make a defensive copy when adding to results.

---

## Pattern 1: Subsets / Combinations / Permutations

**Idea:** Each call extends the current path by one choice, recurses, then removes that choice. Subsets/combinations loop from `start` so order is ignored (`i + 1` = use each element once, `i` = reuse allowed); permutations loop over every index and skip ones already marked `used`. N-Queens places one queen per row and checks column and diagonals before recursing.

```kotlin
// Subsets
fun subsets(nums: IntArray): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    backtrackSubsets(nums, 0, mutableListOf(), result)
    return result
}

private fun backtrackSubsets(nums: IntArray, start: Int, current: MutableList<Int>, result: MutableList<List<Int>>) {
    result.add(ArrayList(current))

    for (i in start until nums.size) {
        current.add(nums[i])
        backtrackSubsets(nums, i + 1, current, result)
        current.removeAt(current.size - 1)
    }
}

// Permutations -- O(n * n!) time
fun permute(nums: IntArray): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    backtrackPermute(nums, BooleanArray(nums.size), mutableListOf(), result)
    return result
}

private fun backtrackPermute(nums: IntArray, used: BooleanArray, current: MutableList<Int>, result: MutableList<List<Int>>) {
    if (current.size == nums.size) {
        result.add(ArrayList(current))
        return
    }

    // used[] instead of `num !in current`: O(1) per check and correct even with duplicate values
    for (i in nums.indices) {
        if (used[i]) continue
        used[i] = true
        current.add(nums[i])
        backtrackPermute(nums, used, current, result)
        current.removeAt(current.size - 1)
        used[i] = false
    }
}

// Combination Sum
fun combinationSum(candidates: IntArray, target: Int): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    candidates.sort()
    backtrackCombinationSum(candidates, target, 0, mutableListOf(), result)
    return result
}

private fun backtrackCombinationSum(candidates: IntArray, target: Int, start: Int, current: MutableList<Int>, result: MutableList<List<Int>>) {
    if (target == 0) {
        result.add(ArrayList(current))
        return
    }

    for (i in start until candidates.size) {
        if (candidates[i] > target) break

        current.add(candidates[i])
        backtrackCombinationSum(candidates, target - candidates[i], i, current, result)
        current.removeAt(current.size - 1)
    }
}

// N-Queens
fun solveNQueens(n: Int): List<List<String>> {
    val result = mutableListOf<List<String>>()
    val board = Array(n) { CharArray(n) { '.' } }
    backtrackNQueens(board, 0, result)
    return result
}

private fun backtrackNQueens(board: Array<CharArray>, row: Int, result: MutableList<List<String>>) {
    if (row == board.size) {
        result.add(board.map { String(it) })
        return
    }

    for (col in board.indices) {
        if (isValidQueen(board, row, col)) {
            board[row][col] = 'Q'
            backtrackNQueens(board, row + 1, result)
            board[row][col] = '.'
        }
    }
}

private fun isValidQueen(board: Array<CharArray>, row: Int, col: Int): Boolean {
    // Check column
    for (i in 0 until row) {
        if (board[i][col] == 'Q') return false
    }

    // Check diagonal (top-left)
    var i = row - 1
    var j = col - 1
    while (i >= 0 && j >= 0) {
        if (board[i][j] == 'Q') return false
        i--
        j--
    }

    // Check diagonal (top-right)
    i = row - 1
    j = col + 1
    while (i >= 0 && j < board.size) {
        if (board[i][j] == 'Q') return false
        i--
        j++
    }

    return true
}
```

## 🧠 Recursive DFS (Stateful Exploration)

**Idea:** Grid backtracking: temporarily overwrite the current cell (`'#'`) so the path cannot reuse it, explore the four neighbours, then restore it. Phone combos: one recursion level per digit, one branch per letter.

```kotlin
// Word Search
fun exist(board: Array<CharArray>, word: String): Boolean {
    if (board.isEmpty() || board[0].isEmpty()) return false

    val rows = board.size
    val cols = board[0].size

    for (i in 0 until rows) {
        for (j in 0 until cols) {
            if (board[i][j] == word[0] && dfsWordSearch(board, word, i, j, 0)) {
                return true
            }
        }
    }

    return false
}

private fun dfsWordSearch(board: Array<CharArray>, word: String, i: Int, j: Int, index: Int): Boolean {
    if (index == word.length) return true

    if (i < 0 || i >= board.size || j < 0 || j >= board[0].size || board[i][j] != word[index]) {
        return false
    }

    val temp = board[i][j]
    board[i][j] = '#' // Mark as visited

    val found = dfsWordSearch(board, word, i - 1, j, index + 1) ||
                dfsWordSearch(board, word, i + 1, j, index + 1) ||
                dfsWordSearch(board, word, i, j - 1, index + 1) ||
                dfsWordSearch(board, word, i, j + 1, index + 1)

    board[i][j] = temp // Backtrack

    return found
}

// Letter combinations of a phone number
fun letterCombinations(digits: String): List<String> {
    if (digits.isEmpty()) return emptyList()

    val phoneMap = mapOf(
        '2' to "abc", '3' to "def", '4' to "ghi",
        '5' to "jkl", '6' to "mno", '7' to "pqrs",
        '8' to "tuv", '9' to "wxyz"
    )

    val result = mutableListOf<String>()
    backtrackLetterCombinations(digits, phoneMap, 0, StringBuilder(), result)
    return result
}

private fun backtrackLetterCombinations(
    digits: String,
    phoneMap: Map<Char, String>,
    index: Int,
    current: StringBuilder,
    result: MutableList<String>
) {
    if (index == digits.length) {
        result.add(current.toString())
        return
    }

    val letters = phoneMap[digits[index]]!!
    for (letter in letters) {
        current.append(letter)
        backtrackLetterCombinations(digits, phoneMap, index + 1, current, result)
        current.deleteCharAt(current.length - 1)
    }
}
```

## ⛔ Constraint-Based Backtracking

**Idea:** Sort first so equal values sit together and anything too large can `break` the loop. At one recursion level, skip a value equal to the previous one (`i > start`), since it would build the same combinations again. Palindrome partitioning only recurses on prefixes that are palindromes.

```kotlin
// Combination Sum II (with duplicates in candidates)
fun combinationSum2(candidates: IntArray, target: Int): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    candidates.sort()
    backtrackCombinationSum2(candidates, target, 0, mutableListOf(), result)
    return result
}

private fun backtrackCombinationSum2(
    candidates: IntArray,
    target: Int,
    start: Int,
    current: MutableList<Int>,
    result: MutableList<List<Int>>
) {
    if (target == 0) {
        result.add(ArrayList(current))
        return
    }

    for (i in start until candidates.size) {
        if (i > start && candidates[i] == candidates[i - 1]) continue
        if (candidates[i] > target) break

        current.add(candidates[i])
        backtrackCombinationSum2(candidates, target - candidates[i], i + 1, current, result)
        current.removeAt(current.size - 1)
    }
}

// Palindrome Partitioning
fun partition(s: String): List<List<String>> {
    val result = mutableListOf<List<String>>()
    backtrackPartition(s, 0, mutableListOf(), result)
    return result
}

private fun backtrackPartition(
    s: String,
    start: Int,
    current: MutableList<String>,
    result: MutableList<List<String>>
) {
    if (start == s.length) {
        result.add(ArrayList(current))
        return
    }

    for (end in start + 1..s.length) {
        val substring = s.substring(start, end)
        if (isPalindrome(substring)) {
            current.add(substring)
            backtrackPartition(s, end, current, result)
            current.removeAt(current.size - 1)
        }
    }
}

private fun isPalindrome(s: String): Boolean {
    var left = 0
    var right = s.length - 1

    while (left < right) {
        if (s[left] != s[right]) return false
        left++
        right--
    }

    return true
}
```
