# Arrays

## Array Concatenation

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


## Contains Duplicate

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
def containsDuplicate(self, nums: List[int]) -> bool:
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

## Anagram

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

