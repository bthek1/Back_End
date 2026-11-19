# Custom Commands

Creating Custom Commands

Author

Benedict Thekkel

## Basic Setup

### Create a Management Directory

> Inside your Django app, create a directory named `management/commands/`.

``` sh
mkdir -p myapp/management/commands/
```

### Add `__init__.py` Files

> Add `__init__.py` files to the `management` and `commands` directories to make them Python packages.

``` sh
touch myapp/management/__init__.py
touch myapp/management/commands/__init__.py
```

### Create Your Command File

> Create a new Python file in the `commands` directory. The file name will be the command name.

``` sh
touch myapp/management/commands/my_custom_command.py
```

## Writing the Custom Command

### Basic Command Structure

> Define the command by creating a subclass of `BaseCommand` or `AppCommand` from `django.core.management.base`

``` python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Description of your custom command'

    def handle(self, *args, **kwargs):
        self.stdout.write('Executing custom command...')
```

### Adding Arguments

> Add arguments and options to the command.

``` python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Description of your custom command'

    def add_arguments(self, parser):
        parser.add_argument('arg1', type=str, help='Argument 1 description')
        parser.add_argument('--option1', type=str, help='Option 1 description', default='default_value')

    def handle(self, *args, **kwargs):
        arg1 = kwargs['arg1']
        option1 = kwargs['option1']
        self.stdout.write(f'Argument: {arg1}')
        self.stdout.write(f'Option: {option1}')
```

### Handling Command Logic

> Implement the logic inside the `handle` method.

``` python
from django.core.management.base import BaseCommand
from myapp.models import ExampleModel

class Command(BaseCommand):
    help = 'Update ExampleModel instances'

    def handle(self, *args, **kwargs):
        examples = ExampleModel.objects.all()
        for example in examples:
            example.description = 'Updated description'
            example.save()
        self.stdout.write(self.style.SUCCESS('Successfully updated all ExampleModel instances'))
```

## Advanced Command Features

### Using Transactions

> Ensure atomicity using database transactions.

``` python
from django.core.management.base import BaseCommand
from django.db import transaction
from myapp.models import ExampleModel

class Command(BaseCommand):
    help = 'Update ExampleModel instances'

    @transaction.atomic
    def handle(self, *args, **kwargs):
        examples = ExampleModel.objects.all()
        for example in examples:
            example.description = 'Updated description'
            example.save()
        self.stdout.write(self.style.SUCCESS('Successfully updated all ExampleModel instances'))
```

### Error Handling

> Implement error handling within your command

``` python
from django.core.management.base import BaseCommand
from django.db import transaction, DatabaseError
from myapp.models import ExampleModel

class Command(BaseCommand):
    help = 'Update ExampleModel instances'

    @transaction.atomic
    def handle(self, *args, **kwargs):
        try:
            examples = ExampleModel.objects.all()
            for example in examples:
                example.description = 'Updated description'
                example.save()
            self.stdout.write(self.style.SUCCESS('Successfully updated all ExampleModel instances'))
        except DatabaseError as e:
            self.stdout.write(self.style.ERROR(f'Error updating ExampleModel instances: {str(e)}'))
```

### Scheduling Commands

> Use task schedulers like cron or Windows Task Scheduler to run your custom commands at scheduled intervals.

``` sh
# Example cron job to run custom command every day at midnight
0 0 * * * /path/to/your/virtualenv/bin/python /path/to/your/project/manage.py my_custom_command
```

Back to top
