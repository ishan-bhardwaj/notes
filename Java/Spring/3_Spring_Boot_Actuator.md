## Spring Boot Actuator
- Exposes endpoints to monitor and manage your application.
- Simply add the dependency to your POM file and REST endpoints are automatically added to your application.
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
- Endpoints are prefixed with `/actuator`, eg - `/health` provides status of your application.
- By default, only `/health` is exposed. The `/info` endpoint can provide more information about your application.
- There are 10+ Spring Boot Actuator enpoints like -
    - `/auditevents` - Audit events for your application
    - `/beans` - List of all beans registered in the Spring application context
    - `/mappings` - List of all `@RequestMapping` paths

> [!NOTE]
> To expose `/info`, make the following changes to `application.properties` file -
> ```
> management.endpoints.web.exposure.include=health,info
> management.info.env.enabled=true
> info.app.name=My App
> info.app.description=Simple fun app
> info.app.version=1.0.0
> ```
> Note that properties starting with `info` will be used by `/info` endpoint.

> [!TIP]
> To expose multiple actuator endpoints over HTTP, either list the individual endpoints with a comma-delimited list or use `*` to expose all endpoints -
> `management.endpoints.web.exposure.include=*` 




