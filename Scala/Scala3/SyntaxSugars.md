## Syntax Sugars

- Methods with one argument -
```
def computation(arg: Int): Int = arg + 1

computation {
    // multiple statements
    42
}
```

- Single abstract method (SAM) pattern - applicable where there is only one abstract method in the trait, but it can also have other implemented fields/methods -
```
trait Action {
    def act(x: Int): Int
}

// Normal instantiation
val action_v1: Action = new Action {
    override def act(x: Int): Int = x + 1
}

// Syntax sugar
val action_v2: Action = (x: Int) => x + 1
```

- SAM usecase -
```
val aThread = new Thread(new Runnable {
    override def run(): Unit = ???
})

val sugarThread = new Thread(() => ???)
```

- Methods ending a colon(:) are right-associative -
```
val list = List(1, 2, 3)

0 :: list       // translated as list.::(0)
```

- Multi-word identifiers -
```
object `Content-Type` {
    val `application/json` = "application/json"
}
```

- Infix types -
```
infix class -->[A, B]

val compositeType_v1: -->[Int, String] = ???
val compositeType_v2: Int -->  String = ???
```

> [!TIP]
> For more readable bytecode & Java interoperability, we can mention a target name for the types. The compiler will then replace the type name with the target name in the bytecode representation -
> ``` 
> import scala.annotation.targetName
>
> @targetName("Arrow")
> infix class -->[A, B]
> ```

- `update` method on mutable containers -
```
val arr = Array(1, 2, 3, 4, 5)
arr.update(2, 45)               // update value at index 2 to 45
arr(2) = 45                     // syntax-sugar for update
```

- Mutable fields -
```
class Mutable {
    private var internalMember: Int = 0

    def member = internalMember             // getter
    def member_=(value: Int): Unit =        // setter
        internalMember = value     
}

val mut = new Mutable
mut.member = 42             // translated as mut.member_=(42)
```

- Varargs - variable arguments -
```
def computation(args: Int*) = ???           // args inside the method is a Seq

val zeroArgs = computation()
val oneArg = computation(1)
val twoArgs = computation(1, 2)

val list = List(1, 2, 3, 4)
val dynamicArgs = computation(list*)
```
