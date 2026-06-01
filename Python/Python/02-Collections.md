# Collections

## Lists

- Creating a list - `ints = ["John", "Doe", "Bob"]`
- Accessing elements - `ints[1]` - returns `Doe`
- Length - `len(ints)` - returns `4`
- Adding element at the end - `ints.append("Mary")`
- Removing an element - `ints.remove("Doe")`
- Sum of elements - `sum(ints)`
- Joining elements of list - `", "join(ints)`
- List comprehension - 
```
nums = [1, 2, 3, 4]
doubled_nums = [n * 2 for n in nums]
```

- List comprehension with conditionals -
```
nums = [1, 2, 3, 4, 5]
odds = [n for n in nums if n % 2 == 1]
```

## Tuples

- Tuples are immutable.
- Creating a tuple - `tuple = ("John", "Doe")`
- Adding an element - `tuple = tuple + ("Bob,")` - creates a new tuple and assigns it to same variable, however the tuple itself doesn't change.

> [!NOTE]
> In `x = "John"`, `x` is a `string`. But in `x = "John,"`, `x` becomes a tuple.

- Destructuring tuples -
```
currencies = 0.8, 1.2
usd, eur = currencies
```

- List comprehension is also applicable on tuples in the same way.

## Sets

- Creating a set - `set = {"John", "Doe"}`
- Adding an element - `set.add("Bob")`
- Removing an element - `set.remove("John")`
- Difference - `s1.difference(s2)`
- Symmetric difference (elements present in either sets but not both) - `s1.symmetric_difference(s2)`
- Intersection - `s1.intersection(s2)`
- Union - `s1.union(s2)`
- Set comprehension -
```
s = { 1, 2, 3, 4}
doubled_s = { n * 2 for n in nums }
```

> [!TIP]
> To output a set from a list comprehension -
> ```
> nums = [1, 2, 3, 4]
> s = { n * 2 for n in nums }
> ```

## Dictionaries

- Creating a dict - `d = { "John": 1, "Doe": 2}`
- Retrieving value by key - `d["John"]` returns `1`
- Adding or updating an element - 
    - `d['John'] = 3` - updates `John` key with value `3`
    - `d['Bob'] = 4`  - adds new key `Bob` with value `4`

> [!TIP]
> In Python 3.7+, dictonaries keep the order of the elements in which they are inserted.

- Converting list of tuples to dict - `dict(list_of_tuples)`
- Dictionary comprehension -
```
keys = [1, 2, 3, 4]
values = ['A', 'B', 'C', 'D']

{
    keys[i]: values[i]
    for i in range(len(keys))
}
```

> [!TIP]
> To check if element is present in a collection, use `in` keyword - `1 in [1, 2, 3, 4]` - returns `True`.

## Slicing

- Extracts sub-collection based on specified starting and ending index.
```
nums = [1, 2, 3, 4, 5]

nums[2:4]      # [3, 4]
nums[1:]       # [2, 3, 4, 5]
nums[:4]       # [1, 2, 3, 4]
nums[:]        # [1, 2, 3, 4, 5]
nums[-3:4]     # [3, 4]
nums[-3:]      # [3, 4, 5]
nums[:-2]      # [1, 2, 3]
nums[-3:-1]    # [3, 4]
```

> [!TIP]
> `nums[:]` returns a new list.

## `zip` function

- Combines elements for two or more collections based on their index.
```
keys = [1, 2, 3, 4]
values = ['A', 'B', 'C', 'D']

pairs = list(zip(keys, values) )     # [(1, 'A'), (2, 'B'), (3, 'C'), (4, 'D')]
```

- If one of the list has more elements than the others, `zip` will ignore the extra elements.

## Enumerate function

- Used for iterating over a collection along with the indices.
```
nums = [10, 20, 30, 40]

for i, n in enumerate(nums):        # i is the index and n is the list element
    <...>
```

- Returns list of tuples just like `zip` but with the indices -
```
list(enumerate(nums))               # [(0, 10), (1, 20), (2, 30), (3, 40)]
```

> By default, `enumerate` starts from 0th index. If we want to specify starting index, use parameter `start` as - `enumerate(nums, start=1)`

