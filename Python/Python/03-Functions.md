# Functions

- Syntax -
```
def <function_name>(<optional_params>):
    <function_body>
    return <optional_return>
```

- If function does not `return` anything explicitly, it returns `None`.

- Default parameter values -
```
def add(x, y=1):
    return x + y 

add(x=5)           # 6
add(5)             # 6
add(x=5, y=10)     # 15
add(5, 10)         # 15
add(5, y=10)       # 15
add(x=5, 10)       # Error - unnamed args cannot be used after named args
```

- If the default parameter value is not a literal, python stores that value and reuse it -
```
default_y = 10

def add(x, y=default_y):
    return x + y

add(2)                  # returns 12

default_y = 15

add(2)                  # still returns 12
```

> [!TIP]
> ```
> print(1, 2, 3, 4)               # (1 2 3 4) - prints elements with spaces in between
> print(1, 2, 3, 4, sep=' - ')    # (1 - 2 - 3 - 4) - prints with separator
> ```

## First class functions

- Functions can be treated as first class citizen - assign to variables, pass function to another function etc.
```
def greet():
    print('Hello')

hello = greet           # function is stored in variable hello

hello()
```

- Lambda functions -
```
avg = lambda seq: sum(seq) / len(seq)
avg([1, 2, 3, 4, 5])                # 3
```

- With multiple parameters -
```
add = lambda x, y: x + y
```
