# Validator

Django **validators** are functions or classes that ensure a value meets certain conditions before being saved to the database. They are typically used in **models, forms, and serializers**

Author

Benedict Thekkel

## **1. Where to Use Validators**

Validators can be applied in: 1. **Django Models** (`models.py`) - Ensures data integrity at the database level. 2. **Django Forms** (`forms.py`) - Validates user input before saving. 3. **Django REST Framework (DRF) Serializers** (`serializers.py`) - Validates API input.

------------------------------------------------------------------------

## **2. Built-in Validators**

Django provides several **built-in validators** in `django.core.validators`.

### **2.1 Common Built-in Validators**

| Validator | Description | Example |
|----|----|----|
| `MinValueValidator(x)` | Ensures the value is at least `x` | `models.IntegerField(validators=[MinValueValidator(1)])` |
| `MaxValueValidator(x)` | Ensures the value is at most `x` | `models.IntegerField(validators=[MaxValueValidator(100)])` |
| `MinLengthValidator(x)` | Ensures the string length is at least `x` | `models.CharField(validators=[MinLengthValidator(5)])` |
| `MaxLengthValidator(x)` | Ensures the string length is at most `x` | `models.CharField(validators=[MaxLengthValidator(50)])` |
| `RegexValidator(regex, message, code)` | Ensures the value matches a regular expression | `models.CharField(validators=[RegexValidator(r'^\d{4}$', message="Enter 4 digits")])` |
| `EmailValidator` | Ensures the value is a valid email format | `models.EmailField(validators=[EmailValidator()])` |
| `URLValidator` | Ensures the value is a valid URL | `models.URLField(validators=[URLValidator()])` |
| `validate_slug` | Ensures the value is a valid slug (alphanumeric + hyphens/underscores) | `models.SlugField(validators=[validate_slug])` |
| `validate_ipv4_address` | Ensures the value is a valid IPv4 address | `models.GenericIPAddressField(validators=[validate_ipv4_address])` |
| `validate_ipv6_address` | Ensures the value is a valid IPv6 address | `models.GenericIPAddressField(validators=[validate_ipv6_address])` |
| `validate_comma_separated_integer_list` | Ensures the value contains only comma-separated integers | `models.CharField(validators=[validate_comma_separated_integer_list])` |

------------------------------------------------------------------------

## **3. Using Validators in Models**

You can **add validators to model fields** to ensure data is stored correctly.

### **Example: Using Multiple Validators in a Model**

``` python
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator, RegexValidator, EmailValidator

class Employee(models.Model):
    name = models.CharField(
        max_length=100,
        validators=[
            RegexValidator(r'^[A-Za-z ]+$', message="Only alphabets and spaces allowed")
        ]
    )
    age = models.IntegerField(validators=[MinValueValidator(18), MaxValueValidator(65)])
    email = models.EmailField(validators=[EmailValidator()])
    phone = models.CharField(
        max_length=10,
        validators=[RegexValidator(r'^\d{10}$', message="Enter a valid 10-digit phone number")]
    )
```

------------------------------------------------------------------------

## **4. Using Validators in Forms**

In **forms**, you can specify **validators** inside `forms.py`.

### **Example: Validating an Input Field**

``` python
from django import forms
from django.core.validators import MinLengthValidator, MaxLengthValidator, RegexValidator

class ContactForm(forms.Form):
    name = forms.CharField(
        max_length=50,
        validators=[RegexValidator(r'^[A-Za-z ]+$', message="Only alphabets allowed")]
    )
    message = forms.CharField(
        widget=forms.Textarea,
        validators=[MinLengthValidator(10, message="Message must be at least 10 characters long")]
    )
```

------------------------------------------------------------------------

## **5. Using Validators in Django REST Framework (DRF) Serializers**

When using **Django REST Framework (DRF)**, validators ensure API inputs are valid.

### **Example: Using Validators in a Serializer**

``` python
from rest_framework import serializers
from django.core.validators import MinValueValidator, MaxValueValidator

class ProductSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=100)
    price = serializers.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0)]
    )
```

------------------------------------------------------------------------

