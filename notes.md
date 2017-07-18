Prerequisites

$ python3 --version
Python 3.6.0
$ virtualenv --version
15.1.0







Install Airflow
$ cd /path/to/my/airflow/workspace
$ virtualenv -p `which python3` venv
$ source venv/bin/activate
(venv) $

(venv) $ pip install airflow==1.8.0

(venv) $ cd /path/to/my/airflow/workspace
(venv) $ mkdir airflow_home
(venv) $ export AIRFLOW_HOME=`pwd`/airflow_home

(venv) $ airflow version

 airflow_home
 ├── airflow.cfg
 └── unittests.cfg







Initialize the Airflow DB

(venv) $ airflow initdb

airflow_home
├── airflow.cfg
├── airflow.db        <- Airflow SQLite DB
└── unittests.cfg


Start the Airflow web server

(venv) $ airflow webserver
  http://localhost:8080/admin/








Your first Airflow DAG

airflow_home
├── airflow.cfg
├── airflow.db
├── dags                <- Your DAGs directory
│   └── hello_world.py  <- Your DAG definition file
└── unittests.cfg








Running your DAG

IN A SECOND TERMINAL WITH THE SERVER RUNNING

$ cd /path/to/my/airflow/workspace
$ export AIRFLOW_HOME=`pwd`/airflow_home
$ source venv/bin/activate
(venv) $ airflow scheduler








Your first Airflow Operator

An Operator is an atomic block of workflow logic, which performs a single action.

Operators are written as Python classes (subclasses of BaseOperator), where the `__init__` function can be used to configure settings for the task and a method named execute is called when the task instance is executed.

Remember that since the execute method can retry many times, it should be idempotent.

airflow_home
├── airflow.cfg
├── airflow.db
├── dags
│   └── hello_world.py
│   └── test_operators.py  <- Second DAG definition file
├── plugins
│   └── my_operators.py    <- Your plugin file
└── unittests.cfg








Debugging an Airflow operator

With `airflow test` command, you can manually start a single operator in the context of a specific DAG run.

DAG run MUST HAVE EXISTED! For DAG run datetime use local time, not UTC (can be found via web UI)

The command takes 3 arguments: the name of the dag, the name of a task and a date associated with a particular DAG Run.

`(venv) $ airflow test my_test_dag my_first_operator_task 2017-03-18T18:00:00.0`

If you want to test a task from a particular DAG run, you can find the needed date value in the logs of a failing task instance.








Debugging an Airflow operator with IPython

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
