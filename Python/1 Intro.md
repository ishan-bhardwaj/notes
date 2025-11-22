# Python

## Variables

```
a = 10
print(a)        # 10

a = 20
print(a)        # 20

PI = 3.14
```

- Generally, we use upper-case letters to denote constants, though they are still variables.

> [!TIP]
> Division operator (`/`)  will always yield a float. To remove any decimal digits i.e. integer division, use `//`.
> ```
> 4 / 2 = 2.0
> 5 / 2 = 2.5
> 4 // 2 = 2
> 5 // 2 = 2.5
> ```

## Strings

- Can be represented either using double (`"`) or single quotes (`'`).
- Multi-line strings -
```
"""Hello

World!
"""
```

- Multi-line strings are useful for documentation.
- Concatenation -
```
"Hello" + " World"          # Hello World
"My age is: " + 30          # Error!
"My age is: " + str(30)     # My age is: 30
```

> [!TIP]
> `"X" * 5` returns `XXXXX`.

- f-Interpolation -
```
age = 30
print(f"Age: {age}")        # Age: 30
```

> [!WARNING]
> ```
> age = 20
> prompt = f"Age: {age}"
> print(prompt)             # Age: 20
> 
> age = 30
> print(prompt)             # Age: 20
> ```

- `format` interpolation -
```
age = 20
prompt = "Age: {}"
print(prompt.format(age))        # Age: 20

age = 30
print(prompt.format(age))        # Age: 30
```

- Using named parameters in `format` -
```
age = 30
prompt = "Age: {age}"
print(promt.format(age=age))     # Age: 30
```

## User Input

```
username = input("Enter your name:")
print(f"Hello {username}")
```

- `input` always return a `string`.

## Boolean

- `True` and `False`
- AND operator - `and`
- OR operator - `or`

```
bool(0)             # False
bool(10)            # True

bool("")            # False
bool("Hello")       # True

bool([])            # False
bool([1, 2, 3, 4])  # True
```

## Lists

- Creating a list - `ints = ["John", "Doe", "Bob"]`
- Accessing elements - `ints[1]` - returns `Doe`
- Length - `len(ints)` - returns `4`
- Adding element at the end - `ints.append("Mary")`
- Removing an element - `ints.remove("Doe")`

## Tuples

- Tuples are immutable.
- Creating a tuple - `tuple = ("John", "Doe")`
- Adding an element - `tuple = tuple + ("Bob,")` - creates a new tuple and assigns it to same variable, however the tuple itself doesn't change.

> [!NOTE]
> In `x = "John"`, `x` is a `string`. But in `x = "John,"`, `x` becomes a tuple.

## Sets

- Creating a set - `set = {"John", "Doe"}`
- Adding an element - `set.add("Bob")`
- Removing an element - `set.remove("John")`

