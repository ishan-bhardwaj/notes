# ZIO Introduction #

### Local Reasoning ###
- Type signature describes the kind of computation that will be performed.

### Referential Transparency ###
- Ability to replace an expression with the value that it evaluates to.
- Not all expressions are referential transparent -
```
val resultOfPrinting: Unit = println("Learning ZIO")
val resultOfPrinting_v2: Unit = () // not the same
```

## Effect ##
- Effect properties
    - the type signature describes what KIND of computation it will perform
    - the type signature describes the type of VALUE that it will produce
    - if side effects are required, construction must be separate from the EXECUTION
- Example
    - Option is an effect
        - type signature describes the kind of computation = a possibly absent value
        - type signature says that the computation returns an A, if the computation does produce something
        - no side effects are needed
    - Future is NOT an effect
        - describes an asynchronous computation
        - produces a value of type A, if it finishes and it's successful
        - side effects are required, construction is NOT SEPARATE from execution

### Simplified ZIO Effect ###
- MyZIO is an effect
    - describes a computation which might perform side effects
    - produces a value of type A if the computation is successful
    - side effects are required, construction IS SEPARATE from execution

```
case class MyZIO[-R, E, A](unsafeRun: R => Either[E, A]) {
    def map[B](f: A => B): MyZIO[R, E, B] = 
        MyZIO(r => unsafeRun(r) match {
            case Left(e) => Left(e)
            case Right(v) => Right(f(v))
        })

    def flatMap[R1 <: R, E1 >: E, B](f: A => MyZIO[R1, E1, B]): MyZIO[R1, E1, B] = 
        MyZIO(r => unsafeRun(r) match {
            case Left(e) => Left(e)
            case Right(v) => f(v).unsafeRun()
        })
}
```

## ZIO Effect ##

- `ZIO[R, E, A]` - core data type in the ZIO library (called **_functional effect type_**) and values of this type are called **functional effects** which are kind of _blueprints_ for concurrent workflows.
- Every ZIO effect is just a description i.e. a blueprint for a concurrent workflow. When we have an effect that describes everything we need to do, we hand it off to the ZIO runtime, which executes the blueprint and produces the result of the program.

- `ZIO[-R, +E, +A]` -
    - `R` is the environment required for the effect to be executed. This could include any dependencies the effect has, for example, access to a database, or an effect might not require any environment, in which case, the type parameter will be `Any`.
    - `E` is the type of value that the effect can fail with. This could be `Throwable` or `Exception`, but it could also be a domain-specific error type, or an effect might not be able to fail at all, in which case the type parameter will be `Nothing`.
    - `A` is the type of value that the effect can succeed with. It can be thought of as the return value or output of the effect.

### Running ZIO Applications ###

- Manually -
    ```
    def main(args: Array[String]): Unit = {
        val runtime = Runtime.default
        implicit val trace: Trace = Trace.empty
        val meaningOfLife = ZIO.succeed(42)
        Unsafe.unsafeCompat { implicit u =>
            val mol = runtime.unsafe.run(meaningOfLife)
            println(mol)
        }
    }
    ```
    - Runtime involves the thread pool and the mechanism by which ZIOs can be evaluated.
    - Trace allows you to debug your code regardless of whether your effects run on the main application thread or on some other thread.

- Extending `ZIOAppDefault` -
    ```
    object Program extends ZIOAppDefault {
        override def run = zioa.debug
    }
    ```
    - `ZIOAppDefault` provides runtime, trace etc.
    - There is also a `ZIOApp` trait that allows us to implement the runtime in manual way.
    - `debug` method prints out the value of evaluated ZIO.

### Useful operations ###

- Constructing ZIO -
    - `ZIO.succced` - constructs a ZIO with success value
    - `ZIO.fail` - constructs a ZIO with a failure
    - `ZIO.attempt` - constructs a ZIO with a value or potential failure
    - `ZIO.suspend` - constructs a ZIO with _suspended_ value or potential failure

> [!TIP]
> The input parameter in `ZIO.attempt` and `ZIO.fail` is _by-name_ (using the `=> A` syntax), which prevents the
code from being evaluated eagerly, hence allowing ZIO to create a value that describes execution.

- Tranforming ZIO - 
    - `ZIO#map` - transforms value 
    - `ZIO#flatMap` - transforms value into another effect
    - `ZIO#delay` - transforms one effect into another effect whose execution is delayed into the future
    - for-comprehension -
    ```
    for {
        _ <- ZIO.succeed(println("Enter name"))
        name <- ZIO.succeed(StdIn.readLine())
        _ <- ZIO.succeed(println(s"Welcome to ZIO, $name"))
    } yield ()
    ```
    - `ZIO#as` - converts the value of a ZIO to something else.
    - `zioa#asUnit` - discards the value of a ZIO to Unit

> [!NOTE]
> In `map` & `flatMap`, the Environment and Error channel stays the same.

> [!TIP]
> ZIO flatMaps implement trampolining technique which involves allocating the ZIO instances on the heap instead of on the stack. This means that evaluating the chain of ZIOs is done in a tail recursive fashion behind the scenes by the ZIO runtime.

- Combining ZIO - 
    - `ZIO#zip` - sequentially combines the results of two effects into a tuple of their results.
    - `ZIO#zipLeft` or `zioA <* zioB` - sequentially combines two effects, returning the result of the _first_.
    - `ZIO#zipRight`or `zioA *> zioB` - sequentially combines two effects returning the result of the _second_.
    - `ZIO#zipWith` - sequentially combines values from both input ZIOs using function provided.
    - `ZIO.foreach` - returns a single effect that describes performing an effect for each element of a collection in sequence.
    - `ZIO#collectAll` - returns a single effect that collects the results of a whole collection of effects.

### ZIO Type Alias ###
- `UIO[A] = ZIO[Any, Nothing, A]` - no requirements, cannot fail, produces `A`
```
val aUIO: UIO[Int] = ZIO.succeed(99)
```

- `URIO[R, A] = ZIO[R, Nothing, A]` - requires `R`, cannot fail, produces `A`
```
val aURIO: URIO[Int, Int] = ZIO.succeed(67)
```

- `RIO[R, A] = ZIO[R, Throwable, A]` - requires `R`, can fail with a `Throwable`, produces `A`
```
val anRIO: RIO[Int, Int] = ZIO.succeed(98)
val aFailedRIO: RIO[Int, Int] = ZIO.fail(new RuntimeException("RIO failed"))
```

- `Task[A] = ZIO[Any, Throwable, A]` - no requirements, can fail with a `Throwable`, produces `A`
```
val aSuccessfulTask: Task[Int] = ZIO.succeed(89)
val aFailedTask: Task[Int] = ZIO.fail(new RuntimeException("Something bad"))
```

- `IO[E,A] = ZIO[Any,E,A]` - no requirements, can fail with `E`, produces `A`
```
val aSuccessfulIO: IO[String, Int] = ZIO.succeed(34)
val aFailedIO: IO[String, Int] = ZIO.fail("Something bad happened")
```
## Default ZIO services ##
- `Clock` - Provides functionality related to time and scheduling. If you are accessing the current time or scheduling a computation to occur at some point in the future you are using this.
- `Console` - Provides functionality related to console input and output.
- `System` - Provides functionality for getting system and environment variables.
- `Random` - Provides functionality for generating random values.