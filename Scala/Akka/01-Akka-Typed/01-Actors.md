# Akka Actors
- Actors are objects that we can't access directly, but only send messages to.
- Behavior defines what an actor will do when it receives a message. `Behavior[String]` will receive messages for type `String` only -
```
import akka.actor.typed.{ActorSystem, Behavior}
import akka.actor.typed.scaladsl.Behaviors

object SimpleActor {
    def apply(): Behavior[String] = Behaviors.receiveMessage { (message: String) => 
        println(s"[simple-actor] I have received a message: $message")
        // perform some computation...
        // Define new behavior for the next message
        Behaviors.same
    }
}

def demoSimpleActor(): Unit = {
    val actorSystem = ActorSystem(SimpleActor(), "SimpleActorSystem")

    actorSystem ! "Hello World!"

    Thread.sleep(1000)
    actorSystem.terminate()
}
```

- The `receiveMessage` inputs a message and returns a new behavior for the next message.

> [!TIP]
> `Behavior.same` keeps the same behavior for the next message.

> [!WARNING]
> `actorSystem ! 20` will not work in above example. The `simpleActorBehavior` only accepts `String` type.

- `Behaviors.receive` is more general API to create behaviors. It has two input parameters - `context` of type `ActorContext` and `message` of type `T`. The `context` is created alongside with the actor which has access to variety of APIs, eg - logging -
```
object SimpleActorV2 {
    def apply(): Behavior[String] = Behaviors.receive { (context, message) => 
        context.log.info(s"[simple-actor] I have received a message: $message")
        Behaviors.same
    }
}
```

- Even more general way is to create an actor is using `setup` method which lets us define actor specific data & methods -
```
object SimpleActorV3 {
    def apply(): Behavior[String] = Behaviors.setup { context => 
        // actor "private" data and methods, behaviors etc
        
        // At the end, define the first behavior that the actor will get on the FIRST message
        Behaviors.receiveMessage { (message: String) => 
            println(s"[simple-actor] I have received a message: $message")
            Behaviors.same
        }
    }
}
```

> [!TIP]
> Message types must be immutable and serializable. To achieve this -
> - Use case classes / objects.
> - Use flat type hierarchy, eg - 
>   ```
>   trait PaymentStatus
>   case object PaymentSucceeded extends PaymentStatus
>   case object PaymentFailed extends PaymentStatus
>   ```

## Managing Actor State

- (Bad Practise) Using mutable variables to hold the state -
```
object StatefulWordCounter {

    def apply(): Behavior[String] = Behaviors.setup { context =>
        var total = 0

        Behaviors.receiveMessage { message =>
            val newCount = message.split(" ").length
            totalCount += newCount
            context.log.info(s"Total count: $totalCount")
            Behaviors.same
        }
    }

}
```

- (Good Practise) Stateless implementation -
```
object StatelessWordCounter {

    def apply(): Behavior[String] = countWords(0)

    def active(totalCount: Int): Behavior[String] = Behaviors.receive { (context, message) =>
        val newCount = message.split(" ").length
        context.log.info(s"Total count: ${totalCount + newCount}")
        active(totalCount + newCount)
    }

}
```

> [!NOTE]
> `active(totalCount + newCount)` is NOT a recursive call.

## Child Actors

- An actor can create multiple child actors, which can then further create child actors - forming a tree like structure, such that every actor can be identified with filesystem like "path".

> [!TIP]
> Root of the actor hierarchy is _guardian_ actors which are created with the `ActorSystem`.
>
> `ActorSystem` creates -
>   - the top-level (root) guardian, with children -
>       - system guardian (for akka internal messages)
>       - user guardian (for our custom actors)

> [!TIP]
> All our actors are child actors of the user guardian.

- `context.spawn` - create child actors and returns `ActorRef[T]` i.e. reference to the child actor that can be used to interact with it, where `T` is the type of messages that child actor can receive.

```
object Parent {
    trait Command
    case class CreateChild(name: String) extends Command
    case class TellChild(name: String) extends Command

    def apply(): Behavior[Command] = Behaviors.receive { (context, message) =>
        message match {
            case CreateChild(name) =>
                context.log.info(s"[parent] creating child with name: $name")
                val childRef: ActorRef[String] = context.spawn(Child(), name)
                active(childRef)
        }
    }

    def active(child: ActorRef[String]): Behavior[Command] = Behaviors.receive { (context, message) =>
        message match {
            case TellChild(msg) =>
                context.log.info(s"[parent] sending message $msg to child")
                childRef ! message
                Behaviors.same
            case _ =>
                context.log.info(s"[parent] command not supported")
                Behaviors.same
        }
    }

}

object Child {
    def apply(): Behavior[String] = Behaviors.receive { (context, message) =>
        context.log.info(s"[child] received message: $message")
        Behaviors.same
    }
}
```

