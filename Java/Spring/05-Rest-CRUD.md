# Rest CRUD APIs

- __Spring Boot Rest Controller__ -
  ```
  @RestController                                     // Adds Rest Support
  @RequestMapping("/test")
  public class TestController {
    
    @GetMapping("/hello")
    public String greet() {
      return "Hello World";
    }
  }
  ```


