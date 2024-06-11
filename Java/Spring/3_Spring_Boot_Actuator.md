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
    - `/threaddump` - List of all threads running in your application

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

>[!TIP]
> We can also exlude the endpoints with a comma-delimited list -
> `management.endpoints.web.exposure.exclude=*` 

## Securing Actuator Endpoints
- Enable Spring Security -
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
- Now when you access `/actuator/beans`, Spring Security will prompt for login. Default user name is `user` and check console logs for the password.
- To override default user name and password, make these changes in `application.properties` -
```
spring.security.user.name=john
spring.security.user.password=Doe@123
```

> [!WARNING]
> Spring Security doesn't secure the `/health` and `/info` endpoint. But we can exclude them -
> `management.endpoints.web.exposure.include=health,info`

