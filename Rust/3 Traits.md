# Trait

- A trait is a contract that requires that a type supports one or more methods.
- Types can vary in their implementation but still implement the same trait.
- A type can implement multiple traits.

## Display Trait

- Requires that a type can be represented as a user-friendly, readable string.
- Mandates `format` method that returns the string.
- When we use `{}` interpolation syntax, Rust relies on `format` method.

## Debug Trait

- Should format the output in a programmer-friendly, debugging context.
- Example Usage -
```
let arr = [1, 2, 3, 4, 5]
println!("{arr:?}")

// pretty-print
println!("{arr:#?}")
```

> [!NOTE]
> Array, Tuple & Range implement Debug trait, but not the Display trait.


### `dbg!` Macro

- Prints and returns the value of a given expression for quick debugging.
- Uses Debug trait's `format` method to ouput several details about the content passed in it.
- Eg -
```
dbg!(2 + 2);        // prints file name, location (line number etc), the operation & the result
```
