# Collection Framework

## `Iterable` Interface

- The “for each” loop works with any object that implements the `Iterable` interface.
```
public interface Iterable<E> {
  Iterator<E> iterator();
}
```

## `Collection` Interface

- The `Collection` interface extends the `Iterable` interface.
- `java.util.Collection<E>` interface -

| Method                                                      | Description                                                                                          |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `Iterator<E> iterator()`                                    | Returns an iterator to traverse the elements in the collection.                                      |
| `int size()`                                                | Returns the number of elements currently stored in the collection.                                   |
| `boolean isEmpty()`                                         | Returns `true` if the collection contains no elements.                                               |
| `boolean contains(Object obj)`                              | Returns `true` if the collection contains an element equal to `obj`.                                 |
| `boolean containsAll(Collection<?> other)`                  | Returns `true` if the collection contains all elements of the specified collection.                  |
| `boolean add(E element)`                                    | Attempts to add an element; returns `true` if the collection changed as a result.                    |
| `boolean addAll(Collection<? extends E> other)`             | Adds all elements from another collection; returns `true` if the collection changed.                 |
| `boolean remove(Object obj)`                                | Removes an element equal to `obj`; returns `true` if removal occurred.                               |
| `boolean removeAll(Collection<?> other)`                    | Removes all elements present in the specified collection; returns `true` if the collection changed.  |
| `boolean removeIf(Predicate<? super E> filter)` *(Java 8+)* | Removes elements matching the predicate; returns `true` if the collection changed.                   |
| `void clear()`                                              | Removes all elements from the collection.                                                            |
| `boolean retainAll(Collection<?> other)`                    | Retains only elements present in the specified collection; returns `true` if the collection changed. |
| `Object[] toArray()`                                        | Returns an array containing all elements of the collection.                                          |
| `<T> T[] toArray(IntFunction<T[]> generator)` *(Java 11+)*  | Returns an array of elements using the provided array constructor (e.g., `String[]::new`).           |


> [!WARNING]
> The Java Collections Framework was designed before generic types were added to Java. For backwards compatibility, the `contains` and `remove` methods have a parameter of type `Object` and not `E`. The `containsAll`, `removeAll`, and `retainAll` methods have a parameter of type `Collection<?>` and not `Collection<? extends E>`.
>
> This means that type errors may not be detected at compile time.
> Example - accidentally a `String` is removed from a collection of `Path` objects -
> ```
> Collection<Path> paths = . . .;
> paths.remove("/tmp");           // Compiles, but can have no effect
> ```

- To make life easier of the implementors, the framework supplies a class `AbstractCollection` that leaves the fundamental methods `size` and `iterator` abstract but implements the routine methods.

> [!TIP]
> A concrete collection class can now extend the `AbstractCollection` class.
> 
> Mutable collections also need an `add` method.  The other methods have been taken care of by the `AbstractCollection` superclass. 

## Iterators

- `java.util.Iterator<E>` -

| Method                                                          | Description                                                                                                                                                     |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `boolean hasNext()`                                             | Returns `true` if there is another element to iterate over.                                                                                                     |
| `E next()`                                                      | Returns the next element in the iteration; throws `NoSuchElementException` if no elements remain.                                                               |
| `void remove()`                                                 | Removes the last element returned by `next()`. Must be called immediately after `next()`. May throw `IllegalStateException` or `UnsupportedOperationException`. |
| `void forEachRemaining(Consumer<? super E> action)` *(Java 8+)* | Performs the given action for each remaining element until all are processed or an exception occurs.                                                            |

- Traverse all the elements -
  ```
  Collection<String> coll = . . .;
  Iterator<String> iter = coll.iterator();
  while (iter.hasNext()) {
    String element = iter.next();
    // do something with element
  }
  ```

  - More concise way - the compiler simply translates the “for each” loop into a loop with an iterator -
  ```
  for (String element : coll) {
    // do something with element
  }
  ```

- Using `Collection#forEach` or `Iterator#forEachRemaining` for traversing with lambda expression -
```
coll.forEach(element -> do something with element);
iter.forEachRemaining(element -> do something with element);
```

> [!NOTE]
> The order in which the elements are visited depends on the collection type, eg - `ArrayList` will visit the elements from index `0` to the last index, but for `HashSet`, the order can be random.

- It is illegal to call `remove` if it wasn’t preceded by a call to `next` - throws `IllegalStateException`, eg -
```
Iterator<String> iter = coll.iterator();
iter.next();          // skip over the first element
iter.remove();        // now remove it
```

## Interfaces in the Collections Framework

![Interfaces in the Collections Framework](assets/collection_interfaces.png)

- __Maps__ -
  - Hold key-value pairs -
  - `V put(K key, V value)` - inserts key-value pairs.
  - `V get(Object key)` - returns value for the key.

