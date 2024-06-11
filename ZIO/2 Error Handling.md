# Error Handling #

- Transforming error channel into something else -
```
val failedZIO = ZIO.fail(new RuntimeException("Error!"))
failedZIO.mapError(_.getMessage)
```

- Attempt - run an effect that might throw an exception
```
val anAttempt: Task[Int] = ZIO.attempt {
    val string: String = null
    string.length
}
```

### Effectfully catch errors ###
```
val catchAllError: ZIO[Any, Nothing, Any] = anAttempt.catchAll(e => ZIO.succeed("Returning default value because $e"))
val catchSelectiveError: ZIO[Any, Throwable, Any] = anAttempt.catchSome {   // partial function
    case e: RuntimeException => ZIO.succeed(s"Ignoring runtime exceptions: $e")
    case _ => ZIO.succeed("Ignoring everything else")
}
```

### Chaining effects ###
```
val aBetterAttempt: ZIO[Any, Nothing, Int] = anAttempt.orElse(ZIO.succeed(45))

// Handling both success and failure
val handleBoth: URIO[Any, String] = anAttempt.fold(ex => s"Something bad happened: $ex", value => s"Length of the string: $value")

// Effectful fold
val handleBothZIO: URIO[Any, String] = anAttempt.fold(ex => ZIO.succeed(s"Something bad happened: $ex"), value => ZIO.succeed(s"Length of the string: $value"))
```

### Conversions between Option/Try/Either to ZIO ###
```
// Try -> ZIO
val aTryToZIO: Task[Int] = ZIO.fromTry(Try(42 / 0))

// Either -> ZIO
val anEither: Either[Int, String] = Right("Success")
val anEitherToZIO: IO[Int, String] = ZIO.fromEither(anEither)

// ZIO -> 
val anAttempt: ZIO[Any, Throwable, Int] = ZIO.attempt(42)
val eitherZIO: ZIO[Any, Nothing, Either[Throwable, Int]] = anAttempt.either

// Reverse: ZIO with Either as the value channel -> ZIO with left side of either as error channel
val anAttempt_v2: ZIO[Any, Throwable, Int] = eitherZIO.absolve

// Option -> ZIO
val anOption: ZIO[Any, Option[Nothing], Int] = ZIO.fromOption(Some(42))
```

## Errors vs Defects ##
- Errors - failures that are present in the ZIO type signature ("checked" errors)
- Defects - failures that are unrecoverable, unforeseen and not present in the ZIO type signature. Eg - `ZIO.succeed(1 / 0)`
- `ZIO[R, E, A]` can finish with `Exit[E, A]` which consists of -
    - `Success[A]` containing a value
    - `Cause[E]` which can be either of the following -
        - `Fail[E]` containing the error
        - `Die(t: Throwable)` an unforeseen `Throwable` <- these are the defects
- Exposing the cause -
```
val failedInt: ZIO[Any, String, Int] = ZIO.fail("I failed!")
val failureCauseExposed: ZIO[Any, Cause[String], Int] = failedInt.sandbox
val failureCauseHidden: ZIO[Any, String, Int] = failureCauseExposed.unsandbox

// fold/foldZIO with cause
val foldedWithCause = failedInt.foldCause(
    cause => s"this failed with ${cause.defects}", 
    value => s"this succeeded with $value"
)
```

### Good Practise ###
- At a lower level, your "errors" should be treated.
- At a higher level, you should hide "errors" and assume they are unrecoverable.

### Turning an error into defect (i.e. unrecoverable) ###
- `orDie` method swallows the error channel and converts it into Nothing i.e. all the errors become defects.
```
def callHTTPEndpoint(url: String): ZIO[Any, IOException, String] =
    ZIO.fail(new IOException("No internet"))

val endpointCallWithDefects: ZIO[Any, Nothing, String] = 
    callHTTPEndpoint("google.com").orDie
```

- Refining the error channel - narrowing the error channel and consider the remaning errors as defects
```
def callHTTPEndpointWideError(url: String): ZIO[Any, Exception, String] =
    ZIO.fail(new IOException("No internet"))

def callHTTPEndpoint_v2(url: String): ZIO[Any, IOException, String] =
    callHTTPEndpointWideError(url).refineOrDie[IOException] {
        case e: IOException => e
        case _: NoRouteToHostException => new IOException("no route to host to $url")
    }
```

- Unrefining the error channel - surface out the exception in the error channel
```
val endpointCallWithError = endpointCallWithDefects.unrefine {
    case e => e.getMessage
}
```

- If value channel is an `Option`, we can move the `None` case to the error channel -
```
val zioa: ZIO[Any, String, Option[Int]] = ???
val ziob: ZIO[Any, Option[String], Int] = zioa.some
```