- When initializing the `ActorSystem(behavior, name)`, the `behavior` argument is the behavior of the user guardian. Therefore, it's a good practise to setup a user guardian behavior separately.
- Example -
```
def demoParentChild(): Unit = {
    import Parent._

    val userGuardianBehavior: Behavior[Unit] = Behaviors.setup { context =>
        // setup all the important actors in your app
        val parent = context.spawn(Parent(), "parent")

        // setup initial interactions between the actors
        parent ! CreateChild("child")
        parent ! TellChild("Hello")

        // user guardian usually has no behavior of its own, but they can have
        Behaviors.empty
    }

    val system = ActorSystem(userGuardianBehavior, "DemoParentChildApp")
    Thread.sleep(1000)
    system.terminate()
}
```

> [!TIP]
> `context.self` returns `ActorRef` of its own actor.
>
> It can be used to get actor hierarchy path (`context.self.path`) or actor's name (`context.self.path.name`).

## Stopping Actors

- To stop an actor - `Behaviors.stopped`
- Optionally, we can pass `() => Unit` to clear up resources after the actor is stopped.

```
object ChildActor {
    def apply(): Behavior[Int] = Behaviors.receive { (context, message) =>
        context.log.info(s"Received: $message")
        if (message == 0) Behaviors.stopped(() => context.log.info("Stopped!"))
        else Behaviors.same
    }
}
```

- If we send any messages to an actor after stopping it, the messages will be sent to the dead letter.

> [!TIP]
> `DeadLetter` is a special actor which is spawned alongside the `ActorSystem` and is the recipient of all the messages that could not find their destinations.

- __`receiveSignal`__ 
    - Used for extending the `Behavior` with special messages like `PostStop` etc.
    - Another way of freeing up resources after the actor is stopped -
    - Signature -
    ```
    def receiveSignal(onSignal: PartialFunction[(ActorContext[T], signal), Behavior[T]]): Behavior[T]
    ```

    - Example -
    ```
    object ChildActor {
        def apply(): Behavior[Int] = Behaviors.receive[Int] { (context, message) =>
            context.log.info(s"Received: $message")
            if (message == 0) Behaviors.stopped
            else Behaviors.same
        }
        .receiveSignal {
            case (context, PostStop) =>
                // clean up resources
                context.log.info("Stopped!")
                Behaviors.same                  // not used in case of stopping

                // or
                apply()                         // Switch the behavior back to the apply()
        }
    }
    ```

- __Stopping Child actors from Parent context__ -
```
context.stop(childRef)
```

> [!NOTE]
> When using `receiveSignal`, the `Behaviors.receive[Int]` has to be typed.

> [!WARNING]
> `context.stop()` can receive any `childRef`, but the `childRef` must be the child of the parent, otherwise it will throw runtime exception.
>
> Therefore, also we cannot call - `context.stop(context.self)`.

## Watching Actors

- Watching an actor means that the watcher will receive a message (`Terminated`) when the actor it is watching dies.
```
context.watch(actorRef)
```

- It can be used for any `ActorRef`, unlike stopping actors.
- Example -
```
receiveSignal {
    case (context, Terminated(diedActorRef)) =>
        ???
}
```

- To deregister an actor from the watch - `context.unwatch(actoRef)`

- Properties -
    - `Terminated` signal gets sent even if the watched actor is already dead at registration time.
    - Registering multiple times may/may not generate multiple terminated signals.
    - Unwatching will not process terminated signals even if they have already been enqueue in your mailbox.

> [!WARNING]
> Never pass mutable state or the context reference to other actors.

## Supervision

- `ActorSystem` kills the actor, if it throws an exception. `Terminated` signal is sent to the watchers.
- We can define `SupervisorStrategy` for the child actors -
```
// In parent
val childBehavior = Behaviors.supervise(WordCounter())
                    .onFailure[RuntimeException](SupervisorStrategy.restart)

val child = context.spawn(childBehavior, "wordCounter")
```

- Since the actor is restarted in this example, `Terminated` signal will not be sent to the watchers.
    - Same actor instance is simply restarted with its state reset.

- Supervisor strategies - `resume`, `restart`, `stop` (default), `restartWithBackoff`.
- `restartWithBackoff(1.second, 1.minute, 0.2)` -
    - `1.second` - actor restart is retried after 1 second, then 2 seconds, then 4 seconds, 8 seconds and so on.
    - `1.minute` - retries happen upto 1 minute.
    - `0.2` - randomness factor added to the first parameter (`1.second`) to make sure multiple actors do not retry at the same time. 

- Applying different strategies for different exception types -
```
val differentStrategies = Behaviors.supervise(
        Behaviors.supervise(WordCounter())
        .onFailure[NullPointerException](SupervisorStrategy.restart)
    )
    .onFailure[RuntimeException](SupervisorStrategy.restart)
```

> [!NOTE]
> The order of exception handling works from inside to outwards. Therefore, always place the most specific exception handlers inside, and the most general exception handlers outside.



