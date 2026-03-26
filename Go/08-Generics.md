# Generics

### `any` limitation

- Let's say we define generic method `add` - 
```
func add(a, b any) any {
    <...>
}
```

- Now, `add` function can accept two integers and return an `int`, but its return type is also `any` -
```
result = add(1, 2)
result + 1              // error - because result is any
```

- Using generics - here Go will know that we are passing two `int`s to `add` and therefore the resulting type will also be an `int` -
```
func add[T any](a, b T) T {
    <...>
}

result = add(1, 2)
result + 1              // works
```

- For multiple generic types -
```
func add[T any, X any] add(a T, b X)
```

- Accepting list of types -
```
func add[T int | float64 | string] (a, b T) T {
    return a + b
}
```
