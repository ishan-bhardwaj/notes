## Braceless Syntax

- If expression -
```
if n > 3 then
    val message = "larger"
    message
else
    val message = "smaller"
    message
```

- for-comprehension -
```
for
    n <- List(1, 2)
    c <- List("black", "white")
yield s"$n-$s"
```

- Pattern matching -
```
meaningOfLife match
    case 1 => "one"
    case 2 => "two"
    case _ => "others"
```

- Methods -
```
def computation(): Int =
    val temp = 10
    temp + 20
```

- Classes / Objects / Traits / Enums -
```
class Animal:               // colon is necessary
    def walk(): Unit = ???
    def sleep(): Unit = ???

end Animal
```

- `end` is optional and signifies the end of expressions. Can be used with everything - if, for-comp, methods, classes etc.

- Anonymous classes -
```
new Animal:
    override def walk(): Unit = ???
```