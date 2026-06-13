# Collections

## The Java Collections Framework

### Design Principles

- Separates interfaces from implementations — program to the interface, construct via the concrete class
- Eg - `Queue<Customer> q = new ArrayDeque<>(100)` — swap implementation by changing only the constructor
- Using the interface type prevents accidental use of implementation-specific methods
- `ArrayDeque` generally outperforms `LinkedList` (less GC pressure)

### `Collection` Interface

- Two fundamental methods: `boolean add(E element)`, `Iterator<E> iterator()`
- `add` returns `true` if the collection changed (Eg - sets return `false` on duplicate)
- Extends `Iterable<E>` — enables for-each loops on all collections
- Key utility methods: `size()`, `isEmpty()`, `contains(Object)`, `containsAll`, `remove(Object)`, `removeAll`, `retainAll`, `addAll`, `clear()`, `toArray()`
- Default method: `removeIf(Predicate<? super E> filter)` — removes elements matching predicate
- `AbstractCollection` provides default implementations of most methods in terms of `size()` and `iterator()`

> [!NOTE]
> `contains`, `remove`, `containsAll`, `removeAll`, `retainAll` use `Object` / `Collection<?>` parameters (not `E`) for backwards compatibility — type errors may not be caught at compile time.

### Iterators

- `Iterator<E>` methods: `boolean hasNext()`, `E next()`, `default void remove()`, `default void forEachRemaining(Consumer)`
- `next()` both advances and returns — think of iterator as sitting between elements
- Calling `next()` past the end throws `NoSuchElementException`
- `remove()` removes the last element returned by `next()` — must be called after `next()`, never twice in a row without an intervening `next()`
- `forEach` / `forEachRemaining` accept lambda expressions:

```java
coll.forEach(element -> ...);
iter.forEachRemaining(element -> ...);
```

---

## Interface Hierarchy

- `Collection` → `List`, `Set`, `Queue`, `Deque`
- `Map` — separate hierarchy (key/value pairs)
- `SortedSet` / `SortedMap` — expose comparator, provide subrange views
- `NavigableSet` / `NavigableMap` — add methods for finding adjacent elements; implemented by `TreeSet`/`TreeMap`
- `SequencedCollection<E>` (Java 21) — uniform `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()`, `reversed()` for lists, sets, deques
- `RandomAccess` — tagging interface; signals efficient random access; used to choose between random and sequential traversal

### `List` vs. `Set`

- `List` — ordered, positional, allows duplicates; `equals` requires same elements in same order
- `Set` — unordered, no duplicates; `equals` requires same elements in any order

---

## Concrete Collections

### `LinkedList<E>`

- Doubly linked — O(1) insert/remove at any position using iterator
- `ListIterator<E>` (extends `Iterator`) — adds:
    - `void add(E element)` — inserts before current position; always modifies list (no boolean return)
    - `void set(E element)` — replaces last visited element
    - `boolean hasPrevious()`, `E previous()`
    - `int nextIndex()`, `int previousIndex()`
- `add` position: immediately before the next element the iterator would return
- `remove` direction depends on last traversal direction (`next` → removes left; `previous` → removes right)
- __Concurrent modification detection__ — collection tracks mutation count; iterator compares on each call; throws `ConcurrentModificationException` if mismatched
    - `set` on iterator does NOT count as structural modification
    - Rule: multiple read-only iterators OR one read-write iterator
- `get(n)` exists but is O(n) — never use in loops; use `ArrayList` for random access

### `ArrayList<E>`

- Dynamic array — O(1) random access; O(n) insert/remove in middle
- Prefer over `Vector` (all `Vector` methods are synchronized — wasteful for single-thread use)

### `HashSet<E>`

- Hash table backed — buckets of linked lists; converts to balanced binary tree when bucket exceeds threshold
- O(1) average `add`, `contains`, `remove`
- Iteration order is essentially random and scrambled per JVM startup
- Load factor default 0.75 — rehashes (doubles bucket count) when exceeded
- Bucket count is always a power of 2 (default 16)
- `HashSet.newHashSet(expectedCount)` — convenience factory that pre-sizes
- Keys should implement `Comparable` if possible — avoids poor performance from hash collisions
- Do not mutate elements after insertion — hash code change breaks the set

### `TreeSet<E>`

- Red-black tree — O(log n) add/contains/remove; always in sorted order
- Elements must implement `Comparable` or a `Comparator` must be supplied
- Implements `NavigableSet` — `higher(v)`, `lower(v)`, `ceiling(v)`, `floor(v)`, `pollFirst()`, `pollLast()`
- Comparator must be compatible with `equals` — otherwise `equals` gives inconsistent results between `TreeSet` and other sets
- Use `TreeSet` only when sorted order is needed; `HashSet` is faster

