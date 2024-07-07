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

- `Behaviors.receive` is more general API to create actors. It has two input parameters - `context` of type `ActorContext` and `message` of type `T`. The `context` is created alongside with the actor which has access to variety of APIs, eg - logging -
```
object SimpleActorV2 {
    def apply(): Behavior[String] = Behaviors.receive { (context, message) => 
        context.log.info(s"[simple-actor] I have received a message: $message")
        Behaviors.same
    }
}
```

- Even more general way is to create an actor is using `setup` method -
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