# Arrays

- Declaring an array - 
  - Syntax - `datatype[] arrayName;`
  - Example - `int[] a;`
  - This only declares the variable, it does not create the array.

- Initialize an array -
  ```
  int[] a = new int[100];
  // or
  var a = new int[100];
  ```

  - This creates an array of 100 integers, all initialized to 0.
  - Array length does not have to be a constant - `var a = new int[n];`

- Array variables store a reference, not the array itself.
- Once created, the array size cannot be changed.
- To access an array element by its index - `a[i]`
- To modify individual elements - `a[0] = 10;`
- Length of the array - `a.length`
- For resizable collections, use `ArrayList` instead.

- Two valid ways to declare an array -
  - `int[] a` - preferred
  - `int a[]` - valid, but less common

- You can declare and initialize an array in one step and the array size is inferred automatically -
  ```
  int[] smallPrimes = { 2, 3, 5, 7, 11, 13 };
  ```

- Creating an array of length 0 -
  ```
  new elementType[0]
  // or
  new elementType[] {}
  ```

- Use `Arrays.toString(a)` to convert an array to a string and print it -
  ```
  int[] a = {2, 3, 5, 7, 11, 13};
  IO.println(Arrays.toString(a));
  ```

  - Output - `[2, 3, 5, 7, 11, 13]`

## Array Copying

- Assigning one array variable to another copies the reference, not the values i.e. both variables point to the same array in memory -
  ```
  int[] luckyNumbers = smallPrimes;
  luckyNumbers[5] = 12;               // smallPrimes[5] is also 12
  ```

- To create a new array with the same values, use `Arrays.copyOf` -
  ```
  int[] copiedLuckyNumbers = Arrays.copyOf(luckyNumbers, luckyNumbers.length);
  ```

  - The second argument is the length of the new array.

- Can also use `copyOf` to resize an array -
  ```
  luckyNumbers = Arrays.copyOf(luckyNumbers, 2 * luckyNumbers.length);
  ```

- Extra elements are filled with -
  - `0` for numeric arrays
  - `false` for boolean arrays
  - `null` for object references
- If new length < original length then only initial values are copied.

- Memory model -
  - Java arrays are heap-allocated.
  - Think of Java arrays like pointers to heap memory.

- To sort an array - `Arrays.sort(a)` - uses a tuned version of the QuickSort algorithm.

## Multi-dimensional Arrays

- Declaring a 2D Array - `double[][] balances`
- Initializing a 2D Array -
  - Fixed size initialization - `balances = new double[NYEARS][NRATES]`
  - Shorthand initialization (if elements known) - 
  
    ```
    int[][] magicSquare = {
      {16, 3, 2, 13},
      {5, 10, 11, 8},
      {9, 6, 7, 12},
      {4, 15, 14, 1}
    };
    ```

- Accessing Elements - `balances[i][j]`
- Printing a 2D Array - `Arrays.deepToString(a)`

## Ragged Arrays (Arrays of Arrays)

- Java has no multidimensional arrays at all, only one-dimensional arrays. 
- Multidimensional arrays are faked as “arrays of arrays.”
- It is legal to construct multi-dimensional arrays where a dimension is zero, eg -
  ```
  new int[3][0]             // 3 rows - each having length 0
  new int[0][3]             // no rows
  ```

> [!TIP]
> In Java, each row is stored separately on the heap.

## for-each loop

- Syntax - `for (variable : collection) statement`
-  The collection expression must be an `array` or an object of a class that implements the `Iterable` interface, such as `ArrayList`.
- Example -
  ```
  for (int element : a)
    IO.println(element);
  ```