### `ArrayDeque<E>`

- Double-ended queue backed by circular array — O(1) add/remove at both ends
- Implements `Deque<E>` and `SequencedCollection<E>`
- Preferred over `Stack` (which extends `Vector`)

### `PriorityQueue<E>`

- Heap-backed — O(log n) `add`, O(log n) `remove` (always removes smallest)
- Iteration does NOT visit in sorted order — only `remove()` guarantees smallest first
- Elements must implement `Comparable` or provide `Comparator`

---

## Maps

### Basic Operations

- `HashMap<K,V>` — hash-based; `TreeMap<K,V>` — sorted by key
- `put(K key, V value)` — returns previous value or `null`
- `get(Object key)` — returns `null` if absent (parameter is `Object` for legacy reasons — type errors may not be caught)
- `getOrDefault(key, defaultValue)` — avoids null checks
- `containsKey`, `containsValue`, `remove(key)`, `size()`
- `forEach((k, v) -> ...)` — iterates all entries

### Updating Entries

- Naive: `counts.put(word, counts.get(word) + 1)` — NPE if first occurrence
- Safe: `counts.put(word, counts.getOrDefault(word, 0) + 1)`
- `putIfAbsent(key, value)` — only puts if key absent or null-mapped
- `merge(key, value, BiFunction)` — associate `value` if absent; combine with existing if present:
    - `counts.merge(word, 1, Integer::sum)`
- `computeIfAbsent(key, Function)` — compute and put only if absent; returns new value (chainable):
    - `index.computeIfAbsent(term, _ -> new TreeSet<>()).add(pageNumber)`
- `compute(key, BiFunction)`, `computeIfPresent(key, BiFunction)`, `replaceAll(BiFunction)`

> [!NOTE]
> `getOrDefault` treats `null` as a valid value; `putIfAbsent` and `computeIfAbsent` treat `null` as absent — inconsistent behaviour to be aware of.

### Map Views

- `keySet()` — `Set<K>` view; removing a key removes the entry
- `values()` — `Collection<V>` view; not a set
- `entrySet()` — `Set<Map.Entry<K,V>>` view; supports `getKey()`, `getValue()`, `setValue()`
- Views are live — changes to map reflect in view and vice versa
- Cannot `add` to `keySet()` or `entrySet()` views
- Disassociate entries with `Map.Entry.copyOf(entry)` or create standalone with `Map.entry(k, v)`

### Specialised Maps

- `WeakHashMap<K,V>` — keys held via `WeakReference`; GC can reclaim entries when key is only weakly reachable
- `LinkedHashMap<K,V>` — insertion order (default) or access order (pass `true` to 3-arg constructor)
    - Override `removeEldestEntry(Map.Entry)` to implement LRU cache
- `LinkedHashSet<E>` — remembers insertion order
- `EnumSet<E extends Enum<E>>` — bit vector backed; no public constructors; use `allOf`, `noneOf`, `range`, `of`
- `EnumMap<K extends Enum<K>, V>` — array-backed; must pass key type to constructor
- `IdentityHashMap<K,V>` — uses `System.identityHashCode` and `==` instead of `hashCode`/`equals`; for object traversal algorithms

---

## Copies and Views

### Small Unmodifiable Collections (Java 9+)

```java
List<String> names = List.of("Peter", "Paul", "Mary");
Set<Integer> nums  = Set.of(2, 3, 5);
Map<String,Integer> scores = Map.of("Peter", 2, "Paul", 3);
Map<String,Integer> scores = Map.ofEntries(entry("Peter", 2), entry("Paul", 3));
```

- Elements, keys, values cannot be `null`; duplicate keys/set elements throw at construction
- Iteration order of `Set` and `Map` is deliberately scrambled per JVM startup — do not rely on it
- Resulting objects are unmodifiable — `UnsupportedOperationException` on mutation

### Unmodifiable Copies (Java 10+)

```java
Set<String> nameSet = Set.copyOf(names);
List<String> nameList = List.copyOf(names);
Map<String,Integer> mapCopy = Map.copyOf(map);
```

- Makes an actual copy — original modification does not affect copy
- If original is already unmodifiable and correct type, `copyOf` returns the same object
- Rejects `null` elements

### Unmodifiable Views (`Collections.unmodifiable*`)

- `Collections.unmodifiableList(list)`, `unmodifiableSet`, `unmodifiableMap`, etc.
- View reflects subsequent changes to the original collection
- Mutator methods throw `UnsupportedOperationException`
- `unmodifiableCollection` inherits `Object.equals` (identity) — use `unmodifiableSet`/`unmodifiableList` for proper equality
- `nCopies(n, obj)` — immutable list of n identical elements; O(1) storage

### Subrange Views

