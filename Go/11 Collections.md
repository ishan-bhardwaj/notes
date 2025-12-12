# Collections

## Arrays

- Defining an array of size `4` -
```
var products [4] string
prices := [4]float64{1.5, 2.5, 3.5, 4.5}

fmt.Println(products)       // prints [   ]
fmt.Println(prices)         // prints [1.5 2.5 3.5 4.5]
```

- Accessing elements - `prices[1]` - returns `2.5`.
- Updating elements by index - `products[2] = "Book"`
- Length - `len(prices)`

## Slices

- Used to get subset of arrays.
```
prices[1:3]         // [2.5 3.5]
prices[:3]          // [1.5 2.5 3.5]
prices[1:]          // [2.5 3.5 4.5]
```

> [NOTE]
> We cannot use negative indices in slices like python does.

> [!TIP]
> If we use larger number in slicing than the array length, Go compiler will throw an error.

- A slice is a conceptal reference on part of the array. Therefore, when we udate any element in the slice, it will also update the underlying array.

- Slices store its length and capacity -
    - Length - number of elements in the slice - `len(prices[1:])`
    - Capacity - length of underlying "first slice" take from the array - `cap(prices[1:])`

### Dynamic Lists with Slices

- Creating a slice - `[]float64{1.5, 2.5}` - if the length is not defined then it's a slice, however, Go also creates an underlying array for this slice.

```
prices := []float64{1.5, 2.5}              

prices[1] = 3.5                             // works
prices[2] = 4.5                             // Error - index out of bounds

updatedPrices := append(prices, 4.5, 5.5)   // works - Go creates a new underlying array
```

### Unpacking List Values

- Concatenating two slices -
```
newPrices := []float64{4.5, 5.5}

append(prices, newPrices...)                // varargs syntax
```

## Maps

- Creating a map of key type `int` and value type `string`-
```
emptyM = map[int]string{}   // empty map

m = map[int]string{
    101: "Hello",
    102: "World"
}

fmt.Println(m)              // prints - map[101:Hello 102:World] 
```

- Accessing elements by key - `m[102]` - returns "World"
- Mutating maps - `m[103] = "Hi there"`
- Deleting key from the map in-place - `delete(m, 103)`

## `make` function

### `make` with slices

- Create a slice with predefined length - `make([]string, 2)` - Go creates an underlying array of length `2` with empty slots with pre-allocated memory.
- If allocate more elements than the defined length, Go will then have to allocate more memory.
- Difference with normal slice definition -
```
// Normal way
s := []string{}         // empty underlying array is created
s[0] = "hello"          // error!

// Using make
s := make([]string, 2)  // underlying array of size 2
s[0] = "hello"          // array is now - [hello  ]                
```
- To define capacity - `make([]string, <length>, <capacity>)`

### `make` with maps

- Create a map with predefined length - `make(map[int]string, <length>)`

## for-loop

- Looping over arrays or slices -
```
for index, value := range <array/slice> {
    <...>
}
```

- If we don't care about the value & index - `for range <array/slice>`
- If we don't case about index - `for range _, val := range <array/slice>`
- For maps, the range returns key and value - `for k, v := range <map>`
