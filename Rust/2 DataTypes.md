# Scalar Data Types

- Rust is a statically typed langugage.
- Provides Type inference - compiler can infer types of variables based on their initial assignments.

## Integers

- Integer types - 
    - Signed - `i8`, `i16`, `i32`, `i64`, `isize`
    - Unsigned - `u8`, `u16`, `u32`, `u64`, `usize`
- Default type infered by Rust for any integer - `i32`
- Eg -
```
let a: i8 = 100;
let b: u16 = 200;
let c: i32 = 300;
```

- Defining datatype with the value -
```
let a = 20i8;        # a will be inferred as i8
```

> [!TIP]
> Use underscores (`_`) to improve long integers readability, eg - `let a = 32_500`

- `isize` & `usize` are aliases for integer types depending on the system architecture, for eg - `isize` will be `i32`/`u32` for a system with 32-bit architecture and `i64`/`u64` for a 64-bit architecture.

## Strings

- String literal - 
```
let greet: &str = "Hello World";
```

- Raw string literals -
```
let greet: &str = r"Hello\n, how are you?"      // `\n` will not be escaped
```

- Interpolation -
```
let count = 10;
println!("Count is: {count}");

// or
println!("Count is: ", count);
```

## Floating points

- Two types - `f32` & `f64` (default)

```
let pi: f64 = 3.14159;
```

- A Format specifier customizes the printed representation of the interpolated value -
```
println!("Value of PI is: {pi:.2}");        // prints only 2 digits after the decimal

// or
println!("Value of PI is: {:.2}", pi)       // same as above
```

## Casting Types (`as`)

```
let x: i32 = 100;
let y: i8 = x as i8;
let z: i32 = x as i32;
```

> [!TIP]
> Integer division results in floor division, eg - `5 / 3` = `1`

## Booleans

```
let x: bool = true;
let y: bool = false;
```

- Operators - 
    - `!` - complement
    - `==` & `!=` - equality & inequality
    - `&&` - AND
    - `||` - OR

## Chars

- Represents utf-8 characters.
- Occupy 4 bytes or 32 bits.
```
let initial: char = 'a';
```

# Compound Data Types

## Array

- Fixed size collection of homogenous data.
```
let nums: [i32; 5] = [1, 2, 3, 4, 5];
```

- Rust knows the elements & length of the array at compile time, that's why it stores the array on a stack at runtime.
- Operations -
    - Length of the array - `nums.len()`
    - Accessing elements by index - `nums[1]` - returns `2`

- Iterating arrays -
```
for num in nums {
    printlns!("{num}");
}
```

> [!NOTE]
> Explicit type annotation is necessary in case of empty array.

> [!TIP]
> Because Rust knows the length of array at compile time, the compiler will throw `index out of bounds` error at compile-time itself.

- Elements of mutable arrays (defined using `let mut`) can be updated, but the length of the array cannot be changed -
```
nums[1] = 10;
```

## Tuple

- Collection of hetrogenous elements.
- Tuple indices start from `0`.

```
let values: (i32, &str, f64) = (1, "Hello", 2.5);
```

- Accessing elements - `value.1` will return `"Hello"`.
- Tuple deconstruction -
```
val (a, b, c) = values      // a = 1, b = "Hello" & c = 2.5
```

## Range 

- Sequence/interval of consecutive values.
- Range is a Generic type implemented in a separate crate included in Rust lang.
```
let days: std::ops::Range<i32> = 1..31;               // Upper bound is exclusive (1 to 30)

let days: std::ops::RangeInclusize<i32> = 1..=31;     // Upper bound is inclusive (1 to 31)
```

- Iterating range -
```
for num in days {
    println!("{num}");
}
```

