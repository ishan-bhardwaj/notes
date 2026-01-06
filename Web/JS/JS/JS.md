## Including JS to HTML ##

- Using inline-scripts -
    ```
    <script>alert("Hello world!")</script>
    ```
    - not a good practise as the code can get large.

- Importing scripts -
    ```
    <script src="assets/scripts/app.js"></script>
    ```
    - Note that closing tags like `<script/>` are not supported for script tags.
    - If we include it inside `<head>`, it will run the script before loading the html page. To render the html first, we write the `<script>` just before closing the `<body>`. 
    - Using `defer` - Tells the browser to download the scripts and parse html simultaneously. The scripts are executed only after parsing html is completed.
        ```
        <script src="assets/scripts/app.js"></script>
        ```
    - Using `async` - tells the browser to download and start loading the scripts along with parsing html simultaneously. Because the scripts are also downloaded asynchronously, the order of their execution is not guranteed.

> [!NOTE]
> `defer` and `async` are ignored for inline scripts.

> [!NOTE]
> We should not combine inline scripts with `src` as external scripts. If we do so, the inline scripts will be ignored.

## Variables & Constants ##

- Variables - 
    - Defining a variable - `let username = 'John'`
    - Reassigning - `userame = 'Alex'`
    - Variable name -
        - Best practise is to use Camel case
        - Only letters and digits allowed, but must start with a letter
        - Starting with `$` or `_` is allowed
- Constants - `const numCount = 10`

## Data Types ##
- Numbers - `2, -3, 12.57`
- Strings - `` 'Hi', "Hi", `Hi` ``
- Boolean - `true, false`
- Objects - `{ name: 'John', age: 20 }`
- Arrays - `[1, 2, 3]`

## String Interpolations ##
- Using `+` - `"Hello " + username`
- Template literals - `` `Hello ${username}` `` - only backticks are allowed

## Functions
- Syntax -
```
function <function_name>(<parameters>) {
    <body>
    <optional_return>
}
```
- Example -
```
function add(a, b) {
    return a + b;
}
```

> [!NOTE]
> Values entered to `<input>` are always String type. To convert to different datatypes - `parseInt`, `parseFloat`.

> [!TIP]
> `+numAsString` can parse the `numAsString` (defined as String but contains numeric value) into a number.

## `+` on Strings ##
- `3 + '3' = '33'` - happens because the + operator also supports strings (for string concatenation).
- It's the only arithmetic operator that supports strings, for eg - `'hi' - 'i' = NaN`
- However, JS can handle below operations -
    - `3 * '3' = 9`
    - `3 - '3' = 0`
    - `3 / '3' = 1`

## Comments ##
- Single line comments - `//`
- Multi-line comments - `/* ... */`

## Arrays ##
- Defining Arrays - `let arr = [1, 2, 3]` or `let arr = []` for empty array.
- Adding a new element - `arr.push(4)`
- Accessing element by index - `arr[index]`

## Objects ##
- Defining objects - `let person = { name: 'John', age: 20 }`
- Access elements by key - `person.name` or `person['name']`

## `null`/ `undefined` / `NaN`
- `undefined` is of type `undefined` and is the default value of uninitialized variables.
- `null` is of type `object` and is assigned to a variable if we want to _reset_ or _clear_ that variable.
- `NaN` (_Not a Number_) - is of type `number` and signifies invalid calculations like - `3 * 'Hi' = NaN`

> [!TIP]
> `typeof <value>` operator can be used to determine the datatype of the value.

> [!TIP]
> `typeof [1, 2, 3]` returns `object` - arrays are special type of objects.