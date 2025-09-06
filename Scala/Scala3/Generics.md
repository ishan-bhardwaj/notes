## Generics

- Provides flexibilty to use same code on different types.
```
class LinkedList[T]
val intList = new LinkedList[Int]
val stringList = new LinkedList[String]
```

## Polymorphic Function Literal

- Syntax - 
```
val identity: [A] => A => A = [A] => (x: A) => x
```

- Applying -
```
identityVal(42)         // 42
identityVal("hello")    // hello
```

- Specifying the type parameters explicitly -
```
identityVal[Int](42)         // 42
identityVal[String]("hello") // "hello"
```

- The type parameter in the literal need not be the same name as that in the declared type -
```
val alternateIdentity: [A] => A => A = [B] => (x: B) => x
```