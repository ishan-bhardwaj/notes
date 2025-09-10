# Non-Functional System Characteristics

## Availability

- Percentage of time a service or infrastructure is accessible and operates normally.
- Measuring availability - `A(%) = (Total Time - Amount Of Time Service Was Down) / Total Time * 100`
- Higher A → better availability.
- We measure availability as a number of nines, eg -
    - 90% (1 nine) - Downtime/Year = 36.5 days
    - 99% (2 nines) - Downtime/Year = 3.65 days
    - 99.5% - Downtime/Year = 1.83 days
    - 99.9% (3 nines) - Downtime/Year = 8.76 hrs
    - 99.999% (5 nines) - Downtime/Year = 5.26 mins
    - 99.99999% (7 nines) - Downtime/Year = 3.15 secs

> [!NOTE]
> Different providers measure availability differently. Always understand how a provider calculates availability before relying on it.

## Reliability

- Probability that a service will perform its functions correctly for a specified time.
- Main focus - how consistently the service operates under varying conditions.
- Key Metrics -
    - MTBF (Mean Time Between Failures) - `(Total Elapsed Time – Sum of Downtime) / Total Number of Failures`
    - MTTR (Mean Time To Repair) - `Total Maintenance Time / Total Number of Repairs`
- Higher MTBF = better reliability.
- Lower MTTR = quicker recovery.

> [NOTE]
> MTTF (Mean Time To Failure): Used for irreparable components (e.g., disk, bulb) instead of MTBF.

- Reliability vs Availability -
    - Reliability → Consistency of operation without failure.
    - Availability → % of time the system is accessible when needed.
    - Mathematically, Availability is a function of Reliability.

## Scalability

- Scalability is the ability of a system to handle an increasing workload without compromising performance.
- Example - A search engine must support more users and index more data as demand grows.
- Types of Workload -
    - Request workload – number of requests served by the system.
    - Data/storage workload – amount of data stored by the system.
- Dimensions of Scalability -
    - Size scalability – ability to add more users/resources without issues.
    - Administrative scalability – ease with which multiple organizations/users can share the system.
    - Geographical scalability – ability to serve users across regions while maintaining performance.
- Approaches to Scalability -
    - Vertical Scalability (Scaling Up) -
        - Add more resources (CPU, RAM) to an existing machine.
        - Advantages - Simple to implement.
        - Limitations - Bound by hardware limits, Expensive (requires high-end/exotic components).
    - Horizontal Scalability (Scaling Out) -
        - Add more machines (commodity nodes) to the system.
        - Advantages - Cost-effective, allows virtually unlimited scaling.
        - Limitations - Requires system design for distributed coordination, Complexity increases (synchronization, fault tolerance, data consistency).

## Maintainability

- Defines how easily a system can be operated, understood, and modified over time.
- Maintainability is the ability of a system to stay operational by - fixing bugs, adding new functionalities, updating the platform, ensuring smooth operations.
- Aspects of Maintainability -
    - Operability – Ease of keeping the system running under normal and faulty conditions.
    - Lucidity – Simplicity of code; easier to understand = easier to maintain.
    - Modifiability – Ability to add/modify features without hassle.
- Measuring Maintainability -
    - Maintainability (M): Probability that the service restores its functions within a specified time after a fault.
    - Example - M = 95% for 30 minutes → 95% chance the system is restored within half an hour.
    - Metric - Mean Time to Repair (MTTR) = `Total Maintenance Time / Total Number of Repairs`
    - Goal - Keep MTTR as low as possible.
- Maintainability vs Reliability -
    - Maintainability – Focuses on time-to-repair.
    - Reliability – Focuses on time-to-failure + time-to-repair.
    - Together, they provide insights into - availability, downtime & uptime.

## Fault Tolerance

- Fault tolerance ensures a system continues to operate correctly despite failures.
- Failures occur at both hardware and software levels and can affect data and services.
- Fault tolerance techniques -
    - Replication -
        - Replicate services and data across multiple nodes/storage.
        - Failed node/data store can be swapped with a replica. Service continues without impacting end users.
        - Key aspects -
            - Multiple data copies are stored separately.
            - Updates must propagate to all replicas.
        - Consistency trade-offs (CAP Theorem) -
            - Synchronous updates → strong consistency but lower availability.
            - Asynchronous updates → higher availability but eventual consistency (stale reads possible).
        - Use Case - Systems requiring high reliability and redundancy (databases, large-scale services).
    - Checkpointing -
        - Periodically save system state in stable storage for recovery after failures - enables restarting from the last saved checkpoint instead of from scratch.
        - Types of Checkpointing -
            - Consistent State - All processes have a coherent view of system state.
            - Inconsistent State - Checkpoints across processes are not aligned - results in discrepancies in recovery
