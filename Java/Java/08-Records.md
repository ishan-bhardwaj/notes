# Records

- A record is a special form of a class designed to model immutable data.
- Its state -
  - is fixed at construction time
  - is publicly readable
- Records are ideal for data carriers whose entire state is represented by a fixed set of fields.

- Declaring a Record - 
  - `record Point(double x, double y) { }`
  - This declaration automatically creates -
    - Instance Fields (called Components) -
    ```
    private final double x;
    private final double y;
    ```

- Automatically Provided Members -
  - Canonical Constructor -
    - automatically defined constructor that sets all instance fields.
    - `Point(double x, double y)`

  - Accessor Methods -
    ```
    double x()
    double y()
    ```

    - Accessors use the component name, not `getX()` or `getY()` -
    ```
    var p = new Point(3, 4);
    IO.println(p.x() + " " + p.y());
    ``` 

  - Automatically Generated Methods - `toString`, `equals`, `hashCode`

- Adding Methods to Records -
```
record Point(double x, double y) {
    double distanceFromOrigin() {
        return Math.hypot(x, y);
    }
}
```

- Can override generated methods if the signature matches (but strongly discouraged as it violates the principle of records as transparent data holders.) -
```
record Point(double x, double y) {
    public double x() { return 2 * x; }     // Legal but misleading
}
```

- No Additional Instance Fields -
```
record Point(double x, double y) {
    private double z;                       // ERROR
}
```

- All components are implicitly `final`, but references may point to mutable objects -
```
record PointInTime(double x, double y, Date when) { }

var pt = new PointInTime(0, 0, new Date());
pt.when().setTime(0);                       // Mutates record state
```

- A record can be declared inside another class. 
  - The enclosing class may access record fields directly.
  - `p.x   // instead of p.x()`
  - This also applies to records in a compact compilation unit.

- Records may declare static fields and methods -
```
record Point(double x, double y) {
    static Point ORIGIN = new Point(0, 0);

    static double distance(Point p, Point q) {
        return Math.hypot(p.x - q.x, p.y - q.y);
    }
}
```

- Compact Constructor -
  - Used to validate or normalize parameters.
  - You cannot read or modify the instance fields in the body of the compact constructor.
  - Example -
  ```
  record Range(int from, int to) {
    Range {
        if (from > to) throw new IllegalArgumentException();
    }
  }
  ```

- Custom Constructors
  - Must delegate to another constructor.
  - Ultimately must invoke the canonical constructor.
  - Example - this record has two constructors: the canonical constructor and a no-argument constructor yielding the origin -
  ```
  record Point(double x, double y) {
    Point() {
        this(0, 0);
    }
  }
  ```
