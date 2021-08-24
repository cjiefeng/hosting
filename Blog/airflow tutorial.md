# Apache Airflow (v2.1.2)

Apache Airflow is a workflow management system. It can be used for various purpose like executing task periodically, monitoring execution status, performing task in a fixed orders, so and so forth just to name a few.

Most part of Airflow is python, but Airflow can be used to execute programs of any language, even shell (more on that later). However before we start using Airflow, one important thing we need to understand is directed acyclic graph (DAG).

## DAG
A finite directed graph with no cycles. There is not path that exist where u start at a vertex, take > 1 step and end up where you started. Refer to below example:

![DAG examples](https://raw.githubusercontent.com/cjiefeng/hosting/master/Blog%20Images/DAG%20example.png)

Notice in `Graph 2`, v3 -> v5 -> v6 -> v3 -> ... formed a cycle, this is a counter example of a DAG.

In Airflow, all workflows are DAG. Each vertex represents a task. This task could be perform anything such as executing bash scripts, python functions, updating database data etc.

##  Installation
1. Let's get started. First, set a home directory for Airflow. I've set it as airflow/ in the current directory. It defaults to `~/airflow`.
  ```
  export AIRFLOW_HOME=$(pwd)/airflow
  ```

2. Next, create a virtual environment if you prefer, it is just good practice to isolate python environments. Go ahead and activate it after the virtual environment set up is complete.
  ```
  python3 -m venv venv
  source venv/bin/activate
  ```

3. Install the required packages using pip, taking into account airflow version (`v2.1.2` in this example) and python version (I'm using `v3.9.5`) that you have.
  ```
  # e.g. 3.9
  PYTHON_VERSION = "$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"

  CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-2.1.2/constraints-${PYTHON_VERSION}.txt"

  pip install apache-airflow=2.1.2 --constraint "${CONSTRAINT_URL}"
  ```

4. Initialise a database for airflow. It defaults to SQLite, lightweight but 0 concurrency, however this is enough for this tutorial, you can change to database of your choice in the settings later. Let's also create an admin user as we need it to log into airflow web server later.
  ```
  airflow db init

  airflow users create -u admin -f admin -l admin -r Admin -e admin
  ```
Go ahead and key in any password and confirmation.

5. We are ready to run airflow now. The command given below runs web server as daemon. Otherwise you could run in 2 different terminals without `-D` flag. Airflow web server defaults to port 8080 but you can simply change it with `-p` flag.
  ```
  airflow webserver -D & airflow scheduler
  ```

6. Visit `localhost:8080` or the port that you have defined earlier to see Airflow UI. Log in with the credentials that we have created earlier.

## Creating a DAG
Airflow preloads with plenty of examples but I learn by doing so let's create a simple example.

Head back to the `AIRFLOW_HOME` directory and create a new directory `dags/`. This is where we will store our custom DAG workflow. Inside which we will create a python file `simple_dag.py`

I would disable `load_examples` in `airflow.cfg` here since I'm not interested in the preloaded examples.

The steps given below results in a messy `py` file, refer to my [github file](https://github.com/cjiefeng/airflow-tutorial/blob/master/airflow/dags/simple_dag.py) if you want to skip all of the explanation and get right into it.

1. Let's define some python function to execute. Notice `timing()` will return an error with `minute == 0`. This is on purpose so please bear with me for the moment.
  ```
  import datetime as dt  

  from airflow import DAG

  def timing():
       now = dt.datetime.now().time()
       minute = now.minute
       return 100 / minute  # fail when 0min

  def respond():
       return 'Greet Responded Again'
  ```

2. We can set some default argument for our DAG as such. Set the `start_date` to a time in the past, airflow will do something called `Backfill`.
  ```
  default_args = {
      'owner': 'airflow',
      'start_date': dt.datetime(2021, 8, 23, 13, 00, 00),
      'concurrency': 1,
      'retries': 0
  }
  ```

3. Next we can define the tasks that we want to run with a DAG id `simple_dag`. Take note of the id as we will use it to locate our DAG later. Let's also import the `operators` so airflow can recognise the how to run them.
  ```
  from airflow.operators.bash_operator import BashOperator
  from airflow.operators.python_operator import PythonOperator

  with DAG('simple_dag',
           default_args=default_args,
           schedule_interval='*/30 * * * *',
      ) as dag:

      opr_fail = PythonOperator(task_id='py_fail',
                                python_callable=timing)

      opr_sleep = BashOperator(task_id='sh_sleep',
                               trigger_rule="all_done",
                               bash_command='sleep 5')

      opr_sleep_strict = BashOperator(task_id='sh_sleep_strict',
                                      trigger_rule="all_success",
                                      bash_command='sleep 5')

      opr_hello = BashOperator(task_id='sh_hi',
                               bash_command='echo "Hi!!"')

      opr_respond = PythonOperator(task_id='py_respond',
                                   trigger_rule="all_done",
                                   python_callable=respond)
  ```
  Simply put, `PythonOperator` executes python functions, `BashOperator` executes bash commands. There are other operators available by airflow, or you could write your own operator if you wish to.

  Amongst all the tasks, something I would like to mention is `trigger_rule`. `all_success` means the task will only be triggered with all its prior tasks are successful, `all_done` means task will trigger regardless of its prior tasks success, as long as they are done.

4. Lastly, we can create our DAG using bitwise operator. The first line `>> [x, y]` means that we fan out to 2 different tasks.
```
opr_fail >> [opr_sleep, opr_sleep_strict]
opr_sleep >> opr_respond << opr_hello
opr_sleep_strict >> opr_respond
```
  This results in a DAG show below.  
  ![simple_dag diagram](https://raw.githubusercontent.com/cjiefeng/hosting/master/Project%20Images/DAG.png)

## Airflow UI
Going back to our webserver `localhost:8080` using default port, let's search for our DAG.

- If you have not disabled `load_examples` in `airflow.cfg`, you might need to search for our DAG id (from step 3 above) from the top right hand corner.

![airflow webpage](https://raw.githubusercontent.com/cjiefeng/hosting/master/Blog%20Images/airflow%20webpage.png)

- We can pause/unpause our DAG by using the toggle buttone beside the DAG id. We can also manually trigger DAG using the action buttons.

![DAG toggle](https://raw.githubusercontent.com/cjiefeng/hosting/master/Blog%20Images/DAG%20toggle.png) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
![DAG actions](https://raw.githubusercontent.com/cjiefeng/hosting/master/Blog%20Images/DAG%20action.png)

- Clicking on the DAG id allows us to see more information about the DAG. Personally I feel that the `Tree View` is confusing so I will skip it for now and go into `Graph View`.

![DAG Graph View](https://raw.githubusercontent.com/cjiefeng/hosting/master/Blog%20Images/DAG%20graph%20view.png)

- Each vertex can tell us what has happened and the status of the task. By cliking on them, we can also see the respective output logs.

- Recall that `def timing()` is set to fail when `minute==0`. From the above DAG, we can see that `sh_sleep` was executed as its `trigger_rule='all_done'`. On the flip side, take a look at `sh_sleep_strict` which was not executed because its prior job has failed.

## Conclusion
Hopefully this has scratched the surface of Apache Airflow for you, it has definitely done so for me. I will continue to explore and perhaps use it for complicated projects.

[Link](https://github.com/cjiefeng/airflow-tutorial) to my github is here for this short tutorial.

That's it for now.  
J