## **6. Creating Custom Validators**

If built-in validators don’t meet your requirements, you can create **custom validators**.

### **6.1 Function-Based Custom Validator**

A function-based validator **raises a `ValidationError`** if input is invalid.

``` python
from django.core.exceptions import ValidationError

def validate_even(value):
    """Ensure the value is an even number."""
    if value % 2 != 0:
        raise ValidationError(f'{value} is not an even number.')

# Using it in a model
class MyModel(models.Model):
    number = models.IntegerField(validators=[validate_even])
```

### **6.2 Class-Based Custom Validator**

A **class-based validator** allows reusability and multiple validation rules.

``` python
from django.core.exceptions import ValidationError

class MinWordsValidator:
    """Ensure the input has at least `min_words` words."""
    def __init__(self, min_words=3):
        self.min_words = min_words

    def __call__(self, value):
        word_count = len(value.split())
        if word_count < self.min_words:
            raise ValidationError(f'Must have at least {self.min_words} words.')

# Using it in a model
class BlogPost(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField(validators=[MinWordsValidator(5)])
```

------------------------------------------------------------------------

## **7. Raising Validation Errors Manually**

Django allows you to raise validation errors in **model’s `clean()` method**.

### **Example: Enforcing a Business Rule in a Model**

``` python
from django.core.exceptions import ValidationError
from django.db import models

class Booking(models.Model):
    date = models.DateField()
    attendees = models.PositiveIntegerField()

    def clean(self):
        if self.attendees > 100:
            raise ValidationError({'attendees': 'Maximum 100 attendees allowed.'})
```

------------------------------------------------------------------------

## **8. Validating in Django Views**

If using function-based or class-based views, you can **manually validate input**.

### **Example: Manually Validating User Input in a View**

``` python
from django.core.exceptions import ValidationError
from django.shortcuts import render
from .models import Booking
from .forms import BookingForm

def booking_view(request):
    if request.method == "POST":
        form = BookingForm(request.POST)
        if form.is_valid():
            try:
                form.clean()  # Manually invoke the validation
                form.save()
                return render(request, "success.html")
            except ValidationError as e:
                form.add_error(None, e)
    else:
        form = BookingForm()
    return render(request, "booking_form.html", {"form": form})
```

------------------------------------------------------------------------

## **9. Skipping Validation**

You can **skip validation** when saving a model using:

``` python
instance.save(validate=False)
```

**Use with caution!** It **bypasses all validation rules**, which may lead to database inconsistencies.

------------------------------------------------------------------------

## **10. Debugging Validators**

If a validation error occurs, check: - If the **validator function is executed** using print statements or logging. - **Raise exceptions** to test: `python raise ValidationError("Custom error message")` - Use Django’s **shell** to test manually: `bash python manage.py shell >>> from myapp.models import Booking >>> b = Booking(date="2025-01-01", attendees=150) >>> b.full_clean() # Manually run all model validation ValidationError: {'attendees': ['Maximum 100 attendees allowed.']}`

------------------------------------------------------------------------

## **11. Summary Table**

| Validator Type | Example Use Case | Notes |
|----|----|----|
| **Built-in Validators** | `MinValueValidator(10)` | Simple and efficient |
| **Function-Based Validators** | `validate_even(value)` | Best for simple conditions |
| **Class-Based Validators** | `MinWordsValidator(min_words=5)` | Best for reusable rules |
| **Form Validation** | `forms.CharField(validators=[RegexValidator()])` | Ensures user input is valid before saving |
| **Serializer Validation** | `serializers.CharField(validators=[MinLengthValidator(5)])` | Ensures API input is correct |
| **Model Clean Method** | `def clean(self): ...` | Allows complex, multi-field validation |

------------------------------------------------------------------------

## **Final Thoughts**

- **Use built-in validators** where possible.
- **Use function-based or class-based validators** for reusable custom rules.
- **Leverage `clean()` in models for multi-field validation**.
- **Debug issues using Django Shell (`python manage.py shell`)**.

Django validators are a powerful way to **enforce data integrity and business rules at different levels**.

------------------------------------------------------------------------

Back to top
