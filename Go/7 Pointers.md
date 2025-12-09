# Pointers

- Pointers are variables that store value **addresses** instead of values.

> [!NOTE]
> Go creates local copy of function arguments (call-by-value) and the return values.

- Get pointer of a variable -
```
var age int = 10
var agePtr *int = &age
```

- Deferencing a pointer i.e. getting value from the pointer - `*agePtr` - will return `10`.
- Null value of a pointer is `nil` - represents the absence of an address value - i.e., a pointer pointing at no address / no value in memory.
