## Values and Types

- `val` is immutable value and `var` is mutable variable.
- For eg -
```
val x: Int = 42
x = 45              // compilation error - "Reassignment to val x"
```

```
var y: Int = 42
y = 45              // works!
```

- Common Types -
    - Short - `val aShort: Short = 10` - 2 bytes representation
    - Int - `val anInt: Int = 42` - 4 bytes representation
    - Long - `val aLong: Long = 23783573680L` - 8 bytes representation. Note that trailng `L` with the number is only used to distinguish the `Int` and `Long` for readability, hence it is optional. 
    - Float - `val aFloat: Float = 2.5f` - 4 bytes representation
    - Double - `val aDouble: Double = 3.14` - 8 bytes representation
    - Boolean - `val aBool: Boolean = false`
    - Char - `val aChar: Char = 'a'`
    - String - `val aString: String = "hello"`

> [!Note]
> Scala `String` type is just an alias for Java `String` i.e.
> ```
> type String = java.lang.String
> ```


