# Airflow

- Core components -
    - Web Server - serves the UI
    - Metadata Database - contains airflow metadata (task instances, workflows etc), eg - postgres, mysql etc.
    - Scheduler - schedules tasks while checking that dependencies are met
    - Executor - part of Scheduler that defines how and on which system to exectue tasks
    - Trigger - used for special type of tasks (deferrable operators)
    - Worker - executes the tasks

- Core concepts -
    - Tasks - basic unit of execution
    - Operator - task templates (eg - BashOperator, PythonOperator etc)
    - DAG - tasks flow with dependencies

- How Airflow works -
    - Add DAG python file into the DAG directory.
    - Scheduler scans the DAG directory for new files every 5 mins by default.
        - Scheduler detects new DAG file and serializes it into Metadata database.
    - Scheduler checks if the DAG is ready to run - creates a DAG run.
        - Then Scheduler checks if there are tasks to run - creates a task instance and sends it to the executor.
    - Executor pushes the tasks instance into a queue and updates the state in Metadata database.
    - Worker picks up the task instances from the queue, executes it and updates the states of the task instances in Metadata database till its completion.
    - Web Server pulls serves data from Metadata database at any point of time.

- Limitations - useful for batch processing, but streaming is not support.

## CLI

- `airflow cheat-sheet` - displays all the CLI commands.

### Database commands

- `airflow db check` - checks if the metadata database can be reached.
- `airflow db clean` - purges old records in database tables in archive tables.
- `airflow db export-archived` - exports archived data from the archived tables - in csv format by default.
- `airflow db drop-archived` - drops archived data from the archived tables.
- `airflow db init` - initializes the database.

### DAG commands

- `airflow dags backfill <dag_name> --reset-dagrun --rerun-failed-tasks --run-backwards -s 2025-11-22 -e 2025-11-25` - runs subsections of a DAG for a specified date range.
- `airflow dags serialize` - re-sync DAGs manually - Airflow does it periodically also (every 5 mins for new DAGs, and 30sec for existing DAGs)
- `airflow dag list` - list all the DAGs.

### Task commands

- `airflow tasks test <dag_name> <task_name>` - tests a task instance.

> [!NOTE]
> Airflow also provides REST API to interact with DAGs.

## Creating DAGs

```
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def first_task():
    print("First Task")
    return 42

def second_task(ti = None):
    print("Second Task")
    print(ti.xcom_pull(task_ids = 'first_task'))


with DAG(
    dag_id = 'stock_market_processor',
    start_date = datetime(2025, 1, 1),
    schedule_interval = '@daily',
    catchup = False,
    tags = ['stock_market']
):

    task_a = PythonOperator(
        task_id = 'first_task',
        python_callable = first_task
    )

    task_b = PythonOperator(
        task_id = 'second_task',
        python_callable = second_task
    )

    task_a >> task_b
```

> [!NOTE]
> `xcom_pull` is used to share data between tasks.

## Taskflow

- Better way to create DAGs using decorators.

```
from airflow.decorators import dag, task
from datetime import datetime

@dag(
    start_date = datetime(2025, 1, 1),
    schedule_interval = '@daily',
    catchup = False,
    tags = ['stock_market']
)
def taskflow():

    @task
    def first_task():
        print("First Task")
        return 42

    @task
    def second_task(value)
        print("Second Task")
        print(value)

    second_task(first_task())    


taskflow()   
```

- `taskflow` is the unique id of the DAG, and similarly, `first_task` & `second_task` are the unique ids of the tasks.

- We can also mix the Taskflow API with previous method - 
```
task_a = PythonOperator(
    task_id = 'first_task',
    python_callable = first_task
)

@task
def second_task(value)
    print("Second Task")
    print(value)

second_task(task_a.output) 
```

- If both tasks do not have any dependencies then we can also write them as - `task_a() >> second_task()` in the mixed approach.

## Sensor decorator

- Used when listening to some events, eg - checking availability of a Rest API.
- To find all sensors - go to https://registry.astronomer.io > Search `sensor`

```
from airflow.decorators import task
from airflow.sensors.base import PokeReturnValue
from airflow.hooks.base import BaseHook

@task.sensor(
    poke_interval = 30,
    timeout = 300,
    mode = 'poke'
)
def is_api_available() -> PokeReturnValue:
    import requests
    api = BaseHook.get_connection("api_unique_id")
    url = f"{api.host}{api.extra_dejson['endpoint']}"
    response = requests.get(url, headers = api.extra_dejson['headers'])

    condition = <condition to return if response is valid>
    return PokeReturnValue(is_done = condition, xcom_value = url)

is_api_available()
```

- `poke_interval` - after how long the sensor should listen to the source again - every `30 seconds`.
- `timeout` - when the sensor should timeout - `300 sec`.
- `poke` is the default mode.
- We don't have to hard-code the connection. Instead, we can configure it from the UI - `Admin` > `Connections` > `+` -
    - Rest API connection example (referenced in `is_api_available`) -
    ```
    Connection Id - <api_unique_id>
    Connection Type - `HTTP`
    Host - <hostname>
    Extra - <endpoint_info>
    ```

    - `<endpoint_info>` will contain API info in json format as below -
    ```
    {
        "endpoint": "<api_endpoint>",
        "headers": {
            "Content-Type": "application/json"
        }
    }
    ```

> [!TIP]
> Generally, we place the DAG files inside `dags` package and all the python "callable" functions inside `include` package.

## Passing arguments to callable

```
get_stock_prices = PythonOperator(
    task_id = 'get_stock_prices',
    python_callable = _get_stock_prices,
    op_kwargs = { 'url': '{{ ti.xcom_pull(task_ids = 'is_api_available') }}', 'endpoint': 'metrics'}
)
```

> [!TIP]
> Templating - In airflow, the code inside `{{ ... }}` is evaluated at runtime.

- Then in the callable -
```
def _get_stock_prices(url, endpoint):
    pass
```

