# Hash Tables in Kotlin

> Kotlin implementation reference for hash maps and sets: frequency counting, hash + sliding window, and set-based uniqueness checks.
> New to this topic? Learn it first in [03. Hashing: Maps and Sets](../learn/03-hashing-maps-sets.md).

## Complexity Overview

| Operation | HashMap | HashSet |
|-----------|---------|---------|
| get/put/containsKey | O(1) avg | O(1) avg |
| remove | O(1) avg | O(1) avg |

> **Key Interview Point:** Kotlin's `getOrDefault()`, `in` operator, and `map[key] = (map[key] ?: 0) + 1` pattern make hash operations cleaner. Use `mutableMapOf<K,V>()` and `mutableSetOf<T>()`.

---

## Basic Operations

```kotlin
// Basic HashMap operations
fun hashMapOperations() {
    val map = mutableMapOf<String, Int>()

    // Put
    map["Alice"] = 25
    map["Bob"] = 30

    // Get
    val age = map["Alice"] // 25

    // Check if key exists
    if ("Alice" in map) {
        println("Alice exists")
    }

    // Remove
    map.remove("Bob")

    // Iterate
    for ((name, age) in map) {
        println("$name is $age years old")
    }
}

// HashSet operations
fun hashSetOperations() {
    val set = mutableSetOf<String>()

    // Add
    set.add("Apple")
    set.add("Banana")

    // Check if contains
    if ("Apple" in set) {
        println("Apple is in set")
    }

    // Remove
    set.remove("Banana")

    // Size
    println(set.size) // 1
}
```

## 📦 Frequency Counting Pattern

**Idea:** Count how often each value appears with `map[x] = (map[x] ?: 0) + 1`, then answer the question from the counts. Two strings are anagrams exactly when their count maps are equal.

```kotlin
// Majority element (appears more than n/2 times)
fun majorityElement(nums: IntArray): Int {
    val frequencyMap = mutableMapOf<Int, Int>()

    for (num in nums) {
        frequencyMap[num] = frequencyMap.getOrDefault(num, 0) + 1
    }

    var majority = nums[0]
    var maxCount = 0

    for ((num, count) in frequencyMap) {
        if (count > maxCount) {
            maxCount = count
            majority = num
        }
    }

    return majority
}

// Find all anagrams in a string
fun findAnagrams(s: String, p: String): List<Int> {
    val result = mutableListOf<Int>()
    if (s.length < p.length) return result

    val pCount = mutableMapOf<Char, Int>()
    val sCount = mutableMapOf<Char, Int>()

    // Initialize pCount
    for (char in p) {
        pCount[char] = pCount.getOrDefault(char, 0) + 1
    }

    // Initialize first window
    for (i in 0 until p.length) {
        sCount[s[i]] = sCount.getOrDefault(s[i], 0) + 1
    }

    if (sCount == pCount) result.add(0)

    // Slide window
    for (i in p.length until s.length) {
        // Remove leftmost character
        val leftChar = s[i - p.length]
        sCount[leftChar] = sCount[leftChar]!! - 1
        if (sCount[leftChar] == 0) {
            sCount.remove(leftChar)
        }

        // Add new character
        val rightChar = s[i]
        sCount[rightChar] = sCount.getOrDefault(rightChar, 0) + 1

        if (sCount == pCount) {
            result.add(i - p.length + 1)
        }
    }

    return result
}
```

## 🧩 Hash + Sliding Window Pattern

**Idea:** The map/set holds what is inside the current window. Expand `right` to add characters; while the window is valid (or invalid, depending on the problem), move `left` and remove characters. `formed` counts how many required characters currently have enough copies.

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

// Minimum window substring
fun minWindow(s: String, t: String): String {
    if (t.isEmpty()) return ""

    val tCount = mutableMapOf<Char, Int>()
    val windowCount = mutableMapOf<Char, Int>()

    // Count characters in t
    for (char in t) {
        tCount[char] = tCount.getOrDefault(char, 0) + 1
    }

    var left = 0
    var right = 0
    var formed = 0
    val required = tCount.size

    var minLen = Int.MAX_VALUE
    var minLeft = 0

    while (right < s.length) {
        val char = s[right]
        windowCount[char] = windowCount.getOrDefault(char, 0) + 1

        if (char in tCount && windowCount[char] == tCount[char]) {
            formed++
        }

        while (left <= right && formed == required) {
            // Update minimum window
            if (right - left + 1 < minLen) {
                minLen = right - left + 1
                minLeft = left
            }

            val leftChar = s[left]
            windowCount[leftChar] = windowCount[leftChar]!! - 1

            if (leftChar in tCount && windowCount[leftChar]!! < tCount[leftChar]!!) {
                formed--
            }

            left++
        }

        right++
    }

    return if (minLen == Int.MAX_VALUE) "" else s.substring(minLeft, minLeft + minLen)
}
```

## 📌 Set for Uniqueness Pattern

**Idea:** A set answers "have I seen this before?" in O(1). For Happy Number, a repeated value means you are stuck in a cycle that will never reach 1.

```kotlin
// Contains duplicate
fun containsDuplicate(nums: IntArray): Boolean {
    val set = mutableSetOf<Int>()

    for (num in nums) {
        if (set.contains(num)) {
            return true
        }
        set.add(num)
    }

    return false
}

// Happy number
fun isHappy(n: Int): Boolean {
    val seen = mutableSetOf<Int>()
    var n = n // parameters are read-only in Kotlin, so shadow with a var

    while (n != 1 && n !in seen) {
        seen.add(n)
        var sum = 0
        var number = n

        while (number > 0) {
            val digit = number % 10
            sum += digit * digit
            number /= 10
        }

        n = sum
    }

    return n == 1
}
```
