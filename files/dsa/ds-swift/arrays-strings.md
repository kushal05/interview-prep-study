# Arrays & Strings in Swift

> Swift implementation reference for arrays and strings: core operations plus sliding window, two pointers, binary search, prefix sum, cyclic sort, Kadane, intervals, and bucket sort.
> New to this topic? Learn it first in [02. Arrays and Strings](../learn/02-arrays-and-strings.md) (then [05. Two Pointers](../learn/05-two-pointers.md) and [06. Sliding Window](../learn/06-sliding-window.md)).

## Complexity Overview

| Operation | Array | String |
|-----------|-------|--------|
| Access by index | O(1) | O(n)* |
| Append | O(1) amortized | O(1) amortized (`var` string) |
| Remove at index | O(n) | O(n) |
| Contains | O(n) | O(n) |

*Swift String indexing is O(n) because characters are variable-width (Unicode). Convert to `Array(str)` for O(1) access.

> **Key Interview Point:** Swift strings are NOT O(1) indexed. Always convert to `let chars = Array(s)` first for O(1) character access. This is the #1 Swift gotcha in interviews.

> **Common Mistake:** Using `s[s.index(s.startIndex, offsetBy: i)]` in a loop is O(n^2). Convert to array first.

---

## Basic Operations

```swift
// Basic array operations
func arrayOperations() {
    // Declaration and initialization
    let arr = [1, 2, 3, 4, 5]
    var mutableArr = [1, 2, 3, 4, 5]

    // Access elements
    print(arr[0]) // 1

    // Add element (for Array)
    mutableArr.append(6)

    // Remove element
    mutableArr.remove(at: 2)

    // Search
    let index = arr.firstIndex(of: 3) ?? -1 // 2

    // Length
    print(arr.count) // 5
}

// String operations
func stringOperations() {
    let str = "Hello World"

    // Length
    print(str.count) // 11

    // Access characters
    if let firstChar = str.first {
        print(firstChar) // 'H'
    }

    // Substring
    let startIndex = str.startIndex
    let endIndex = str.index(startIndex, offsetBy: 5)
    let substr = String(str[startIndex..<endIndex]) // "Hello"

    // Split
    let words = str.split(separator: " ") // ["Hello", "World"]

    // Replace
    let replaced = str.replacingOccurrences(of: "World", with: "Swift")
}
```

## 🌀 Sliding Window Pattern

**Idea:** Keep a window `[left, right]` over the array. Grow it by moving `right`; when the window breaks a rule (duplicate char, too large), shrink it from `left`. Each index enters and leaves once, so it is O(n).

```swift
// Longest substring without repeating characters
func lengthOfLongestSubstring(_ s: String) -> Int {
    var charSet = Set<Character>()
    var left = 0
    var maxLength = 0

    let chars = Array(s)

    for right in 0..<chars.count {
        while charSet.contains(chars[right]) {
            charSet.remove(chars[left])
            left += 1
        }
        charSet.insert(chars[right])
        maxLength = max(maxLength, right - left + 1)
    }

    return maxLength
}

// Max sum subarray of size K
func maxSumSubarray(_ arr: [Int], _ k: Int) -> Int {
    var maxSum = Int.min
    var windowSum = 0

    // Initialize first window
    for i in 0..<k {
        windowSum += arr[i]
    }
    maxSum = windowSum

    // Slide window
    for i in k..<arr.count {
        windowSum += arr[i] - arr[i - k]
        maxSum = max(maxSum, windowSum)
    }

    return maxSum
}
```

## 🔁 Two Pointers Pattern

**Idea:** Use two indices that move toward each other (reverse, pair-sum on sorted input) or in the same direction (slow = write position, fast = read position) to avoid a nested loop.

```swift
// Reverse array in-place
func reverseArray(_ arr: inout [Int]) {
    var left = 0
    var right = arr.count - 1

    while left < right {
        let temp = arr[left]
        arr[left] = arr[right]
        arr[right] = temp
        left += 1
        right -= 1
    }
}

// Remove duplicates from sorted array
func removeDuplicates(_ nums: inout [Int]) -> Int {
    if nums.isEmpty { return 0 }

    var i = 0
    for j in 1..<nums.count {
        if nums[j] != nums[i] {
            i += 1
            nums[i] = nums[j]
        }
    }
    return i + 1
}
```

## 🔍 Binary Search Pattern

**Idea:** On sorted (or "sorted-ish") data, look at the middle and discard the half that cannot contain the answer. Halving each step gives O(log n). For a rotated array, compare `nums[mid]` with `nums[right]` to tell which half holds the minimum.

```swift
// Binary search on sorted array
func binarySearch(_ arr: [Int], _ target: Int) -> Int {
    var left = 0
    var right = arr.count - 1

    while left <= right {
        let mid = left + (right - left) / 2

        if arr[mid] == target {
            return mid
        } else if arr[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }

    return -1
}

// Find minimum in rotated sorted array
func findMin(_ nums: [Int]) -> Int {
    var left = 0
    var right = nums.count - 1

    while left < right {
        let mid = left + (right - left) / 2

        if nums[mid] > nums[right] {
            left = mid + 1
        } else {
            right = mid
        }
    }

    return nums[left]
}
```

## ➕ Prefix Sum Pattern

