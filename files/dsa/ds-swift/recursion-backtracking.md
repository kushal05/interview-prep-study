# Recursion & Backtracking in Swift

> Swift implementation reference for backtracking: subsets, permutations, combinations, N-Queens, grid DFS and duplicate pruning.
> New to this topic? Learn it first in [04. Recursion](../learn/04-recursion.md) and [18. Backtracking](../learn/18-backtracking.md).

## When to Use
- "Return **all** combinations/permutations/subsets"
- Explore every valid configuration
- Exponential solution space with pruning

## Template
```
func backtrack(_ path: inout [Int], ...) {
    if isComplete(path) {
        result.append(path)  // Swift copies value types automatically!
        return
    }
    for choice in choices {
        path.append(choice)
        backtrack(&path, ...)
        path.removeLast()
    }
}
```

> **Key Interview Point:** Swift arrays are value types -- they copy on assignment. So `result.append(path)` already makes a copy (unlike Java/JS). Use `inout` parameters for mutations.

---

## Pattern 1: Subsets / Combinations / Permutations

**Idea:** Each call extends the current path by one choice, recurses, then removes that choice. Subsets/combinations loop from `start` so order is ignored (`i + 1` = use each element once, `i` = reuse allowed); permutations loop over every index and skip ones already marked `used`. N-Queens places one queen per row and checks column and diagonals before recursing.

```swift
// Subsets
func subsets(_ nums: [Int]) -> [[Int]] {
    var result = [[Int]]()
    var current = [Int]()
    backtrackSubsets(nums, 0, &current, &result)
    return result
}

private func backtrackSubsets(_ nums: [Int], _ start: Int, _ current: inout [Int], _ result: inout [[Int]]) {
    result.append(current)

    for i in start..<nums.count {
        current.append(nums[i])
        backtrackSubsets(nums, i + 1, &current, &result)
        current.removeLast()
    }
}

// Permutations
func permute(_ nums: [Int]) -> [[Int]] {
    var result = [[Int]]()
    var current = [Int]()
    var used = [Bool](repeating: false, count: nums.count)
    backtrackPermute(nums, &used, &current, &result)
    return result
}

private func backtrackPermute(_ nums: [Int], _ used: inout [Bool], _ current: inout [Int], _ result: inout [[Int]]) {
    if current.count == nums.count {
        result.append(current)
        return
    }

    for i in 0..<nums.count {
        if used[i] { continue }

        used[i] = true
        current.append(nums[i])
        backtrackPermute(nums, &used, &current, &result)
        current.removeLast()
        used[i] = false
    }
}

// Combination Sum
func combinationSum(_ candidates: [Int], _ target: Int) -> [[Int]] {
    var result = [[Int]]()
    var current = [Int]()
    let sortedCandidates = candidates.sorted()
    backtrackCombinationSum(sortedCandidates, target, 0, &current, &result)
    return result
}

private func backtrackCombinationSum(_ candidates: [Int], _ target: Int, _ start: Int, _ current: inout [Int], _ result: inout [[Int]]) {
    if target == 0 {
        result.append(current)
        return
    }

    for i in start..<candidates.count {
        if candidates[i] > target { break }

        current.append(candidates[i])
        backtrackCombinationSum(candidates, target - candidates[i], i, &current, &result)
        current.removeLast()
    }
}

// N-Queens
func solveNQueens(_ n: Int) -> [[String]] {
    var result = [[String]]()
    var board = Array(repeating: Array(repeating: ".", count: n), count: n)
    backtrackNQueens(&board, 0, &result)
    return result
}

private func backtrackNQueens(_ board: inout [[String]], _ row: Int, _ result: inout [[String]]) {
    if row == board.count {
        result.append(board.map { $0.joined() })
        return
    }

    for col in 0..<board.count {
        if isValidQueen(board, row, col) {
            board[row][col] = "Q"
            backtrackNQueens(&board, row + 1, &result)
            board[row][col] = "."
        }
    }
}

private func isValidQueen(_ board: [[String]], _ row: Int, _ col: Int) -> Bool {
    let n = board.count

    // Check column
    for i in 0..<row {
        if board[i][col] == "Q" { return false }
    }

    // Check diagonal (top-left)
    var i = row - 1
    var j = col - 1
    while i >= 0 && j >= 0 {
        if board[i][j] == "Q" { return false }
        i -= 1
        j -= 1
    }

    // Check diagonal (top-right)
    i = row - 1
    j = col + 1
    while i >= 0 && j < n {
        if board[i][j] == "Q" { return false }
        i -= 1
        j += 1
    }

    return true
}
```

