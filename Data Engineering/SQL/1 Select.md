# Select

## Select & From

- Syntax -
```
SELECT <columns>
FROM <table>
```

- Eg -
```
SELECT name, age
FROM employee
```

- Use `*` to select all the columns.

## Where

- `WHERE` clause filters data based on the specified condition - `WHERE <condition>`

- Eg -
```
SELECT name, age
FROM employee
WHERE country = 'US'
```

- Order of execution - `FROM` > `WHERE` > `SELECT`

## Order By

- Orders the resulting rows based on specified columns in ascending or descending order - 
```
ORDER BY <column> <order>
```

- `order` is optional and can be - `ASC` (default, if not specified) or `DESC`

- Eg - 
```
SELECT name, age
FROM employee
ORDER BY salary DESC
```

- Using multiple columns -
```
SELECT name, age
FROM employee
ORDER BY 
    country ASC,
    salary DESC
```

## Group By

- Groups rows based on the specified columns -
```
GROUP BY <columns>
```

- Eg -
```
SELECT 
    country,
    department,
    SUM(salary) AS total_score
FROM employee
GROUP BY country, department
```

- `SUM(salary)` sums all the salaries in a group and `AS total_score` is a alias for the new column generated.

> [!WARNING]
> All the non-aggregated columns in `SELECT` must be present in `GROUP BY`.

## Having

- Filters data after aggregation.
- Can only be used with `GROUP BY` and applicable on aggregated columns.
- Eg -
```
SELECT 
    country,
    department,
    SUM(salary) AS total_score
FROM employee
GROUP BY country, department
HAVING SUM(salary) > 500
```

> [!NOTE]
> Most SQL dialects don’t allow aliases (like `total_salary`) in the `HAVING` clause — you must use the aggregate function itself.

- To filter the data before aggregation, we use `WHERE` -
```
SELECT 
    country,
    department,
    SUM(salary) AS total_salary
FROM employee
WHERE department = 'IT'
GROUP BY country, department
HAVING SUM(salary) > 500
```

- Order of execution - `FROM` > `WHERE` > `GROUP BY` > `HAVING` > `SELECT`

## Distinct

- Removes duplicate rows.
- Eg - 
```
SELECT DISTINCT country
from employee
```

## TOP

- Returns top `N` rows from the table based on the row number.
- Syntax -
```
SELECT TOP N <columns>
FROM <table>
```

> [!TIP]
> Query Execution order -
> `FROM` > `WHERE` > `GROUP BY` > `HAVING` > `SELECT DISTINCT` > `ORDER BY` > `TOP`

> [!TIP]
> Getting static data from query results instead of getting it from a table -
> ```
> SELECT 50 AS static_int, 'Hello' AS static_string
> ```
>
> Using mix of table and static data -
> ```
> SELECT name, age, 'New' AS customer_type
> FROM customers
> ```

