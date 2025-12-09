# Object-Oriented Programming

```
class Student:
    def __init__(self, name, grade):             # constructor
        self.name = name
        self.grade = grade

    def average(self):
        return sum(self.grades) / len(self.grades)


# Instantiation
john = Student('John', [10, 20, 30, 40])    
```

> [!TIP]
> Functions that start and end with double underscores are called dunder functions. 