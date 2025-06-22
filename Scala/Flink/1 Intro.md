# Flink

- Flink is a distributed system and engine for stateful big data streaming.
- Can be controlled to offer exactly-once, at-least-once or at-most-once guarantees.

### Use cases

- Event-driven applications - react to data elements as events like trigger computations, external actions, updates etc. Example -
    - recommendation systems
    - fraud detection
    - alerting and monitoring

- Low-latency data pipelines -
    - transform/enrich/transport data as it arrives
    - much lower latency than regular ETL scheduled jobs
    - Example - real-time aggregation like search indices

- Real-time data analytics - process data, extract insights and make decisions as soon as new data arrives. Example -
    - data-driven business decisions
    - IoT applications
    - measuring data from consumer applications

### Concepts

- **Dataflow** - description of data processing steps (called _operators_) in the applications.
- **Operator** - an independent logical processing step which can be parallelized into tasks.
- **Task / Operator Task** - an independent instance of an operator. It works on a data partition and runs on a physical machine.
- **Event Time vs Processing Time** - all events in Flink are timestamped.
    - Event time - time when the event was created in the source system.
    - Processing time - time when the event arrived at the Flink processor.