- `list.subList(from, to)` — inclusive/exclusive; mutations propagate to backing list
- `SortedSet`: `subSet(from, to)`, `headSet(to)`, `tailSet(from)`
- `NavigableSet`: same with boolean `inclusive` flags
- `SortedMap`/`NavigableMap`: `subMap`, `headMap`, `tailMap` analogues

### Set from Map

```java
Set<String> cache = Collections.newSetFromMap(new LinkedHashMap<>() {
    protected boolean removeEldestEntry(Map.Entry<String,Boolean> e) {
        return size() > 100;
    }
});
```

### Reversed Views

```java
for (String e : collection.reversed()) { ... }
```

- `SequencedCollection.reversed()`, `SequencedSet.reversed()`, `SequencedMap.reversed()`

### Checked Views

```java
List<String> safeList = Collections.checkedList(strings, String.class);
```

- Throws `ClassCastException` immediately on wrong-type insertion — helps debug generic type smuggling

### Synchronized Views

```java
Map<String,Employee> map = Collections.synchronizedMap(new HashMap<>());
```

- All methods are synchronized — safe for multi-threaded access
- Better alternatives exist in `java.util.concurrent` (Chapter 10)

---

## Algorithms (`Collections` and `Arrays`)

### Sorting

```java
Collections.sort(staff);
Collections.sort(staff, Comparator.comparingDouble(Employee::getSalary));
list.sort(comparator);
Arrays.sort(array);
```

- Uses modified merge sort — O(n log n) guaranteed, __stable__ (equal elements preserve relative order)
- `Collections.sort` dumps to array internally, sorts, copies back — works for `LinkedList` too
- `Comparator.reverseOrder()` — reverses natural order; `.reversed()` — reverses any comparator

### Shuffling

```java
Collections.shuffle(cards, RandomGenerator.getDefault());
```

- For non-`RandomAccess` lists: copies to array, shuffles, copies back

### Binary Search

```java
int i = Collections.binarySearch(sortedList, target);
int i = Collections.binarySearch(sortedList, target, comparator);
```

- List must already be sorted
- Returns index if found (≥0); returns `-(insertionPoint) - 1` if not found
- Insertion point: `insertionPoint = -i - 1`
- Degrades to linear search for non-`RandomAccess` lists

### Simple Algorithms

- `Collections.min(coll)` / `max(coll)` — with optional comparator
- `Collections.copy(dest, src)` — dest must be at least as long as src
- `Collections.fill(list, value)` — fills all positions
- `Collections.replaceAll(list, oldValue, newValue)`
- `Collections.reverse(list)` — O(n)
- `Collections.rotate(list, d)` — element at index i moves to `(i + d) % size`
- `Collections.swap(list, i, j)`
- `Collections.frequency(coll, obj)` — count of elements equal to obj
- `Collections.disjoint(c1, c2)` — true if no common elements
- `Collections.addAll(coll, values...)` — adds varargs to collection

### Bulk Operations

- `coll1.removeAll(coll2)` — removes elements present in coll2
- `coll1.retainAll(coll2)` — keeps only elements present in coll2 (intersection)
- Works on views: `staffMap.keySet().removeAll(terminatedIDs)`
- Works on subranges: `staff.subList(0, 10).clear()`

### Collection ↔ Array

```java
List<String> list = List.of(array);       // array to list
String[] arr = list.toArray(String[]::new); // list to typed array
```

- `toArray()` with no args returns `Object[]` — cannot cast
- Pre-Java 11: `list.toArray(new String[0])`

### Writing Generic Algorithms

- Accept `Collection<Item>` (or `Iterable<Item>`) rather than `ArrayList<Item>` — accept the most general interface that does the job
- Return `List<Item>` rather than `ArrayList<Item>` — leave room for future improvements

---

## Legacy Collections

- `Hashtable` — synchronized `HashMap`; use `HashMap` or `ConcurrentHashMap` instead
- `Vector` — synchronized `ArrayList`; no reason to use
- `Stack` — extends `Vector`; use `ArrayDeque` instead
- `Enumeration` — predecessor to `Iterator`; convert with `Collections.list(enum)` or `.asIterator()`
- `Properties` — string-key/string-value map with file I/O and default chain
    - `getProperty(key)`, `getProperty(key, default)`, `setProperty(key, value)`
    - `load(Reader)`, `store(Writer, comment)` — use readers/writers for UTF-8 (ISO 8859-1 with streams)
    - Chain defaults: `new Properties(defaultSettings)`
- `BitSet` — packed bit array; `get(i)`, `set(i)`, `clear(i)`, `and`, `or`, `xor`, `andNot`, `cardinality()`, `stream()`
    - Use for sets of non-negative integers with an upper bound — far more efficient than `Set<Integer>`