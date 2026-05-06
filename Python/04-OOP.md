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
> `__init__` method gets called _after_ the object is created.

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

## Inheritance

```
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

    def weekly_salary(self):
        return self.salary * 40


class Developer(Employee):
    def __init__(self, name, salary, department):
        super().__init__(name, salary)                 // call to parent constructor
        self.department = department
    
    @property
    def develop():
        return 'Developing a new feature!'


john = Developer('John Doe', 100, 'IT')
john.weekly_salary()
john.develop()
```

- `@property` decorator allows no-args methods to be used as fields i.e. `john.develop` is valid code.

- `@classmethod` decorator passes object's class into method argument -
    ```
    class Foo:
        @classmethod
        def hi(cls):
            print(cls.__name__)         # Foo

    my_obj = Foo()
    my_obj.hi()       
    ```

- Static methods -
    ```
    class Bar:
        @staticmethod
        def hi():
            print('inside static method')

    Bar.hi()

    my_obj = Bar()
    my_obj.hi()                         # also works!
    ```

- Prefer `@classmethod` over `@staticmethod` in inheritance -
    ```
    class FixedFloat:
        def __init__(self, amount):
            self.amount = amount
        
        def __repr__(self):
            return f'<FixedFloat f{self.amount:.2f}>'

        @classmethod
        def from_sum(cls, value1, value2):
            return cls(value1 + value2)

        @staticmethod
        def from_static_sum(value1, value2):
            return FixedFloat(value1 + value2)


    class Euro(FixedFloat):
        def __init__(self, amount):
            super().__init__(amount)
            self.symbol = '€'

        def __repr__(self):
            return f'<Euro {self.symbol}{self.amount:.2f}>'

    
    money1 = Euro.from_sum(16.758, 9.999)
    print(money1)                                                # <Euro €26.76>

    money2 = Euro.from_static_sum(16.758, 9.999)
    print(money2)                                                # <FixedFloat 26.76>
    ```
