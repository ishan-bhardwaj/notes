# Golang

- Create a directory and intialize it as go module -
    - Using VS code - `Search` > Type `Go - Initialize go.mod` > Enter module name
    - This will create a `go.mod` file inside the directory.
    - The `go.mod` file declares your module’s import path and depedencies & their versions.

## Hello World

```
package main

import "fmt"

function main() {
    fmt.Println("Hello World!")
}
```

> [!NOTE]
> Every Go file must start with a package declaration to specify which package it belongs to, enabling the compiler and import system to organize and build your code. `package main` and `func main()` signifies that it is the entry point.

- To import multiple modules at the same time -
```
import (
    "fmt",
    "strings"
)
```

> [!NOTE]
> Anything starting with only upper-case letter is public. That's why we're able to access `fmt.Println`.

- Running the program - `go run helloWorld.go`
    - Compiles / builds the files into a temp directory and then executes it.

- Build - `go build`
    - If no path is mentioned here, it will write the output files into current working directory.
    - By default, builds the binaries for current OS, but to cross-compile for others (for eg - linux) - `GOOS=linux go build`

- Cleanup the binaries - `go clean`

> [!WARNING]
> Unused variables & methods are compilation-errors in Go.