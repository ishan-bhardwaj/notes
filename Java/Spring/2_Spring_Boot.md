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

