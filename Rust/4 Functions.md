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
  ```
  fun add(x: i32, y: i32) -> i32 {
    x + y                          // Implicitly returned
  }
  ```

  - When returning implicitly, the semi-colon must be ommitted because semi-colon creates a _statement_ which prevents the implicit return and thus, the function returns `unit` instead.
  - Example - below code will return `unit` -
  ```
  fn shoe_size() -> i32 {
    12;
  }
  ```

  - This rule is applicable for code blocks as well.

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

