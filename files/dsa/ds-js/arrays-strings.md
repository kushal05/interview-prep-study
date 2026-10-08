# Arrays & Strings in JavaScript

> JavaScript implementation reference for arrays and strings: core operations plus sliding window, two pointers, binary search, prefix sum, cyclic sort, Kadane, intervals and bucket sort.
> New to this topic? Learn it first in [02. Arrays and Strings](../learn/02-arrays-and-strings.md) (then [05. Two Pointers](../learn/05-two-pointers.md) and [06. Sliding Window](../learn/06-sliding-window.md)).

## Complexity Overview

| Operation | Array | String |
|-----------|-------|--------|
| Access by index | O(1) | O(1) |
| Push/Pop (end) | O(1) | - |
| Shift/Unshift (front) | O(n) | - |
| Splice (middle) | O(n) | - |
| indexOf / includes | O(n) | O(n*m) |

> **Key Interview Point:** JS strings are immutable. Use array of chars or `split('')` for in-place manipulation. `Array.shift()` is O(n) -- avoid in BFS loops; use index pointer instead.

> **Common Mistake:** Using `==` instead of `===` for comparisons. Always use strict equality in interviews.

---

## Basic Operations

```javascript
// Basic array operations
function arrayOperations() {
    // Declaration and initialization
    const arr = [1, 2, 3, 4, 5];
    const mutableArr = [1, 2, 3, 4, 5];

    // Access elements
    console.log(arr[0]); // 1

    // Add element
    mutableArr.push(6);

    // Remove element
    mutableArr.splice(2, 1);

    // Search
    const index = arr.indexOf(3); // 2

    // Length
    console.log(arr.length); // 5
}

// String operations
function stringOperations() {
    const str = "Hello World";

    // Length
    console.log(str.length); // 11

    // Access characters
    console.log(str[0]); // 'H'

    // Substring
    const substr = str.substring(0, 5); // "Hello"

    // Split
    const words = str.split(" "); // ["Hello", "World"]

    // Replace
    const replaced = str.replace("World", "JavaScript");
}
```

## Pattern 1: Sliding Window

**When to use:** Contiguous subarray/substring with a constraint.

```javascript
// Longest substring without repeating characters
function lengthOfLongestSubstring(s) {
    const charSet = new Set();
    let left = 0;
    let maxLength = 0;

    for (let right = 0; right < s.length; right++) {
        while (charSet.has(s[right])) {
            charSet.delete(s[left]);
            left++;
        }
        charSet.add(s[right]);
        maxLength = Math.max(maxLength, right - left + 1);
    }
    return maxLength;
}

// Max sum subarray of size K
function maxSumSubarray(arr, k) {
    let maxSum = -Infinity;
    let windowSum = 0;

    // Initialize first window
    for (let i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    maxSum = windowSum;

    // Slide window
    for (let i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

## 🔁 Two Pointers Pattern

**Idea:** Keep two indices that move toward each other (or one slow, one fast) so each element is visited once, giving O(n) instead of the O(n^2) of checking every pair.

```javascript
// Reverse array in-place
function reverseArray(arr) {
    let left = 0;
    let right = arr.length - 1;

    while (left < right) {
        const temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left++;
        right--;
    }
}

