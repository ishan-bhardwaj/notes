## Select

### Select & From

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

### Where

- `WHERE` clause filters data based on the specified condition - `WHERE <condition>`

- Eg -
```
SELECT name, age
FROM employee
WHERE country = 'US'
```

- Order of execution - `FROM` > `WHERE` > `SELECT`

### Order By

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

### Group By

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

### Having

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

### Distinct

- Removes duplicate rows.
- Eg - 
```
SELECT DISTINCT country
from employee
```

