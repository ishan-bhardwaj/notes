## Class Based Access Privileges

- A method can access the `private` fields of the object it is invoked on.
- A method of a class can access `private` fields of any other object of the same class -

    ```
    class Employee {
      private int id;
      // other fields

      public boolean equals(Employee other) {
          return id == other.id;                // accesses private field of 'other'
      }
    }

    if (harry.equals(boss)) { ... }               // valid
    ```

  - Legal because `boss` is an `Employee` and the method belongs to the same class.
  - This enables methods like `equals`, `compareTo`, or `copy` constructors to work efficiently.

## Private Methods

- Instance fields are made `private` to protect data.
- Similarly, helper methods (internal computations) are often not part of the public interface.
- Reasons to make methods private -
  - They are implementation details.
  - May require a special protocol or calling order.
  - Not meant to be used outside the class.
