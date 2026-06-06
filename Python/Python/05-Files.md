# Files

- Reading file content -
  ```
  my_file = open('data.txt', 'r')
  file_content = my_file.read()
  lines = my_file.readlines()         
  my_file.close()
  ```

- Writing content -
  ```
  my_file = open('data.txt', 'w')       # 'w' overwrites the existing content
  my_file.write('hello')
  my_file.close()
  ```

## `with` statement

- File is closed automatically when it reaches the end of the block -
  ```
  with open('data.txt', 'r') as file:
    file_contents = file.read()
  ```  

## CSV Files

- Reading CSV file as `list` -
  ```
  import csv
  file = open('file.csv', 'r')
  reader = csv.reader(file)
  for line in reader:
    # line is a list
  ```

- Writing `list` to CSV file -
  ```
  import csv
  file = open('file.csv', 'w')
  writer = csv.writer(file)
  writer.writerows(list)
  ```

- Reading CSV file as `dict` -
  ```
  import csv
  file = open('file.csv', 'r')
  reader = csv.DictReader(file)
  for line in reader:
    # line is a dict
  ```

- Writing `dict` to CSV file -
  ```
  import csv
  file = open('file.csv', 'w')
  writer = csv.DictWriter(file, fieldnames=['name', 'age'])
  writer.writeheader()
  writer.writerows(dict)
  ```

## JSON Files

- Reading from JSON file -
  ```
  import json
  file = open('friends_json.txt', 'r')
  file_contents = json.load(file)         # reads file as a dict
  ```

- Writing to JSON file -
  ```
  import json
  file = open('friends_json.txt', 'w')
  cars = [
    {'make': 'Ford', 'model': 'Fiesta'},
    {'make': 'Ford', 'model': 'Focus'}
  ]

  json.dump(cars, file)
  ```

- Convert json string to dict -
  ```
  import json
  raw = '[{"first_name": "John", "last_name": "Doe"}]'
  parsed = json.loads(raw)
  ```

- Similary, `json.dumps()` convert dict to json string.

## Imports

- Assume `utils/file_operations.py` contains functions like (`save_to_file`) -
  - Import entire module - `import utils.file_operations`
  - Import specific functions - `from utils.file_operations import save_to_file`

- If two things have the same thing, then Python will use the one which is imported at the last.
- Older python versions require creating empty `__init__.py` file inside directories to mark them as python package.

> [!TIP]
> Python executes the files that are imported.

- __Relative imports__ -
  - Assume two files - `utils/common/file_operations.py` and `utils/finder.py`
  - To import any function from `file_operations` into `finder.py` -
    ```
    from utils.common.file_operations import save_to_file         # correct - absolute path
    from .common.file_operations import save_to_file              # correct - relative path
    from common.file_operations import save_to_file               # incorrect - python assumes absolute path by default

    from ..find import NotFoundError                              # relative parent dir import - imports from utils package
    ```

> [!WARNING]
> A file with relative import will throw an error when executed directly, but will work if imported as a module in another file.
> 
> `__name__` on the executing script returns `__main__`, and the imported modules like `utils.common.finder` returns `utils.common.finder`
>
> So, when running the file with the relative path directly, it's `__name__` return `__main__.common.finder` which doesn't exist - throws an error - `No module named __main__.common`
