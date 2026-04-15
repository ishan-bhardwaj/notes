# Creating DAGs

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

> [!TIP]
> Airflow can send notifications to external systems (eg, email, slack etc) using "Notifiers". To add a notifier - add parameters `on_success_callback` & `on_failure_callback` in `@dag` decorator.

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

## Docker Operator

```
DockerOperator(
    task_id='format_prices',
    image='airflow/stock-app',
    container_name='format_prices',
    api_version='auto',
    auto_remove='success,
    docker_url='tcp://docker-proxy:2375',
    network_mode='container:spark-master'   # same network as spark-master container is using,
    tty=True,
    xcom_all=False,
    mount_tmp_dir=False,
    environment={
        'SPARK_APPLICATION_ARGS': '{{ ti.xcom_pull(task_ids='store_prices') }}'
    }
)
```
