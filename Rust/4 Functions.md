# Functions

- Function with no-args -
```
fn greet() {
  println!("Hello World!");
}
```

- Function with multiple args -
```
fn greet(username: &str) {
  println!("Hello {username}");
}

fun add(x: i32, y: i32) {
  println!("Sum is: {x + y}");
}
```

- Returning a value -
```
fun add(x: i32, y: i32) -> i32 {
  return x + y;
}
```

> [!NOTE]
> Every function must have a return value.

- Implicitly returning a result - 
  - The last line of the function is returned.
  - Can omit semi-colon.
  ```
  fun add(x: i32, y: i32) -> i32 {
    x + y                          // Implicitly returned
  }
  ```

## `Unit` as a return type

- A __Unit__ is an empty tuple.
- Unit is the default return value of a function if no return value is specified either implicitly or explicitly.
```
fn example() {}

let result: () = example();
```

## Blocks

- Defines scope of code.
```
fn main() {
  let x = 10;

  {
    let y = x + 20;
    println!("Value of y is: {y}");
  }

  // Blocks can be assigned to variables with last line returning a value
  let z = {
    let total = x * y;
    total + 1;
  }
}
```

