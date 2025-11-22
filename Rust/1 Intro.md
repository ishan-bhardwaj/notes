# Rust

- Each version of Rust aims to be backward compatible.
- Check version - `rustc --version`
- Updating rust - `rustup update`
- To uninstall - `rustup self uninstall`
- Local documentation - `rustup doc`
- Cargo is a command line tool that helps manage Rust projects.

- Creating a Rust binary project - `cargo new <project_name>`

> [!TIP]
> In Rust, we store project settings and dependencies in `Cargo.toml` file. TOML = Tom's Obvious Minimal Language.

## Hello World

- `main.rs` -
```
fn main() {
    println!("Hello World");
}
```

- Compiling - 
    - `rustc main.rs` - creates executable binary (`.exe` on Windows).
    - `cargo build` or `cargo b` - creates executable binary for the entire project.
- Formatting - 
    - `rustfmt main.rs` - formats `mains.rs` file.
    - `cargo fmt` - formats entire project.
- `cargo build` runs in two modes -
    - Debug mode -
        - default 
        - fast and unoptimized version
        - binary file is generated in `target/debug/` directory.
    - Release mode - 
        - `cargo build --release`
        - slower but optimized for runtime performance
        - binary file is generated in `target/release/` directory.

- Cleaning target directory - `cargo clean`
- Running project directly (build + run) in single step - `cargo run` or `cargo r`
- To run project without showing steps - `cargo run --quiet`
- To check any violations without compiling - `cargo check`
- Comments - 
    - Single line - `//`
    - Multi-line - `/* */`

## Variables

- Immutable by default.
- Can only be declared inside a function.
- Defining an immutable variable - `let num_users = 10;`
- Defining a mutable variable - `let mut num_users = 10;`
- Explicit type annotation - `let num_users: i32 = 10;`
- Interpolating a variable - 
```
let active_users = 2;

println!("Number of users: {}, active users: {}", num_users, active_users);

// or - gives the ability to reuse variables based on the index
println!("Number of users: {0}, active_users: {1}, total users: {0}", num_users, active_users);

// or
println!("Number of users: {num_users}, active users: {ative_users}")
```

> [!TIP]
> If a variable is unused, rust compiler will show us a warning. To avoid it, append the unused variable with underscore (`_`).

## Constants

- Immutable values.
- Constant value must be known at compile-time.
- Can be declared at both function and file level.
- Explicit type annotation is necessary.
- Defining a constant - `const PI: f64 = 3.14;`

## Type Alias

- Declares an alias for an existing type.
```
type Meters = i32;

fn main() {
    let race_length: Meters = 1600;
}
```

## Compiler Directive

- Annotation that tells the compiler how to parse the source code.
- Can be applied to individual lines, functions or the entire file.
- Eg - to allow unused variable -
```
#[allow(unused_variables)]
let race_length: Meters = 1600;
```

- To apply directive to the whole file - `#![allow(unused_variables)]` - by `!`, the compiler knows that the directive is applied to the whole file, not just the next line.
