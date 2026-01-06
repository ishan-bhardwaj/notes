# Data Manipulation (DML)

## Insert

- Add new rows to the table -
```
INSERT INTO <table_name> (<column_1>, <column_2>, ... , <column_N>)
VALUES 
    (<value_11>, <value_12>, ... , <value_1N>),
    (<value_21>, <value_22>, ... , <value_2N>)
```

- If we want to specify values for all the columns, we can ignore writing the column names.

- `INSERT` using `SELECT` - used to copy data from a source table to target table, eg - copy data from `customers` table into `persons` -
```
INSERT INTO persons (id, name, phone)
SELECT id, first_name, 'Unknown'
FROM customers
```

## Update

- Update content of existing rows -
```
UPDATE <table_name>
SET 
    <column_1> = <value_1>,
    <column_2> = <value_2>,
    ...
    <column_N> = <value_N>
WHERE <condition>
```

> [!WARNING]
> Without `WHERE`, all rows will be updated.

- Eg - update the score of customer id 6 to 100 -
```
UPDATE customers
SET score = 100
WHERE id = 6
```

- Eg - Update all customers with a NULL score by setting their score to 0 -
```
UPDATE customers
SET score = 0
WHERE score IS NULL
```

## Delete

- Deleting rows -
```
DELETE FROM <table_name>
WHERE <condition>
```

> [!WARNING]
> Without `WHERE`, all rows will be deleted.

- Eg - Delete all customers with id greater than 5 -
```
DELETE FROM customers
WHERE id > 5
```

- Clear the whole table at one without checking or logging anything -
```
TRUNCATE TABLE <table_name>

-- or

DELETE FROM <table_name>
```

> [!TIP]
> `TRUNCATE` is faster than `DELETE` for large table as it skips a lot of things behind the scenes that `DELETE` does.
