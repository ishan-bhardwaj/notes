# Golang

- Go is a type-safe compiled language.
- Check version - `go version`
- Print all the Go env variables - `go env`

### Hello World

```
package main

import "fmt"

func main() {
    fmt.Println("Hello World")
}
```


### `go run`

- Executes the Go file(s) - `go run hello.go`
- Compiles the code into binary which is stored in a temporary directory. Once built, it executes those binaries and then deletes the binary after your program finishes.
- Useful for testing out small programs or using Go like a scripting language.


### `go build`

- Builds the binary for later use - `go build hello.go`
- The name of the binary matches the name of the file or package in the same directory where the Go file is.
- To use different name or location, use `-o` option. Eg - this will compile code to binary named `hello_world` -
```
go build -o hello_world hello.go
```


### `go install`

- Instalsl third-party Go tools - `go install`. By default, this workspace is located in `$HOME/go` -
    - `$HOME/go/src` - source code
    - `$HOME/go/bin` - compiled binaries
- To specify different workspace, set `$GOPATH` environment variable.
- Go developers do not rely on centrally hosted service like Maven Central for Java, NPM registry for JS.
- `go install` takes the source code repository of the third-party project, followed by the version. 
- Downloads, compiles and installs the tool in `$GOPATH/bin` directory.
- Eg -
```
go install github.com/rakyll/hey@latest
```
- The contents of Go repositories are cached in proxy servers. Depending on the repository and the values in your `GOPROXY` environment variable, `go install` may download from a proxy or directly from a repository.
- Git is required to download from repository.
- To update an installed tool to a newer version, rerun `go install` with the newer version specified.

> [!TIP]
> To download the latest version, use `@latest`.


### `go fmt` / `goimports`

-  Go defines a standard way of formatting code.
- `go fmt` automatically reformats your code to match the standard format.
- `goimports` cleans up the import statements.
- To download `goimports` -
```
go install golang.org/x/tools/cmd/goimports@latest
```

- Running `goimports` across the project -
```
goimports -l -w .
```
where -
    - `-l` prints the files with incorrect formatting
    - `-w` to modify the files in-place
    - `.` specifies files to be scanned - everything in the current directory and all of its subdirectories.


### Semi-colon Insertion Rule

-  Go requires a semicolon at the end of every statement. 
- However, Go developers never put the semicolons in themselves. The Go compiler does it automatically based on the rule - If the last token before a newline is any of the following, the lexer inserts a semicolon after the token -
    - An identifier (which includes words like int and float64)
    - A basic literal such as a number or string constant
    - One of the tokens: `break`, `continue`, `fallthrough`, `return`, `++`, `--`, `)` or `}`
- That's why `go fmt` command won’t fix braces on the wrong line.
- For eg - in `func main()`, the semicolon insertion rule sees the `)` at the end of the line and turns that into `func main();` which is not valid Go.


### Linting & Vetting

- `golint` ensure your code follows style guidelines.
- `golint ./...` runs golint over your entire project.
- `golint` has been deprecated. Use other tools like - `golangci-lint`, `staticcheck` and `revive`.
- `go vet` points out code that is syntactically valid, but there are mistakes that are not what you meant to do like assigning values to variables that are never used.
- `go vet ./...` runs go vet over your entire project.
- `golangci-lint` combines `golint`, `go vet`, and an ever-increasing set of other code quality tools. You can configure which linters are enabled and which files they analyze by including a file named `.golangci.yml` at the root of your project - `golangci-lint run`


## Makefiles

- Used for automating build steps that can be run by anyone, anywhere, at any time. Eg -
```
.DEFAULT_GOAL := build

fmt:
        go fmt ./...
.PHONY:fmt

lint: fmt
        golint ./...
.PHONY:lint

vet: fmt
        go vet ./...
.PHONY:vet

build: vet
        go build hello.go
.PHONY:build
```

- Each possible operation is called a target.
- The `.DEFAULT_GOAL` defines which target is run when no target is specified.
- Target definitions - The word before the colon (`:`) is the name of the target. Any words after the target (like `vet` in the line `build: vet`) are the other targets that must be run before the specified target runs.
- The tasks that are performed by the target are on the indented lines after the target. 
- The `.PHONY` line keeps make from getting confused if you ever create a directory in your project with the same name as a target.
- Makefile is run on modules. Create module - `go mod init my_app` and then run `make` to execute the Makefile.


### Upgrading Go version

- Install a secondary Go environment, for eg - if you are currently running version `1.15.2` and wanted to try out version `1.15.6` -
```
$ go get golang.org/dl/go.1.15.6
$ go1.15.6 download
```

- Then use the command `go1.15.6` instead of the `go` to see if version `1.15.6` works for your programs -
```
$ go1.15.6 build
```

- Once you have validated that your code works, you can delete the secondary environment by finding its `GOROOT`, deleting it, and then deleting its binary from your `$GOPATH/bin` directory -
```
$ go1.15.6 env GOROOT
/Users/gobook/sdk/go1.15.6
$ rm -rf $(go1.15.6 env GOROOT)
$ rm $(go env GOPATH)/bin/go1.15.6
```