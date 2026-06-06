# Multi-Tenancy & Resource Management

## Fair Scheduler for Multi-Tenancy

- __Fair Scheduler__ - Spark's intra-application scheduling mode that shares resources across concurrent jobs (within one SparkSession) using weighted pools; the primary tool for multi-tenant workloads sharing a single Spark application
- Enabled via `spark.scheduler.mode=FAIR`; configured via `fairscheduler.xml`
- Critical distinction - Fair Scheduler operates WITHIN a single SparkApplication (across multiple concurrent jobs/threads); it does NOT control resource allocation between separate Spark applications (that is the cluster manager's job)

### Why Fair Scheduler for Multi-Tenancy

- Single SparkSession shared by multiple users (notebook server, multi-tenant SQL engine like Spark Thrift Server)
- Without Fair Scheduler - first user's job monopolizes all executor cores; second user's job waits entirely
- With Fair Scheduler - resources interleaved across pools; each user/tenant gets guaranteed minimum share

### Pool Configuration

```xml
<!-- fairscheduler.xml -->
<allocations>
    <pool name="production">
        <schedulingMode>FAIR</schedulingMode>
        <weight>4</weight>
        <minShare>20</minShare>    <!-- minimum tasks guaranteed even under contention -->
    </pool>
    <pool name="interactive">
        <schedulingMode>FAIR</schedulingMode>
        <weight>2</weight>
        <minShare>10</minShare>
    </pool>
    <pool name="batch">
        <schedulingMode>FIFO</schedulingMode>
        <weight>1</weight>
        <minShare>0</minShare>
    </pool>
    <pool name="default">
        <schedulingMode>FIFO</schedulingMode>
        <weight>1</weight>
        <minShare>0</minShare>
    </pool>
</allocations>
```

```python
spark = SparkSession.builder \
    .config("spark.scheduler.mode", "FAIR") \
    .config("spark.scheduler.allocation.file", "/path/to/fairscheduler.xml") \
    .getOrCreate()
```

### Pool Assignment

- Jobs assigned to pools at submission time via `SparkContext.setLocalProperty` -
```python
    # Thread-local property - each thread/user submits to their pool
    spark.sparkContext.setLocalProperty("spark.scheduler.pool", "interactive")
    df.count()    # this job goes to interactive pool

    spark.sparkContext.setLocalProperty("spark.scheduler.pool", "batch")
    df.write.parquet("/output")   # this job goes to batch pool

    # Clear to use default pool
    spark.sparkContext.setLocalProperty("spark.scheduler.pool", None)
```

- Thread-local semantics - each thread has its own pool assignment; concurrent threads with different pool assignments run simultaneously; critical for multi-tenant notebook servers where each user request runs in its own thread

### Scheduling Algorithm Deep Dive

- `FairSchedulingAlgorithm` determines task assignment order across pools -
    - For each resource offer, iterates pools sorted by `runningTasks / weight` (ascending)
    - Pool with lowest `runningTasks / weight` ratio has highest priority (most "starved")
    - Among tied pools - `minShare` tiebreaker: pool below `minShare` wins
    - Further tie - alphabetical by pool name

- Weight semantics - if pool A has `weight=2` and pool B has `weight=1` with equal pending tasks - pool A gets approximately $\frac{2}{3}$ of resources, pool B gets $\frac{1}{3}$

- `minShare` semantics - hard minimum; if pool has pending tasks and `runningTasks < minShare`, it always wins resource offers over any pool above its `minShare`; guarantees latency SLA for interactive queries

### Pool Hierarchy (Nested Pools)

- Pools can be nested for hierarchical resource management -
```xml
    <allocations>
        <pool name="team-A">
            <weight>3</weight>
            <pool name="team-A-production">
                <weight>2</weight>
                <minShare>5</minShare>
            </pool>
            <pool name="team-A-development">
                <weight>1</weight>
                <minShare>0</minShare>
            </pool>
        </pool>
        <pool name="team-B">
            <weight>1</weight>
        </pool>
    </allocations>
```

- Resources allocated to team-A vs team-B first (3:1 ratio); within team-A, split production:development (2:1)

### Fair Scheduler Limitations

- Only intra-application - cannot prevent one application from starving others at cluster level
- `minShare` is a soft guarantee - if total cluster tasks < sum of `minShare` values, all pools get their minimum; but if cluster is overloaded, actual enforcement depends on task completion patterns
- No preemption - Fair Scheduler does not kill running tasks to give resources to a higher-priority pool; it only controls allocation of newly freed slots
- Pool config changes require application restart - `fairscheduler.xml` read at `SparkContext` creation; runtime changes not picked up

### Multi-Tenant Thrift Server Pattern

- Spark Thrift Server (HiveServer2-compatible SQL endpoint) shares one SparkSession across multiple JDBC clients
- Each client connection gets its own thread → thread-local pool assignment → different pools per client SLA tier
- Configuration -
```python
    # Thrift Server sets pool based on client connection metadata
    # In HiveContext session configuration:
    # SET spark.scheduler.pool=interactive;   -- client sets this per session
```

---

## YARN Queues & Capacity Scheduler

- __YARN Capacity Scheduler__ - YARN's default queue-based resource manager; allocates cluster capacity across queues; each queue gets a guaranteed minimum and maximum share of the cluster
- Operates at the application level (across Spark applications); not within a single application
- Every Spark application on YARN runs in exactly one queue; queue assignment determines resource limits

### Capacity Scheduler Architecture

```
root
├── production        (40% capacity, 60% max)
│   ├── etl           (60% of production = 24% cluster)
│   └── streaming     (40% of production = 16% cluster)
├── interactive       (30% capacity, 50% max)
│   ├── team-a        (50% of interactive = 15% cluster)
│   └── team-b        (50% of interactive = 15% cluster)
└── default           (30% capacity, 40% max)
└── development   (100% of default = 30% cluster)
```

### Capacity Scheduler Configuration

- `capacity-scheduler.xml` on YARN ResourceManager -
```xml
    <property>
        <name>yarn.scheduler.capacity.root.queues</name>
        <value>production,interactive,default</value>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.production.capacity</name>
        <value>40</value>    <!-- 40% of cluster -->
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.production.maximum-capacity</name>
        <value>60</value>    <!-- can use up to 60% when others idle -->
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.production.user-limit-factor</name>
        <value>1</value>     <!-- one user cannot take more than 1× their fair share -->
    </property>
```

### Submitting Spark to a Specific Queue

```python
spark = SparkSession.builder \
    .config("spark.yarn.queue", "production.etl") \   # dot-separated queue path
    .getOrCreate()

# Or via spark-submit
# spark-submit --queue production.etl my_job.py
```

### Queue ACLs

```xml
<!-- Restrict who can submit to production queue -->
<property>
    <name>yarn.scheduler.capacity.root.production.acl_submit_applications</name>
    <value>spark-prod-user prod-group</value>   <!-- users or groups -->
</property>
<property>
    <name>yarn.scheduler.capacity.root.production.acl_administer_queue</name>
    <value>spark-admin</value>
</property>
```

### Queue Elasticity

- __Capacity__ - guaranteed minimum; queue always gets this fraction even if others need resources
- __Maximum capacity__ - queue can expand up to this fraction when other queues have idle capacity
- __User limit factor__ - within a queue, limits how much one user can consume relative to others
- Preemption (if enabled) - YARN kills containers from over-allocated queues to give capacity back to under-served queues
```xml
    <property>
        <name>yarn.scheduler.capacity.preemption.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.scheduler.capacity.preemption.natural_termination_factor</name>
        <value>0.2</value>   <!-- preempt 20% of over-allocation per cycle -->
    </property>
```

### YARN Fair Scheduler (Alternative)

- YARN also has its own Fair Scheduler (distinct from Spark's intra-app Fair Scheduler)
- YARN Fair Scheduler - allocates YARN containers across applications fairly; similar to Capacity Scheduler but with different scheduling semantics
- `yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler`
- Configured via `fair-scheduler.xml`; supports queues with weights, `minResources`, `maxResources`
- Less commonly used in enterprise than Capacity Scheduler; better for dynamic, bursty workloads

### Dynamic Queue Configuration (YARN 3.x)

- YARN 3.x supports runtime queue configuration changes without ResourceManager restart
- Add queues, change capacity, adjust ACLs while cluster runs
- `yarn rmadmin -refreshQueues` - reload capacity-scheduler.xml at runtime

### Spark Dynamic Allocation and YARN Queues

```python
spark = SparkSession.builder \
    .config("spark.dynamicAllocation.enabled", "true") \
    .config("spark.dynamicAllocation.minExecutors", "2") \
    .config("spark.dynamicAllocation.maxExecutors", "100") \
    .config("spark.dynamicAllocation.initialExecutors", "10") \
    .config("spark.yarn.queue", "interactive") \
    .getOrCreate()
```

- Dynamic allocation respects queue limits - `maxExecutors` bounded by queue's available capacity
- If queue at capacity limit - new executor requests queued until another app releases containers
- Queue preemption + dynamic allocation = Spark scales down when preempted; scales back up when capacity available

---

## Cost Attribution & Chargeback Patterns

- __Cost attribution__ - tracking which team, project, or user consumed which cluster resources; enables chargeback (billing tenants for actual usage) and showback (showing usage without billing)
- Critical for multi-tenant shared clusters; without it, high-cost teams have no incentive to optimize

### Resource Consumption Metrics for Attribution

- CPU core-hours - `numCores × durationHours` per executor × number of executors
- Memory GB-hours - `memoryGB × durationHours` per executor × number of executors
- Storage I/O - shuffle bytes, HDFS read/write bytes
- GPU hours - for GPU-enabled clusters

### Attribution via Spark Application Metadata

```python
# Tag applications at submission time
spark = SparkSession.builder \
    .appName("team-analytics-daily-etl") \
    .config("spark.yarn.tags", "team=analytics,project=daily-etl,cost-center=CC-1234") \
    .config("spark.yarn.queue", "analytics") \
    .getOrCreate()

# YARN application tags queryable via RM API
# GET http://rm-host:8088/ws/v1/cluster/apps?applicationTags=team=analytics
```

### Spark History Server for Cost Data

```python
# Query History Server REST API for application resource usage
import requests

apps = requests.get("http://history-server:18080/api/v1/applications").json()
for app in apps:
    app_id = app["id"]
    stages = requests.get(
        f"http://history-server:18080/api/v1/applications/{app_id}/stages"
    ).json()
    executors = requests.get(
        f"http://history-server:18080/api/v1/applications/{app_id}/executors"
    ).json()

    # Calculate core-hours
    total_core_seconds = sum(
        e["totalCores"] * e["totalDuration"] / 1000
        for e in executors if e["id"] != "driver"
    )
    core_hours = total_core_seconds / 3600
    print(f"{app['name']}: {core_hours:.2f} core-hours")
```

### YARN Resource Usage API

```bash
# Get per-application resource usage from YARN RM
curl "http://rm-host:8088/ws/v1/cluster/apps?state=FINISHED&startedTimeBegin=1700000000000" \
  | jq '.apps.app[] | {
      id: .id,
      name: .name,
      queue: .queue,
      user: .user,
      vcoreSeconds: .vcoreSeconds,
      memorySeconds: .memorySeconds,
      startedTime: .startedTime,
      finishedTime: .finishedTime
    }'
```

### SparkListener-Based Attribution

- Custom `SparkListener` collects fine-grained per-job/stage resource usage -
```scala
    import org.apache.spark.scheduler._

    class CostAttributionListener extends SparkListener {
        private val jobStartTimes = mutable.Map[Int, Long]()
        private val jobPools = mutable.Map[Int, String]()

        override def onJobStart(event: SparkListenerJobStart): Unit = {
            jobStartTimes(event.jobId) = System.currentTimeMillis()
            // Pool set via local property
            jobPools(event.jobId) = event.properties.getProperty(
                "spark.scheduler.pool", "default")
        }

        override def onTaskEnd(event: SparkListenerTaskEnd): Unit = {
            val metrics = event.taskMetrics
            // Accumulate per-pool: CPU time, shuffle bytes, input bytes
            val pool = jobPools.getOrElse(event.stageId, "unknown")
            attributeResources(pool,
                cpuMs = metrics.executorRunTime,
                shuffleReadBytes = metrics.shuffleReadMetrics.totalBytesRead,
                shuffleWriteBytes = metrics.shuffleWriteMetrics.bytesWritten,
                inputBytes = metrics.inputMetrics.bytesRead
            )
        }

        override def onJobEnd(event: SparkListenerJobEnd): Unit = {
            val duration = System.currentTimeMillis() - jobStartTimes(event.jobId)
            // Emit cost record: pool, duration, accumulated resources
            emitCostRecord(jobPools(event.jobId), duration)
            jobStartTimes.remove(event.jobId)
            jobPools.remove(event.jobId)
        }
    }

    // Register
    spark.sparkContext.addSparkListener(new CostAttributionListener)
```

### Chargeback Models

- __Core-hour model__ - charge per `executor_cores × executor_hours`; simplest; ignores memory variance
cost = executor_count × cores_per_executor × duration_hours × price_per_core_hour

- __Resource-unit model__ - normalize cores and memory into "Resource Units" (RU) -
RU = max(cores / cores_per_RU, memory_GB / GB_per_RU)
cost = RU × duration_hours × price_per_RU_hour

- Eg - 1 RU = 1 core + 4 GB; a 4-core 32-GB executor = max(4/1, 32/4) = 8 RU
    - Penalizes memory-heavy jobs appropriately

- __Shuffle-weighted model__ - add surcharge for shuffle-heavy jobs (network and disk cost) -

base_cost = core_hours × price_per_core_hour
shuffle_cost = shuffle_TB × price_per_TB_shuffled
total_cost = base_cost + shuffle_cost

- __Spot/on-demand blended model__ - for cloud clusters; spot instances cheaper but interruptible -
cost = (spot_executor_hours × spot_price) + (on_demand_executor_hours × on_demand_price)

### Tagging Strategy for Attribution

```python
# Comprehensive tagging for cost attribution
spark = SparkSession.builder \
    .appName(f"{team}-{project}-{environment}") \
    .config("spark.yarn.tags", ",".join([
        f"team={team}",
        f"project={project}",
        f"environment={environment}",
        f"cost_center={cost_center}",
        f"job_type={job_type}",         # batch, streaming, interactive
        f"run_date={run_date}"
    ])) \
    .config("spark.yarn.queue", f"{team}-{environment}") \
    .getOrCreate()
```

### Showback Dashboard Pattern

```python
# Daily cost report from YARN + History Server data
def compute_daily_costs(date: str) -> pd.DataFrame:
    # 1. Fetch all apps for the day from YARN RM API
    apps = fetch_yarn_apps(date)

    # 2. Parse tags for attribution dimensions
    records = []
    for app in apps:
        tags = parse_tags(app["applicationTags"])
        core_hours = app["vcoreSeconds"] / 3600
        memory_gb_hours = app["memorySeconds"] / 1024 / 3600

        records.append({
            "date": date,
            "team": tags.get("team", "unknown"),
            "project": tags.get("project", "unknown"),
            "cost_center": tags.get("cost_center", "unknown"),
            "queue": app["queue"],
            "core_hours": core_hours,
            "memory_gb_hours": memory_gb_hours,
            "cost_usd": core_hours * PRICE_PER_CORE_HOUR
        })

    return pd.DataFrame(records)

# 3. Aggregate and publish to cost dashboard
daily_df = compute_daily_costs("2024-01-15")
daily_df.groupby(["team", "cost_center"]).agg({
    "core_hours": "sum",
    "cost_usd": "sum"
}).reset_index().sort_values("cost_usd", ascending=False)
```

### Quota Enforcement

- Chargeback is reactive (bill after consumption); quota enforcement is proactive (prevent over-consumption)
- YARN queue `maximum-capacity` - hard limit on cluster fraction; jobs queued when queue full
- YARN per-user `user-limit-factor` - within queue, limit one user's share
- Spark `spark.dynamicAllocation.maxExecutors` - application-level executor cap; set per-team based on quota

```python
# Per-team quota enforcement via dynamic allocation limits
TEAM_QUOTAS = {
    "analytics": {"maxExecutors": 100, "queue": "analytics"},
    "ml": {"maxExecutors": 200, "queue": "ml"},
    "interactive": {"maxExecutors": 50, "queue": "interactive"}
}

team = get_current_team()
quota = TEAM_QUOTAS[team]

spark = SparkSession.builder \
    .config("spark.dynamicAllocation.enabled", "true") \
    .config("spark.dynamicAllocation.maxExecutors", quota["maxExecutors"]) \
    .config("spark.yarn.queue", quota["queue"]) \
    .getOrCreate()
```

> [!NOTE]
> The Spark Fair Scheduler and YARN Capacity Scheduler solve different problems and are used together:
> - YARN Capacity Scheduler - allocates cluster resources BETWEEN Spark applications (inter-application)
> - Spark Fair Scheduler - allocates executor slots WITHIN a single Spark application (intra-application)
> - For a multi-tenant SQL service - YARN queues separate teams at cluster level; Spark Fair Scheduler gives different response time guarantees to concurrent queries within the shared Spark Thrift Server session

> [!TIP]
> Cost attribution implementation priority -
> ```
> 1. Tag all applications at submission time (team, project, cost_center, environment)
>    - Zero-overhead; just metadata
>    - Enables all downstream attribution queries
>
> 2. YARN vcoreSeconds + memorySeconds from RM API
>    - Available without any custom instrumentation
>    - Accurate to the container level
>
> 3. SparkListener for finer-grained job/stage attribution
>    - Needed when one app runs jobs for multiple tenants (Thrift Server, notebook server)
>    - Links resource usage to specific user sessions via pool assignment
>
> 4. Publish to cost dashboard with weekly chargeback reports
>    - Makes cost visible to teams; incentivizes optimization
>    - High-cost teams self-regulate when they see their bills
> ```