// Remove duplicates from sorted array
function removeDuplicates(nums) {
    if (nums.length === 0) return 0;

    let i = 0;
    for (let j = 1; j < nums.length; j++) {
        if (nums[j] !== nums[i]) {
            i++;
            nums[i] = nums[j];
        }
    }
    return i + 1;
}
```

## 🔍 Binary Search Pattern

**Idea:** When the search space is sorted (or has a yes/no boundary), compare against the middle and throw away the half that cannot contain the answer. Each step halves the range, so it is O(log n).

```javascript
// Binary search on sorted array
function binarySearch(arr, target) {
    let left = 0;
    let right = arr.length - 1;

    while (left <= right) {
        const mid = Math.floor(left + (right - left) / 2);

        if (arr[mid] === target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return -1;
}

// Find minimum in rotated sorted array
function findMin(nums) {
    let left = 0;
    let right = nums.length - 1;

    while (left < right) {
        const mid = Math.floor(left + (right - left) / 2);

        if (nums[mid] > nums[right]) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    return nums[left];
}
```

## ➕ Prefix Sum Pattern

**Idea:** `prefix[i]` stores the sum of the first `i` elements, so any range sum is `prefix[r + 1] - prefix[l]` in O(1). Combined with a hash map of seen prefix sums, it counts subarrays with a target sum in one pass.

```javascript
// Subarray sum equals K
function subarraySum(nums, k) {
    const prefixSum = new Map();
    prefixSum.set(0, 1);
    let sum = 0;
    let count = 0;

    for (const num of nums) {
        sum += num;
        count += prefixSum.get(sum - k) || 0;
        prefixSum.set(sum, (prefixSum.get(sum) || 0) + 1);
    }
    return count;
}

// Range sum query (Immutable)
class NumArray {
    constructor(nums) {
        this.prefixSum = new Array(nums.length + 1).fill(0);
        for (let i = 0; i < nums.length; i++) {
            this.prefixSum[i + 1] = this.prefixSum[i] + nums[i];
        }
    }

    sumRange(left, right) {
        return this.prefixSum[right + 1] - this.prefixSum[left];
    }
}
```

## 🔄 Cyclic Sort Pattern

**Idea:** When values are in the range 1..n, value `v` belongs at index `v - 1`. Swap each value into its home slot; afterwards any index whose value is wrong reveals a missing or duplicate number. O(n) time, O(1) extra space.

```javascript
// First missing positive (LC 41); also finds the missing number when values are 1..N
function findMissingNumber(nums) {
    let i = 0;
    while (i < nums.length) {
        const correctIndex = nums[i] - 1;
        if (nums[i] > 0 && nums[i] <= nums.length && nums[i] !== nums[correctIndex]) {
            swap(nums, i, correctIndex);
        } else {
            i++;
        }
    }

    for (let j = 0; j < nums.length; j++) {
        if (nums[j] !== j + 1) {
            return j + 1;
        }
    }
    return nums.length + 1;
}

// Find all duplicates in array
function findDuplicates(nums) {
    const result = [];

    let i = 0;
    while (i < nums.length) {
        const correctIndex = nums[i] - 1;
        if (nums[i] !== nums[correctIndex]) {
            swap(nums, i, correctIndex);
        } else {
            i++;
        }
    }

    for (let j = 0; j < nums.length; j++) {
        if (nums[j] !== j + 1) {
            result.push(nums[j]);
        }
    }
    return result;
}

function swap(nums, i, j) {
    const temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

## 🧠 Kadane's Algorithm Pattern

**Idea:** At each index decide: extend the best subarray ending at the previous index, or start fresh here (whichever is larger). Track the best value seen overall.

```javascript
// Maximum Subarray Sum
function maxSubarraySum(nums) {
    let maxSum = nums[0];
    let currentSum = nums[0];

    for (let i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
```

## 🔀 Merge Intervals Pattern

**Idea:** Sort by start time so overlapping intervals become neighbours, then sweep once: extend the current interval while the next one starts before it ends, otherwise close it and start a new one. O(n log n) for the sort.

```javascript
// Merge Intervals
function mergeIntervals(intervals) {
    if (intervals.length === 0) return [];

    // Sort by start time
    intervals.sort((a, b) => a[0] - b[0]);
    const result = [];
    let current = intervals[0];

    for (const interval of intervals) {
        if (current[1] >= interval[0]) {
            current[1] = Math.max(current[1], interval[1]);
        } else {
            result.push(current);
            current = interval;
        }
    }
    result.push(current);
    return result;
}
```

## 🪜 Dutch National Flag Pattern (3-Way Partition)

**Idea:** Maintain three regions: `[0, low)` holds 0s, `[low, mid)` holds 1s, `(high, end]` holds 2s. Inspect `nums[mid]` and swap it into the right region; one pass, O(1) space.

```javascript
// Sort Colors (0, 1, 2)
function sortColors(nums) {
    let low = 0;
    let mid = 0;
    let high = nums.length - 1;

    while (mid <= high) {
        switch (nums[mid]) {
            case 0:
                swap(nums, low, mid);
                low++;
                mid++;
                break;
            case 1:
                mid++;
                break;
            case 2:
                swap(nums, mid, high);
                high--;
                break;
        }
    }
}

function swap(nums, i, j) {
    const temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

## 🧺 Bucket Sort Pattern

**Idea:** A frequency can never exceed `n`, so use an array of `n + 1` buckets indexed by frequency. Reading buckets from high to low yields the most frequent elements in O(n) without sorting.

```javascript
// Top K frequent elements
function topKFrequent(nums, k) {
    const frequencyMap = new Map();
    for (const num of nums) {
        frequencyMap.set(num, (frequencyMap.get(num) || 0) + 1);
    }

    const buckets = Array.from({ length: nums.length + 1 }, () => []);
    for (const [num, freq] of frequencyMap) {
        buckets[freq].push(num);
    }

    const result = [];
    for (let i = buckets.length - 1; i >= 1; i--) {
        for (const num of buckets[i]) {
            result.push(num);
            if (result.length === k) return result;
        }
    }
    return result;
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 1 | Two Sum | Hash Map | Easy |
| 3 | Longest Substring Without Repeating Characters | Sliding Window | Medium |
| 11 | Container With Most Water | Two Pointers | Medium |
| 53 | Maximum Subarray | Kadane's | Medium |
| 56 | Merge Intervals | Sort + Merge | Medium |
| 75 | Sort Colors | Dutch National Flag | Medium |
| 76 | Minimum Window Substring | Sliding Window | Hard |
| 347 | Top K Frequent Elements | Bucket Sort / Heap | Medium |
| 560 | Subarray Sum Equals K | Prefix Sum | Medium |