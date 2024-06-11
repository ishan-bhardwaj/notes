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

## Configuring Spring Boot Server -
- Core properties -
```
# Log Levels severity mapping
logging.level.org.springframework=DEBUG
logging.level.org.hibernate=TRACE
logging.level.com.example=INFO

# Log file name
logging.file.name=my-app.log
logging.file.path=C:/projects/logs
```
- Web properties -
```
# HTTP server port
server.port=8000

# Context path of the application, default is /
server.servlet.context-path=/my-app

# Default HTTP session timeout (15 mins), default is 30 mins
server.servlet.session.timeout=15m
```

> [!NOTE]
> `server.servlet.context-path=myapp` property is used to define the api parent path. Now, we can access the APIs at thi path -
> `http://localhost:8000/myapp/health`

- Actuator properties -
```
# Endpoints to include by name or wildcard
management.endpoints.web.exposure.exclude=*

# Base path for actuator endpoints
management.endpoints.web.base-path=/actuator
```

- Security Properties -
```
# Default username and password
spring.security.user.name=admin
spring.security.user.password=topsecret
```

- Data Properties -
```
# JDBC URL of the database
spring.datasource.url=jdbc://localhost:3306/ecommerce

# Login username and password of the database
spring.datasource.username=admin
spring.datasource.password=admin@123
```