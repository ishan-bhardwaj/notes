# Dependency Injection #
- `ZIO.service[EmailService]` - returns an `EmailService` "service" with type - `ZIO[EmailService, Nothing, EmailService]` i.e. it requires an `EmailService`, throws `Nothing` and returns an `EmailService` instance.
- We can then create a program which requires the `EmailService` as dependency -
```
val program = for {
    emailService <- ZIO.service[EmailService]
    _ <- emailService.sendEmail()
} yield ()
```
- Finally, we can inject the `EmailService` dependency to the program -
```
program.provideLayer(new EmailService)
```
- Benefits -
    - We only need to worry about injecting the dependencies at time of the call.
    - All ZIOs requiring this dependency will use the same instance, hence no memory leaks.
    - Can use different instances of the same type for different needs (like testing).

# ZLayer #
- While creating ZLayers, the best practise is to write "factory" methods exposing layers in the companion objects of the services.
- Creating a ZLayer that doesn't require any dependency -
```
class ConnectionPool(n: Int)

object ConnectionPool {
    val live(n: Int): ZLayer[Int, Nothing, ConnectionPool] = 
        ZLayer.succeed(new ConnectionPool(n))
}
```
- Layers that require a dependency can be built with `ZLayer.fromFunction` which can automatically fetch the function arguments and place them into the ZLayer's dependency/environment type argument.
```
class UserDatabase(pool: ConnectionPool)

object UserDatabase {
    val live: ZLayer[ConnectionPool, Nothing, UserDatabase] =
        ZLayer.fromFunction(UserDatabase.create _)
}
```
- Composing layers -
1. Vertical Composition (`>>>`) -
    ```
    val completeDatabaseLayer: ZLayer[Any, Nothing, UserDatabase] = connectionPoolLayer >>> databaseLayer
    ```
2. Horizontal Composition (`++`) -
    - Eg - there is a `UserRegister` service that requires two dependencies - `UserDatabase`, `EmailService`
    ```
    val userRegisterLayer: ZLayer[Any, Nothing, UserDatabase with EmailService] = databaseLayerFull ++ emailServiceLayer
    ```
    - The error channel is combined by returning the lowest common ancestor of both.

- We can provide all the dependencies in the `provide` or `provideLayer` method (in any order) and ZIO will take care of wiring them together as needed.
- ZIO will also tells us if any required dependency is missing or if there are multiple layers of the same type - at compilation time.
- To get the dependency graph, inject this layer also - `ZLayer.Debug.tree`
- To get the dependency graph on browser UI, inject this layer - `ZLayer.Debug.mermaid`
- To directly create the ZLayer by defining the required service and its services -
```
val subscriptionLayer: ZLayer[Any, Nothing, UserSubscription] =
    ZLayer.make[UserSubscription](layer1, layer2, layerN)
```
- Passthrough - if we want to inject the dependency (environment channel) into the value channel - 
```
val dbWithPoolLayer: ZLayer[ConnectionPool, Nothing, ConnectionPool with UserDatabase] =
    UserDatabase.live.passthrough
```
- Service - take a dependency and expose it as a value to further layers -
```
val dbService = ZLayer.service[UserDatabase]
```
- Launch - creates a ZIO that uses the services and never finishes -
```
val subscriptionLaunch: ZIO[EmailService with UserDatabase, Nothing, Nothin] = UserSubscription.live.launch
```
- Note that the value channel is `Nothing` because the ZIO will never finish.
- Memoization - by default, the dependency layer is instantiated once and used everywhere. To create a new instance - `UserDatabase.live.fresh`
- ZIO provided services - Clock, Random, Systen, Console
```
val getTime = Clock.currentTime(TimeUnit.SECONDS)
val randomValue = Random.nextInt
val sysVariable = System.env("JAVA_HOME")
val printlnEffect = Console.printLine("Hello World")
```


