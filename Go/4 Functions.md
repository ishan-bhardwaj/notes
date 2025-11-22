# Functions

- Function declaration -
```
func add(v1 int, v2 int) int {
    return v1 + v2
}
```

- If input parameters are of same type - `func add(v1, v2 int)`

- Functions can return multiple values -
```
func calculateBounds(v1, v2 int) (int, int) {
    upper = v1 + v2
    lower = v1 - v2
    return upper, lower
}

upper_b, lower_b = calculateBounds(10, 2)
```

- Alternatively, return by variable names -
```
func calculateBounds(v1, v2 int) (upper int, lower int) {
    upper = v1 + v2
    lower = v1 - v2
}

upper_b, lower_b = calculateBounds(10, 2)
```