## Final Instance Fields

- An instance field can be declared as `final`.
- Rules for `final` fields -
  - Must be initialized during object construction (in every constructor).
  - Cannot be reassigned after the constructor completes.

- Example -

```
class Employee {
    private final String name; // cannot change after construction
}
```

- Use-cases -
  - Useful for fields that never change -
    - Primitive types (int, double, etc.)
    - Immutable classes (e.g., String, LocalDate)
- Benefits -
  - Guarantees immutability of the reference.
  - Improves clarity and safety of your class design.

- Final Fields in Mutable Classes -
  - `final` applies to the reference, not the object itself.
  - Example -

  ```
  private final StringBuilder evaluations;

  evaluations = new StringBuilder(); // initialized in constructor
  ```

  - You cannot reassign evaluations to a new object, but the object itself can be mutated -

  ```
  void giveGoldStar() {
    evaluations.append(LocalDate.now() + ": Gold star!\n");
  }
  ```

- A `final` field can be `null`, but once set, it cannot change -

```
name = n != null && n.length() == 0 ? null : n
```