- __`List`__ -
  - Ordered collection.
  - Elements are added into a particular position in the container.
  - Elements are accessed in two ways -
    - By an iterator - Linked list implementation - elements are visited _sequentially_.
    - By an integer index - Array implementation - called _random access_ because elements can be visited in any order.
    - Array implementation -
      - Has fast random access.
      - 
  - The `List` interface defines several methods for random access -
    ```
    void add(int index, E element)
    E remove(int index)
    E get(int index)
    E set(int index, E element)
    ```

  - `equals` method returns `true` if both the lists have the same elements in the same order.

> [!WARNING]
> `List<Integer` has two `remove` methods -
>   - `boolean remove(int index)` - removes the element with the given index
>   - `boolean remove(Integer o)` - removes the element equal to o

> [!NOTE]
> `RandomAccess` is a tagging interface used to test whether a particular collection supports efficient random access.

- __`Set`__ -
  - `add` method reject duplicates. 
  - `equals` method returns `true` if both the sets have the same elements, irrespective of the ordering of their elements.
  - `hashCode` method is defined such that two sets with the same elements always yield the same hash code.

- The `SortedSet` and `SortedMap` interfaces expose the comparator object used for sorting, and they define methods to obtain views of subsets of the collections.

- The interfaces `NavigableSet` and `NavigableMap` contain additional methods for finding the next or previous element in sorted sets and maps -
  - The `TreeSet` and `TreeMap` classes implement these interfaces.
  - The navigation operations can be efficiently implemented in tree-based data structures.

- __`SequencedCollection<E>`__ (Java 21+) -
  - Provides uniform access to the first and last elements of a collection and reverse traversal.
  ```
  E getFirst()
  E getLast()
  void addFirst(E e)
  void addLast(E e)
  E removeFirst()
  E removeLast()
  SequencedCollection<E> reversed()
  ```

  - The `SequencedSet` subinterface sharpens the return type of the `reversed` method to `SequencedSet`. 
  - The `SequencedMap` interface has analogous methods for maps.

## Concrete Collections

- All classes in the below table implement the `Collection` interface, with the exception of the classes with names ending in Map. Those classes implement the `Map` interface instead.

| Collection Type     | Description                                                                     |
| ------------------- | ------------------------------------------------------------------------------- |
| __ArrayList__       | An indexed sequence that grows and shrinks dynamically                          |
| __LinkedList__      | An ordered sequence that allows efficient insertion and removal at any location |
| __ArrayDeque__      | A double-ended queue implemented as a circular array                            |
| __HashSet__         | An unordered collection that rejects duplicates                                 |
| __TreeSet__         | A sorted set                                                                    |
| __EnumSet__         | A set of enumerated type values                                                 |
| __LinkedHashSet__   | A set that remembers insertion order                                            |
| __PriorityQueue__   | A collection that allows efficient removal of the smallest element              |
| __HashMap__         | Stores key–value associations                                                   |
| __TreeMap__         | A map with sorted keys                                                          |
| __EnumMap__         | A map whose keys are from an enum type                                          |
| __LinkedHashMap__   | A map that remembers insertion order of entries                                 |
| __WeakHashMap__     | A map whose entries can be garbage-collected when keys are no longer referenced |
| __IdentityHashMap__ | A map that compares keys using `==` instead of `equals()`                       |

- Relationships between these classes -
![Collection Classes](assets/collection_classes.png)

## Linked Lists

- A linked list stores each object in a separate link - each link also stores a reference to the next link in the sequence.
- All linked lists are actually doubly linked lists.
- Removing an element from the middle of a linked list is an inexpensive operation—only the links around the element to be removed need to be updated.
- Important difference between linked lists and generic collections -
  - LinkedList is ordered, so element position matters.
  - `LinkedList.add(x) only` appends to the end.
  - Inserting in the middle requires knowing the current position.
  - Iterators represent positions between elements, so position-based insertion belongs to them.
  - Therefore, the Java Collections Framework supplies a subinterface `ListIterator` that contains an `add` method -
  ```
  interface ListIterator<E> extends Iterator<E> {
    void add(E element);
    ...
  }
  ```

  - Unlike `Collection.add`, this method does not return a `boolean` — it is assumed that the add operation always modifies the list.

- In addition, the `ListIterator` interface has two methods that you can use for traversing a list backwards -
  - `boolean hasPrevious()`
  - `E previous()`

- The `listIterator` method of the `LinkedList` class returns an `iterator` object that implements the `ListIterator` interface - `ListIterator<String> iter = staff.listIterator()`

