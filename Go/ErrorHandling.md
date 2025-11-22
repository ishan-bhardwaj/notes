# Error Handling

- `error` type -
```
import "errors"

func divide(a, b int) (float64, error) {
    if b == 0 {
        return -1, errors.New("Cannot divide by 0")
    } else {
        res := a / b
        return res, nil
    }
}

var x, err := divide(10, 0)

if err != nil {
    fmt.Println("ERROR")
    fmt.Println(err)
}
```

- Panics - exits the application -
```
panic("Fatal error, exiting.")
```

