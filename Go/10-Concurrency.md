# Goroutines

- Goroutines execute code in parallel.
- Creating a goroutine - `go <function_call>`
- Creating a channel - `make(chan <datatype>)` - `datatype` defines what type of value the channel will emit.
- Example -
```
func greet(name string, isDone chan bool) {
    time.Sleep(2 * time.Second)
    fmt.Println("Hello, " name)
    isDone <- true
}

func main() {
    isDone := make(chan bool)
    go greet("John", isDone)
    <- isDone                   // can also print the value
}
```

> [!TIP]
> We can also print the value emit by channel - `fmt.Println(<- isDone)`

- One channel can be used in multiple goroutines. If we care about any one goroutine that finishes first, we call `<- isDone` once. But if we need to wait for all the goroutines to complete, we need to call `<- isDone` as many times as the number of goroutines.

- We can also use a for-loop to wait for all the goroutines to return the result -
```
for range isDone {
    <...>
}
```

> [!WARNING]
> Looping for channel will throw a fatal error if do not "close" the channel. When we know that one goroutine will take the longest to return, we should close the channel after it's done.

- To close the channel - `close(isDone)`

- Creating slice of channels (if we want to keep track of multiple goroutines with different channels) -
```
dones := make([] chan bool, 2)

for i, _ := range dones {
    dones[i] = make(chann bool)
}

// waiting for all the channels to finish
for _, done := dones {
    <- done
}
```

> [!TIP]
> Channels can also be used to transmit errors.

- If we have multiple channels and we only need to wait for any of them (for eg - doneChan and errChan) -
```
select {
    case err := <- errChan:
        if err != nil {
            fmt.Println(err)
        }
    case <- doneChan:
        fmt.Println("Done!")
}
```

## `defer`

- Defers the execution of function until the surrounding method is finished -
```
func ReadLines(path string) (string, error) {
    file, err := os.Open(path)

    if err != nil {
        return nil, errors.New("Failed to open file")
    }

    defer file.Close()              // won't be called here

    <...>

    return "File read sucessfully!"
}                                   // defer will be called here
```