**Idea:** `prefix[i]` = sum of the first `i` elements, so any range sum is `prefix[r+1] - prefix[l]` in O(1). To count subarrays summing to `k`, store how many times each running sum has appeared and look up `sum - k`.

```swift
// Subarray sum equals K
func subarraySum(_ nums: [Int], _ k: Int) -> Int {
    var prefixSum = [0: 1]
    var sum = 0
    var count = 0

    for num in nums {
        sum += num
        count += prefixSum[sum - k] ?? 0
        prefixSum[sum] = (prefixSum[sum] ?? 0) + 1
    }

    return count
}

// Range sum query (Immutable)
class NumArray {
    private var prefixSum: [Int]

    init(_ nums: [Int]) {
        prefixSum = [Int](repeating: 0, count: nums.count + 1)
        for i in 0..<nums.count {
            prefixSum[i + 1] = prefixSum[i] + nums[i]
        }
    }

    func sumRange(_ left: Int, _ right: Int) -> Int {
        return prefixSum[right + 1] - prefixSum[left]
    }
}
```

## 🔄 Cyclic Sort Pattern

**Idea:** When values are in the range `1...n`, value `v` belongs at index `v - 1`. Swap each value into its home slot; afterwards any index `j` where `nums[j] != j + 1` reveals a missing or duplicate number. O(n) time, O(1) extra space.

```swift
// Find missing number (1 to N)
func findMissingNumber(_ nums: inout [Int]) -> Int {
    var i = 0
    while i < nums.count {
        let correctIndex = nums[i] - 1
        if nums[i] > 0 && nums[i] <= nums.count && nums[i] != nums[correctIndex] {
            swap(&nums, i, correctIndex)
        } else {
            i += 1
        }
    }

    for j in 0..<nums.count {
        if nums[j] != j + 1 {
            return j + 1
        }
    }

    return nums.count + 1
}

// Find all duplicates in array
func findDuplicates(_ nums: inout [Int]) -> [Int] {
    var result = [Int]()

    var i = 0
    while i < nums.count {
        let correctIndex = nums[i] - 1
        if nums[i] != nums[correctIndex] {
            swap(&nums, i, correctIndex)
        } else {
            i += 1
        }
    }

    for j in 0..<nums.count {
        if nums[j] != j + 1 {
            result.append(nums[j])
        }
    }

    return result
}

// Helper function to swap elements
func swap(_ nums: inout [Int], _ i: Int, _ j: Int) {
    let temp = nums[i]
    nums[i] = nums[j]
    nums[j] = temp
}
```

## 🧠 Kadane's Algorithm Pattern

**Idea:** At each element, either extend the best subarray ending at the previous element or start fresh here, whichever is larger. Track the best value seen overall. O(n), O(1) space.

```swift
// Maximum Subarray Sum
func maxSubarraySum(_ nums: [Int]) -> Int {
    var maxSum = nums[0]
    var currentSum = nums[0]

    for i in 1..<nums.count {
        currentSum = max(nums[i], currentSum + nums[i])
        maxSum = max(maxSum, currentSum)
    }

    return maxSum
}
```

## 🔀 Merge Intervals Pattern

**Idea:** Sort by start. Walk through the intervals; if the next one starts before the current one ends, they overlap, so extend the current end. Otherwise close the current interval and start a new one. O(n log n) for the sort.

```swift
// Merge Intervals
func mergeIntervals(_ intervals: [[Int]]) -> [[Int]] {
    if intervals.isEmpty { return [] }

    let sortedIntervals = intervals.sorted { $0[0] < $1[0] }
    var result = [[Int]]()
    var current = sortedIntervals[0]

    for interval in sortedIntervals {
        if current[1] >= interval[0] {
            current[1] = max(current[1], interval[1])
        } else {
            result.append(current)
            current = interval
        }
    }

    result.append(current)
    return result
}
```

## 🪜 Dutch National Flag Pattern (3-Way Partition)

**Idea:** Three pointers: everything before `low` is 0, everything after `high` is 2, and `mid` scans the unknown middle. Do not advance `mid` after swapping with `high`, because the swapped-in value has not been checked yet.

```swift
// Sort Colors (0, 1, 2)
func sortColors(_ nums: inout [Int]) {
    var low = 0
    var mid = 0
    var high = nums.count - 1

    while mid <= high {
        switch nums[mid] {
        case 0:
            swap(&nums, low, mid)
            low += 1
            mid += 1
        case 1:
            mid += 1
        case 2:
            swap(&nums, mid, high)
            high -= 1
        default:
            break
        }
    }
}
```

## 🧺 Bucket Sort Pattern

**Idea:** A frequency can be at most `n`, so make bucket `i` hold the numbers that appear exactly `i` times. Read buckets from high to low until you have `k` numbers. O(n) instead of O(n log n) sorting.

```swift
// Top K frequent elements
func topKFrequent(_ nums: [Int], _ k: Int) -> [Int] {
    var frequencyMap = [Int: Int]()

    for num in nums {
        frequencyMap[num, default: 0] += 1
    }

    var buckets = [[Int]](repeating: [], count: nums.count + 1)

    for (num, freq) in frequencyMap {
        buckets[freq].append(num)
    }

    var result = [Int]()
    for i in stride(from: buckets.count - 1, to: 0, by: -1) {
        for num in buckets[i] {
            result.append(num)
            if result.count == k {
                return result
            }
        }
    }

    return result
}
```
