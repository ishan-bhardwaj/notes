## Spring Container
- Spring Container functions - 
    - Create and manage objects (Inversion of Control)
    - Inject object dependencies (Dependency Injection)
- Inversion of Control - The approach of outsourcing the construction and management of objects.
- Dependency Inversion - the client delegates the responsibility of providing its dependencies to another object.
- Ways to configure Spring Container -
    - Java Source Code
    - Java Annotations
    - XML Configuration file (legacy)
- Recommended Dependency Injection Types -
1. Constructor Injection -
    - Use this when you have required dependencies
    - Recommended by the spring.io development team
2. Setter Injection - 
    - Use this when you have optional dependencies
    - If dependency is not provided, we can specify reasonable default logic

## AutoWiring
- Used for dependency injection
- Spring looks for the class that matches by type - class or interface - by scanning the `@Component` annotations and inject it automatically if that class implements the desired interface.
- Example -
```
public interface Coach {
    String getWorkout();
}

@Component
public class CricketCoach implements Coach {
    @Override
    public String getWorkout() {
        return "Run for 15 mins";
    }
}
```
```
@RestController
public class SimpleController {
    private Coach myCoach;

    @Autowired
    public SimpleController(Coach theCoach) {
        myCoach = theCoach;
    }

    @GetMapping("/workout")
    public String getWorkout() {
        return myCoach.getWorkout();
    }
}
```
- `@Component` marks the class as a Spring Bean which is just a regular Java class managed by Spring. It also makes the bean available for dependency injection.
- `@Autowired` annotation tells Spring to inject a dependency.
> [!NOTE]
> If you only have one constructor then `@Autowired` on constructor is optional.

## Annotations
- `@SpringBootApplication` is composed of the following annotations -
    - `@EnableAutoConfiguration` enables Spring Boot's auto-configuration support
    - `@ComponentScan` enables component scanning of current package (where the main `SpringBootApplication` exists) and its sub-packages recursively
    - `@Configuration` provides ability to register extra beans with `@Bean` or import other configuration class
> [!TIP]
> To explicitly list base packages to scan -
> ```
> @SpringBootApplication(
>   scanBasePackages = {
>       "com.example.myapp",
>       "com.example.utils"
> })
> ```

## Setter Injection
- Create setter method(s) in our class for injections and then configure the dependency injection with `@Autowired` annotation.
- Example -
```
@RestController
public class SimpleController {
    private Coach myCoach;

    @Autowired
    public void setCoach(Coach theCoach) {
        myCoach = theCoach;
    }
}
```
> [!TIP]
> We can inject dependencies by calling ANY method in our class by simply using `@Autowired` annotation.
> ```
> @RestController
> public class SimpleController {
>     private Coach myCoach;
>
>     @Autowired
>     public void setCoach(Coach theCoach) {
>         myCoach = theCoach;
>     }
> }
> ```

## Qualifiers
- For a given interface, if we have multiple implementations then the Spring will get confused on which one to inject. To overcome this issue, we use `@Qualifier` annotation and specify the bean id which is same as the class name where the first character is lowercase.
- Example -
```
@RestController
public class SimpleController {
    private Coach myCoach;

    @Autowired
    public SimpleController(@Qualifier("cricketCoach") Coach theCoach) {
        myCoach = theCoach;
    }
}
```
- `@Qualifier` also works with setter injection.

## Primary
- Instead of using `@Qualifier` and specifying the bean name, we can also specify the "primary" bean to use in case of multiple implementations by annotating that implementation with `@Primary`.
- Example -
```
@RestController
public class SimpleController {
    private Coach myCoach;

    @Autowired
    public SimpleController(Coach theCoach) {
        myCoach = theCoach;
    }
}

public interface Coach {
    String getWorkout();
}

@Component
@Primary
public class CricketCoach implements Coach {
    @Override
    public String getWorkout() {
        return "Run for 15 mins";
    }
}
```
> [!NOTE]
> We can have only one class annotated as `@Primary` for multiple implementations. If we mark more that one class as `@Primary`, we'll get an error - `more than one 'primary' bean found among candidates: [.....]`

> [!TIP]
> If we mix both `@Primary` and `@Qualifier`, the `@Qualifier` will have higher priority.

## Field Injection
- Type of dependency injection that inject dependencies by setting field values on our class directly (even private fields) and it's accomplished by using Java Reflection.
- It is not recommended by the spring.io team because it makes the code harder to unit test.
- Example -
```
@RestController
public class SimpleController {
    @Autowired
    private Coach myCoach;  // No need for constructors or setters
}
```

## Lazy Initialization
- By default, when the application starts, all beans are initialized, but we can also specify lazy initialization using `@Lazy` annotation on the class.
- Then a bean will only be initialized in the following cases -
    - needed for dependency injection
    - explicitly requested
- Setting the lazy initialization for all the beans, set the below property in `application.properties` file -
```
spring.main.lazy-initialization=true
```

## Bean Scope
