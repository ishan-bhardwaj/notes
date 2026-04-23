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

## Magic / Dunder Methods

- Functions that start and end with double underscores are called dunder functions.

```
class Garage:
    def __init__(self, cars):
        self.cars = cars

    # len(ford)
    def __len__(self):
        return len(self.cars)
    
    # ford[i]
    def __getitem__(self, i):
        return self.cars[i]

    # Used in debugging
    # code-oriented description
    def __repr__(self):
        return f'<Garage {self.cars}>'    

    # print(ford)
    # user-oriented description
    def __str__(self):
        return f'<Garage with {len(self)} cars>'
```

- `__getitem__` enables for-loop -
```
ford = Garage(['Fiesta', 'Focus'])

for car in ford:
    print(car)
```

- If `__str__` is not present, then `print` calls `__repr__`.

