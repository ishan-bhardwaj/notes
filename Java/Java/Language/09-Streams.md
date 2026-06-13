# Streams

## Why Streams

- Streams express computations at a higher conceptual level — __what__, not __how__
- Collections require explicit iteration; streams declare the operation and leave scheduling to the implementation
- Same pipeline can run in parallel by changing `stream()` to `parallelStream()`
- Three key differences from collections:
    - Streams do not store elements — elements may be generated on demand
    - Stream operations do not mutate the source — they produce new streams
    - Stream operations are __lazy__ — only executed when the terminal operation forces them

### Pipeline Structure

1. Create a stream
2. Apply intermediate operations (transformations, each returns a new stream)
3. Apply a terminal operation (forces execution; stream cannot be reused after)

---

## Creating Streams

```java
Stream.of(array)                         // from array or varargs
Stream.of("a", "b", "c")
Arrays.stream(array, from, to)           // subrange of array
collection.stream()                      // from any Collection
Stream.empty()                           // zero elements
Stream.generate(supplier)               // infinite; calls supplier each time
Stream.iterate(seed, f)                  // infinite; seed, f(seed), f(f(seed)), ...
Stream.iterate(seed, hasNext, f)        // finite; stops when hasNext fails
Stream.ofNullable(obj)                   // empty if null, else singleton
```

- `"text".lines()` — stream of lines in a string
- `Pattern.compile(regex).splitAsStream(input)` — split by regex
- `new Scanner(str).tokens()` — stream of tokens
- `Files.lines(path)` — stream of file lines; use in try-with-resources
- `Stream.Builder<T>` — build a stream incrementally with `builder.add(e)` / `builder.build()`
- Convert `Iterator`/`Iterable` to stream via `StreamSupport.stream(spliterator, parallel)`

> [!NOTE]
> Do NOT modify the backing collection while a stream operation is in progress (__noninterference__ rule). Modification before the terminal operation is technically allowed but strongly discouraged.

---

## Intermediate Operations

### `filter`, `map`, `flatMap`, `mapMulti`

```java
stream.filter(predicate)          // keep elements matching predicate
stream.map(function)              // transform each element
stream.flatMap(f)                 // f returns Stream<R>; results concatenated
stream.mapMulti((e, consumer) -> { consumer.accept(...); }) // push multiple results per element
```

- `flatMap` flattens a stream of streams into a single stream
- `mapMulti` is more efficient when producing results via a loop — avoids creating intermediate streams

### Substreams and Combining

```java
stream.limit(n)                   // first n elements
stream.skip(n)                    // drop first n elements
stream.takeWhile(predicate)       // take while predicate holds, then stop
stream.dropWhile(predicate)       // drop while predicate holds, then yield rest
Stream.concat(a, b)               // concatenate two streams (a must be finite)
```

### Other Transformations

```java
stream.distinct()                 // suppress duplicates (order preserved)
stream.sorted()                   // natural order (elements must be Comparable)
stream.sorted(comparator)         // custom order
stream.peek(consumer)             // pass each element to consumer; useful for debugging
```

---

## Terminal Operations

### Simple Reductions

```java
stream.count()                    // number of elements
stream.max(comparator)            // Optional<T>
stream.min(comparator)            // Optional<T>
stream.findFirst()                // Optional<T>; first element
stream.findAny()                  // Optional<T>; any element (useful in parallel)
stream.anyMatch(predicate)        // true if any element matches
stream.allMatch(predicate)        // true if all elements match
stream.noneMatch(predicate)       // true if no elements match
```

### Collecting Results

```java
stream.toList()                   // unmodifiable List (Java 16+)
stream.toArray()                  // Object[]
stream.toArray(String[]::new)     // typed array
stream.forEach(consumer)          // process in arbitrary order
stream.forEachOrdered(consumer)   // process in stream order (matters for parallel)
stream.collect(collector)         // general collection into target structure
stream.iterator()                 // get Iterator<T>
```

---

## `Optional<T>`

### Getting a Value Safely

```java
opt.orElse(defaultValue)                        // value or default
opt.orElseGet(() -> computeDefault())           // lazy default
opt.orElseThrow(ExceptionType::new)             // throw if absent
```

### Consuming a Value

```java
opt.ifPresent(consumer)                         // run only if present
opt.ifPresentOrElse(consumer, emptyAction)      // branch on presence
```

### Pipelining