- The `add` method adds the new element before the iterator position -
  - Example -  the following code skips over the first element in the linked list and adds "Juliet" before the second element -
  ```
  var staff = new LinkedList<String>();
  staff.add("Amy");
  staff.add("Bob");
  staff.add("Carl");
  ListIterator<String> iter = staff.listIterator();
  iter.next();                                          // skip past first element
  iter.add("Juliet");
  ```

- If you call the `add` method multiple times, the elements are simply added in the order in which you supplied them.
- When you use the add operation with an iterator that was freshly returned from the `listIterator` method and that points to the first element of the list, the newly added element becomes the first element. 
- When the iterator has passed the last element of the list (that is, when `hasNext` returns `false`), the added element becomes the new tail of the list.

> [!NOTE]
> `remove()` depends on the last movement i.e. it deletes the element that was last returned by `next()` or `previous()`.
>
> This also means that you cannot call `remove()` twice because after `remove()`, there is no “last returned element”. You must move again (`next()` or `previous()`) before removing again.

- A `set` method replaces the last element, returned by a call to `next` or `previous`, with a new element.

- If an iterator finds that its collection has been modified by another iterator or by a method of the collection itself, it throws a `ConcurrentModificationException`. For example, consider the following code -
```
List<String> list = . . .;
ListIterator<String> iter1 = list.listIterator();
ListIterator<String> iter2 = list.listIterator();
iter1.next();
iter1.remove();
iter2.next();                      // throws ConcurrentModificationException
```

- Concurrent modification detection - 
  - The collection keeps track of the number of mutating operations (such as adding and removing elements). 
  - Each iterator keeps a separate count of the number of mutating operations that it was responsible for.
  - At the beginning of each iterator method, the iterator simply checks whether its own mutation count equals that of the collection. 
  - If not, it throws a `ConcurrentModificationException`.

> [!NOTE]
> The linked list only keeps track of structural modifications to the list, such as adding and removing links. The `set` method does not count as a structural modification. You can attach multiple iterators to a linked list, all of which call `set` to change the contents of existing links.

- The LinkedList class has `get` method that lets you access element at specified index.
  - But it is very inefficient as LinkedList does not support fast random access.
  - Each time you look up another element, the search starts again from the beginning of the list. 
  - The LinkedList object makes no effort to cache the position information.

> [!TIP]
> The `get` method has one slight optimization - If the index is at least $size() / 2$, the search for the element starts at the end of the list.

- The list iterator interface also has a method to tell you the index of the current position -
  - `nextIndex`/`previousIndex ` method returns the integer index of the element that would be returned by the next call to `next`/`previous`.
  - If you have an integer index `n`, then `list.listIterator(n)` returns an iterator that points just before the element with index `n`.

- `java.util.List<E>` -

| Method                                     | Description                                                  |
| ------------------------------------------ | ------------------------------------------------------------ |
| `listIterator()`                           | Returns a list iterator starting before the first element    |
| `listIterator(int index)`                  | Returns a list iterator positioned before element at `index` |
| `add(int i, E e)`                          | Inserts element at position `i`                              |
| `addAll(int i, Collection<? extends E> c)` | Inserts all elements of a collection starting at `i`         |
| `remove(int i)`                            | Removes and returns element at position `i`                  |
| `get(int i)`                               | Returns element at position `i`                              |
| `set(int i, E e)`                          | Replaces element at position `i`, returns old value          |
| `indexOf(Object o)`                        | Index of first matching element, or `-1`                     |
| `lastIndexOf(Object o)`                    | Index of last matching element, or `-1`                      |

- `java.util.ListIterator<E>` -

| Method            | Description                                                |
| ----------------- | ---------------------------------------------------------- |
| `add(E e)`        | Inserts element at iterator’s current position             |
| `set(E e)`        | Replaces last element returned by `next()` or `previous()` |
| `hasPrevious()`   | Checks if backward traversal is possible                   |
| `previous()`      | Returns previous element                                   |
| `nextIndex()`     | Index of element that `next()` would return                |
| `previousIndex()` | Index of element that `previous()` would return            |

- `java.util.LinkedList<E>` -

| Method                                  | Description                                      |
| --------------------------------------- | ------------------------------------------------ |
| `LinkedList()`                          | Creates an empty linked list                     |
| `LinkedList(Collection<? extends E> c)` | Creates list containing elements of a collection |
| `addFirst(E e)`                         | Inserts element at the beginning                 |
| `addLast(E e)`                          | Inserts element at the end                       |
| `getFirst()`                            | Returns first element                            |
| `getLast()`                             | Returns last element                             |
| `removeFirst()`                         | Removes and returns first element                |
| `removeLast()`                          | Removes and returns last element                 |

## ArrayLists

- `ArrayList` class also implements the `List` interface.
- An ArrayList encapsulates a dynamically reallocated array of objects.

## Hash Sets

