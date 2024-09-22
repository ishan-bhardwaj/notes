# Go Intro
- Go is a compiled language
- Install latest version of [Go](https://go.dev/dl/)
- Check version - `go version`
- Get the list of environment variables recognized by the `go` tool - `go env`

## First program
```
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```
- Execute program - `go run hello.go`
- The `go run` command compiles your code into a binary. However, the binary is built in a temporary directory. The `go run` command builds the binary, executes the binary from that temporary directory, and then deletes the binary after your program finishes. This makes the `go run` command useful for testing out small programs or using Go like a scripting language.
- To build the binaries, we use `go build` command. This creates an executable called `hello` (or `hello.exe` on Windows) in the current directory.
- The name of the binary matches the name of the file or package that you passed in. If you want a different name for your application, or if you want to store it in a different location, use the `-o` flag - 
```
go build -o hello_world hello.go
```

## Third-party Go tools
- To install third-party tools - `go install`
- By default, the third-party tools are installed in a workspace located in `$HOME/go` with source code stored in `$HOME/go/src` and the compiled binaries stored in `$HOME/go/bin`
- You can use this default or specify a different workspace by setting the `$GOPATH` environment variable. it’s a good idea to explicitly define `$GOPATH` and to put the `$GOPATH/bin` directory in your executable path.
- The `go install` command takes an argument, which is the location of the source code repository of the project you want to install, followed by an `@` and the version of the tool you want (if you just want to get the latest version, use `@latest`). It then downloads, compiles, and installs the tool into your `$GOPATH/bin` directory.
- `go install` is a very convenient way to distribute Go programs to other Go developers.

## Code formatting
- To auto-format the code to match the standard format - `go fmt`
- To format imports - `goimports`
- To install `goimports` - 
```
go install golang.org/x/tools/cmd/goimports@latest
```
where -
- The `-l` flag tells goimports to print the files with incorrect formatting to the console
- The `-w` flag tells goimports to modify the files in-place
- The `.` specifies the files to be scanned: everything in the current directory and all of its subdirectories

## The Semicolon Insertion Rule
- Go requires a semicolon at the end of every statement. However, Go developers never put the semicolons in themselves; the Go compiler does it for them.
- If the last token before a newline is any of the following, the lexer inserts a semicolon after the token -
    - An identifier (which includes words like int and float64)
    - A basic literal such as a number or string constant
    - One of the tokens: `“break,” “continue,” “fallthrough,” “return,” “++,” “--,” “),” or “}”`

> [!TIP]
> Always run `go fmt` or `goimports` before compiling your code!

## Linting and Vetting
- `golint` tool ensure your code follows style guidelines, but it has been deprecated. Some other recommended replacements are `golangci-lint`, `staticcheck` and `revive`.
- `go vet` checks for things like passing the wrong number of parameters to formatting methods or assigning values to variables that are never used.
- Rather than using separate tools, you can run multiple tools together with `golangci-lint`. It combines `golint`, `go vet`, and an ever-increasing set of other code quality tools -
```
golangci-lint run
```

## Makefiles
- `make` is used to specify your build steps.
- Example - 
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
- Each possible operation is called a `target`. The `.DEFAULT_GOAL` defines which target is run when no target is specified. In our case, we are going to run the `build` target.
- Next we have the target definitions. The word before the colon (:) is the name of the target. Any words after the `target` (like `vet` in the line `build: vet`) are the other targets that must be run before the specified target runs.
- The tasks that are performed by the target are on the indented lines after the target. (The `.PHONY` line keeps `make` from getting confused if you ever create a directory in your project with the same name as a target.)
- Before you can use this `Makefile`, you need to make this project a Go module -
```
go mod init <directory-name>
```
Now you can use the Makefile -
```
make
```
You should see the following output:
```
go fmt ./...
go vet ./...
go build hello.go
```

- Installation on Windows - use a package manager like `Chocolatey` and then use it to install make -
```
choco install make
```

## Migrating Apps to latest versions
- One option is to install a secondary Go environment. For example, if you are currently running version `1.15.2` and wanted to try out version `1.15.6`, you would use the following commands -
```
$ go get golang.org/dl/go.1.15.6
$ go1.15.6 download
```
- You can then use the command `go1.15.6` instead of the `go` command to see if version `1.15.6` works for your programs -
```
$ go1.15.6 build
```
- Once you have validated that your code works, you can delete the secondary environment by finding its `GOROOT`, deleting it, and then deleting its binary from your `$GOPATH/bin directory`. Here’s how to do that on Mac OS, Linux, and BSD:
```
$ go1.15.6 env GOROOT
/Users/gobook/sdk/go1.15.6
$ rm -rf $(go1.15.6 env GOROOT)
$ rm $(go env GOPATH)/bin/go1.15.6
```
- For Windows and Mac, you cab download the latest installer, which removes the old version when it installs the new one.
