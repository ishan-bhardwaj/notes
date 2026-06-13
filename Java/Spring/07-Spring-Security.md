# Spring Security

- Implemented using Servlet Flters in the backgroun -
    - Servlet Filters are used to pre-process / post-process web requests
    - Servlet Filters can route web requests based on security logic
- Two methods of securing an app -
    - Declarative -
        - Define application security constraints in config
            - All Java config - `@Configuration`
        - Provides separation of concerns between application code and security
    - Programmatic -
        - Spring Security provides an API for custom application coding
        - Provides greater customization for specific app requirements
- Enabling Spring Security -
    - `pom.xml` -
        ```
        <dependency>
            <groupId>com.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        ```

    - This will automatically secure all endpoints -
        - Default user name - `user`
        - Password - check console logs

- Override default user name and password in `application.properties` -
    ```
    spring.security.user.name=john
    spring.security.user.password=john@123
    ```

- Password storage - 
    - Format -`{id}encodedPassword`
    - Eg - `{noop}john@123`
    - Where `id` can be -
        - `noop` - plain text passwords
        - `bcrypt` - bcrypt password hashing
        - etc

- Security config -
    - Spring Boot will not use the user/password from the `application.properties` file 
    - Adding users, passwords and roles using `Basic Auth` -
    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;

    @Configuration
    public class DemoSecurityConfig {

        @Bean
        public InMemoryUserDetailsManager userDetailsManager() {

            UserDetails john = User.builder()
                                   .username("john")
                                   .password("{noop}john@123")
                                   .roles("EMPLOYEE", "MANAGER")
                                   .build();

            UserDetails mary = User.builder()
                                   .username("mary")
                                   .password("{noop}mary@123")
                                   .roles("EMPLOYEE")
                                   .build();

            return new InMemoryUserDetailsManager(john, mary);
        }
    }
    ```

## Restrict access based on roles

- Syntax -
    ```java
    requestMatchers("/api/employees")                       // can be multiple
        .hasRole("EMPLOYEE")                                // single role

    requestMatchers(HttpMethod.GET, "/api/employee")        // with http method
        .hasRole("EMPLOYEE")

    requestMatchers(HttpMethod.GET, "/api/employee/**")     // ** matches on all sub-paths
        .hasRole("EMPLOYEE")
    ```

- `hasAnyRole` - matching with any role in the list of roles
- Example - in `DemoSecurityConfig` -
    ```java
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity key) throws Exception {
        http.authorizeHttpRequests(configurer ->
            configurer
                .requestMatchers(HttpMethod.GET, "/api/employees", hasRole("EMPLOYEE"))
                .requestMatchers(HttpMethod.POST, "/api/employees", hasRole("MANAGER"))
                .requestMatchers(HttpMethod.DELETE, "/api/employees/**", hasRole("ADMIN"))
        )

        // use HTTP Basic authentication
        http.httpBasic(Customizer.withDefaults());

        // disable CSRF
        http.csrf(csrf -> csrf.disable());

        return http.build();
    }
    ```

## Cross-Site Request Forgery (CSRF)

- Spring Security can prevent against CSRF attacks
- Embed additional authentication data/token into all HTML forms
- On subsequent requests, web app will verify token before processing
- Primary use case is traditional web apps (HTML forms etc)
- When to use CSRF protection -
    - Recommended to use for normal browser web requests
    - You may disable CSRF protection when building REST APIs for non-browser clients
    - In general, not required for stateless REST APIs that use POST, PUT, DELETE and/or PATCH

## Database Support

- Follows Spring Security's predefined table schemas
- Develop SQL script to setup up database tables - `users` and `authorities` -
    ```
    CREATE TABLE users (
        username varchar(50) NOT NULL,
        password varchar(50) NOT NULL,
        enabled tinyint NOT NULL,

        PRIMARY KEY (username)

    ) ENGINE=InnoDB DEFAULT CHARSET=latin1;

    INSERT INTO users
    VALUES
    ('john', '{noop}john@123', 1),
    ('mary', '{noop}mary@123', 1);

    CREATE TABLE authorities (
        username varchar(50) NOT NULL,
        authority varchar(50) NOT NULL,
            
        UNIQUE KEY authorities_idx_1 (username, authority),

        CONSTRAINT authorities_ibfk_1
        FOREIGN KEY (username)
        REFERENCES users (username)
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1;

    INSERT INTO authorities
    VALUES
    ('john', 'ROLE_EMPLOYEE'),                  // Internally Spring Security user "ROLE_" prefix
    ('mary', 'ROLE_EMPLOYEE'),
    ('mary', 'ROLE_MANAGER');
    ```

- Add database support to maven POM file -
    ```
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
    ```

- Create JDBC properties file -
    ```
    spring.datasource.url=jdbc:mysql://localhost:3306/employee_directory
    spring.datasource.username=springstudent
    spring.datasource.password=springstudent
    ```

- Update Spring Security config to use JDBC -
    ```java
    @Configuration
    public class DemoSecurityConfig {

        @Bean
        public UserDetailsManager userDetailsManager(DataSource dataSource) {
            return new JdbcUserDetailsManager(dataSource);                      // no hard-coded users
        }

    }
    ```

## Bcrypt

- Performs one-way encrypted hashing - password is never decrypted
- Adds a random salt to the password for additional protection
- Includes support to defeat brute force attacks
- Syntax - `{bcrypt}encodedPassword`
    - Password column must be at least $68$ chars wide
        - `{bcrypt}` - $8$ chars
        - `encodedPassword` - $60$ chars
- Example with JPA/Hibernate - `DemoSecurityConfig.java` -
    ```java
    @Bean 
    public BCryptPasswordEncoder passwordEncoder() { 
        return new BCryptPasswordEncoder(); 
    } 
 
    //authenticationProvider bean definition 
    @Bean 
    public DaoAuthenticationProvider authenticationProvider(UserService userService) { 
        DaoAuthenticationProvider auth = new DaoAuthenticationProvider(); 
        auth.setUserDetailsService(userService);                    //set the custom user details service 
        auth.setPasswordEncoder(passwordEncoder());                 //set the password encoder - bcrypt 
        return auth; 
    }
    ```

## Custom Tables

- Update Spring Security Configuration -
    - Provide query to find user by username
    - Provide query to find autorities / roles by username
- Example -
    ```java
    @Bean
    public UserDetailsManager userDetailsManager(DataSource dataSource) {
        JdbcUserDetailsManager theUserDetailsManager = new JdbcUserDetailsManager(dataSource);

        theUserDetailsManager
            .setUsersByUsernameQuery("select user_id, pw, active from members where user_id=?");

        theUserDetailsManager
            .setAuthoritiesByUsernameQuery("select user_id, role from roles where user_id=?");

        return theUserDetailsManager;
    }
    ```
