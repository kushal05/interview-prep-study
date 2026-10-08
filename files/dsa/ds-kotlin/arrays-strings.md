# Arrays & Strings in Kotlin

> Kotlin implementation reference for arrays and strings: core operations plus sliding window, two pointers, binary search, prefix sum, cyclic sort, Kadane, intervals, and bucket sort.
> New to this topic? Learn it first in [02. Arrays and Strings](../learn/02-arrays-and-strings.md) (then [05. Two Pointers](../learn/05-two-pointers.md) and [06. Sliding Window](../learn/06-sliding-window.md)).

## Complexity Overview

| Operation | IntArray | MutableList | String |
|-----------|----------|-------------|--------|
| Access by index | O(1) | O(1) | O(1) |
| Add to end | - | O(1) amortized | - |
| Remove at index | - | O(n) | - |
| indexOf | O(n) | O(n) | O(n*m) |
| Sort | O(n log n) | O(n log n) | - |

> **Key Interview Point:** In Kotlin, use `IntArray` for primitive arrays (no boxing overhead) and `Array<Int>` only when needed. Use `StringBuilder` for string building. Kotlin's `maxOf()`, `minOf()` are handy alternatives to `Math.max/min`.

> **Common Mistake:** Kotlin ranges are inclusive on both ends: `0..n` includes `n`, while `0 until n` excludes `n`. Use `until` for array indices.

---

## Basic Operations

```kotlin
// Basic array operations
fun arrayOperations() {
    // Declaration and initialization
    val arr = arrayOf(1, 2, 3, 4, 5)
    val mutableArr = mutableListOf(1, 2, 3, 4, 5)

    // Access elements
    println(arr[0]) // 1

    // Add element (for MutableList)
    mutableArr.add(6)

    // Remove element
    mutableArr.removeAt(2)

    // Search
    val index = arr.indexOf(3) // 2

    // Length
    println(arr.size) // 5
}

// String operations
fun stringOperations() {
    val str = "Hello World"

    // Length
    println(str.length) // 11

    // Access characters
    println(str[0]) // 'H'

    // Substring
    val substr = str.substring(0, 5) // "Hello"

    // Split
    val words = str.split(" ") // ["Hello", "World"]

    // Replace
    val replaced = str.replace("World", "Kotlin")
}
```

## 🌀 Sliding Window Pattern

**Idea:** Keep a window `[left, right]` over the input. Grow it by moving `right`; when it breaks the rule (a repeated char, too big), shrink it from `left`. Each index enters and leaves once, so it is O(n).

```kotlin
// Longest substring without repeating characters
fun lengthOfLongestSubstring(s: String): Int {
    val charSet = mutableSetOf<Char>()
    var left = 0
    var maxLength = 0

    for (right in s.indices) {
        while (s[right] in charSet) {
            charSet.remove(s[left])
            left++
        }
        charSet.add(s[right])
        maxLength = maxOf(maxLength, right - left + 1)
    }
    return maxLength
}

// Max sum subarray of size K
fun maxSumSubarray(arr: IntArray, k: Int): Int {
    var maxSum = 0
    var windowSum = 0

    // Initialize first window
    for (i in 0 until k) {
        windowSum += arr[i]
    }
    maxSum = windowSum

    // Slide window
    for (i in k until arr.size) {
        windowSum += arr[i] - arr[i - k]
        maxSum = maxOf(maxSum, windowSum)
    }
    return maxSum
}
```

## 🔁 Two Pointers Pattern

**Idea:** Use two indices that move toward each other (reverse, pair sum on sorted data) or in the same direction (slow writer / fast reader) to avoid a nested loop.

```kotlin
// Reverse array in-place
fun reverseArray(arr: IntArray) {
    var left = 0
    var right = arr.size - 1

    while (left < right) {
        val temp = arr[left]
        arr[left] = arr[right]
        arr[right] = temp
        left++
        right--
    }
}

// Remove duplicates from sorted array
fun removeDuplicates(nums: IntArray): Int {
    if (nums.isEmpty()) return 0

    var i = 0
    for (j in 1 until nums.size) {
        if (nums[j] != nums[i]) {
            i++
            nums[i] = nums[j]
        }
    }
    return i + 1
}
```

## 🔍 Binary Search Pattern

**Idea:** If you can throw away half of the remaining range with one comparison, you get O(log n). For the rotated array, compare `mid` with `right` to tell which half is sorted.

```kotlin
// Binary search on sorted array
fun binarySearch(arr: IntArray, target: Int): Int {
    var left = 0
    var right = arr.size - 1

    while (left <= right) {
        val mid = left + (right - left) / 2

        when {
            arr[mid] == target -> return mid
            arr[mid] < target -> left = mid + 1
            else -> right = mid - 1
        }
    }
    return -1
}

// Find minimum in rotated sorted array
fun findMin(nums: IntArray): Int {
    var left = 0
    var right = nums.size - 1

    while (left < right) {
        val mid = left + (right - left) / 2

        if (nums[mid] > nums[right]) {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return nums[left]
}
```

## ➕ Prefix Sum Pattern

**Idea:** `prefix[i]` stores the sum of the first `i` elements, so any subarray sum is `prefix[r+1] - prefix[l]`. Storing counts of prefix sums in a map lets you count subarrays summing to `k` in one pass.