```java
opt.map(f)                     // transform value; empty if absent
opt.filter(predicate)          // empty if value doesn't match
opt.flatMap(f)                 // f returns Optional; flattens nesting
opt.or(() -> alternativeOpt)   // substitute another Optional if empty
opt.stream()                   // Stream of 0 or 1 elements
```

### Creating Optionals

```java
Optional.of(value)             // throws NPE if null
Optional.ofNullable(value)     // empty if null
Optional.empty()               // empty
```

### Anti-Patterns

- Do NOT use `get()` or `isPresent()`/`isEmpty()` — equivalent to null checks, defeats the purpose
- Use `orElseThrow()` (noisier synonym for `get()`) only when you are certain the value is present
- Do NOT: fields of type `Optional` (not serializable, extra object cost), `Optional` method params, Optionals in sets/maps

### `Optional` + `flatMap` for Chaining

```java
Optional.of(arg)
    .flatMap(this::inverse)
    .flatMap(this::squareRoot)
```

- Each step returns `Optional`; if any step is empty, the whole chain is empty

### Converting `Optional` to Stream

```java
ids.map(Users::lookup)
   .flatMap(Optional::stream)   // drops empty Optionals, unwraps present ones
```

---

## Collectors (`java.util.stream.Collectors`)

### String Joining

```java
Collectors.joining()
Collectors.joining(", ")
Collectors.joining(", ", "[", "]")  // prefix, suffix
```

### Summary Statistics

```java
Collectors.summarizingInt(String::length)   // → IntSummaryStatistics
// .getCount(), .getSum(), .getAverage(), .getMax(), .getMin()
```

### Collecting into Collections

```java
Collectors.toList()                         // mutable (type unspecified)
Collectors.toUnmodifiableList()
Collectors.toSet()
Collectors.toUnmodifiableSet()
Collectors.toCollection(TreeSet::new)       // specific collection type
```

### Collecting into Maps

```java
Collectors.toMap(Person::id, Person::name)
Collectors.toMap(Person::id, Function.identity())
Collectors.toMap(keyFn, valueFn, mergeFunction)           // resolve key conflicts
Collectors.toMap(keyFn, valueFn, mergeFunction, TreeMap::new) // specific map type
Collectors.toConcurrentMap(...)             // for parallel streams
```

- Duplicate keys throw `IllegalStateException` unless a merge function is provided

### Grouping and Partitioning

```java
Collectors.groupingBy(classifier)           // Map<K, List<T>>
Collectors.groupingBy(classifier, downstream)
Collectors.partitioningBy(predicate)        // Map<Boolean, List<T>>
Collectors.groupingByConcurrent(classifier) // for parallel streams
```

### Downstream Collectors

```java
Collectors.counting()
Collectors.summingInt(fn) / summingLong / summingDouble
Collectors.averagingInt(fn) / averagingLong / averagingDouble
Collectors.maxBy(comparator)   // Optional<T>
Collectors.minBy(comparator)   // Optional<T>
Collectors.mapping(fn, downstream)
Collectors.flatMapping(fn, downstream)
Collectors.filtering(predicate, downstream)
Collectors.collectingAndThen(downstream, finisher)
Collectors.teeing(downstream1, downstream2, merger)  // two parallel collections, combine results
```

Eg - nested grouping:

```java
Locale.availableLocales().collect(
    groupingBy(Locale::getCountry,
        mapping(Locale::getDisplayLanguage, toSet())))
```

### Implementing Custom Collectors

- Implement four functions: `supplier()`, `accumulator()`, `combiner()`, `finisher()`
- Declare characteristics: `IDENTITY_FINISH`, `UNORDERED`, `CONCURRENT`
- Use `Collector.of(supplier, accumulator, combiner, finisher, characteristics...)` factory

```java
Collector.of(ArrayList<T>::new,
    (rc, e) -> { if (condition) rc.add(e); },
    (rc1, rc2) -> { rc1.addAll(rc2); return rc1; },
    Function.identity(),
    Collector.Characteristics.IDENTITY_FINISH)
```

- For downstream-compatible collectors, delegate to `downstream.supplier()`, `downstream.combiner()`, `downstream.finisher()` and adapt only the accumulator

---

## Reduction Operations

```java
stream.reduce((x, y) -> x + y)              // Optional<T>; no identity
stream.reduce(0, (x, y) -> x + y)           // T; identity provided; empty → identity
stream.reduce(identity, accumulator, combiner) // for mixed types (e.g. sum of string lengths)
stream.collect(BitSet::new, BitSet::set, BitSet::or) // mutable reduction into container
```

