## Classes & Objects

- Class consists of states (variables) and behaviors (methods).
```
class Student {

    // Variables
    int id;
    String name;
    String gender;

    // Methods
    boolean updateProfile(String newName) {
        name = newName;
        return true;
    }

}
```

- Objects are the instances of the class.
```
// Creating a new Student object
Student s = new Student();

// Setting student's state - not a good practise, use constructors instead
s.id = 1000;
s.name = "John";
s.gender = "Male"; 

// Calling methods
s.updateProfile("Doe");
```

## Variables & Constants

- Variables - can be updated, eg - 
```
int count = 0;
count = 5;
```

> [!WARNING]
> Such assignment statements cannot appear at class-level. They can appear inside class memebers like methods, constructors etc.

- Constants - cannot be updated.


