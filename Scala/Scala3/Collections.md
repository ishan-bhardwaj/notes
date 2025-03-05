## Linear Collections

- `Seq`, `List`, `Array`, `Vector`, `Set`, `Range` are commonly used linear collections.

### `Seq`

- `Seq` is a trait for a collection that has a well-defined ordering and indexing support.
```
val values = Seq(1, 2, 3, 4, 5)

val secondElem_v1 = values.apply(1)     // 2
val secondElem_v2 = values(1)           // 2

// utility methods
val reversed = values.reverse           // List(5, 4, 3, 2, 1)
val concat = values ++ Seq(6, 7)        // List(1, 2, 3, 4, 5, 6, 7)
val sorted = values.sorted              // sorted using numerical ordering provided implicitly
```

> [!TIP]
> `Seq.apply` method returns a particular implementation of `Seq` trait - which in our case is returning a `List`.

### `List`

- `List` is an implementation of `Seq` trait -
```
val list = List(1, 2, 3, 4, 5)

val firstElem = list.head               // 1
val lastElem = list.tail                // 5

val appeneded = list +: 6               // List(1, 2, 3, 4, 5, 6)
val preppended_v1 = 0 +: list           // List(0, 1, 2, 3, 4, 5)
val prepended_v2 = 0 :: list            // List(0, 1, 2, 3, 4, 5)

// utility methods
val scalax5 = List.fill(2)("Scala")     // List("Scala", "Scala")
List("Hello", "World").mkString(" ")    // Hello World
```

### `Range`

- `Range` is another implementation of `Seq` trait -
```
val aRange: Seq[Int] = 1 to 10
```

- Two subtypes -
    - `Range.Inclusive` - `1 to 10` - where `10` is inclusive
    - `Range.Exclusive` - `1 until 10` - where `10` is not inclusive

### `Array`

- `Array` is NOT an implementation of `Seq` trait, but has access to most `Seq` APIs.
- To convert an `Array` into `Seq` - `Array#toIndexedSeq`

> [!TIP]
> Arrays are mutable i.e. the elements of an array can be mutated in-place, whereas all the other collections are immutable.

- Updating an array - `arr.update(index, newElement)` - but no new array is allocated, the element is updated in-place.

### `Vector`

- Vectors are fast implementation of `Seq` for a large amount of data.
- Creating a vector - `Vector(1, 2, 3, 4, 5)`

### `Set`

- A set does not store duplicate elements and does not mantain the order of its elements.
- `TreeSet` is a sub-type of a `Set` which gurantees element ordering.
- `Set` is implemented using `equals` & `hashCode` methods.
- Main usage is to check whether the set contains a particular element -
```
val set = Seq(1, 2, 3, 4, 5)

set.contains(3)     // true
set.apply(3)        // true - same as contains
set(3)              // true
```

- Adding an element - `set + 4` - returns a new set, even if the set already contained that element.
- Removing an element - `set - 4`
- Union two sets - `setA ++ setB` or `setA.union(setB)` or `setA | setB`
- Difference - `setA -- setB` or `setA.diff(setB)`
- Intersection - `setA.intersect(setB)` or `setA & setB`

## Tuples

- Tuples are finite order list of element that can contain hetrogenous elements.
```
val tuple: Tuple2[Int, String] = (1, "Hello")

tuple._1            // 1
tuple._2            // "Hello"
```

- `Tuple2[Int, String]` can simply be written as `(Int, String)`.
- Updating elements -
```
val anotherTuple = tuple.copy(_1 = 2)
anotherTuple._1     // 3
```

- Another way of creating `Tuple2` -
```
val tuple: (Int, String) = 1 -> "Hello"
```

## Maps

- Key-value pairs.
- Does not gurantee element ordering.
```
val map: Map[String, Int] = Map(
    "Hello" -> 1,
    "World" -> 2
)
```

- `List[Tuple2[K, V]]#toMap` converts the list to `Map[String, Int]`.
- Core APIs -
```
map.contains("Hello")           // check if map contains the key "hello"
map("hello")                    // 1 - value present at the key "hello"; throws `NoSuchElementException` if key is not present
map.get("hello")                // returns the value in Option
```

- Return some default value for a non-existing key - `Map#withDefaultValue`
- Add new key-value pair - `map + ("greet" -> 3)`
- Removing key - `map - "greet"`

- Filtering keys -
```
map.view.filterKeys(_ == "hello").toMap
```

- Mapping values -
```
map.view.mapValues(_ * 2).toMap
```

- `groupBy` on collections returns a `Map` -
```
val l = List(1, 2, 3, 4, 5)
l.groupBy(_ % 2 == 0)               // Map[Boolean, List[Int]] = Map(true -> List(2, 4), false -> List(1, 3, 5))
```