- For parallel streams: operation must be __associative__ and ideally have an __identity element__
- Prefer `mapToInt(...).sum()` over `reduce` for numeric aggregations — simpler and avoids boxing

---

## Gatherers (Java 24+)

Gatherers are to intermediate operations what Collectors are to terminal operations.

```java
stream.gather(gatherer)  // transforms stream; can be chained
```

### Predefined Gatherers

```java
Gatherers.windowFixed(n)        // non-overlapping windows of size n → Stream<List<T>>
Gatherers.windowSliding(n)      // sliding windows → Stream<List<T>>
Gatherers.mapConcurrent(max, fn) // concurrent map via virtual threads; max concurrent tasks
Gatherers.fold(() -> identity, (acc, e) -> newAcc)   // sequential fold → stream of 1 element
Gatherers.scan(() -> identity, (acc, e) -> newAcc)   // all intermediate fold results
```

### Implementing Custom Gatherers

```java
Gatherer.ofSequential(
    initializer,          // Supplier<A> — state factory
    integrator)           // Gatherer.Integrator<A, T, R>

Gatherer.of(
    initializer,
    integrator,
    combiner,             // BinaryOperator<A> — for parallel
    finisher)             // BiConsumer<A, Downstream<R>> — after last element
```

- Integrator: `boolean integrate(A state, T element, Downstream<R> downstream)` — return `false` to short-circuit; call `downstream.push(result)` for each output
- Return `downstream.push(e)` result when not short-circuiting yourself — propagates downstream rejection
- Use `Gatherer.Integrator.ofGreedy(...)` when your gatherer never short-circuits (allows optimisation)
- Use `Gatherer.ofSequential(...)` when elements must be processed in order (no combiner)
- Sequential vs parallel determined by whether combiner is `Gatherer.defaultCombiner()`

---

## Primitive Type Streams

- `IntStream`, `LongStream`, `DoubleStream` — avoid boxing overhead
- Use `IntStream` for `short`, `char`, `byte`, `boolean`; `DoubleStream` for `float`

### Creating

```java
IntStream.of(1, 2, 3)
IntStream.range(0, 100)           // [0, 100)
IntStream.rangeClosed(0, 100)     // [0, 100]
Arrays.stream(intArray, from, to)
words.mapToInt(String::length)    // Stream<String> → IntStream
stream.mapToObj(i -> ...)         // IntStream → Stream<R>
intStream.boxed()                 // IntStream → Stream<Integer>
"text".codePoints()               // IntStream of Unicode code points
RandomGenerator.getDefault().ints() // infinite random IntStream
```

- Conversion from `Stream<Integer>` to `IntStream`: `numbers.mapToInt(i -> i)` (NOT `Function.identity()`)

### Unique Methods (not on `Stream<T>`)

```java
intStream.sum()
intStream.average()               // OptionalDouble
intStream.max()                   // OptionalInt
intStream.min()                   // OptionalInt
intStream.summaryStatistics()     // IntSummaryStatistics (all four at once)
intStream.toArray()               // int[]
```

---

## Parallel Streams

```java
collection.parallelStream()
stream.parallel()
```

- All intermediate operations execute in parallel when terminal operation runs
- Operations must be __stateless__ and executable in __arbitrary order__

### Rules for Safe Parallel Streams

- Never mutate shared mutable state in stream operations (race condition)
- Use collectors like `groupingBy` + `counting()` instead of external arrays
- For concurrent map accumulation: `groupingByConcurrent` — uses shared `ConcurrentHashMap`; values not in stream order
- `stream.unordered()` — drops ordering guarantee; enables faster `distinct()` and `limit()` in parallel
- Avoid blocking operations (file I/O, network) in parallel stream lambdas — starves the fork-join pool

### Performance Considerations

- Parallelism has overhead — only pay off for large data sets and computationally intensive tasks
- Data source must be splittable (arrays and lists split well; linked lists do not)
- By default uses `ForkJoinPool.commonPool()`; can substitute a custom pool:

```java
ForkJoinPool customPool = ...;
result = customPool.submit(() ->
    stream.parallel().map(...).collect(...)).get();
```

- For random numbers in parallel: use `RandomGenerator.SplittableGenerator.of(...)`
- `mapConcurrent` gatherer is better than parallel streams for blocking I/O — uses virtual threads, not the fork-join pool