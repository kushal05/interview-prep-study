# Hash Tables in Java

> Java implementation reference for `HashMap`/`HashSet`: frequency counting, hash + sliding window, and set-based uniqueness checks.
> New to this topic? Learn it first in [03. Hashing: Maps and Sets](../learn/03-hashing-maps-sets.md).

## Complexity Overview

| Operation | HashMap | HashSet | TreeMap |
|-----------|---------|---------|---------|
| Get/Put | O(1) avg | O(1) avg | O(log n) |
| ContainsKey | O(1) avg | O(1) avg | O(log n) |
| Remove | O(1) avg | O(1) avg | O(log n) |
| Iteration | O(n) | O(n) | O(n) |

> **Key Interview Point:** Use `HashMap` for key-value pairs, `HashSet` for unique element tracking. Both have O(1) average operations. Use `getOrDefault()` and `merge()` to write cleaner code.

---

## Basic Operations

```java
// HashMap
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 25);
map.get("Alice");                          // 25
map.getOrDefault("Bob", 0);               // 0
map.containsKey("Alice");                  // true
map.remove("Alice");
map.merge("count", 1, Integer::sum);      // increment pattern
for (var entry : map.entrySet()) { /* entry.getKey(), entry.getValue() */ }

// HashSet
Set<Integer> set = new HashSet<>();
set.add(1);
set.contains(1);   // true
set.remove(1);
set.size();
```

---

## Pattern 1: Frequency Counting

**When to use:** Count occurrences, find majority element, check anagrams.

```java
// Majority element -- Time: O(n), Space: O(n)
public int majorityElement(int[] nums) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    for (var e : freq.entrySet())
        if (e.getValue() > nums.length / 2) return e.getKey();
    return -1;
}

// Find all anagrams -- Time: O(n), Space: O(1) -- alphabet is fixed
public List<Integer> findAnagrams(String s, String p) {
    List<Integer> result = new ArrayList<>();
    if (s.length() < p.length()) return result;

    Map<Character, Integer> need = new HashMap<>(), have = new HashMap<>();
    for (char c : p.toCharArray()) need.merge(c, 1, Integer::sum);

    // Initialize first window
    for (int i = 0; i < p.length(); i++) have.merge(s.charAt(i), 1, Integer::sum);
    if (have.equals(need)) result.add(0);

    // Slide window
    for (int i = p.length(); i < s.length(); i++) {
        have.merge(s.charAt(i), 1, Integer::sum);                      // add right
        char left = s.charAt(i - p.length());
        have.merge(left, -1, Integer::sum);                             // remove left
        if (have.get(left) == 0) have.remove(left);
        if (have.equals(need)) result.add(i - p.length() + 1);
    }
    return result;
}
```

---

## Pattern 2: Hash + Sliding Window

**Idea:** The map/set holds what is inside the current window. Expand `right` to add characters; while the window is valid (or invalid, depending on the problem), move `left` and remove characters. `formed` counts how many required characters currently have enough copies.

```java
// Longest substring without repeating characters -- Time: O(n)
public int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int left = 0, max = 0;
    for (int right = 0; right < s.length(); right++) {
        while (window.contains(s.charAt(right))) window.remove(s.charAt(left++));
        window.add(s.charAt(right));
        max = Math.max(max, right - left + 1);
    }
    return max;
}

// Minimum window substring -- Time: O(n), Space: O(alphabet)
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>(), have = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

    int left = 0, formed = 0, required = need.size();
    int minLen = Integer.MAX_VALUE, minLeft = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        have.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && have.get(c).equals(need.get(c))) formed++;

        while (formed == required) {
            if (right - left + 1 < minLen) { minLen = right - left + 1; minLeft = left; }
            char lc = s.charAt(left);
            have.merge(lc, -1, Integer::sum);
            if (need.containsKey(lc) && have.get(lc) < need.get(lc)) formed--;
            left++;
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(minLeft, minLeft + minLen);
}
```

---

## Pattern 3: Set for Uniqueness

**Idea:** A set answers "have I seen this before?" in O(1). `Set.add` returns `false` for a repeat, which doubles as the check. For Happy Number, a repeated value means you are stuck in a cycle that will never reach 1.

```java
// Contains duplicate -- Time: O(n), Space: O(n)
public boolean containsDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int n : nums) if (!seen.add(n)) return true; // add returns false if already present
    return false;
}

// Happy number -- Time: O(log n), Space: O(log n)
public boolean isHappy(int n) {
    Set<Integer> seen = new HashSet<>();
    while (n != 1 && seen.add(n)) {
        int sum = 0;
        while (n > 0) { int d = n % 10; sum += d * d; n /= 10; }
        n = sum;
    }
    return n == 1;
}
```

> **Common Mistake:** Using `Integer` equality with `==` instead of `.equals()` for values > 127. Java caches Integer objects only for -128 to 127.

---

## Common Interview Questions

| # | Problem | Pattern | Difficulty |
|---|---------|---------|------------|
| 1 | Two Sum | Hash Map Lookup | Easy |
| 169 | Majority Element | Frequency Count | Easy |
| 217 | Contains Duplicate | Set | Easy |
| 242 | Valid Anagram | Frequency Count | Easy |
| 3 | Longest Substring Without Repeating | Hash + Window | Medium |
| 49 | Group Anagrams | Hash Map | Medium |
| 76 | Minimum Window Substring | Hash + Window | Hard |
| 438 | Find All Anagrams | Hash + Window | Medium |
| 202 | Happy Number | Set for Cycle | Easy |
| 560 | Subarray Sum Equals K | Prefix Sum + Hash | Medium |
