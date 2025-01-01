## ZStream

- `ZStream[R, E, O]` - an effectful stream that requires an environment `R`, may fail with an error `E`, and may succeed with zero or more values of type `O`.
- A ZIO produces its result all at once whereas a ZStream produces its result _incrementally_.
- A `ZStream` is defined in terms of one operator - `process` - which can be evaluated repeatedly to pull more elements from the stream.
```
trait ZStream[-R, +E, +O] {
    def process: ZIO[R with Scope, Option[E], Chunk[O]]
}
```

- Each time the `process` is evaluated, it will either -
    - Succeed with a `Chunk` of new `O` values that have been pulled from the stream,
    - Fail with an error `E`, or
    - Fail with `None` indicating the end of the stream.

> [!TIP]
> The `process` is a scoped ZIO so that resources can be safely acquired and released in the stream context.

## Type Parameters

- `ZStream[R, E, O]` -
    - `R` represents the set of services required to run the stream.
        - To provide the environment, we can use - `provide`, `provideLayer`, `ZStream.provideSomeLayer`, and `ZStream#provideCustomLayer\index{ZStream\#provideCustomLayer}`.
        - Accessing enviroment (all of them returns a `ZStream`) -
            - `ZStream#environment` - Access the whole environment and return a value that depends on it
            - `ZStream#environmentWith` - Access the given environment and return a stream that depends on it
            - `ZStream#environmentWithZIO` - Access the given environment and return a ZIO that depends on it
            - `ZStream#environmentWithStream` - Access the given environment and return a ZStream that depends on it
    - `E` represents the type of errors the stream can potentially fail with.
        - We can use `ZStream#catchAll` or `ZStream#catchSome` to recover from errors in a stream and fall back to a new stream instead.
        - Note that if a stream fails with an `E`, we can use an operator like `ZStream#catchAll` to ignore that error and execute the logic of a new stream instead. However, it is not well-defined to ignore that error and then pull from the original stream again.
    - `O` represents the type of values output by the stream.

## Constructing ZStream -

### From existing values
- `apply` method -
```
val stream: ZStream[Any, Nothing, Int] = ZStream(1, 2, 3, 4, 5)
```
    
- From collections -
```
object ZStream {
    def fromIterable[O](os: Iterable[O]): ZStream[Any, Nothing, O] = ???
}

// Example
val stream = ZStream[Any, Nothing, Int] = ZStream.fromIterable(List(1, 2, 3, 4, 5))
```

### From effects
```
object ZStream {
    def fromZIO[R, E, O](zio: ZIO[R, E, O]): ZStream[R, E, O] = ???
    def scoped[R, E, O](zio: ZIO[R with Scope, E, O]): ZStream[R, E, O] = ???
}
```

### From repetitions
```
object ZStream {
    def repeat[O](o: => O): ZStream[Any, Nothing, O] = ???
    def repeatZIO[R, E, O](zio: ZIO[R, E, O]): ZStream[R, E, O] = ???
    def repeatZIOChunk[R, E, O](zio: ZIO[R, E, Chunk[O]]): ZStream[R, E, O] = ???
    def repeatZIOOption[R, E, O](zio: ZIO[R, Option[E], O]): ZStream[R, E, O] = ???
    def repeatZIOChunkOption[R, E, O](zio: ZIO[R, Option[E], Chunk[O]]): ZStream[R, E, O] = ???
}

// Example
lazy val ones: ZStream[Any, Nothing, Int] = ZStream.repeat(1)

trait Tweet
lazy val getTweets: ZIO[Any, Nothing, Chunk[Tweet]] = ???
lazy val tweetStream: ZStream[Any, Nothing, Tweet] = ZStream.repeatZIOChunk(getTweets)
```

- The basic variants create streams that repeat those effects forever, while the Option variants return streams that terminate when the provided effect fails with `None`.

