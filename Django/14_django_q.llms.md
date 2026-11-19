# django_q

Django-Q is a task queue and scheduler designed for Django, allowing you to run tasks asynchronously in the background, schedule recurring tasks, and handle distributed task queues. It’s an alternative to tools like Celery, providing a simpler, more Pythonic interface for handling asynchronous tasks in Django projects.

Author

Benedict Thekkel

### 1. **Core Concepts of Django-Q**

- **Task Queue**: You can offload long-running or resource-heavy tasks to the background using Django-Q. These tasks are processed asynchronously without blocking the main Django app.
- **Scheduling**: Django-Q has built-in support for periodic or scheduled tasks, similar to cron jobs.
- **Result Handling**: It supports tracking the result of tasks and retrieving them later.
- **Clusters**: Django-Q operates with “clusters” of worker processes that handle tasks in parallel, allowing you to scale task processing horizontally.
- **Multiple Backends**: Django-Q supports Redis, ORM, and other storage options for task persistence and broker functionality.
- **Failover and Recovery**: It has robust task recovery mechanisms, ensuring tasks are retried in case of failure.

### 2. **Installation**

To install **Django-Q**, you can use `pip`:

``` bash
pip install django-q
```

Once installed, add `django_q` to your `INSTALLED_APPS` in `settings.py`:

``` python
INSTALLED_APPS = [
    # Other apps...
    'django_q',
]
```

After that, run the necessary migrations:

``` bash
python manage.py migrate
```

### 3. **Basic Setup**

After installation, configure Django-Q in `settings.py`. You need to define the broker, which will handle communication between Django and the background workers. Common brokers include **Redis** and **Django ORM**.

#### Example configuration using Redis:

``` python
Q_CLUSTER = {
    'name': 'DjangoQCluster',
    'workers': 4,  # Number of workers to handle tasks
    'recycle': 500,  # How many tasks before a worker is recycled
    'timeout': 60,  # Maximum time to wait for a task
    'queue_limit': 50,  # Limit the number of tasks in the queue
    'bulk': 10,  # Process tasks in bulk
    'orm': 'default',  # Use Django ORM for task storage
    'redis': {
        'host': 'localhost',
        'port': 6379,
        'db': 0,
    }
}
```

For **Django ORM** as the broker (no external dependencies):

``` python
Q_CLUSTER = {
    'name': 'DjangoQCluster',
    'workers': 4,
    'timeout': 60,
    'queue_limit': 50,
    'bulk': 10,
    'orm': 'default'  # Use Django ORM to store tasks and results
}
```

### 4. **Running Django-Q**

Start Django-Q’s cluster to begin processing tasks by running:

``` bash
python manage.py qcluster
```

The cluster will spawn worker processes that consume tasks from the queue and execute them in the background.

### 5. **Asynchronous Tasks**

To define an asynchronous task, you create a Python function and call it asynchronously using the `async_task()` function.

#### Example:

``` python
from django_q.tasks import async_task

def send_email(user_id):
    # Send email logic here
    print(f'Sending email to user {user_id}')

# Running the task asynchronously
async_task('myapp.tasks.send_email', user_id)
```

- **`async_task()`**: Schedules the function to be executed in the background.

#### Tracking Task Results

You can track the result of a task if needed. Use the `result` function to retrieve the result:

``` python
from django_q.tasks import result

task_id = async_task('myapp.tasks.send_email', user_id)

# Later in the code, you can fetch the result
res = result(task_id)
if res:
    print(f'Task Result: {res}')
```

If the task fails, Django-Q will retry the task until it either succeeds or exceeds the retry limit.

### 6. **Scheduled Tasks**

Django-Q has built-in support for **scheduled tasks** (periodic tasks). You can schedule tasks to run at specific intervals, similar to cron jobs.

#### Example:

``` python
from django_q.models import Schedule

# Schedule a task to run every minute
Schedule.objects.create(
    func='myapp.tasks.send_email',
    args='user_id',
    schedule_type=Schedule.MINUTES,
    minutes=1,
    repeats=-1,  # Repeat indefinitely
)
```

- **`Schedule.MINUTES`**: Schedule type to run tasks periodically (other options include `Schedule.HOURLY`, `Schedule.DAILY`, `Schedule.WEEKLY`).
- **`repeats`**: Set to `-1` to repeat indefinitely, or to a positive integer to repeat a fixed number of times.

#### Available Schedule Types:

- **One-time task**: `Schedule.ONCE`
- **Repeating tasks**:
  - `Schedule.MINUTES`
  - `Schedule.HOURLY`
  - `Schedule.DAILY`
  - `Schedule.WEEKLY`
  - `Schedule.MONTHLY`

### 7. **Using ORM as the Broker**

If you prefer not to use Redis or another external service, Django-Q can use the Django ORM as the broker for storing tasks. This is a great option for simpler setups or testing environments.

To use the ORM as the broker, update your settings:

