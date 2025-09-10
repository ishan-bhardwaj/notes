# Arrays

## Array Concatenation (E)

- Problem - Given an integer array `nums` of length `n`, return a new array of length `2n` formed by concatenating `nums` with itself.
- Example - 
```
Input: nums = [1,4,1,2]
Output: [1,4,1,2,1,4,1,2]
```

### Solution 1
```
def getConcatenation(nums: List[int]) -> List[int]:
    n = len(nums)
    ans = [None] * (2 * n)                # preallocate
    for i, x in enumerate(nums):
        ans[i] = ans[i + n] = x
    return ans
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(n)`

### Solution 2
```
def getConcatenation(nums: List[int]) -> List[int]:
    return nums + nums
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(n)`

- For very large arrays, Solution 2 can increase runtime and peak memory usage slightly -
    - Python creates a new list of size `2n`.
    - Then it copies all elements of nums twice into the new list - Copy the first `nums → ans[0:n]` and then Copy the second `nums → ans[n:2n]`.
    - Internally, this involves __two__ memory copy loops.
    - Also, during the copy, Python may temporarily use internal buffers, increasing peak memory usage slightly.


## Contains Duplicate (E)

- Problem - Given an integer array `nums`, return `true` if any value appears more than once in the array, otherwise return `false`.
- Example -
```
Input: nums = [1, 2, 3, 3]
Output: true
```

### Solution 1
```
def containsDuplicate(nums: List[int]) -> bool:
    seen = set()
    for num in nums:
        if num in seen:
            return True             # early exit
        seen.add(num)
    return False
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(n)`

### Solution 2
```
def containsDuplicate(nums: List[int]) -> bool:
    return len(nums) != len(set(nums))
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(n)`

- Does not allow early exit.

### Solution 3
- If only constant space is allowed -
```
def containsDuplicate(nums: List[int]) -> bool:
    nums.sort()                         # in-place sorting
    for i in range(1, len(nums)):
        if nums[i] == nums[i-1]:
            return True
    return False
```

- Approach -
    - Sort the array in-place (`O(n log n)` time, `O(1)` extra space).
    - Compare adjacent elements -
        - If any two neighbors are equal → duplicate found → return `True`.
        - Otherwise → continue scanning.
        - If no duplicates → return `False`.

- Complexity -
    - Time - `O(n log n)`
    - Space - `O(1)`

- Supports early exit.

### Solution 4
- Using bitmask -
```
def containsDuplicate(nums: List[int]) -> bool:
    bitmask = 0
    for num in nums:
        if (bitmask >> num) & 1:
            return True       # Duplicate found
        bitmask |= 1 << num   # Mark number as seen
    return False
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(1)`

- Approach -
    - bitmask is a single integer where each bit represents whether a number has been seen.
        - Bit at position `i = 1` → number i has been seen.
        - Bit at position `i = 0` → number i has not been seen.
    - For each number `num` in `nums` -
        - Check `(bitmask >> num) & 1` → if `1` → duplicate found → return `True`.
        - Otherwise, set the bit → `bitmask |= 1 << num`.
    - If the loop completes without finding duplicates → return `False`.

- Pros -
    - Supports early exits.
    - Constant space — only one integer needed.
    - Very fast — bit operations are extremely cheap.

- Cons -
    - Works only for small non-negative integers (e.g., 0–31 for 32-bit integer, 0–63 for 64-bit).
    - Cannot handle negative numbers without modification.
    - Cannot handle large numbers directly — would need multiple integers or a bit array.

## Anagram (E)

- Problem - Given two strings `s` and `t`, return `true` if the two strings are anagrams of each other, otherwise return `false`.
- An anagram is a string that contains the exact same characters as another string, but the order of the characters can be different.
- Example -
```
Input: s = "racecar", t = "carrace"
Output: true
```

### Solution 1

- Using 2 hashmaps -
```
def isAnagram(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False

    countS, countT = {}, {}

    for i in range(len(s)):
        countS[s[i]] = 1 + countS.get(s[i], 0)
        countT[t[i]] = 1 + countT.get(t[i], 0)
    return countS == countT
```

- Complexity -
    - Time - `O(n + m)`, where `n` is length of `s` and `m` is length of `t`
    - Space - `O(1)`, since we have at most 26 different characters.

### Solution 2

- Using single hashmap -
```
def isAnagram(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False

    count = {}
    for i in range(len(s)):
        count[s[i]] = count.get(s[i], 0) + 1
        count[t[i]] = count.get(t[i], 0) - 1

    return all(v == 0 for v in count.values())
```

