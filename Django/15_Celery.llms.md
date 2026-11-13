# Celery

**Celery** is an asynchronous task queue/job queue system based on distributed message passing. It’s widely used in Django and other Python web frameworks to manage background tasks, perform distributed computation, and handle scheduled tasks. Celery is particularly powerful for running time-consuming tasks asynchronously to improve user experience and system responsiveness.

Author

Benedict Thekkel

### 1. **What is Celery?**

**Celery** is a distributed system to process messages asynchronously by running tasks in the background. It uses a **message broker** to deliver messages between the main application and worker nodes. It is designed to handle distributed task processing at scale.

### Key Features:

- **Asynchronous Task Execution**: Run tasks in the background, freeing up your main application.
- **Task Scheduling**: Execute tasks at specific intervals or after certain delays (like cron jobs).
- **Distributed Processing**: Run tasks across multiple worker nodes for scalability and fault tolerance.
- **Task Retrying**: Tasks can be automatically retried if they fail.

### 2. **Celery Architecture**

#### a. **Workers**

Workers are background processes that execute tasks. You can have multiple workers across different machines to handle tasks in parallel.

#### b. **Message Broker**

A message broker (such as **Redis**, **RabbitMQ**, or **Amazon SQS**) is used to send messages from your main application to Celery workers. The broker handles task queuing and routing.

#### c. **Result Backend**

The result backend stores the results of tasks. Celery supports several backends like **Redis**, **Django ORM**, **AMQP**, **Memcached**, and **SQLAlchemy**.

### 3. **Installation and Setup**

#### a. **Install Celery**

To install Celery in your project, use `pip`:

``` bash
pip install celery
```

#### b. **Basic Celery Setup for a Django Project**

1.  **Create a `celery.py` file** in your Django project directory (where `settings.py` resides):

    ``` python
    # project/celery.py
    from __future__ import absolute_import, unicode_literals
    import os
    from celery import Celery

    # set the default Django settings module for the 'celery' program.
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')

    app = Celery('project')

    # Using a string here means the worker doesn’t have to serialize
    # the configuration object to child processes.
    app.config_from_object('django.conf:settings', namespace='CELERY')

    # Load task modules from all registered Django app configs.
    app.autodiscover_tasks()
    ```

2.  **Update `__init__.py`** to ensure that Celery is imported when Django starts.

    ``` python
    # project/__init__.py

    from __future__ import absolute_import, unicode_literals

    # This will make sure the app is always imported when
    # Django starts so that shared_task will use this app.
    from .celery import app as celery_app

    __all__ = ('celery_app',)
    ```

3.  **Configure Celery in `settings.py`**:

    Add the configuration for the message broker (for example, Redis):

    ``` python
    # project/settings.py

    # Celery settings
    CELERY_BROKER_URL = 'redis://localhost:6379/0'  # Redis as message broker
    CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'  # Redis to store task results
    CELERY_ACCEPT_CONTENT = ['json']
    CELERY_TASK_SERIALIZER = 'json'
    CELERY_RESULT_SERIALIZER = 'json'
    CELERY_TIMEZONE = 'UTC'
    ```

#### c. **Starting Celery Workers**

Once you’ve configured Celery, you need to run the Celery workers to process the tasks. Run this command from the Django project root:

``` bash
celery -A project worker --loglevel=info
```

This command starts a Celery worker that will process tasks defined in your project. The `--loglevel=info` flag ensures you see task output.

### 4. **Defining and Executing Tasks**

Tasks in Celery are just Python functions that you register as tasks using the `@task` decorator. You can create tasks in any app within the Django project.

#### a. **Defining a Task**

``` python
# tasks.py in any Django app

from celery import shared_task
from time import sleep

@shared_task
def add(x, y):
    sleep(10)  # Simulate a time-consuming task
    return x + y
```

- **`@shared_task`**: This decorator marks the function as a Celery task, meaning it can be executed asynchronously.

#### b. **Calling a Task**

To execute a Celery task asynchronously, you use the `delay()` method:

``` python
from .tasks import add

# Call the task asynchronously
result = add.delay(4, 6)

# You can also retrieve the result (if needed)
print(result.get())  # This will block until the task is done
```

This will queue the `add` task, and Celery workers will execute it in the background.

#### c. **Task Results**

You can track task execution status and retrieve the result using the task’s ID:

``` python
# Retrieve the result using the task ID
from celery.result import AsyncResult

result = AsyncResult(task_id)
if result.successful():
    print(result.result)
```

### 5. **Task Retries and Error Handling**

Celery allows you to automatically retry tasks that fail due to transient errors.

#### a. **Retrying a Task**

To make a task retryable, use the `retry()` method in the task function:

