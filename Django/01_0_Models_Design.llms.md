# Model Design

Model Design

Author

Benedict Thekkel

## Basic Model Definition

> Django models are defined as Python classes that inherit from `django.db.models.Model`. Each attribute of the class represents a database field.

------------------------------------------------------------------------

``` python
from django.db import models

class MyModel(models.Model):
    my_field = models.CharField(max_length=100)
    my_number = models.IntegerField()
```

## Field Types

> This is a comprehensive table of **Django model fields**, grouped by **category**, including their **parameters**, **description**, and **usage**.

------------------------------------------------------------------------

### 🏗️ **Django Model Field Types**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **AutoField** | `models.AutoField` | Auto-incrementing primary key (default for `id` fields). | `primary_key` |
| **BigAutoField** | `models.BigAutoField` | 64-bit auto-incrementing primary key. | `primary_key` |
| **SmallAutoField** | `models.SmallAutoField` | Small integer auto-incrementing field. | `primary_key` |

------------------------------------------------------------------------

### 📝 **Text-Based Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **CharField** | `models.CharField` | Fixed-length text field. Requires `max_length`. | `max_length`, `blank`, `null`, `default` |
| **TextField** | `models.TextField` | Variable-length, large text field. | `blank`, `null`, `default` |
| **SlugField** | `models.SlugField` | Short label for URLs, typically used for slugs. | `max_length`, `unique`, `blank` |
| **EmailField** | `models.EmailField` | Text field that validates email addresses. | `max_length`, `blank`, `null` |
| **URLField** | `models.URLField` | Validates URLs. | `max_length`, `blank`, `null` |
| **UUIDField** | `models.UUIDField` | Stores UUIDs as strings. | `primary_key`, `default` |

------------------------------------------------------------------------

### 🔢 **Numeric Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **IntegerField** | `models.IntegerField` | Integer values. | `blank`, `null`, `default` |
| **BigIntegerField** | `models.BigIntegerField` | 64-bit integer values. | `blank`, `null`, `default` |
| **PositiveIntegerField** | `models.PositiveIntegerField` | Positive integers only. | `blank`, `null`, `default` |
| **PositiveSmallIntegerField** | `models.PositiveSmallIntegerField` | Small positive integers. | `blank`, `null`, `default` |
| **SmallIntegerField** | `models.SmallIntegerField` | Small integer values. | `blank`, `null`, `default` |
| **FloatField** | `models.FloatField` | Floating-point numbers. | `blank`, `null`, `default` |
| **DecimalField** | `models.DecimalField` | Fixed-point decimal numbers (exact precision). | `max_digits`, `decimal_places`, `blank`, `null` |

------------------------------------------------------------------------

### 📅 **Date/Time Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **DateField** | `models.DateField` | Date (year, month, day). | `auto_now`, `auto_now_add`, `blank`, `null` |
| **TimeField** | `models.TimeField` | Time (hour, minute, second). | `auto_now`, `auto_now_add`, `blank`, `null` |
| **DateTimeField** | `models.DateTimeField` | Date and time combined. | `auto_now`, `auto_now_add`, `blank`, `null` |
| **DurationField** | `models.DurationField` | Stores periods of time (timedelta). | `blank`, `null` |

------------------------------------------------------------------------

### 🎨 **File/Media Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **FileField** | `models.FileField` | Uploads files to a specified directory. | `upload_to`, `blank`, `null` |
| **ImageField** | `models.ImageField` | Inherits from `FileField`, with image validation. | `upload_to`, `blank`, `null` |

------------------------------------------------------------------------

### ✅ **Boolean Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **BooleanField** | `models.BooleanField` | True/False values. | `default` |
| **NullBooleanField** | `models.NullBooleanField` | True/False/Null (deprecated in Django 3.1+). | `default` |

------------------------------------------------------------------------

### 🔗 **Relational Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **ForeignKey** | `models.ForeignKey` | Many-to-one relationship. | `to`, `on_delete`, `related_name` |
| **OneToOneField** | `models.OneToOneField` | One-to-one relationship. | `to`, `on_delete`, `related_name` |
| **ManyToManyField** | `models.ManyToManyField` | Many-to-many relationship. | `to`, `related_name`, `through` |

------------------------------------------------------------------------

### 📦 **Miscellaneous / Special Fields**

| **Field Type** | **Field Class** | **Description** | **Common Arguments** |
|----|----|----|----|
| **GenericIPAddressField** | `models.GenericIPAddressField` | Stores IPv4 or IPv6 addresses. | `protocol`, `blank`, `null` |
| **JSONField** | `models.JSONField` | Stores structured JSON data (Django 3.1+). | `default`, `blank`, `null` |

------------------------------------------------------------------------

### ✅ **Arguments Explained (Quick Reference)**

