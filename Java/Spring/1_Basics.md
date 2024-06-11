## Maven Commands
- If maven is not installed in your system, use maven wrapper files i.e. `mvnw`, otherwise use the installed maven commands i.e. `mvn`.
- Compile and test - `mvnw clean compile test`
- Packaging - `mvnw package`
- Run spring boot app - `mvnw spring-boot:run`

> [!CAUTION]
> Do not use the `src/main/webapp` directory if your application is packaged as a JAR. It only works with WAR packaging. It is silently ignored by most build tools if you generate a JAR.

## Create Spring Boot Application
- `@SpringBootApplication` signifies the spring boot app starting point.
```
@SpringBootApplication
public class MyApp {

    public static void main(String[] args) {
        SpringBootApplication.run(MyApp.class, args);
    }

}
```

## Create a REST Controller
- `@RestContoller` sets up the rest controller
- `@GetMapping` handles the HTTP GET requests which exposes `"/"` path that returns `Hello World!`
```
import org.springframework.web.bind.annotation.*;

@RestController
public class SimpleRestContoller {

    @GetMapping("/")
    public String hello() {
        return "Hello World!";
    }

}
```

## Application Properties
- `@Value` reads the required config from `application.properties` -
```
@RestController
public class SimpleRestController {

    @Value("${user.name}")
    private String userName;

}
```

