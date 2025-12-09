# Interfaces

- Creating an interface -
```
type Walker interface {
    Walk(int) error
}
```

- Any struct 

> [!TIP]
> If an interface has only one method, then general convention is to name the interface as - method name suffixed with "er", eg - `Walker`.

- Polymorphic method -
```
func warmup(entity Walker) error {
    entity.Walk(20)
}
```
