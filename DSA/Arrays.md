# Arrays

## Array Concatenation

- Problem - Given an integer array `nums` of length `n`, return a new array of length `2n` formed by concatenating `nums` with itself.
- Example - 
```
Input: nums = [1,4,1,2]
Output: [1,4,1,2,1,4,1,2]
```

**Solution 1**
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

**Solution 2**
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