```kotlin
// Subarray sum equals K
fun subarraySum(nums: IntArray, k: Int): Int {
    val prefixSum = mutableMapOf(0 to 1)
    var sum = 0
    var count = 0

    for (num in nums) {
        sum += num
        count += prefixSum.getOrDefault(sum - k, 0)
        prefixSum[sum] = prefixSum.getOrDefault(sum, 0) + 1
    }
    return count
}

// Range sum query (Immutable)
class NumArray(nums: IntArray) {
    private val prefixSum = IntArray(nums.size + 1)

    init {
        for (i in nums.indices) {
            prefixSum[i + 1] = prefixSum[i] + nums[i]
        }
    }

    fun sumRange(left: Int, right: Int): Int {
        return prefixSum[right + 1] - prefixSum[left]
    }
}
```

## 🔄 Cyclic Sort Pattern

**Idea:** When values are in the range 1..n, value `v` belongs at index `v - 1`. Swap each value into its home; afterwards any index whose value is wrong reveals a missing or duplicate number.

```kotlin
// Find missing number (1 to N)
fun findMissingNumber(nums: IntArray): Int {
    var i = 0
    while (i < nums.size) {
        val correctIndex = nums[i] - 1
        if (nums[i] > 0 && nums[i] <= nums.size && nums[i] != nums[correctIndex]) {
            swap(nums, i, correctIndex)
        } else {
            i++
        }
    }

    for (j in nums.indices) {
        if (nums[j] != j + 1) {
            return j + 1
        }
    }
    return nums.size + 1
}

// Find all duplicates in array
fun findDuplicates(nums: IntArray): List<Int> {
    val result = mutableListOf<Int>()

    var i = 0
    while (i < nums.size) {
        val correctIndex = nums[i] - 1
        if (nums[i] != nums[correctIndex]) {
            swap(nums, i, correctIndex)
        } else {
            i++
        }
    }

    for (j in nums.indices) {
        if (nums[j] != j + 1) {
            result.add(nums[j])
        }
    }
    return result
}

private fun swap(nums: IntArray, i: Int, j: Int) {
    val temp = nums[i]
    nums[i] = nums[j]
    nums[j] = temp
}
```

## 🧠 Kadane's Algorithm Pattern

**Idea:** At each element, either extend the best subarray ending at the previous element or start fresh here, whichever is larger. Track the best value seen.

```kotlin
// Maximum Subarray Sum
fun maxSubarraySum(nums: IntArray): Int {
    var maxSum = nums[0]
    var currentSum = nums[0]

    for (i in 1 until nums.size) {
        currentSum = maxOf(nums[i], currentSum + nums[i])
        maxSum = maxOf(maxSum, currentSum)
    }
    return maxSum
}
```

## 🔀 Merge Intervals Pattern

**Idea:** Sort by start time. Then an interval overlaps the current merged block only if its start is <= the block's end; otherwise close the block and start a new one.

```kotlin
// Merge Intervals
fun mergeIntervals(intervals: Array<IntArray>): Array<IntArray> {
    if (intervals.isEmpty()) return emptyArray()

    intervals.sortBy { it[0] }
    val result = mutableListOf<IntArray>()
    var current = intervals[0]

    for (interval in intervals) {
        if (current[1] >= interval[0]) {
            current[1] = maxOf(current[1], interval[1])
        } else {
            result.add(current)
            current = interval
        }
    }
    result.add(current)
    return result.toTypedArray()
}
```

## 🪜 Dutch National Flag Pattern (3-Way Partition)

**Idea:** Keep three regions: `[0, low)` holds 0s, `[low, mid)` holds 1s, `(high, end]` holds 2s. Look at `nums[mid]` and swap it into the right region. Don't advance `mid` after swapping with `high`, because the swapped-in value hasn't been checked yet.

```kotlin
// Sort Colors (0, 1, 2)
fun sortColors(nums: IntArray) {
    var low = 0
    var mid = 0
    var high = nums.size - 1

    while (mid <= high) {
        when (nums[mid]) {
            0 -> {
                swap(nums, low, mid)
                low++
                mid++
            }
            1 -> mid++
            2 -> {
                swap(nums, mid, high)
                high--
            }
        }
    }
}
```

## 🧺 Bucket Sort Pattern

**Idea:** A frequency can be at most `n`, so put each number in bucket `freq`. Then walk the buckets from high to low to collect the top k in O(n), with no sorting.

```kotlin
// Top K frequent elements
fun topKFrequent(nums: IntArray, k: Int): IntArray {
    val frequencyMap = mutableMapOf<Int, Int>()
    for (num in nums) {
        frequencyMap[num] = frequencyMap.getOrDefault(num, 0) + 1
    }

    val buckets = Array<MutableList<Int>>(nums.size + 1) { mutableListOf() }
    for ((num, freq) in frequencyMap) {
        buckets[freq].add(num)
    }

    val result = mutableListOf<Int>()
    for (i in buckets.size - 1 downTo 1) {
        for (num in buckets[i]) {
            result.add(num)
            if (result.size == k) return result.toIntArray()
        }
    }
    return result.toIntArray()
}
```
