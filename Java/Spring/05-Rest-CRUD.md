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

