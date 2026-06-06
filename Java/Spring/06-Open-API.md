# Open API

- [Springdoc](springdoc.org) provides a Swagger web UI for accessing endpoints
- Maven dependency -
    ```
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>x.y.z</version>
    </dependency>
    ```

- By default, Swagger UI is available at <http://localhost:8080/swagger-ui/index.html>
- Docs for API endpoints are available as JSON or YAML -
    - Useful for integration with other dev tools
    - Client SDK generation, API mocking, contract testing etc
- By default, JSON docs are available at - <http://localhost:8080/v3/api-docs>
- YAML docs are available at - <http://localhost:8080/v3/api-docs.yaml>
- Configure custom path in `applicaiton.properties` -
    ```
    springdoc.swagger-ui.path=/my-fun-ui.html         // swagger UI path
    springdoc.api-docs.path=/my-api-docs              // api docs path          
    ```

> [!WARNING]
> Open API / Swagger does not currently work with Spring Data REST in Spring Boot 4
>
> It does work with regular `@RestController` based projects

