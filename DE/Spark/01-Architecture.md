# Spark Architecture

- Distributed data processing and execution engine designed to process large-scale datasets across clusters of machines.
- Spark Application consists of -
    - Cluster Manager
    - Driver Process
    - Executor Processes

- __The cluster manager__ -
    - Tracks available cluster resources
    - Allocates CPU and memory
    - Launches executor processes
    - Handles container isolation
    - Monitors node availability
    - Schedules Spark applications
    - Eg - Spark standalone, YARN, Mesos, Kubernetes

- __Driver Process__ -
    - Control plane of the Spark application
    - Runs the application's `main()` function and coordinates all distributed execution

- __Executors__ -
    - Distributed worker JVM processes running on cluster nodes
    - Perform actual computation
    - Launched by the cluster manager

## Local Mode

- Driver and executors run on same machine
- Executors run as threads
- No real distributed network exists

## SparkSession

- Main entry point into Spark
- 1:1 relationship between `SparkSession` and the spark application
- Exists only on driver - executor can never contain SparkSession objects
- Creating spark session -
    - Scala -
        ```
        val spark = SparkSession.builder()
                                .appName("my-app")
                                .getOrCreate()
        ```

    - Python -
        ```
        spark = SparkSession.builder \
                            .appName("my-app") \
                            .master("<master-url>") \
                            .config("<config-name>", "config-value") \
                            .getOrCreate()
        ```

## `spark-submit`

- Command-line tool used to submit Spark applications to a cluster for execution
- Responsibilities -
    - Upload application code to cluster
    - Launch driver process
    - Request cluster resources/executors
    - Start application execution
- Eg -
    ```
    spark-submit \
        --master yarn \
        --deploy-mode cluster \
        --executor-memory 4G \
        app.py
    ```
    