- Complexity -
    - Time - `O(n + m)`, where `n` is length of `s` and `m` is length of `t`
    - Space - `O(1)`, since we have at most 26 different characters.

## Two Sum (E)

- Problem - Given an array of integers `nums` and an integer `target`, return the indices `i` and `j` such that `nums[i] + nums[j] == target` and `i != j`.
- Example -
```
Input: nums = [3,4,5,6], target = 7
Output: [0,1]
```

### Solution

```
def twoSum(nums: List[int], target: int) -> List[int]:
    seen = {}  # value -> index

    for i, n in enumerate(nums):
        diff = target - n
        if diff in seen:
            return [seen[diff], i]
        seen[n] = i

    return []
```

- Complexity - 
    - Time - `O(n)`
    - Space - `O(n)`

## Longest Common Prefix (E)

- Problem - You are given an array of strings `strs`. Return the longest common prefix of all the strings.
- Example -
```
Input: strs = ["bat","bag","bank","band"]
Output: "ba"
```

### Solution 1
- Horizontal scanning -
```
def longestCommonPrefix(strs: List[str]) -> str:
    if not strs:
        return ""
    
    prefix = strs[0]
    for s in strs[1:]:
        i = 0
        while i < len(prefix) and i < len(s) and prefix[i] == s[i]:
            i += 1

        prefix = prefix[:i]
        if not prefix:
            break

    return prefix
```

- Complexity -
    - Time - `O(S)`, where `S` is total number of characters in all Strings.
    - Space - `O(m)`, where `m` is length of first string.

### Solution 2
- Vertical Scanning -
```
def longestCommonPrefix(strs: List[str]) -> str:
    if not strs:
        return ""
    
    first = strs[0]
    for i in range(len(first)):
        char = first[i]
        for s in strs[1:]:
            if i >= len(s) or s[i] != char:
                return first[:i]
    return first
```

- Complexity -
    - Time - `O(S)`, where `S` is total number of characters in all Strings.
    - Space - `O(1)`

### Solution 3
- Prefix Shrinking -
```
def longestCommonPrefix(strs: List[str]) -> str:
    if not strs:
        return ""
    
    prefix = strs[0]
    for s in strs[1:]:
        while not s.startswith(prefix):
            prefix = prefix[:-1]
            if not prefix:
                return ""
    return prefix
```

- Complexity -
    - Time - `O(S)`, where `S` is total number of characters in all Strings.
    - Space - `O(m)`, where `m` is length of first string.

### Solution 4
- Divide and Conquer -
```
def longestCommonPrefix(strs: List[str]) -> str:
    if not strs:
        return ""
    def lcp(left: int, right: int) -> str:
        if left == right:
            return strs[left]
        mid = (left + right) // 2
        l = lcp(left, mid)
        r = lcp(mid + 1, right)
        i = 0
        while i < len(l) and i < len(r) and l[i] == r[i]:
            i += 1
        return l[:i]
    
    return lcp(0, len(strs) - 1)
```

- Complexity -
    - Time - `O(S)`, where `S` is total number of characters in all Strings.
    - Space - `O(m log n)`, where `m` is the length of the common prefix at each recursion step (≤ length of shortest string) and `n` is the number of strings.

## Group Anagrams (M)

- Problem - Given an array of strings strs, group all anagrams together into sublists. You may return the output in any order.
- Example -
```
Input: strs = ["act","pots","tops","cat","stop","hat"]
Output: [["hat"],["act", "cat"],["stop", "pots", "tops"]]
```

### Solution 1
```
def groupAnagrams(strs: List[str]) -> List[List[str]]:
    groups = {}
    
    for s in strs:
        # Count characters a-z
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        key = tuple(count)
        
        if key not in groups:
            groups[key] = []
        groups[key].append(s)
    
    return list(groups.values())
```

- Explanation -
    - `ord(c)` gives the ASCII code of character `c` i.e. `a` → `97`, `b` → `98`, `c` → `99`, ... `z` → `122`.
    - `ord(c) - ord('a')` maps letters `'a'-'z'` → `0-25`.
    - `count[ord(c) - ord('a')] += 1` - Increments the count of that letter in count.
    - `count` for "eat" - `[1,0,0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0]`

- Complexity -
    - Time - `O(n * m)`, where `n` is number of strings in `strs` and `m` is the maximum length of a string.
    - Space - `O(n * m)`
        - Dictionary keys - tuple of length `26` for each unique anagram group → `O(n * 26) ≈ O(n)`
        - Dictionary values - storing all input strings → `O(n * m)`