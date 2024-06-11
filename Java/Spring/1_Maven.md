## Maven Commands
- If maven is not installed in your system, use maven wrapper files i.e. `mvnw`, otherwise use the installed maven commands i.e. `mvn`.
- Compile and test - `mvnw clean compile test`
- Packaging - `mvnw package`
- Run spring boot app - `mvnw spring-boot:run`

> [!TIP]
> `Spring Boot Starters` provide a curated list of maven dependencies grouped together which reduces the amount of maven configuration. It contains spring-web, spring-webmvc, hibernate-validator, json, tomcat etc.
> ```
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-starter-web</artifactId>
> </dependency>
> ```

> [!CAUTION]
> Do not use the `src/main/webapp` directory if your application is packaged as a JAR. It only works with WAR packaging. It is silently ignored by most build tools if you generate a JAR.

