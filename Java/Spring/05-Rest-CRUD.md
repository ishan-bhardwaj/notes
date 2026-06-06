# Rest CRUD APIs

- Maven dependency - `spring-boot-starter-webmvc`
- __Spring Boot Rest Controller__ -
  ```
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RestController;

  @RestController                                     // Adds Rest Support
  @RequestMapping("/test")
  public class TestController {
    
    @GetMapping("/hello")
    public String greet() {
      return "Hello World";
    }
  }
  ```

## JSON Data Binding

- Jackson handles data binding between JSON and Java POJO internally, by calling appropriate getter / setter methods.
- Spring Boot Starter Web automatically includes dependency for Jackson.

## Path Variables

```
@GetMapping("/students/{studentId}")
public Student getStudent(@PathVariable int studentId) {
  ...
}
```

- By default, `studentId` variable name must match with the path variable.

## Exception Handling

- Create custom error response class -
  ```
  public class StudentErrorResponse {
    private int status;
    private String message;
    private long timestamp;
    
    // constructors

    // getters/setters
  }
  ```

- Create custom student exception -
  ```
  public class StudentNotFoundException extends RuntimeException {
    public StudentNotFoundException(String message) {
      super(message);
    }
  }
  ```

- Update REST service to throw exception.
- Add exception handler method with `@ExceptionHandler` -
  - Exception handler will return a `ResponseEntity` (a wrapper for the HTTP response object).
  - `ResponseEntity` provides fine-grained control to specify HTTP status codes, headers and response body.
  - Accept `Exception` as input argument to handle all kinds of exceptions.
    ```
    @ExceptionHandler
    public ResponseEntity<StudentErrorResponse> handleException(StudentNotFoundException e) {
      StudentErrorResponse error = new StudentErrorResponse();

      error.setStatus(HttpStatus.NOT_FOUND.value());
      error.setMessage(e.getMessage);
      error.setTimeStamp(System.currentTimeMillis());

      return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    ```

## Global Exception Handling

- `@ControllerAdvice` -
  - pre-processes requests to controllers
  - post-processes responses to handle exceptions
    ```
    @ControllerAdvice
    public class StudentRestExceptionHandler {
      
      @ExceptionHandler
      public ResponseEntity<StudentErrorResponse> handleException(StudentNotFoundException e) {
        StudentErrorResponse error = new StudentErrorResponse();

        error.setStatus(HttpStatus.NOT_FOUND.value());
        error.setMessage(e.getMessage);
        error.setTimeStamp(System.currentTimeMillis());

        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
      }

    }
    ```

## Service Facade Pattern

```
@Service
public class EmployeeServiceImpl extends EmployeeService {
  // inject EmployeeDAO

  @Override
  public List<Employee> findAll() {
    return employeeDAO.findAll();
  }
}
```

- Best practise is to apply transactional boundaries at the service layer i.e. apply `@Transactional` on service methods.

## CRUD Mappings

- Post request -
  ```
  @PostMapping("/employees")
  public Employee addEmployee(@RequestBody Employee employee) {
  
    // setting id to 0/null to force inserting a new item, instead of updating
    employee.setId(0);                  // if id in int
    employee.setId(null);               // if id is integer

    Employee dbEmployee = employeeService.save(employee);

    return dbEmployee;
  }
  ```

- Put request - `@PutMapping`
- Delete request - `@DeleteMapping`
- Patch request -
  - Inject the helper class - JsonMapper - a helper class in the Jackson library for JSON processing.
  
  ```
  @PatchMapping("/employees/{employeeId}")
  public Employee patchEmployee(@PathVariable int employeeId, @RequestBody Map<String, Object> patchPayload) {
    Employee tempEmployee = employeeService.findById(employeeId);

    if (tempEmployee == null) {
      throw new RuntimeException("Employee id not found: " + employeeId);
    }

    // Apply the partial updates to the existing employee object
    Employee patchedEmployee = jsonMapper.updateValue(tempEmployee, patchPayload);

    return employeeService.save(patchedEmployee);
  }
  ```

- Delete request -
  ```
  @DeleteMapping("/employees/{employeeId}")
  public String deleteEmployee(@PathVariable String employeeId) {
    ...
  }
  ```

## Spring Data REST

- Provides REST CRUD implementations out-of-the-box - just by adding Spring Data REST to the pom file
  ```
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
  </dependency>
  ```

- Spring Data REST will scan your project for `JpaRepository`
- Expose REST APIs for each entity type for your `JpaRepository`
- By default, it will create endpoints based on the entity type -
  - Simple pluralized form -
    - First character of entity type is lowercase
    - Then just adds an "s" to the entity
  - Eg -
    - Entity type = `Employee`
    - Rest endpoints - `/employees`

- Customize endpoint base path in `application.properties` -
  ```
  spring.data.rest.base-path=/api
  ```

> [!NOTE]
> Spring Data REST only uses ID on the url, eg - `PUT /employees/{employeeId}`

- Specifying custom path - 
  ```
  @RepositoryRestResource(path="members")                                           // Rest api - /members
  public interface EmployeeRepository extends JpaRepository<Employee, Integer> {}
  ```

## Pagination

- By default, Spring Data REST uses page size = $20$
- Navigating using query param -
  ```
  /employees?page=0           // pages are 0-based
  /employees?page=1
  ```

- Configs -
  - `spring.data.rest.default-page-size` - page size
  - `spring.data.rest.max-page-size` - max page size

## Sorting

- Sort by last name (ascending by default) - `/employees?sort=lastName`
- Sort by first name descending - `/employees?sort=firstName,desc`
- Sort by last name, then first name, ascending - `/employees?sort=lastName,firstName,asc`

## HATEOAS

- HATEOAS = Hypermedia As The Engine Of Application State
- Uses Hypertext Application Language (HAL) data format
- Spring Data REST endpoints are HATEOAS compliant
- Hypermedia-driven sites provide information to access REST interfaces
- Eg - 
  ```
  GET /employees/3

  // Response
  {
    "firstName": "John",
    "lastName": "Doe",
    "age": 25,
    "_links": {
      "self": {
        "href": "http://localhost:8080/employees/3"
      },
      "employee": {
        "href": "http://localhost:8080/employees/3"
      }
    }
  }
  ```

- For a collection, metadata includes page size, total elements, pages etc
