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

## Remove Element

- Problem - You are given an integer array `nums` and an integer `val`. Your task is to remove all occurrences of val from `nums` "in-place". After removing all occurrences of `val`, return the number of remaining elements, say `k`, such that the first `k` elements of `nums` do not contain `val`.
- Note - The order of the elements which are not equal to val does not matter.
- Example -
```
Input: nums = [1,1,2,3,4], val = 1
Output: [2,3,4]
```

### Solution
```
def removeElement(self, nums: list[int], val: int) -> int:
    k = 0
    for x in nums:
        if x != val:
            nums[k] = x
            k += 1
    return k
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(1)`

## Majority Element

- Problem - Given an array nums of size n, return the majority element.
- Example - 
```
Input: nums = [5,5,1,1,1,5,5]
Output: 5
```

### Solution 1
```
def majorityElement(self, nums: list[int]) -> int:
    freq = {}
    for x in nums:
        freq[x] = freq.get(x, 0) + 1
    max_elem = max(freq, key=freq.get)
    return max_elem
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(n)`

### Solution 2 - Boyer-Moore Voting Algorithm

> [!WARNING]
> Only works when majority element has greater than `n/2` occurrences.

```
def majorityElement(self, nums):
    res = count = 0
    for num in nums:
        if count == 0:
            res = num
        count += (1 if num == res else -1)
    return res
```

- Complexity -
    - Time - `O(n)`
    - Space - `O(1)`

## Design HashSet

- Problem - Design a HashSet without using any built-in hash table libraries. Implement `MyHashSet` class -
    - `void add(key)` - Inserts the value key into the HashSet.
    - `bool contains(key)` - Returns whether the value key exists in the HashSet or not.
    - `void remove(key)` - Removes the value key in the HashSet. If key does not exist in the HashSet, do nothing.

### Solution 1 - Boolean Array

- Only works if we have finite elements, say 1 million.

```
class MyHashSet:

    def __init__(self):
        self.data = [False] * 1000001

    def add(self, key: int) -> None:
        self.data[key] = True

    def remove(self, key: int) -> None:
        self.data[key] = False

    def contains(self, key: int) -> bool:
        return self.data[key]
```

- Complexity -
    - Time - `O(1)`
    - Space - `O(1000000)`

### Solution 2 - Linked List

```
class ListNode:
    def __init__(self, key: int, next=None):
        self.key = key
        self.next = next

class MyHashSet:
    def __init__(self, initial_capacity: int = 16, load_factor: float = 0.75):
        self.capacity = initial_capacity
        self.size = 0
        self.load_factor = load_factor
        self.buckets = [None] * self.capacity

    def _hash(self, key: int) -> int:
        return key % self.capacity

    def _resize(self):
        old_buckets = self.buckets
        self.capacity *= 2
        self.buckets = [None] * self.capacity
        self.size = 0

        for head in old_buckets:
            cur = head
            while cur:
                self.add(cur.key)
                cur = cur.next

    def add(self, key: int) -> None:
        if self.contains(key):
            return

        if self.size + 1 > self.capacity * self.load_factor:
            self._resize()

        idx = self._hash(key)
        new_node = ListNode(key, next=self.buckets[idx])
        self.buckets[idx] = new_node
        self.size += 1

    def remove(self, key: int) -> None:
        idx = self._hash(key)
        cur = self.buckets[idx]
        prev = None
        while cur:
            if cur.key == key:
                if prev:
                    prev.next = cur.next
                else:
                    self.buckets[idx] = cur.next
                self.size -= 1
                return
            prev, cur = cur, cur.next

    def contains(self, key: int) -> bool:
        idx = self._hash(key)
        cur = self.buckets[idx]
        while cur:
            if cur.key == key:
                return True
            cur = cur.next
        return False
```

