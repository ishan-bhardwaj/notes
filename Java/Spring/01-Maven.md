## Maven Commands
- If maven is not installed in your system, use maven wrapper files i.e. `mvnw`, otherwise use the installed maven commands i.e. `mvn`.
- Compile and test - `mvnw clean compile test`
- Packaging - `mvnw package`
- Two options for running the app -
    1. Use `java -jar myapp.jar`
    2. Use Spring Boot Maven plugin - `mvnw spring:boot run`

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

## Spring Boot Starter Parent
- Spring Boot provides a "Starter Parent" which provide maven defaults like default compiler level, UTF-8 source encoding etc.
```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.0.0-RC1</version>
    <relativePath/>     <!-- lookup parent from repository -->
</parent>
```
- For the rest of `spring-boot-starter-*` dependencies, no need to list version because they will be inherited from the starter parent -
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
- Spring boot plugin is used to run commands like `mvn spring-boot:run` -
```
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        <plugin>
    <plugins>
</build>    
```

>[!TIP]
> To override a default (eg Java version), set as a property -
> ```
> <properties>
>   <java.version>17</java.version>
> </properties>
> ```

## Spring Boot Dev Tools
- `spring-boot-devtools` automatically restarts your application when code is updated i.e. Hot Reloading.
> [!TIP]
> Intellij community edition does not support DevTools by default. To make it work, we need to -
> 1. `Select Preferences > Build, Execution, Deployment > Compiler > Check Build project automatically`
> 2. `Select Preferences > Advanced Setting > Check Allow auto-make to start...`
