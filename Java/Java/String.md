# Strings

- Strings are sequences of characters.
- Immutable.
- String is a reference type but special — can be created without `new`
- Implements `CharSequence` (along with `StringBuilder`, etc.).
- Creating Strings -
```
String s1 = "Fluffy";                  // String literal
String s2 = new String("Fluffy");      // Using new
String s3 = """
             Fluffy""";                // Text block
```

- Concatenation Rules -
    - Evaluated left to right.
    - Examples -
    ```
    1 + 2          - 3
    "a" + "b"      - ab
    "a" + "b" + 3  - ab3
    1 + 2 + "c"    - 3c
    "c" + 1 + 2    - c12
    "c" + null     - cnull
    ```

    - With variables -
    ```
    int three = 3;
    String four = "4";
    System.out.println(1 + 2 + three + four); // 64
    ```

## String Methods -
- Length - `str.length()`
- Accessing Characters - `str.charAt(i) `
- Code Points (Unicode support) -
```
str.codePointAt(i)
str.codePointBefore(i)
str.codePointCount(start, end)
```

- Substrings -
    ```
    str.substring(begin)
    str.substring(begin, end)   // end is exclusive
    ```

    - `substring(3,3)` - empty string
    - `substring(3,2)` - exception
    - `substring(3,8)` - exception
    
- Finding Index - returns `-1` if not found.
```
str.indexOf(ch/str)
str.indexOf(ch/str, fromIndex)
str.indexOf(ch/str, fromIndex, endIndex)
```

- Case Conversion - 
```
toLowerCase()
toUpperCase()
```

- Equality - 
```
equals(Object obj)
equalsIgnoreCase(String str)
```

- Substring Search (case-sensitive)
```
startsWith(prefix)
startsWith(prefix, fromIndex)
endsWith(suffix)
contains(CharSequence seq)
```

- Replacing -
```
replace(char old, char new)
replace(CharSequence target, CharSequence replacement)
```

- Trimming & Stripping -
```
strip()          // removes leading & trailing Unicode whitespace
stripLeading()   // removes only leading
stripTrailing()  // removes only trailing
trim()           // removes ASCII whitespace only
```

- Indentation -
```
indent(n)     // +ve = add spaces, -ve = remove spaces
stripIndent() // removes common leading whitespace
```

    - Normalizes line breaks to `\n`.
    - `indent()` always adds a line break at end (if missing).
    - `stripIndent()` does not add line break.

- Empty & Blank Strings -
```
isEmpty()  // length == 0
isBlank()  // empty or only whitespace
```

- Overriding Object Methods -
    - `toString()` → customize object representation.
    - `equals(Object)` → check logical equality.
hashCode() → must be consistent with equals.

If a.equals(b) == true, then a.hashCode() == b.hashCode().