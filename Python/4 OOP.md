# Object-Oriented Programming

```
class Student:
    def __init__(self, name, grade):             # constructor
        self.name = name
        self.grade = grade

    def average(self):
        return sum(self.grades) / len(self.grades)


# Instantiation
s1 = Student('John', [10, 20, 30, 40])
s1.name                   # 'John'
s1.grade                  # [10, 20, 30, 40]
s1.average()
```

- In `s1.average()`, python will automatically pass the object reference `s1` to `self` parameter in the method i.e. converted to - `Student.average(s1)`.

> [!TIP]
> Functions that start and end with double underscores are called dunder functions.
