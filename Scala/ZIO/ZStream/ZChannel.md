## ZChannel

- `ZChannel[-Env, -InErr, -InElem, -InDone, +OutErr, +OutElem, +OutDone]` represents -
    - Combination of `ZStream`, `ZPipeline` and `ZSink` where `Elem` type represents incremental output like the stream and pipeline and `Done` or `Err` type represents final done value or failure like the sink.
    - “duals” to each of our output types.

- `ZChannel` is a description of a program which -
    - requires a set of services `Env`
    - accepts zero or more inputs of type `InElem` along with one final input of either type `InErr` or `InDone`
    - produces zero or more outputs of type `OutElem` along with one final output of either type `OutErr` or `OutDone`.

> [!TIP]
> We use `Any` for the input channels and `Nothing` for the output channels if they are not being used.

- Streams as channels - `ZStream[-R, +E, +A] = ZChannel[R, Any, Any, Any, E, Chunk[A], Any]` -
    - A stream is a producer of zero or more values of type `Chunk[A]`. It doesn’t accept any inputs, and it doesn’t produce any terminal value, so we can replace each of the input types with `Any` just like we do for the environment type when we don’t require any services.
    - A stream also doesn’t produce any meaningful “done” value, so we can model this with the `Unit` value or with the `Any` type.

- Sinks as channels - `ZSink[-R, +E, -In, +L, +Z] = ZChannel[R, Nothing, Chunk[In], Any, E, L, Z]`
    - A sink uses an environment of type `R`, consumes elements of type `Chunk[In]`, can potentially emit leftovers of type `L`, and eventually produces either a summary value of type `Z` or fails with an error of type `E`.
    - A sink doesn’t work with any more information about the done value of the upstream than the fact that it is done, so we can make that type `Any`.
    - The sink itself doesn’t know how to handle any errors, so the error type has to be `Nothing`.

- Pipeline as channels - `ZPipeline[-R, +E, -In, +Out] = ZChannel[R, Nothing, Chunk[In], Any, E, Chunk[Out], Any]`
    - A pipeline both accepts multiple inputs and produces multiple outputs. 
    - It also doesn’t know how to handle errors and doesn’t accept or produce a meaningful done value.

- ZIO as channels - `ZIO[-R, +E, +A] = ZChannel[R, Any, Any, Any, E, Nothing, A]`
    - A ZIO is a workflow that requires a set of services `R`, doesn’t require any inputs, doesn’t produce any incremental outputs, and either produces a terminal done value `A` or fails with an error `E`.

- ZIO does not actually define sink, pipeline and sink as a type alias for a channel but instead defines them as a new data type that wraps a channel -
```
final case class ZStream[-R, +E, +A](
    channel: ZChannel[R, Any, Any, Any, E, Chunk[A], Any]
)
```

## Channel Constructors

- Each of the output type parameters of ZChannel has a corresponding constructor that outputs a value in the appropriate channel -
```
object ZChannel {
    def fail[E](cause: Cause[E]): ZChannel[Any, Any, Any, Any, E, Nothing, Nothing] = ???
    def succeed[Done](done: Done): ZChannel[Any, Any, Any, Any, Nothing, Nothing, Done] = ???
    def write[Out](out: Out): ZChannel[Any, Any, Any, Any, Nothing, Out, Any] = ???
}
```

> [!NOTE]
> `write` writes an element to the output channel.

- `ZChannel.readWith` - reads an input and executes a new channel based on the result. Channels are pull-based, so when we read, we might get either an `InElem` or a terminal `InDone` or `InErr` value, so `readWith` requires us to handle all of those possibilities.
- `ZChannel.fromZIO` - convert a ZIO workflow into a ZChannel.
- `ZChannel.scoped` - imports a scoped ZIO into the channel world, removing the Scope as a requirement and subsuming it in the Scope of the channel.

## Channel Operators

- `ZChannel#foldChannel` - run one channel and then run another channel after the first one is done based on its result.
- `ZChannel#map` - maps elements while leaving errors and done values unchanged.
- Similaryly, we have `ZChannel#flatMap` and `*>` just like in ZIO.
- `ZChannel#pipeTo` - “pipes” all the outputs from one channel to the inputs of another channel. It has a symbolic alias `>>>` sending all of the inputs through the given channel.

> [!TIP]
> All of the operators for combining streams, sinks, and pipelines are implemented in terms of `ZChannel#pipeTo`.

- `ZChannel#concatMap` - like `flatMap` but for the `Out` channel instead of the `Done` channel. It takes an existing channel, maps each of the elements output by that channel to a new channel, and then sequentially runs those channels and emits their elements.

> [TIP]
> `ZChannel#flatMap` is implemented in terms of `ZChannel#concatMap`.

- `ZChannel#mergeWith` runs two channels concurrently and merges their outputs. Both channels are run at the same time, and both have the ability to read from the upstream and write to the downstream. Once one of the channels is done, what happens next is controlled by the `MergeDecision`, which is an algebraic data type that describes either terminating the slower channel or waiting for the slower channel to complete and doing something with its result.

- `ZChannel#ensuring` - allows adding a finalizer that will be run after a channel is done.

## Channel Scopes

- A ZIO workflow never produces any incremental output, so when we use an operator like `ZIO#ensuring`, it is clear that the finalizer should be run immediately after ZIO completes execution and before any subsequent workflow is run.
- Example - 
```
zio1.ensuring(finalizer).flatMap(_ => zio2)
```
Here, `finalizer` will be run immediately after `zio1` completes execution and before `zio2` begins execution.

- In the context of channels -
    - `channel1.ensuring(finalizer).flatMap(_ => channel2)`
        - `ZChannel#flatMap` is not executed until `channel1` completes execution because `flatMap` needs the `Done` value of `channel1` to determine what to do next, so finalizer will run immediately after `channel1` completes execution and before `channel2` begins execution. This is why we sometimes say that `ZChannel#flatMap` “closes” the scope of the channel.
    - `channel1.ensuring(finalizer).pipeTo(channel2)`
        - `channel1` is not done until `channel2` has finished reading from it, so while `channel2` is processing elements emitted by `channel1`, it can rely on `finalizer` not being run yet.
    - `channel1.ensuring(finalizer).concatMap(_ => channel2)`
        - `channel2` needs to be run before `finalizer` is run.
        - Since `ZStream#flatMap` is implemented in terms of `ZChannel#concatMap`, streams have a “dynamic scope” where in `stream1.ensuring.flatMap(_ => stream2)` the `finalizer` will not be run until `stream2` completes execution since the stream is not really “done” until the processing of its downstream is done.
    - `channel1.ensuring(finalizer).mergeWith(channel2)`
        - the `finalizer` will again be deferred to the scope of the merged stream because even though `channel1` is done emitting elements someone downstream may be processing those elements.