## 🧠 Recursive DFS (Stateful Exploration)

**Idea:** Grid backtracking: temporarily overwrite the current cell (`"#"`) so the path cannot reuse it, explore the four neighbours, then restore it. Phone combos: one recursion level per digit, one branch per letter.

```swift
// Word Search
func exist(_ board: [[Character]], _ word: String) -> Bool {
    let chars = Array(word) // convert once, not once per cell; also guards empty word below
    guard !board.isEmpty, !board[0].isEmpty, !chars.isEmpty else { return chars.isEmpty }

    let rows = board.count
    let cols = board[0].count
    var board = board

    for i in 0..<rows {
        for j in 0..<cols {
            if board[i][j] == chars[0] && dfsWordSearch(&board, chars, i, j, 0) {
                return true
            }
        }
    }

    return false
}

private func dfsWordSearch(_ board: inout [[Character]], _ word: [Character], _ i: Int, _ j: Int, _ index: Int) -> Bool {
    if index == word.count { return true }

    guard i >= 0, i < board.count, j >= 0, j < board[0].count, board[i][j] == word[index] else {
        return false
    }

    let temp = board[i][j]
    board[i][j] = "#" // Mark as visited

    let found = dfsWordSearch(&board, word, i - 1, j, index + 1) ||
                dfsWordSearch(&board, word, i + 1, j, index + 1) ||
                dfsWordSearch(&board, word, i, j - 1, index + 1) ||
                dfsWordSearch(&board, word, i, j + 1, index + 1)

    board[i][j] = temp // Backtrack

    return found
}

// Letter combinations of a phone number
func letterCombinations(_ digits: String) -> [String] {
    guard !digits.isEmpty else { return [] }

    let phoneMap: [Character: String] = [
        "2": "abc", "3": "def", "4": "ghi",
        "5": "jkl", "6": "mno", "7": "pqrs",
        "8": "tuv", "9": "wxyz"
    ]

    var result = [String]()
    backtrackLetterCombinations(Array(digits), phoneMap, 0, "", &result)
    return result
}

private func backtrackLetterCombinations(_ digits: [Character], _ phoneMap: [Character: String], _ index: Int, _ current: String, _ result: inout [String]) {
    if index == digits.count {
        result.append(current)
        return
    }

    guard let letters = phoneMap[digits[index]] else { return }

    for letter in letters {
        backtrackLetterCombinations(digits, phoneMap, index + 1, current + String(letter), &result)
    }
}
```

## ⛔ Constraint-Based Backtracking

**Idea:** Sort first so equal values sit together and anything too large can `break` the loop. At one recursion level, skip a value equal to the previous one (`i > start`), since it would build the same combinations again. Palindrome partitioning only recurses on prefixes that are palindromes.

```swift
// Combination Sum II (with duplicates in candidates)
func combinationSum2(_ candidates: [Int], _ target: Int) -> [[Int]] {
    var result = [[Int]]()
    var current = [Int]()
    let sortedCandidates = candidates.sorted()
    backtrackCombinationSum2(sortedCandidates, target, 0, &current, &result)
    return result
}

private func backtrackCombinationSum2(_ candidates: [Int], _ target: Int, _ start: Int, _ current: inout [Int], _ result: inout [[Int]]) {
    if target == 0 {
        result.append(current)
        return
    }

    for i in start..<candidates.count {
        if i > start && candidates[i] == candidates[i - 1] { continue }
        if candidates[i] > target { break }

        current.append(candidates[i])
        backtrackCombinationSum2(candidates, target - candidates[i], i + 1, &current, &result)
        current.removeLast()
    }
}

// Palindrome Partitioning
func partition(_ s: String) -> [[String]] {
    var result = [[String]]()
    var current = [String]()
    backtrackPartition(Array(s), 0, &current, &result)
    return result
}

private func backtrackPartition(_ chars: [Character], _ start: Int, _ current: inout [String], _ result: inout [[String]]) {
    if start == chars.count {
        result.append(current)
        return
    }

    for end in (start + 1)...chars.count {
        let substring = String(chars[start..<end])
        if isPalindrome(substring) {
            current.append(substring)
            backtrackPartition(chars, end, &current, &result)
            current.removeLast()
        }
    }
}

private func isPalindrome(_ s: String) -> Bool {
    let chars = Array(s)
    var left = 0
    var right = chars.count - 1

    while left < right {
        if chars[left] != chars[right] { return false }
        left += 1
        right -= 1
    }

    return true
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