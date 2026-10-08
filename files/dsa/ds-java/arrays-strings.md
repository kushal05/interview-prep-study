# Arrays & Strings in Java

> Java implementation reference for arrays and strings: core operations plus sliding window, two pointers, binary search, prefix sum, cyclic sort, Kadane, intervals, Dutch flag and bucket sort.
> New to this topic? Learn it first in [02. Arrays and Strings](../learn/02-arrays-and-strings.md) (then [05. Two Pointers](../learn/05-two-pointers.md) and [06. Sliding Window](../learn/06-sliding-window.md)).

## Complexity Overview

| Operation | Array | ArrayList | String |
|-----------|-------|-----------|--------|
| Access by index | O(1) | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) | O(n) |
| Insert/Delete at end | - | O(1) amortized | O(n) (immutable) |
| Insert/Delete at index | - | O(n) | O(n) (immutable) |

> **Key Interview Point:** Java Strings are immutable. Use `StringBuilder` for building strings character by character -- avoids O(n^2) string concatenation.

---

## Basic Operations

```java
import java.util.*;

public class ArrayOperations {
    public static void arrayOperations() {
        // Fixed-size array
        int[] arr = {1, 2, 3, 4, 5};

        // Dynamic array
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

        // Access: O(1)
        System.out.println(arr[0]); // 1

        // Add to end: O(1) amortized
        list.add(6);

        // Remove at index: O(n)
        list.remove(2);

        // Binary search (array must be sorted): O(log n)
        int index = Arrays.binarySearch(arr, 3);

        // Sort: O(n log n)
        Arrays.sort(arr);
    }

    public static void stringOperations() {
        String str = "Hello World";
        str.length();           // 11
        str.charAt(0);          // 'H'
        str.substring(0, 5);    // "Hello"
        str.split(" ");         // ["Hello", "World"]
        str.replace("World", "Java");

        // Efficient string building
        StringBuilder sb = new StringBuilder();
        for (char c : str.toCharArray()) {
            sb.append(c);
        }
        String result = sb.toString();
    }
}
```

---

## Pattern 1: Sliding Window

**When to use:** Contiguous subarray/substring problems with a constraint.

```java
// Longest substring without repeating characters
// Time: O(n), Space: O(min(n, alphabet_size))
public static int lengthOfLongestSubstring(String s) {
    Set<Character> charSet = new HashSet<>();
    int left = 0, maxLength = 0;

    for (int right = 0; right < s.length(); right++) {
        // Shrink window until no duplicate
        while (charSet.contains(s.charAt(right))) {
            charSet.remove(s.charAt(left));
            left++;
        }
        charSet.add(s.charAt(right));
        maxLength = Math.max(maxLength, right - left + 1);
    }
    return maxLength;
}

// Max sum subarray of fixed size K
// Time: O(n), Space: O(1)
public static int maxSumSubarray(int[] arr, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i];
    int maxSum = windowSum;

    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k]; // slide: add right, remove left
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

> **Common Mistake:** In variable-size sliding window, forgetting to shrink the window. Always have a while-loop inside the for-loop to maintain the constraint.

---

## Pattern 2: Two Pointers

**When to use:** Sorted arrays, in-place modifications, comparing from both ends.

```java
// Reverse array in-place
// Time: O(n), Space: O(1)
public static void reverseArray(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int temp = arr[left];
        arr[left++] = arr[right];
        arr[right--] = temp;
    }
}

// Remove duplicates from sorted array (return new length)
// Time: O(n), Space: O(1)
public static int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int i = 0; // slow pointer: last unique position
    for (int j = 1; j < nums.length; j++) {
        if (nums[j] != nums[i]) {
            nums[++i] = nums[j];
        }
    }
    return i + 1;
}
```

---

## Pattern 3: Binary Search

**When to use:** Sorted data, need O(log n), find boundary/minimum/maximum.

```java
// Standard binary search
// Time: O(log n), Space: O(1)
public static int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoids integer overflow
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}

// Find minimum in rotated sorted array
// Time: O(log n), Space: O(1)
public static int findMin(int[] nums) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] > nums[right]) left = mid + 1; // min is in right half
        else right = mid;                              // min is in left half (including mid)
    }
    return nums[left];
}
```

> **Common Mistake:** Using `(left + right) / 2` which can overflow. Always use `left + (right - left) / 2`.

---

## Pattern 4: Prefix Sum

**When to use:** Multiple range sum queries, counting subarrays with a given sum.

```java
// Subarray sum equals K -- uses prefix sum + hash map
// Time: O(n), Space: O(n)
public static int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0, 1); // empty prefix has sum 0
    int sum = 0, count = 0;

    for (int num : nums) {
        sum += num;
        count += prefixCount.getOrDefault(sum - k, 0); // how many prefixes give sum-k?
        prefixCount.merge(sum, 1, Integer::sum);
    }
    return count;
}

// Range sum query (immutable) -- precompute prefix sums
public static class NumArray {
    private int[] prefix;
    public NumArray(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++)
            prefix[i + 1] = prefix[i] + nums[i];
    }
    public int sumRange(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}