| **Argument** | **Description** |
|----|----|
| `max_length` | Maximum length of the field (required for `CharField` and `SlugField`). |
| `blank` | Boolean; whether the field is allowed to be blank in forms. |
| `null` | Boolean; whether the field is allowed to be NULL in the database. |
| `default` | Default value for the field. |
| `choices` | Choices for limiting field values (e.g., for `CharField` or `IntegerField`). |
| `primary_key` | Boolean; whether the field is the primary key for the model. |
| `unique` | Boolean; whether this field must be unique throughout the table. |
| `upload_to` | Function or string to specify upload location (for `FileField`/`ImageField`). |
| `auto_now` | Boolean; updates field to `now()` every time the object is saved. (Date/Time) |
| `auto_now_add` | Boolean; sets field to `now()` when object is first created. (Date/Time) |

------------------------------------------------------------------------

### ✅ **Common Use Cases and Recommendations**

| **Use Case**             | **Recommended Field**               |
|--------------------------|-------------------------------------|
| Text content             | `CharField`, `TextField`            |
| Numeric data             | `IntegerField`, `DecimalField`      |
| Timestamps               | `DateTimeField` with `auto_now_add` |
| User-uploaded files      | `FileField`, `ImageField`           |
| IP address logging       | `GenericIPAddressField`             |
| JSON or dynamic settings | `JSONField`                         |
| UUID primary keys        | `UUIDField` with `primary_key=True` |
| Foreign model relations  | `ForeignKey`, `OneToOneField`       |

------------------------------------------------------------------------

### 🔨 **New Fields in Recent Django Versions**

| **Version** | **Field** | **Notes** |
|----|----|----|
| Django 3.1 | `JSONField` | Cross-database support (previously Postgres only) |
| Django 3.2+ | `BigAutoField` | Default primary key field for new Django projects |
| Django 4.0 | `SmallAutoField` | AutoField for small integers (optimizes small datasets) |

------------------------------------------------------------------------

### ✅ **Summary Table by Category**

| **Category** | **Fields** |
|----|----|
| **Text** | `CharField`, `TextField`, `SlugField`, `EmailField`, `URLField`, `UUIDField` |
| **Numeric** | `IntegerField`, `FloatField`, `DecimalField`, `SmallIntegerField`, `BigIntegerField`, `PositiveIntegerField` |
| **Date/Time** | `DateField`, `TimeField`, `DateTimeField`, `DurationField` |
| **Boolean** | `BooleanField`, `NullBooleanField` |
| **File/Media** | `FileField`, `ImageField` |
| **Relations** | `ForeignKey`, `OneToOneField`, `ManyToManyField` |
| **Special** | `GenericIPAddressField`, `JSONField` |

------------------------------------------------------------------------

## Meta Options

> Meta options are used to define model-level metadata, such as ordering, verbose names, and database table names.

------------------------------------------------------------------------

``` python
class MyModel(models.Model):
    my_field = models.CharField(max_length=100)

    class Meta:
        ordering = ['my_field']
        verbose_name = 'My Model'
        verbose_name_plural = 'My Models'
```

## Model Relationships

Django ORM supports three types of relationships: - One-to-One (OneToOneField): Each instance of a model is related to one instance of another model. - Many-to-One (ForeignKey): Many instances of a model are related to one instance of another model. - Many-to-Many (ManyToManyField): Many instances of a model are related to many instances of another model.

``` python
class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
```

## Field Options

> Field options allow you to specify constraints and attributes for model fields.

------------------------------------------------------------------------

- max_length: Maximum length of the field (for CharField and TextField).
- default: Default value for the field.
- null: If True, Django will store empty values as NULL in the database.
- blank: If True, the field is allowed to be blank.
- choices: A list of valid choices for this field.
- unique: If True, this field must be unique throughout the table.

## Model Methods

> You can add methods to your model to encapsulate business logic or utility functions.

------------------------------------------------------------------------

``` python
class Book(models.Model):
    title = models.CharField(max_length=100)

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return f"/books/{self.id}/"
```

## Custom Managers and QuerySets

> Custom managers and querysets allow you to define reusable query logic.

------------------------------------------------------------------------

``` python
class BookManager(models.Manager):
    def published(self):
        return self.filter(published=True)

class Book(models.Model):
    title = models.CharField(max_length=100)
    published = models.BooleanField(default=False)

    objects = BookManager()
```

## Model Inheritance

> Django supports model inheritance, allowing you to reuse common fields and methods.

------------------------------------------------------------------------

- Abstract Base Classes: Used when you want to put some common information into a base class.
- Multi-table Inheritance: Each model in the hierarchy gets its own database table.
- Proxy Models: Used to modify the Python-level behavior without changing the model’s fields.

``` python
class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        abstract = True

class Student(CommonInfo):
    grade = models.CharField(max_length=2)
```

## Signals

> Signals allow decoupled applications to get notified when certain actions occur (e.g., a model instance is saved or deleted).

------------------------------------------------------------------------

``` python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Book)
def book_saved(sender, instance, **kwargs):
    print(f"Book saved: {instance.title}")
```

## Model Validations

> You can define custom validation logic within your model.

------------------------------------------------------------------------

``` python
class Book(models.Model):
    title = models.CharField(max_length=100)
    publication_date = models.DateField()

    def clean(self):
        if self.publication_date > datetime.date.today():
            raise ValidationError('Publication date cannot be in the future.')
```

Back to top
