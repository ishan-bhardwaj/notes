# Interfaces

- Creating an interface -
```
type Walker interface {
    Walk(int) error
}
```

- Any struct that has same methods as `Walker` interface will automatically inherit `Walker` interface.

> [!TIP]
> If an interface has only one method, then general convention is to name the interface as - method name suffixed with "er", eg - `Walker`.

- Polymorphic method -
```
func warmup(entity Walker) error {
    entity.Walk(20)
}
```

## Embedded Interfaces

```
type saver interface {
    Save() error
}

type outputable interface {
    saver           // existing interface
    Display()       // new method
}
```

## Special "Any Value" Type

- Accepts any type of value -
```
func printSomething(value interface{})
```

Or, use `any` which is an alias for `interface{}`
```
func.printSomething(value any)
```

## Switch on type -

```
func printSomething(value interface{}) {
    switch value.(type) {
        case int:       //
        case flat64:    //
        case string:    //
        default:        // optional
    }
}
```

> [!NOTE]
> When no case matches with the value type and there is no default, Go will not throw an exception and silently skip it.

> [!NOTE]
> `value.(type)` only works within `switch` statement.

- Alternative way to get both the value and the type -
```
typedValue, ok := value.(int)
```

- `typedValue` is the `value` converted to `int`, and `ok` is a boolean which is true if `value` is of type `int`.

## Type Aliases

- Defining type alias -
```
type floatMap map[string]float64
```

- Adding functions to type aliases -
```
func (floatMap) output() {
    <...>
}
```