```

---

## Pattern 5: Cyclic Sort

**When to use:** Array contains values in range 1..N, find missing or duplicate.

**Idea:** Value `v` belongs at index `v - 1`. Swap each value into its home slot; afterwards any index whose value is wrong reveals a missing or duplicate number. O(n) time, O(1) extra space.

```java
// First missing positive (LC 41); also finds the missing number when values are 1..N
// Time: O(n), Space: O(1)
public static int findMissingNumber(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int correct = nums[i] - 1;
        if (nums[i] > 0 && nums[i] <= nums.length && nums[i] != nums[correct]) {
            int temp = nums[i]; nums[i] = nums[correct]; nums[correct] = temp;
        } else {
            i++;
        }
    }
    for (int j = 0; j < nums.length; j++) {
        if (nums[j] != j + 1) return j + 1;
    }
    return nums.length + 1;
}

// Find all duplicates (LC 442: values in 1..n, each appears once or twice)
// Time: O(n), Space: O(1) (output excluded)
public static List<Integer> findDuplicates(int[] nums) {
    List<Integer> result = new ArrayList<>();
    int i = 0;
    while (i < nums.length) {
        int correct = nums[i] - 1;
        if (nums[i] != nums[correct]) {
            int temp = nums[i]; nums[i] = nums[correct]; nums[correct] = temp;
        } else {
            i++;
        }
    }
    for (int j = 0; j < nums.length; j++) {
        if (nums[j] != j + 1) result.add(nums[j]);
    }
    return result;
}
```

---

## Pattern 6: Kadane's Algorithm

**Idea:** At each index decide: extend the best subarray ending at the previous index, or start fresh here (whichever is larger). Track the best value seen overall.

```java
// Maximum subarray sum
// Time: O(n), Space: O(1)
public static int maxSubarraySum(int[] nums) {
    int maxSum = nums[0], currentSum = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]); // extend or restart
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
```

---

## Pattern 7: Merge Intervals

**Idea:** Sort by start time so overlapping intervals become neighbours, then sweep once: extend the current interval while the next one starts before it ends, otherwise close it and start a new one. O(n log n) for the sort.

```java
// Time: O(n log n) for sort, Space: O(n)
public static int[][] mergeIntervals(int[][] intervals) {
    if (intervals.length == 0) return new int[0][];
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));

    List<int[]> result = new ArrayList<>();
    int[] current = intervals[0];
    for (int[] interval : intervals) {
        if (current[1] >= interval[0]) {
            current[1] = Math.max(current[1], interval[1]); // merge
        } else {
            result.add(current);
            current = interval;
        }
    }
    result.add(current);
    return result.toArray(new int[result.size()][]);
}
```

---

## Pattern 8: Dutch National Flag (3-Way Partition)

**Idea:** Maintain three regions: `[0, low)` holds 0s, `[low, mid)` holds 1s, `(high, end]` holds 2s. Inspect `nums[mid]` and swap it into the right region; one pass, O(1) space.

```java
// Sort Colors -- sort array of 0s, 1s, 2s in-place
// Time: O(n), Space: O(1)
public static void sortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.length - 1;
    while (mid <= high) {
        switch (nums[mid]) {
            case 0 -> { swap(nums, low++, mid++); }
            case 1 -> { mid++; }
            case 2 -> { swap(nums, mid, high--); }
        }
    }
}

private static void swap(int[] nums, int i, int j) {
    int temp = nums[i]; nums[i] = nums[j]; nums[j] = temp;
}
```

> **Common Mistake:** Incrementing `mid` when swapping with `high`. After swapping with `high`, the new value at `mid` is unknown -- do not increment `mid`.

---

## Pattern 9: Bucket Sort

**Idea:** A frequency can never exceed `n`, so use an array of `n + 1` buckets indexed by frequency. Reading buckets from high to low yields the most frequent elements in O(n) without sorting.

```java
// Top K frequent elements
// Time: O(n), Space: O(n)
public static int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) freq.merge(num, 1, Integer::sum);

    @SuppressWarnings("unchecked")
    List<Integer>[] buckets = new List[nums.length + 1];
    for (int i = 0; i < buckets.length; i++) buckets[i] = new ArrayList<>();
    for (var entry : freq.entrySet()) buckets[entry.getValue()].add(entry.getKey());

    List<Integer> result = new ArrayList<>();
    for (int i = buckets.length - 1; i >= 1 && result.size() < k; i--) {
        result.addAll(buckets[i]);
    }
    return result.stream().limit(k).mapToInt(Integer::intValue).toArray();
}
```

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 1 | Two Sum | Hash Map | Easy |
| 3 | Longest Substring Without Repeating Characters | Sliding Window | Medium |
| 11 | Container With Most Water | Two Pointers | Medium |
| 15 | 3Sum | Two Pointers | Medium |
| 33 | Search in Rotated Sorted Array | Binary Search | Medium |
| 53 | Maximum Subarray | Kadane's | Medium |
| 56 | Merge Intervals | Sort + Merge | Medium |
| 75 | Sort Colors | Dutch National Flag | Medium |
| 76 | Minimum Window Substring | Sliding Window | Hard |
| 347 | Top K Frequent Elements | Bucket Sort / Heap | Medium |
| 560 | Subarray Sum Equals K | Prefix Sum | Medium |
