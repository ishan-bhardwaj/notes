# Managing Documents

- Indexing documents -
  ```
  POST /<index>/_doc
  // JSON
  ```

- Indexing documents with explicit id -
  ```
  PUT /<index>/_doc/<id>
  // JSON
  ```

> [!TIP]
> `auto_create_index` (set to `true` by default) allows creating an index automatically if we try to insert a document to a non-existing index.

- Get document by id -
  ```
  GET /<index>/_doc/<id>
  ```

- Updating document by id -
  ```
  POST /<index>/_update/<id>
  {
    "doc": {
      // key-value pairs
    }
  }
  ```

> [!TIP]
> ES documents are immutable.
>   - Updating a document creates a new document and replaces the existing one.

- __Scripted updates__ -
  - Enables us to write custom scripts
  - Eg - increment existing value -
    ```
    POST /<index>/_update/<id>
    {
      "script": {
        "source": "ctx._source.in_stock--"        // Reduce the in_stock field value by 1
      }
    }
    ```

  - For advanced scripts, use triple quotes to use multi-line strings.
  - Using params -
    ```
    POST /<index>/_update/<id>
    {
      "script": {
        "source": "ctx._source.in_stock -= params.quantity"   // Reduce the in_stock by 4
        "params": {
          "quantity": 4
        }
      }
    }
    ```

> [!TIP]
> If we try to update a field with same value as the existing one, the `result` key in the response will return `noop`, but this is not the case for scripted updates.

- __Upsert__ -
  ```
  POST /<index>/_update/<id>
  {
    "upsert": {
      // key-value pairs
    }
  }
  ```

- __Replacing document__ -
  ```
  PUT /<index>/_doc/<id>
  {
    // key-value pairs
  }
  ```

- __Deleting document__ -
  ```
  DELETE /<index>/_doc/<id>
  ```

## Routing

- Process of resolving a shard for a document.
- ES uses following formula to determine which shard the document should be stored -
  ```
  shard_num = hash(_routing) % num_primary_shards
  ```

  - `_routing`, by default, refers to the document's id.
- It is possible to customize routing.
- The default routing strategy ensures that documents are distributed evenly.

## How ES reads data

- A read request is received and handled by a coodinating node.
- Routing is used to resolve the document's replication group.
- _Adaptive Replica Selection (ARS)_ is used to send the query to the best available shard -
  - ARS helps reduce query response times.
  - ARS is essentially an intelligent load balancer.
- The coordinating node collects the response and sends it to the client.

## How ES writes data


