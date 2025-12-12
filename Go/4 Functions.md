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

## Functions as values

### Accepting functions as values

```
def double(num int) {
    return num * 2
}

def triple(num int) {
    return num * 3
}

// HOF
def transformNumbers(nums *[]int, transform func(int) int) []int {
    transformed := []int{}
    for _, val := range nums {
        transformed = append(transformed, transform(val))
    }
    return transformed
}


// calling
numvers := []int{1, 2, 3, 4, 5}
transformNumbers(&numbers, double)
transformNumbers(&numbers, triple)
```

- Using custom types for function definitions - `type transformFn func(int) int`

### Returning functions as values

```
func getTransformer() func(int) int {
    return double
}
```

## Anonymous Functions

```
transformNumbers(&numbers, func(num int) int {
    return num * 2
})
```

## Closures

- Factor function pattern.
- In following example, `factor` is accessible even though it is a parameter of `createTransformer`. 
- Therefore, the anonymous function created in the body is "closed" over the variables or parameters (`factor` in this example) of the scope in which it was created.
```
func createTransformer(factor int) func(int) int {
    return func(num int) int {
        return num * factor
    }
}
```

## Variadic Functions

- `nums` will be inferred as a slice -
```
func add(nums ...int) int {
    sum := 0
    for _, val := range nums {
        sum += val
    }

    return sum
}

add(1, 2, 3, 4, 5)

// passing slice to add function
numbers := []int{1, 2, 3, 4, 5}
add(numbers...)
```