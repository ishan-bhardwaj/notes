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

## If-else

```
if <condition1>:
    <...>
elif <condition2>:
    <...>
else:
    <...>
```

## Loops

### While loops

```
while <condition>:
    <...>
```

### for loops

- Iterating collections -
```
l = [1, 2, 3, 4]

for i in l:
    <...>
```

> [!TIP]
> Use underscore (`_`) if the variables are unused, eg - `for _ in l`

- Iterating collections by index - 
```
for i in range(len(l)):
    <...>
```

- Iterating dictionaries -
    - Iterating keys - `for k in my_dict`
    - Iterating values - `for v in my_dict.values()`
    - Iterating both - `for k, v in my_dict.items()`

> [!TIP]
> Use `break` to break out of the loop and `continue` to skip to the next iteration.

- `else` keyword with loops -
    - Applicable on both `for` and `while` loops -
    - Runs the block of code when loop iterates over all the elements without encountering a `break` or an error -
    ```
    for i in my_dict:
        <...>
    else:
        <...>
    ```