### From unfolding
```
object ZStream {
    def unfold[S, A](s: S)(f: S => Option[(A, S)]): ZStream[Any, Nothing, A] = ???
    def unfoldZIO[R, E, A, S](s: S)(f: S => ZIO[R, E, Option[(A, S)]]): ZStream[R, E, A] = ???
    def unfoldChunk[S, A](s: S)(f: S => Option[(Chunk[A], S)]): ZStream[Any, Nothing, A] = ???
    def unfoldChunkZIO[R, E, A, S](s: S)(f: S => ZIO[R, E, Option[(Chunk[A], S)]]): ZStream[R, E, A] = ???
}
```

- The `S` type parameter allows us to maintain some internal state when constructing the stream.
- The way these constructors work is as follows:
    a. The first time values are pulled from the stream, the function `f` will be applied to the initial state `s`
    b. If the result is `None`, the stream will end
    c. If the result is `Some`, the stream will return `A`, and the next time values are pulled, the first step will be repeated with the new `S`.

- Using `ZStream#unfold` (and `ZStream#unfoldZIO`) to construct a `ZStream` from a `List` where the state we maintain is the remaining list elements -
```
def fromList[A](list: List[A]): ZStream[Any, Nothing, A] =
    ZStream.unfold(list) {
        case h :: t => Some((h, t))
        case Nil => None
    }
```
    
- If there are remaining list elements, we emit the first one and return a new state with the remaining elements. If there are no remaining list elements, we return `None` to signal the end of the stream.

## Running Streams

- Running a ZStream involves compiling it down to a single ZIO effect that describes running the streaming program all at once and producing a single value. The second step actually runs that ZIO effect.

### Folding over stream values

- **`ZStream#foldWhile`** -
    - `ZStream#foldWhile` (and `ZStream#foldWhileZIO`) allows reducing a stream to a summary value and also signaling early termination.
    ```
    trait ZStream[-R, +E, +O] {
        def foldWhile[S](s: S)(cont: S => Boolean)(f: (S, O) => S): ZIO[R, E, S]
    }
    ```

- **`ZStream#runDrain`** -
    - `ZStream#runDrain` runs a stream to completion purely for its effects. The `ZStream#runDrain` operator always continues but ignores the values produced by the stream, simply returning an effect that succeeds with the `Unit` value.

    - Usecase - `ZStream#runDrain` operator is useful if your entire logic for this part of your program is described as a stream. For example, we could have a stream that describes getting tweets, performing some filtering and transformation on them, and then writing the results to a database.

    - Implementing `ZStream#runDrain` using `ZStream#foldWhile` -
    ```
    def runDrain: ZIO[R, E, Unit] = foldWhile(())(_ => true)((s, _) => s)
    ```

- **`ZStream#foreach`** -
    - `ZStream#runDrain` in that it runs a stream to completion for its effects, but now it lets us specify an effect we want to perform for every stream element.
    - For example, if we had a stream of tweets, we could call `ZStream#foreach` to print each tweet to the console or write each tweet to a database.
    - Works well with for-comprehensions -
    ```
    val effect: ZIO[Any, Nothing, Unit] =
        for {
            x <- ZStream(1, 2)
            y <- ZStream(x, x + 3)
        } Console.printLine((x, y).toString).orDie
    ```
    
### Returning Stream values

- Returning head (`runHead`) or last (`runLast`) element from stream -
```
def runHead: ZIO[R, E, Option[O]] = foldWhile[Option[O]](None)(_.isEmpty)((_, o) => Some(o))
def runLast: ZIO[R, E, Option[O]] = foldWhile[Option[O]](None)(_ => true)((_, o) => Some(o))
```

- To collect all the values produced by the stream into a `Chunk` -
```
def runCollect: ZIO[R, E, Chunk[O]] = foldWhile[Chunk[O]](Chunk.empty)(_ => true)((s, o) => s :+ o)
```

- To get sample of first N elements, we can use `ZStream.take(N)`

- Count and sum of the stream -
```
def runCount: ZIO[R, E, Long] = foldWhile(0L)(_ => true)((s, _) => s + 1L)
def runSum[O1 >: O](implicit ev: Numeric[O1]): ZIO[R, E, O1] = foldWhile(ev.zero)(_ => true)(ev.plus)
```
