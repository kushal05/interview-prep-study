# Hash Tables in JavaScript

> JavaScript implementation reference for hash tables: `Map`, `Set` and plain objects, plus frequency counting, hash + sliding window and set-based uniqueness patterns.
> New to this topic? Learn it first in [03. Hashing: Maps and Sets](../learn/03-hashing-maps-sets.md).

## Complexity Overview

| Operation | Map | Set | Object |
|-----------|-----|-----|--------|
| Get/Set/Has | O(1) avg | O(1) avg | O(1) avg |
| Delete | O(1) avg | O(1) avg | O(1) avg |
| Size | O(1) | O(1) | O(n)* |

*`Object.keys(obj).length` is O(n). Use `Map` for better performance.

> **Key Interview Point:** Prefer `Map` over plain objects for hash maps -- `Map` preserves insertion order, supports any key type, and has O(1) `.size`. Use `Set` for unique element tracking.

> **Common Mistake:** JS `Map.get()` returns `undefined` for missing keys, not `null`. Use `map.get(key) || 0` for safe defaults with numbers, but be careful with falsy values (0, false, "").

---

## Basic Operations

```javascript
// Basic Map operations
function mapOperations() {
    const map = new Map();

    // Put
    map.set("Alice", 25);
    map.set("Bob", 30);

    // Get
    const age = map.get("Alice"); // 25

    // Check if key exists
    if (map.has("Alice")) {
        console.log("Alice exists");
    }

    // Remove
    map.delete("Bob");

    // Iterate
    for (const [name, age] of map) {
        console.log(`${name} is ${age} years old`);
    }
}

// Set operations
function setOperations() {
    const set = new Set();

    // Add
    set.add("Apple");
    set.add("Banana");

    // Check if contains
    if (set.has("Apple")) {
        console.log("Apple is in set");
    }

    // Remove
    set.delete("Banana");

    // Size
    console.log(set.size); // 1
}
```

## 📦 Frequency Counting Pattern

**Idea:** One pass builds a `value -> count` map in O(n); afterwards questions like "most frequent" or "same letters?" become simple map lookups or comparisons. (For majority element specifically, Boyer-Moore voting does it in O(1) space.)

```javascript
// Majority element (appears more than n/2 times)
function majorityElement(nums) {
    const frequencyMap = new Map();

    for (const num of nums) {
        frequencyMap.set(num, (frequencyMap.get(num) || 0) + 1);
    }

    let majority = nums[0];
    let maxCount = 0;

    for (const [num, count] of frequencyMap) {
        if (count > maxCount) {
            maxCount = count;
            majority = num;
        }
    }

    return majority;
}

// Find all anagrams in a string
function findAnagrams(s, p) {
    const result = [];
    if (s.length < p.length) return result;

    const pCount = new Map();
    const sCount = new Map();

    // Initialize pCount
    for (const char of p) {
        pCount.set(char, (pCount.get(char) || 0) + 1);
    }

    // Initialize first window
    for (let i = 0; i < p.length; i++) {
        sCount.set(s[i], (sCount.get(s[i]) || 0) + 1);
    }

    if (mapsEqual(sCount, pCount)) result.push(0);

    // Slide window
    for (let i = p.length; i < s.length; i++) {
        // Remove leftmost character
        const leftChar = s[i - p.length];
        sCount.set(leftChar, sCount.get(leftChar) - 1);
        if (sCount.get(leftChar) === 0) {
            sCount.delete(leftChar);
        }

        // Add new character
        const rightChar = s[i];
        sCount.set(rightChar, (sCount.get(rightChar) || 0) + 1);

        if (mapsEqual(sCount, pCount)) {
            result.push(i - p.length + 1);
        }
    }

    return result;
}

function mapsEqual(map1, map2) {
    if (map1.size !== map2.size) return false;

    for (const [key, value] of map1) {
        if (map2.get(key) !== value) return false;
    }

    return true;
}
```

## 🧩 Hash + Sliding Window Pattern

**Idea:** Grow the window with `right`, track what is inside it in a map/set, and shrink from `left` while the window is invalid (or, for minimum window, while it is still valid). Each index enters and leaves once, so O(n).

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

// Minimum window substring
function minWindow(s, t) {
    if (t.length === 0) return "";

    const tCount = new Map();
    const windowCount = new Map();

    // Count characters in t
    for (const char of t) {
        tCount.set(char, (tCount.get(char) || 0) + 1);
    }

    let left = 0;
    let right = 0;
    let formed = 0;
    const required = tCount.size;

    let minLen = Number.MAX_SAFE_INTEGER;
    let minLeft = 0;

    while (right < s.length) {
        const char = s[right];
        windowCount.set(char, (windowCount.get(char) || 0) + 1);

        if (tCount.has(char) && windowCount.get(char) === tCount.get(char)) {
            formed++;
        }

        while (left <= right && formed === required) {
            // Update minimum window
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minLeft = left;
            }

            const leftChar = s[left];
            windowCount.set(leftChar, windowCount.get(leftChar) - 1);

            if (tCount.has(leftChar) && windowCount.get(leftChar) < tCount.get(leftChar)) {
                formed--;
            }

            left++;
        }

        right++;
    }

    return minLen === Number.MAX_SAFE_INTEGER ? "" : s.substring(minLeft, minLeft + minLen);
}
```

## 📌 Set for Uniqueness Pattern

**Idea:** A `Set` answers "have I seen this before?" in O(1), which detects duplicates and also cycles (if a state repeats, you are looping forever).

```javascript
// Contains duplicate
function containsDuplicate(nums) {
    const set = new Set();

    for (const num of nums) {
        if (set.has(num)) {
            return true;
        }
        set.add(num);
    }

    return false;
}

// Happy number
function isHappy(n) {
    const seen = new Set();
    let number = n;

    while (number !== 1 && !seen.has(number)) {
        seen.add(number);
        let sum = 0;
        let temp = number;

        while (temp > 0) {
            const digit = temp % 10;
            sum += digit * digit;
            temp = Math.floor(temp / 10);
        }

        number = sum;
    }

    return number === 1;
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