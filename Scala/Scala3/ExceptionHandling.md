## Exception Handling

- Throwing an exception is an expression of type `Nothing` -
```
val x: Nothing = throw new RuntimeException("Error!")
```

- The exception is of type `Throwable` which has below subtypes -
    - `Error` - eg - Stackoverflow error, Out of memory error etc.
    - `Exception` - eg - `NullPointerException`, `RuntimeException` etc.

- Catching an exception -
```
try {
    throw new RuntimeException("Error!")
} catch {
    case e: NullPointerException => 54
    case e: RuntimeException => 45
} finally {

}
```

- `finally` block is optional and will be executed no matter what. It is usually used to close the resources.

> [!TIP]
> `finally` has no impact on the return type of the try-catch expression.

- Custom exceptions -
```
class MyException(message: String) extends RuntimeException
```

