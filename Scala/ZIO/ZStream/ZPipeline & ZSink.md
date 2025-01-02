## ZPipeline & ZSink

- `ZPipeline` & `ZSink` treat stream transfomers & consumers as a _first-class entities_ respectively.
- `ZPipeline[-Env, +Err, -In, +Out]` -
    - transforms a stream of values of type `In` to a stream of values of type `Out`. 
    - Use an environment of type `Env` to perform the transformation.
    - The transformation can potentially fail with an error of type `Err`.
- `ZSink[-Env, +Err, -In, +Leftover, +Summary]` -
    - Consume a stream of values of type `In` to produce a value of type `Summary`
    - Use an environment of type `Env`
    - Can potentially fail with an error of type `Err`.
    - May also choose not to consume some of the original stream elements of type `Leftover`

```
val stream: ZStream[Any, Nothing, Int] = ZStream(1, 2, 3, 4, 5)
val pipeline: ZPipeline[Any, Nothing, Int, Int] = ZPipeline.map(_ * 2)
val sink: ZSink[Any, Nothing, Int, Nothing, Unit] = ZSink.foreach(Console.printLine(_).orDie)

stream.via(pipeline).run(sink)

// OR, using symbolic operator >>>
stream >>> pipeline >>> sink
```

- The `ZSink#zipPar` operator combines two sinks into a new sink that will send all of its inputs to both of the original sinks.

## `ZSink`

- Keeps accepting input until it produces either an error or a summary value, along with potentially some leftovers.
- Leftovers arise because some sinks may not consume all of their inputs.
- Example - (the remaining values - 4, 5 - will be returned as Leftovers)
```
val stream: ZStream[Any, Nothing, Int] = ZStream(1, 2, 3, 4, 5)
val sink: ZSink[Any, Nothing, Int, Int, Chunk[Int]] = ZSink.collectAllN(3)
stream.run(sink)        // Chunk(1, 2, 3)
```

- `ZStream#transduce` - repeatedly run the stream with the sink, emitting each summary value produce by the sink as a new stream -
```
stream.transduce(sink).runCollect       // Chunk(Chunk(1, 2, 3), Chunk(4, 5))
```

- `ZSink.collectAll` - collects all the elements of a stream

- Constructing `ZSink` -
    - `ZSink.fromFile` - writes the elements of a stream to a _file_.
    - `ZSink.fromQueue` - writes the elements of a stream to a _queue_.
    - `ZSink.fromHub` - writes the elements of a stream to a _hub_.
    - `ZSink.foreach` - perform a ZIO workflow for each stream element, such as writing it to a file or printing it to the console.
    - `ZSink.unwrapScoped` - construct a sink from a scoped ZIO workflow returning a sink, so you could, for example, open a database connection and then return a sink that writes to the database for each element.

> [!TIP]
> There are also versions of many of these stream constructors that work on chunks with the Chunk suffix. These can be useful for performance-sensitive applications where you want to work on the level of chunks instead of individual elements.
> `ZSink.fromPush` lets you construct a sink directly from the low-level representation.

- Transforming `ZSink` -
    - `ZSink#contramap` - transforms the _input type_ of the sink -
    ```
    lazy val sink: ZSink[Any, Nothing, Byte, Nothing, Unit] = ???
    sink.contramapChunks[String](strings =>
        strings.flatMap(string => Chunk.fromArray(string.getBytes))
    )
    ```
    - `ZSink#map` / `ZSink#mapZIO` / `ZSink#mapChunks` - transform the _output type_ of a sink.
    - `ZSink#flatMap` - construct a new sink based on the result of the original sink.

> [!TIP]
> `ZSink#zipPar` - sends the elements of a stream to both sinks.

> [!TIP]
> `ZSink#foldSink` - handle the failure value of a sink in addition to its success value.