``` python
Q_CLUSTER = {
    'name': 'DjangoQCluster',
    'workers': 4,
    'timeout': 60,
    'queue_limit': 50,
    'orm': 'default'  # ORM mode
}
```

Tasks and results will now be stored in the database, using Django models.

### 8. **Task Retry and Timeout**

Django-Q automatically retries tasks if they fail, with configurable retry limits and timeouts.

- **Timeout**: Defined in the cluster configuration (`timeout` key). If a task exceeds this time, it is marked as failed and retried.
- **Retries**: If a task fails, Django-Q retries the task based on the retry policy you define.

#### Example:

``` python
Q_CLUSTER = {
    'timeout': 60,  # Tasks timeout after 60 seconds
    'retry': 2,  # Retries each task twice if it fails
}
```

### 9. **Task Chaining and Grouping**

You can chain multiple tasks together or group tasks to run them in parallel.

#### Task Chaining:

``` python
from django_q.tasks import chain

# Chain tasks, one task will be executed after the previous one completes
chain('myapp.tasks.task_one', 'myapp.tasks.task_two', args=(1,))
```

#### Task Grouping (Run in Parallel):

``` python
from django_q.tasks import async_task, result_group

group = async_task('myapp.tasks.task_one', 1, group='task_group')
async_task('myapp.tasks.task_two', 2, group='task_group')

# Retrieve the results for the group
results = result_group('task_group')
for r in results:
    print(f'Task Result: {r}')
```

### 10. **Task Priority**

Django-Q supports priority queues, where higher priority tasks are processed before lower-priority ones.

#### Example:

``` python
# Set a high priority for the task
async_task('myapp.tasks.send_email', user_id, priority=1)
```

Lower numbers have higher priority (1 being the highest).

### 11. **Monitoring and Admin**

Django-Q provides a Django Admin interface for monitoring tasks, schedules, and clusters. You can view:

- **Failed tasks**
- **Running tasks**
- **Scheduled tasks**
- **Task results**

In your `admin.py`:

``` python
from django_q.admin import TaskAdmin, ScheduleAdmin
from django_q.models import Task, Schedule

admin.site.register(Task, TaskAdmin)
admin.site.register(Schedule, ScheduleAdmin)
```

This will give you visibility into the task queue and schedules via the Django Admin.

### 12. **Advanced Settings**

Django-Q comes with a variety of settings to fine-tune task processing and behavior.

#### Advanced Options:

``` python
Q_CLUSTER = {
    'name': 'DjangoQCluster',
    'workers': 4,  # Number of worker processes
    'timeout': 60,  # Time limit per task
    'recycle': 500,  # Restart worker after processing this many tasks
    'compress': True,  # Compress task data to reduce memory usage
    'save_limit': 100,  # Only keep 100 task results in the database
    'group_save_limit': 100,  # Limit the number of task group results
    'queue_limit': 50,  # Maximum number of tasks in the queue
    'cpu_affinity': 1,  # Allocate one core for worker processing
    'log_level': 'DEBUG',  # Log level (DEBUG, INFO, WARNING, etc.)
}
```

- **`recycle`**: Recycles workers after a specified number of tasks to avoid memory leaks.
- **`compress`**: Compresses task data to reduce memory footprint.
- **`save_limit`**: Limits how many task results are saved (useful for reducing database size).
- **`queue_limit`**: Prevents too many tasks from being queued up

.

### 13. **Failover and High Availability**

Django-Q offers **failover support** in case a task or worker process fails. It retries tasks and restarts failed workers as needed.

- **Distributed Mode**: You can deploy Django-Q in a distributed mode by running multiple clusters across different servers. This allows horizontal scaling and high availability.

### 14. **Logging and Debugging**

Django-Q provides detailed logging options, which you can configure in your settings:

``` python
Q_CLUSTER = {
    'log_level': 'DEBUG',  # DEBUG, INFO, WARNING, ERROR
    'log_file': '/path/to/your/logfile.log',  # Specify a log file
}
```

Logs include task execution details, errors, and retries, which can help in debugging and monitoring your task queue.

### 15. **Django Signals**

Django-Q integrates with Django’s signals framework. For example, you can trigger tasks based on model signals like `post_save`:

``` python
from django.db.models.signals import post_save
from django.dispatch import receiver
from django_q.tasks import async_task
from myapp.models import MyModel

@receiver(post_save, sender=MyModel)
def my_model_post_save(sender, instance, **kwargs):
    # Trigger async task when a MyModel object is saved
    async_task('myapp.tasks.process_data', instance.id)
```

### Conclusion

**Django-Q** is a powerful and flexible task queue and scheduler for Django applications, providing asynchronous task execution, scheduling, and result tracking with ease. It is particularly well-suited for smaller to mid-sized projects, or as a simpler alternative to Celery, thanks to its minimal configuration requirements and built-in ORM support.

Back to top
