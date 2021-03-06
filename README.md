http://michal.karzynski.pl/blog/2017/03/19/developing-workflows-with-apache-airflow/


__________________________________________________________________________________________________


### Prerequisites

$ python3 --version
Python 3.6.0
$ virtualenv --version
15.1.0



__________________________________________________________________________________________________



### Install Airflow

$ cd /path/to/my/airflow/workspace
$ virtualenv -p `which python3` venv
$ source venv/bin/activate
(venv) $

(venv) $ pip install airflow==1.8.0

(venv) $ cd /path/to/my/airflow/workspace
(venv) $ mkdir airflow_home
(venv) $ export AIRFLOW_HOME=`pwd`/airflow_home

(venv) $ airflow version

```
 airflow_home
 ├── airflow.cfg
 └── unittests.cfg
```


__________________________________________________________________________________________________




### Initialize the Airflow DB

(venv) $ airflow initdb

```
airflow_home
├── airflow.cfg
├── airflow.db        <- Airflow SQLite DB
└── unittests.cfg
```

Start the Airflow web server

(venv) $ airflow webserver
  http://localhost:8080/admin/




__________________________________________________________________________________________________



### Your first Airflow DAG

```
airflow_home
├── airflow.cfg
├── airflow.db
├── dags                <- Your DAGs directory
│   └── hello_world.py  <- Your DAG definition file
└── unittests.cfg
```



__________________________________________________________________________________________________



### Running your DAG

IN A SECOND TERMINAL WITH THE SERVER RUNNING

$ cd /path/to/my/airflow/workspace
$ export AIRFLOW_HOME=`pwd`/airflow_home
$ source venv/bin/activate
(venv) $ airflow scheduler




__________________________________________________________________________________________________



### Your first Airflow Operator

An Operator is an atomic block of workflow logic, which performs a single action.

Operators are written as Python classes (subclasses of BaseOperator), where the `__init__` function can be used to configure settings for the task and a method named execute is called when the task instance is executed.

Remember that since the execute method can retry many times, it should be idempotent.
```
airflow_home
├── airflow.cfg
├── airflow.db
├── dags
│   └── hello_world.py
│   └── test_operators.py  <- Second DAG definition file
├── plugins
│   └── my_operators.py    <- Your plugin file
└── unittests.cfg
```



__________________________________________________________________________________________________



### Debugging an Airflow operator

With `airflow test` command, you can manually start a single operator in the context of a specific DAG run.

DAG run MUST HAVE EXISTED! For DAG run datetime use local time, not UTC (can be found via web UI)

The command takes 3 arguments: the name of the dag, the name of a task and a date associated with a particular DAG Run.

`(venv) $ airflow test my_test_dag my_first_operator_task 2017-03-18T18:00:00.0`

If you want to test a task from a particular DAG run, you can find the needed date value in the logs of a failing task instance.




__________________________________________________________________________________________________




### Debugging an Airflow operator with IPython

`(venv) $ pip install ipython`

You can then place IPython’s `embed()` command in your code, for example in the execute method of an operator

Example:
  ```
  def execute(self, context):
    log.info("Hello World!")

    from IPython import embed; embed()

    log.info('operator_param: %s', self.operator_param)
  ```

  You could of course also drop into Python’s interactive debugger pdb (import pdb; pdb.set_trace()) or the IPython enhanced version ipdb (import ipdb; ipdb.set_trace()).




__________________________________________________________________________________________________





### Your first Airflow Sensor

An Airflow Sensor is a special type of Operator, typically used to monitor a long running task on another system.

To create a Sensor, we define a subclass of BaseSensorOperator and override its poke function. The poke function will be called over and over every poke_interval seconds until one of the following happens:

poke returns True – if it returns False it will be called again.
poke raises an AirflowSkipException from airflow.exceptions – the Sensor task instance’s status will be set to Skipped.
poke raises another exception, in which case it will be retried until the maximum number of retries is reached.

Predefined sensors exist

To add a new Sensor to your my_operators.py file, add the following code:

```
from datetime import datetime
from airflow.operators.sensors import BaseSensorOperator

class MyFirstSensor(BaseSensorOperator):

    @apply_defaults
    def __init__(self, *args, **kwargs):
        super(MyFirstSensor, self).__init__(*args, **kwargs)

    def poke(self, context):
        current_minute = datetime.now().minute
        if current_minute % 3 != 0:
            log.info("Current minute (%s) not is divisible by 3, sensor will retry.", current_minute)
            return False

        log.info("Current minute (%s) is divisible by 3, sensor finishing.", current_minute)
        return True
```

Remember to also change the plugin class, to add the new sensor to the operators it exports:
```
class MyFirstPlugin(AirflowPlugin):
    name = "my_first_plugin"
```

You can now place the operator in your DAG:
```
from datetime import datetime
from airflow import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators import MyFirstOperator, MyFirstSensor


dag = DAG('my_test_dag', description='Another tutorial DAG',
          schedule_interval='0 12 * * *',
          start_date=datetime(2017, 3, 20), catchup=False)

dummy_task = DummyOperator(task_id='dummy_task', dag=dag)

sensor_task = MyFirstSensor(task_id='my_sensor_task', poke_interval=30, dag=dag)

operator_task = MyFirstOperator(my_operator_param='This is a test.',
                                task_id='my_first_operator_task', dag=dag)

dummy_task >> sensor_task >> operator_task
```

Restart your webserver and scheduler and try out your new workflow.




__________________________________________________________________________________________________




### Communicating between operators with Xcom

In most workflow scenarios downstream tasks will have to use some information from an upstream task.

Since each task instance will run in a different process, perhaps on a different machine, Airflow provides a communication mechanism called Xcom for this purpose.

Each task instance can store some information in Xcom using the xcom_push function and another task instance can retrieve this information using xcom_pull.

The information passed using Xcoms will be pickled and stored in the Airflow database (xcom table), so it’s better to save only small bits of information, rather then large objects.

Let’s enhance our Sensor, so that it saves a value to Xcom. We’re using the xcom_push() function which takes two arguments – a key under which the value will be saved and the value itself.

```
class MyFirstSensor(BaseSensorOperator):
    ...

    def poke(self, context):
        ...
        log.info("Current minute (%s) is divisible by 3, sensor finishing.", current_minute)
        task_instance = context['task_instance']
        task_instance.xcom_push('sensors_minute', current_minute)
        return True
```

Now in our operator, which is downstream from the sensor in our DAG, we can use this value, by retrieving it from Xcom.

Here we’re using the xcom_pull() function providing it with two arguments – the task ID of the task instance which stored the value and the key under which the value was stored.

```
class MyFirstOperator(BaseOperator):
    ...

    def execute(self, context):
        log.info("Hello World!")
        log.info('operator_param: %s', self.operator_param)
        task_instance = context['task_instance']
        sensors_minute = task_instance.xcom_pull('my_sensor_task', key='sensors_minute')
        log.info('Valid minute as determined by sensor: %s', sensors_minute)
```

If you trigger a DAG run now and look in the operator’s logs, you will see that it was able to display the value created by the upstream sensor.