``` python
@shared_task(bind=True, max_retries=3)
def example_task(self):
    try:
        # Simulate a task that may fail
        risky_operation()
    except SomeError as exc:
        raise self.retry(exc=exc, countdown=5)
```

- **`max_retries=3`**: The task will be retried a maximum of 3 times.
- **`countdown=5`**: The task will wait 5 seconds before retrying.

#### b. **Error Handling**

If a task raises an exception, it can be caught and retried or handled with a fallback:

``` python
@shared_task(bind=True)
def risky_task(self):
    try:
        risky_operation()
    except Exception as exc:
        self.update_state(state='FAILURE', meta={'error': str(exc)})
        raise exc
```

- **`self.update_state()`**: Updates the task’s state manually. You can store custom error metadata using the `meta` argument.

### 6. **Periodic and Scheduled Tasks**

You can schedule tasks to run at specific times or intervals, similar to cron jobs.

#### a. **Using Celery Beat**

**Celery Beat** is a scheduler that kicks off tasks at regular intervals. It works alongside your Celery workers.

1.  **Install the necessary package**:

    ``` bash
    pip install django-celery-beat
    ```

2.  **Add `django_celery_beat` to your `INSTALLED_APPS`** in `settings.py`:

    ``` python
    INSTALLED_APPS = [
        # other apps
        'django_celery_beat',
    ]
    ```

3.  **Run migrations** for Celery Beat:

    ``` bash
    python manage.py migrate django_celery_beat
    ```

4.  **Configure a Periodic Task** using the Django Admin or in code:

    Example of a periodic task running every 10 minutes:

    ``` python
    from django_celery_beat.models import PeriodicTask, IntervalSchedule

    # Create an interval schedule (every 10 minutes)
    schedule, created = IntervalSchedule.objects.get_or_create(
        every=10,
        period=IntervalSchedule.MINUTES,
    )

    # Create the periodic task
    PeriodicTask.objects.create(
        interval=schedule,                  # Use the schedule created above
        name='My periodic task',
        task='myapp.tasks.my_task',  # The name of the task function
    )
    ```

5.  **Start the Celery Beat Scheduler**:

    ``` bash
    celery -A project beat --loglevel=info
    ```

    This command will run the scheduler, which triggers periodic tasks based on your schedule.

#### b. **Using the `crontab` Scheduler**

You can also configure tasks to run at specific times using cron-like syntax:

``` python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'task_name': {
        'task': 'myapp.tasks.my_task',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),  # Every Monday at 7:30 AM
    },
}
```

### 7. **Optimizing Celery for Production**

#### a. **Concurrency**

You can control how many worker processes or threads Celery spawns using the `--concurrency` option when starting workers:

``` bash
celery -A project worker --concurrency=4
```

This will start 4 worker processes.

#### b. **Prefetch Limit**

Celery workers can fetch multiple tasks in advance, but this may overload workers in some cases. You can limit the number of prefetched tasks with `CELERYD_PREFETCH_MULTIPLIER`:

``` python
app.conf.worker_prefetch_multiplier = 1  # Pref

etch only 1 task at a time
```

#### c. **Task Time Limits**

If a task takes too long, you can set time limits to prevent worker exhaustion:

``` python
app.conf.task_soft_time_limit = 60  # Raise exception after 60 seconds
app.conf.task_time_limit = 120      # Hard limit (kill task after 120 seconds)
```

#### d. **Monitoring and Admin**

Use **Flower**, a web-based tool to monitor Celery workers and tasks:

1.  Install Flower:

    ``` bash
    pip install flower
    ```

2.  Start Flower:

    ``` bash
    celery -A project flower
    ```

    Flower provides a web interface that shows the current status of workers, tasks, and queues. You can view task progress, task failures, and retry counts.

### 8. **Broker Options**

Celery supports a wide range of message brokers, including:

- **Redis**: A simple key-value store that can act as a message broker.
- **RabbitMQ**: A robust, feature-rich message broker that supports advanced features like routing and exchanges.
- **Amazon SQS**: A scalable message queue service.

#### Redis Example:

``` python
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

#### RabbitMQ Example:

``` python
CELERY_BROKER_URL = 'amqp://guest:guest@localhost:5672//'
```

### 9. **Best Practices**

- **Use Task Queues**: Use task queues to categorize and separate different types of tasks, especially if they have different priority or resource requirements.

  Example:

  ``` python
  celery -A project worker -Q queue_name
  ```

- **Idempotent Tasks**: Ensure tasks are idempotent, meaning they can run multiple times without unintended side effects. This is essential because Celery retries tasks after failures.

- **Error Handling**: Always handle potential errors inside tasks using `try/except` and consider using `retry()` for transient issues.

- **Monitor Tasks**: Use monitoring tools like **Flower** or **Prometheus** to track worker health and task performance.

Back to top
