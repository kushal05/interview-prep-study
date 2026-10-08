# Hash Tables in Swift

> Swift implementation reference for hash tables: `Dictionary` and `Set` basics, frequency counting, hash + sliding window, and uniqueness checks.
> New to this topic? Learn it first in [03. Hashing: Maps and Sets](../learn/03-hashing-maps-sets.md).

## Complexity Overview

| Operation | Dictionary | Set |
|-----------|-----------|-----|
| Get/Set | O(1) avg | O(1) avg |
| Contains | O(1) avg | O(1) avg |
| Remove | O(1) avg | O(1) avg |

> **Key Interview Point:** Swift's `Dictionary` subscript returns an optional. Use `dict[key, default: 0] += 1` for clean frequency counting.

> **Common Mistake:** Using `dict[key]! += 1` will crash if key doesn't exist. Use `dict[key, default: 0] += 1` instead.

---

## Basic Operations

```swift
// Basic Dictionary operations
func dictionaryOperations() {
    var map = [String: Int]()

    // Put
    map["Alice"] = 25
    map["Bob"] = 30

    // Get
    let age = map["Alice"] // Optional(25)

    // Check if key exists (idiomatic, always O(1) avg)
    if map["Alice"] != nil {
        print("Alice exists")
    }

    // Remove
    map.removeValue(forKey: "Bob")

    // Iterate
    for (name, age) in map {
        print("\(name) is \(age) years old")
    }
}

// Set operations
func setOperations() {
    var set: Set<String> = []

    // Add
    set.insert("Apple")
    set.insert("Banana")

    // Check if contains
    if set.contains("Apple") {
        print("Apple is in set")
    }

    // Remove
    set.remove("Banana")

    // Size
    print(set.count) // 1
}
```

## 📦 Frequency Counting Pattern

**Idea:** Count occurrences in a dictionary in one pass, then answer the question from the counts. For anagram windows, keep a count for the current fixed-size window and update it by one character in and one out as it slides. (Majority Element can also be solved in O(1) space with Boyer-Moore voting.)

```swift
// Majority element (appears more than n/2 times)
func majorityElement(_ nums: [Int]) -> Int {
    var frequencyMap = [Int: Int]()

    for num in nums {
        frequencyMap[num, default: 0] += 1
    }

    var majority = nums[0]
    var maxCount = 0

    for (num, count) in frequencyMap {
        if count > maxCount {
            maxCount = count
            majority = num
        }
    }

    return majority
}

// Find all anagrams in a string
func findAnagrams(_ s: String, _ p: String) -> [Int] {
    var result = [Int]()
    guard s.count >= p.count else { return result }

    var pCount = [Character: Int]()
    var sCount = [Character: Int]()
    let sChars = Array(s)
    let pChars = Array(p)

    // Initialize pCount
    for char in pChars {
        pCount[char, default: 0] += 1
    }

    // Initialize first window
    for i in 0..<p.count {
        sCount[sChars[i], default: 0] += 1
    }

    if sCount == pCount {
        result.append(0)
    }

    // Slide window
    for i in p.count..<s.count {
        // Remove leftmost character
        let leftChar = sChars[i - p.count]
        sCount[leftChar]! -= 1
        if sCount[leftChar] == 0 {
            sCount.removeValue(forKey: leftChar)
        }

        // Add new character
        let rightChar = sChars[i]
        sCount[rightChar, default: 0] += 1

        if sCount == pCount {
            result.append(i - p.count + 1)
        }
    }

    return result
}
```

## 🧩 Hash + Sliding Window Pattern

**Idea:** A variable-size window plus a dictionary/set describing what is inside it. Expand `right` until the window is valid, then shrink `left` while it stays valid, recording the best answer. `formed` counts how many distinct required characters currently have enough copies.

```swift
// Longest substring without repeating characters
func lengthOfLongestSubstring(_ s: String) -> Int {
    var charSet: Set<Character> = []
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

// Minimum window substring
func minWindow(_ s: String, _ t: String) -> String {
    guard !t.isEmpty else { return "" }

    var tCount = [Character: Int]()
    var windowCount = [Character: Int]()
    let sChars = Array(s)
    let tChars = Array(t)

    // Count characters in t
    for char in tChars {
        tCount[char, default: 0] += 1
    }

    var left = 0
    var right = 0
    var formed = 0
    let required = tCount.count

    var minLen = Int.max
    var minLeft = 0

    while right < sChars.count {
        let char = sChars[right]
        windowCount[char, default: 0] += 1

        if let count = tCount[char], windowCount[char] == count {
            formed += 1
        }

        while left <= right && formed == required {
            // Update minimum window
            if right - left + 1 < minLen {
                minLen = right - left + 1
                minLeft = left
            }

            let leftChar = sChars[left]
            windowCount[leftChar]! -= 1

            if let count = tCount[leftChar], windowCount[leftChar]! < count {
                formed -= 1
            }

            left += 1
        }

        right += 1
    }

    return minLen == Int.max ? "" : String(sChars[minLeft..<minLeft + minLen])
}
```

## 📌 Set for Uniqueness Pattern

**Idea:** A `Set` answers "have I seen this before?" in O(1) average. Use it to detect duplicates, or to detect a cycle in a sequence of states (Happy Number repeats a value if it never reaches 1).

```swift
// Contains duplicate
func containsDuplicate(_ nums: [Int]) -> Bool {
    var set: Set<Int> = []

    for num in nums {
        if set.contains(num) {
            return true
        }
        set.insert(num)
    }

    return false
}

// Happy number
func isHappy(_ n: Int) -> Bool {
    var seen: Set<Int> = []
    var number = n

    while number != 1 && !seen.contains(number) {
        seen.insert(number)
        var sum = 0
        var temp = number

        while temp > 0 {
            let digit = temp % 10
            sum += digit * digit
            temp /= 10
        }

        number = sum
    }

    return number == 1
}
```

---

## Summary: Hash Table Pattern Map

| Pattern                | Keywords / Use Case                        |
|-----------------------|-------------------------------------------|
| Frequency Counting    | majority element, anagrams, most frequent |
| Hash + Sliding Window | longest substring, minimum window         |
| Set for Uniqueness    | duplicates, cycles, unique elements       |
| Dictionary Operations | key-value mapping, counting, lookup       |

---