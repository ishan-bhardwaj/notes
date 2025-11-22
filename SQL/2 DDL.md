# Data Definition (DDL)

## Create Table

- Creating a new table -
```
CREATE TABLE <table_name> (
    <column_1> <datatype> <constraints>
    <column_2> <datatype> <constraints>
    ... 
    <column_N> <datatype> <constraints>
    CONSTRAINT <primary_key_constraint_name> PRIMARY KEY (<primary_key_column>)
)
```

- Eg -
```
CREATE TABLE persons (
    id INT NOT NULL,
    name VARCHAR(50) NOT NULL,
    birth_date DATE,
    phone VARCHAR(15) NOT NULL,
    CONSTRAINT pk_persons PRIMARY KEY (id)
)
```

## Alter Table

- Updating table definition -
```
ALTER TABLE <table_name> <operation> 
```

- Adding a new columns -
```
ALTER TABLE persons
ADD email VARCHAR(50) NOT NULL
```

> [!TIP]
> New columns are appended at the end of the table by default. To add a new column in middle, we have to create a new table.

- Removing a column -
```
ALTER TABLE persons
DROP COLUMN phone
```

## Drop Table

- Deleting the table -
```
DROP TABLE <table_name>
```
