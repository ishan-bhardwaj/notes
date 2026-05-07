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

