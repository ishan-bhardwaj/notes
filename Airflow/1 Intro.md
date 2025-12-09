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
