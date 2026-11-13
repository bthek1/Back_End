# Forms

Forms

Author

Benedict Thekkel

## Introduction to Django Forms

- Forms in Django: Django forms are used to handle the input from a user, process it, and then use it for various purposes such as saving data to the database, sending an email, etc.
- Two Types of Forms:
  - Forms: `django.forms.Form` is used to create a form from scratch.
  - ModelForms: `django.forms.ModelForm` is used to create a form directly from a Django model.

## Creating a Basic Form

- Form Class: Define a form by creating a subclass of `forms.Form`.
- Form Fields: Add fields using the various field classes (e.g., `CharField`, `EmailField`, `ChoiceField`).

``` python
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
```

- Rendering the Form: Forms can be rendered in templates using Django template tags.

``` html
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Send</button>
</form>
```

## Handling Form Submission

- Views: Handle form submission in a Django view.
- Form Validation: When a form is submitted, you can check if it’s valid using `form.is_valid()`.

> Example

``` python
from django.shortcuts import render
from .forms import ContactForm

def contact_view(request):
    if request.method == 'POST':
        form = ContactForm(request.POST)
        if form.is_valid():
            # Process the data in form.cleaned_data
            # For example, send an email or save the data
            return render(request, 'contact_success.html')
    else:
        form = ContactForm()
    return render(request, 'contact_form.html', {'form': form})
```

## Form Fields

> Django provides a variety of form fields, each corresponding to a different HTML input element:

------------------------------------------------------------------------

- CharField: `<input type="text">`
- EmailField: `<input type="email">`
- ChoiceField: `<select>`
- BooleanField: `<input type="checkbox">`
- DateField: `<input type="date">`
- FileField: `<input type="file">`
- ImageField: A special subclass of `FileField` used for uploading images.

> Example

``` python
class ExampleForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    birth_date = forms.DateField(widget=forms.SelectDateWidget)
```

## Widgets

- Customizing Widgets: Widgets define how the form field is rendered. Each form field has an associated widget that determines its appearance.
- Common Widgets:
  - `TextInput`, `Textarea`, `Select`, `CheckboxInput`, `FileInput`, etc.
- Custom Widgets: You can create custom widgets by subclassing forms.Widget.

> Example:

``` python
class CustomForm(forms.Form):
    name = forms.CharField(widget=forms.TextInput(attrs={'class': 'special', 'size': '40'}))
    email = forms.EmailField(widget=forms.EmailInput(attrs={'placeholder': 'you@example.com'}))
```

## Form Validation

- Field Validation: Basic validation is provided by the field types (e.g., `EmailField` validates that the input is a valid email address).
- Custom Validation: Add custom validation by defining a `clean_<fieldname>()` method in your form class.
- Form-Level Validation: For validation that involves multiple fields, override the `clean()` method.

> Example

``` python
class RegistrationForm(forms.Form):
    username = forms.CharField(max_length=100)
    email = forms.EmailField()
    password = forms.CharField(widget=forms.PasswordInput)
    confirm_password = forms.CharField(widget=forms.PasswordInput)

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        confirm_password = cleaned_data.get('confirm_password')

        if password != confirm_password:
            raise forms.ValidationError("Passwords do not match")
```

## ModelForms

- Introduction: A ModelForm is a form that automatically creates fields based on the fields in a Django model. It’s used to create forms that directly interact with the database.
- Creating a ModelForm:

``` python
from django import forms
from .models import Person

class PersonForm(forms.ModelForm):
    class Meta:
        model = Person
        fields = ['first_name', 'last_name', 'email']
```

- Saving Data: When the form is valid, you can call form.save() to save the data to the database.

- Customizing ModelForms: You can override or add fields, customize widgets, and add validation.

## Formsets

- Introduction: Formsets are a layer of abstraction to work with multiple forms on the same page. They are often used to manage multiple instances of a model.

- Creating a Formset:

``` python
from django.forms import formset_factory

class SimpleForm(forms.Form):
    name = forms.CharField()

SimpleFormSet = formset_factory(SimpleForm, extra=3)
```

- ModelFormsets: ModelFormsets work similarly to Formsets but are based on models.

``` python
from django.forms import modelformset_factory
from .models import Person

PersonFormSet = modelformset_factory(Person, fields=('first_name', 'last_name', 'email'), extra=2)
```

Back to top
