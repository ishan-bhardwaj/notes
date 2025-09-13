# DataTypes

## Primitive Data Types

### Signed Integers

- Four types - `byte`, `short`, `int`, `long`.
- `long` literals require `L` suffix, eg `10L`.
- No literal syntax for `byte`/`short`, use casting `(byte) 127`.
- Literals - 
    - Hexadecimal - `0xCAFEBABE`
    - Binary - `0b1001` (= 9)
    - Octal - `011` (= 9, confusing, avoid leading zeros)
    - Underscores allowed for readability - `1_000_000`
- Special Values - `Integer.MIN_VALUE`, `Integer.MAX_VALUE` (also in Long, Short, Byte).

> [!TIP] 
> Use `BigInteger` if `long` is insufficient.

> [!TIP] 
> `Byte.toUnsignedInt(b)` maps signed `byte` (–128–127) to unsigned `int` (0–255).

### Floating-Point Types

- Default type - `double`
- Use suffix - `3.14F` (`float`), `3.14D` (`double`, optional).
- Special Values - `Double.POSITIVE_INFINITY`, `Double.NEGATIVE_INFINITY`, `Double.NaN`

> [!WARNING] 
> `x == Double.NaN` is always `false`. Use `Double.isNaN(x)` instead.

> [!TIP] 
> Use `Double.isInfinite()` or `Double.isFinite()`.

> [!WARNING] 
> Floating-point arithmetic is imprecise - `2.0 - 1.7 = 0.30000000000000004`. For precise math (finance, etc.), use `BigDecimal`.

### `char` Type
- Stores UTF-16 code units.
- Example - `'J' = 74` (`0x4A`)
- Unicode escapes - `'\u004A' == 'J'`
- Escape sequences - `\n`, `\r`, `\t`, `\b`, `\'`, `\\`

### boolean Type

- Values - `true`, `false`
- Not related to integers (0/1).

## Variables

- Declare - `int count = 0;`
- Multiple in one line - `int total = 0, count;`
- Type inference - `var random = new Random();`

> [!WARNING] 
> Variables must be initialized before use.

## Identifiers

- Names of variables, methods, classes.
- Rules - must start with letter; can contain letters, digits, `_`, currency symbols, and Unicode letters.
- Examples - `π`, `élévation` are valid.

## Constants

- Declared with final. Example - `final int DAYS_PER_WEEK = 7;`
- Convention - UPPERCASE with underscores.
- For global constants  `static final`, eg - `Calendar.DAYS_PER_WEEK`
- Must be initialized exactly once before use (can defer initialization with conditionals).

## Operators

- Assignment operators (`=`, `+=`, `-=` etc.) are right-associative, eg - `i -= j -= k` - `i -= (j -= k)`
- Integers - truncates fraction (`17 / 5 = 3`)
- Floating-point - keeps fraction (`17.0 / 5 = 3.4`)
- `&`, `|` on booleans - evaluate both operands.

> [!WARNING] 
> Integer division by zero - Exception.
> Floating-point division by zero - `Infinity` or `NaN` (no exception).

> [!TIP] 
> Use `Math.multiplyExact`, `addExact` etc. to detect overflow.

> [!WARNING] 
> Casting can silently truncate. Use `Math.toIntExact` to be safe.

- Conditional Operator - `String period = time < 12 ? "am"  - "pm";`

## `BigInteger` & `BigDecimal`

- BigInteger - arbitrary-precision integers.
- BigDecimal - arbitrary-precision decimal (for exact values).
- Examples -
```
BigInteger n = BigInteger.valueOf(876543210123456789L);
BigInteger m = new BigInteger("9876543210123456789");
BigInteger r = BigInteger.valueOf(5).multiply(n.add(m));

BigDecimal exact = BigDecimal.valueOf(2, 0)
                   .subtract(BigDecimal.valueOf(17, 1));  // 0.3 exactly
```

> [!TIP] 
> Use `parallelMultiply` for faster computation with multiple cores.

## Strings

- Immutable sequence of characters.
- Concatenation - `+`
- Join - `String.join(" | ", "Peter", "Paul", "Mary")`
- For efficient concatenation - use `StringBuilder`.
- Substring & Split -
```
String names = "Peter, Paul, Mary";
String[] arr = names.split(", ");  // ["Peter", "Paul", "Mary"]

"Hello World!".substring(7, 12);  // "World"
```

- Comparison -
    - `equals()` - value equality.
    - `==` - reference equality (use only for null check).
    - `"World".equals(location)` - safe even if location == `null`
    - `equalsIgnoreCase` ignores case.
    - `compareTo` - lex order; returns - negative, positive or 0.

> [!TIP] 
> Use Collator for locale-aware sorting.

## Number/String Conversions

- Int to String - `Integer.toString(42)`
- Int to Binary String - `Integer.toString(42, 2)` = `"101010"`
- String to Int - `Integer.parseInt("10")`
- Binary String to Int - `Integer.parseInt("101010", 2)` = `42`
- For floats - `Double.toString`, `Double.parseDouble`