- Complexity -
    - Let -
        - `n` is number of elements in the hash set
        - `m` is number of buckets (capacity)
        - Load factor `α` is `n / m`
    - Add Operation -
        - Compute hash - `O(1)`
        - Check contains (traverse chain) - `O(α)` on average
        - Insert at head of chain - `O(1)`
        - Average case - `O(1)` because `α ≤ load_factor` (usually `0.75`).
        - Worst case - `O(n)` if all keys collide in the same bucket.
        - Amortized - `O(1)` even with resizing because resizing doubles capacity and spreads keys.
    - Remove Operation -
        - Compute hash - `O(1)`
        - Traverse chain to find key - `O(α)` on average
        - Remove node - `O(1)`
        - Average case - `O(1)`
        - Worst case - `O(n)` if all keys collide
    - Contains Operation -
        - Compute hash - `O(1)`
        - Traverse chain - `O(α)` on average
        - Average case - `O(1)`
        - Worst case - `O(n)` if all keys collide
    - Resizing -
        - Occurs when `size > capacity * load_factor`
        - Copies all elements - `O(n)`
        - Amortized cost per insertion - `O(1)`
        - So all operations are `O(1)` average/amortized, `O(n)` worst case.
    - Space Complexity -
        - Buckets array - `O(m)`, `m` = capacity
        - Nodes - `O(n)` for actual keys
        - Total space - `O(n + m)`
        - Since `m` grows proportionally with `n` during resizing, effectively `O(n)`
    
## Design HashMap

- Problem - Design a HashMap without using any built-in hash table libraries. Implement the MyHashMap class -
    - `MyHashMap()` initializes the object with an empty map.
    - `void put(int key, int value)` inserts a `(key, value)` pair into the HashMap. If the key already exists in the map, update the corresponding value.
    - `int get(int key)` returns the value to which the specified key is mapped, or `-1` if this map contains no mapping for the key.
    - `void remove(key)` removes the key and its corresponding value if the map contains the mapping for the key.

### Solution 1 - Array

- Works only when keys have bounded range.

```
class MyHashMap:

    def __init__(self):
        self.map = [-1] * 1000001

    def put(self, key: int, value: int) -> None:
        self.map[key] = value

    def get(self, key: int) -> int:
        return self.map[key]

    def remove(self, key: int) -> None:
        self.map[key] = -1
```

- Complexity -
    - Time - `O(1)` for each function call.
    - Space - `O(1000000)` since the key is in the range `[0, 1000000]`

### Solution 2 - Linked List

```
class ListNode:
    def __init__(self, key = -1, val = -1, next = None):
        self.key = key
        self.val = val
        self.next = next

class MyHashMap:

    def __init__(self):
        self.map = [ListNode() for _ in range(1000)]

    def hash(self, key: int) -> int:
        return key % len(self.map)

    def put(self, key: int, value: int) -> None:
        cur = self.map[self.hash(key)]
        while cur.next:
            if cur.next.key == key:
                cur.next.val = value
                return
            cur = cur.next
        cur.next = ListNode(key, value)

    def get(self, key: int) -> int:
        cur = self.map[self.hash(key)].next
        while cur:
            if cur.key == key:
                return cur.val
            cur = cur.next
        return -1

    def remove(self, key: int) -> None:
        cur = self.map[self.hash(key)]
        while cur.next:
            if cur.next.key == key:
                cur.next = cur.next.next
                return
            cur = cur.next
```

- Complexity -
    - Let - 
        - `n` is the number of keys.
        - `k` is the size of the map (1000).
        - `m` is the number of unique keys.
    - Time - `O(n/k)` for each function call.
    - Space - `O(k + m)`

### Quick Sort

- QuickSort is a divide-and-conquer sorting algorithm.
- Approach -
    - Pick a pivot element from the array - can pick first, last, middle, or random element. Random pivot better average-case performance.
    - Partition the array into two parts -
        - Elements less than the pivot go to the left.
        - Elements greater than the pivot go to the right.
    - Recursively sort the left and right subarrays.
    - Combine results — because each pivot ends up in its final sorted position, no merging is needed.

```
def quicksort(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pi = partition(arr, low, high)
        quicksort(arr, low, pi - 1)
        quicksort(arr, pi + 1, high)


def partition(arr, low, high):
    pivot = arr[high]  # Choose the last element as pivot
    i = low - 1        # Index of smaller element

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    # Place pivot in the correct position
    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

- Complexity -
    - Best / Average Time - `O(n logn)`
    - Worst Time - `O(n^2)` - pivot always largest or smallest
    - Space - `O(logn)` for recursion stack

> [!TIP]
> worst-case is rare if pivot is chosen randomly.

- Pros -
    - In-place sorting - doesn’t need extra arrays (unlike merge sort).
    - Cache-friendly - works well on large arrays.
    - Very fast in practice; widely used in standard libraries.