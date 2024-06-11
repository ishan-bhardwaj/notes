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

```
import zio._

val meaningOfLife: ZIO[Any, Nothing, Int] = ZIO.succeed(42)
val aFailure: ZIO[Any, String, Nothing] = ZIO.fail("Something went wrong")
val aSuspendedZIO: ZIO[Any, Throwable, Int] = ZIO.suspend(meaningOfLife)

// map & flatMap
val improvedMOL = meaningOfLife.map(_ * 2)
val printingMOL = meaningOfLife.flatMap(mol => ZIO.succeed(println(mol)))

// for comprehensions
val smallProgram = for {
    _ <- ZIO.succeed(println("what's your name"))
    name <- ZIO.succeed(StdIn.readLine())
    _ <- ZIO.succeed(println(s"Welcome to ZIO, $name"))
  } yield ()

// zip, zipWith
val anotherMOL = ZIO.succeed(100)
val tupledZIO = meaningOfLife.zip(anotherMOL)
val combinedZIO = meaningOfLife.zipWith(anotherMOL)(_ * _)
```

### Some useful operations ###
- Evaluate two ZIOs in sequence and take value of the LAST one - `zioa *> ziob`
- Evaluate two ZIOs in sequence and take value of the FIRST one - `zioa <* ziob`
- Convert the value of a ZIO to something else - `zioa.as(value)`
- Discard the value of a ZIO to Unit - `zioa.asUnit`
Note that ZIO flatMaps implement trampolining technique which involves allocating the ZIO instances on the heap instead of on the stack. This means that evaluating the chain of ZIOs is done in a tail recursive fashion behind the scenes by the ZIO runtime.

- In `ZIO.suspend`, the error channel is `Throwable` because `ZIO.suspend` is used for effects whose construction itself may fail.
- In `map` & `flatMap`, the Environment and Error channel stays the same.

### ZIO Type Alias ###
- `UIO[A] = ZIO[Any,Nothing,A]` - no requirements, cannot fail, produces A
```
val aUIO: UIO[Int] = ZIO.succeed(99)
```
- `URIO[R,A] = ZIO[R,Nothing,A]` - cannot fail
```
val aURIO: URIO[Int, Int] = ZIO.succeed(67)
```
- `RIO[R,A] = ZIO[R,Throwable, A]` - can fail with a Throwable
```
val anRIO: RIO[Int, Int] = ZIO.succeed(98)
val aFailedRIO: RIO[Int, Int] = ZIO.fail(new RuntimeException("RIO failed"))
```
- `Task[A] = ZIO[Any, Throwable, A]` - no requirements, can fail with a Throwable, produces A
```
val aSuccessfulTask: Task[Int] = ZIO.succeed(89)
val aFailedTask: Task[Int] = ZIO.fail(new RuntimeException("Something bad"))
```
- `IO[E,A] = ZIO[Any,E,A]` - no requirements
```
val aSuccessfulIO: IO[String, Int] = ZIO.succeed(34)
val aFailedIO: IO[String, Int] = ZIO.fail("Something bad happened")
```

### Running ZIO Application from main() ###
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

### Running ZIO Application in ZIO way ###
```
object Program extends ZIOAppDefault {
    override def run = zioa.debug
}
```
- ZIOAppDefault provides runtime, trace etc.
- There is also a `ZIOApp` trait that allows us to implement the runtime in manual way.
- `debug` method prints out the value of evaluated ZIO.