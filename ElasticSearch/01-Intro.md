# ElasticSearch

- Written in Java, built on Apache Lucene.
- Data is stored as documents, containing fields.
  ```
  {
    "firstName": "John",
    "lastName": "Doe",
    "interests": ["ElasticSearch", "Logstash", "Kibana"]
  }
  ```

- Start ES - `bin/elasticsearch`
- Reset password - `bin/elasticsearch-reset-password -u elastic`
- Enrollment tokens are valid for 30 minutes.
  - Create new - `bin/elasticsearch-create-enrollment-token --scope kibana`
- Start kibana - `bin/kibana`

## Architecture

- A __Node__ is an instance of elasticsearch that stores data.
- A __Cluster__ is a collection of related nodes.
  - We can have multiple clusters for different purposes.
  - Clusters are completely independent of each other by default.
    - It is possible to perform cross-cluster search, but it's not that common.
- When a node is started, it either joins an existing cluster or it will create its own cluster automatically of just that node.
- __Document__ - unit of data stored within the cluster.
  - Documents are JSON objects.
  - When you index a document, the original JSON object that you sent to ES is stored along with some metadata used internally by ES.
- __Index__ - every document within ES is stored within an index.
  - An index groups documents together logically, and provides configuration options related to scalability and availability.
  - Search queries are run against indices.

## Sharding

- Sharding is a way to divide indices into smaller pieces to horizontally scale the data volume.
- Each piece is called a _shard_.
- Sharding is done at index level.
- Each shard is an Apache Lucene index.
- An ES index consists of one or more Lucene indices.
- A Shard has no predefined size - it grows as documents are added to it.
- A Shard may store up to 2 billion documents.
- Improves performance - parallelization of queries increases the throughput of an index.
- An index contains a primary shard and a replica shard by default.
- Number of shards -
  - _Split API_ - increase
  - _Shrink API_ - reduce

## Replication

- Replication is supported natively and enabled by default.
- Replication is configured at index level.
- Replication works by creating copies of shards - _replica shards_
- A Shard that has been replicated - _primary shard_
- A Primary shard and its replica shards collectively called _replication group_
- A Replica shard can serve search requests, exactly like its primary shard.
- The number of replicas can be configured at index creation.
- Replica shards are never stored on the same node as their primary shard.
- Replication also helps in increasing the number of requests that can be handled in parallel - increasing system throughput.

> [!NOTE]
> Elasticsearch will only add replica shards for clusters with multiple nodes.
>
> i.e. for single node, configuring an index to contain one or more replicas will not have any effect.

## Snapshots

- ES supports taking snapshots as backups.
- Snapshots can be used to restore state to a given point in time.
- Snapshots can be taken at the index level, or for the entire cluster.

> [!TIP]
> Use snapshots for backups, and replication of high availability (and performance).

## Kibana Console Tool

- Kibana > `Dev Tools` > `Console`
- Cluster health - 
  ```
  GET /_cluster/health
  ```

> [!TIP]
> Configure cluster name - `$ES_HOME > config > elasticsearch.yml`

- Listing nodes -
  ```
  GET /_cat/nodes?v             // v adds a descriptive header row to the output
  ```

- Listing indices -
  ```
  GET /_cat/indices?v&expand_wildcards=all
  ```  

  - `expand_wilcards=all` shows both user and system indices.
  - Leading period (`.`) in an index name indicates a system index - meaning it will be hidden by default.

- Listing shards -
  ```
  GET /_cat/shards?v
  ```

- Create index -
  ```
  PUT /<index>
  ```

- Create index with settings -
  ```
  PUT /<index>
  {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 2
    }
  }
  ```

- Delete index -
  ```
  DELETE /<index>
  ```

> [!TIP]
> Kibana indices are configured with `auto_expand_replicas` setting with a value of `0-1` - dynamically changes the number of replicas dependending on how many nodes are there in our cluster.

## Node Roles

- __Master Node__ -
  - Responsible for creating and deleting indices, keeping track of nodes and allocating shards to nodes.
  - `node.master` - boolean 
  - A Node with this role will not automatically become the master node, unless there are no other master-eligible nodes.
  - The cluster's master node is elected based on a voting process.

- __Data Node__ -
  - Enables a node to store data.
  - `node.data` - boolean
  - Storing data includes performing queries related to that data, such as search and modification queries.

- __Ingest Node__ -
  - Enables a node to run ingest pipelines.
  - `node.ingest` - boolean
  - Ingest pipelines are series of steps (processors) that are performed when indexing documents.
  - Processors may manipulate documents, eg - resolving an IP to lat/lon

- __Machine Learning Node__ -
  - `node.ml` identifies a node as a machine learning node.
    - This lets the node run machine learning jobs.
  - `xpack.ml.enabled` enables the node's capability of responding to machine learning API requests.

- __Coordination Node__ -
  - Coordination refers to the distribution of queries and the aggregation of the results.
  - Such a node does not search any data on its own - delegates it to data nodes.
  - Configured by disabling all the other roles.
  - Dedicated coordination nodes are only useful for large clusters, as it can be used as a load balancer.

- __Voting-only Node__ -
  - Participates in the voting for a new master node.
  - This node itself cannot be elected as the master node though.
  - `node.voting_only` - boolean
  - Rarely used - only useful in large clusters.


