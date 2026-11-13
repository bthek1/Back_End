# Crispy Forms

Forms

Author

Benedict Thekkel

## Installation of crisply and bootstrap

``` sh
poetry add django-crispy-forms crispy-bootstrap5
```

## Update `settings.py`

``` python
INSTALLED_APPS = [
    # Other installed apps
    'crispy_forms',
    'crispy_bootstrap5',
]

# Set the Crispy Forms template pack to Bootstrap 5
CRISPY_ALLOWED_TEMPLATE_PACKS = "bootstrap5"
CRISPY_TEMPLATE_PACK = "bootstrap5"
```

## Form Layouts

> Django Crispy Forms provides a powerful layout system that allows you to control the form layout directly in your form class.

------------------------------------------------------------------------

> Example

``` python
from django import forms
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Layout, Field, Row, Column, Submit

class ExampleForm(forms.Form):
    first_name = forms.CharField()
    last_name = forms.CharField()
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.form_method = 'post'
        self.helper.layout = Layout(
            Row(
                Column('first_name', css_class='form-group col-md-6 mb-0'),
                Column('last_name', css_class='form-group col-md-6 mb-0'),
                css_class='form-row'
            ),
            'email',
            'message',
            Submit('submit', 'Send Message', css_class='btn btn-primary')
        )
```

- Layout Elements:
  - Field: Wraps a form field. You can customize its attributes (e.g., CSS classes, placeholders).
  - Row and Column: Define grid layouts for your form fields (e.g., using Bootstrap’s grid system).
  - Submit: Adds a submit button to the form.

## Helper Class

> The FormHelper class in Django Crispy Forms is where most of the customization happens.

- Common Attributes:
  - form_method: Specifies the form submission method (‘post’ or ‘get’).
  - form_action: The URL where the form should be submitted.
  - form_class: CSS classes for the form.
  - layout: Defines the form layout using Layout, Field, Row, Column, etc.

> Example

``` python
from crispy_forms.helper import FormHelper

class ExampleForm(forms.Form):
    ...

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.form_id = 'example-form-id'
        self.helper.form_class = 'form-horizontal'
        self.helper.form_method = 'post'
        self.helper.form_action = 'submit_form'
```

## Update Your Forms

``` python
from django import forms
from .models import ToDo
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Submit

class ToDoForm(forms.ModelForm):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.add_input(Submit('Add todo', 'Add todo'))
        
    class Meta:
        model = ToDo
        fields = ['title', 'description', 'completed']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 4}),
        }
```

## Update Views

``` python
@login_required
def add_todo(request):
    if request.method == 'POST':
        form = ToDoForm(request.POST)
        if form.is_valid():
            todo = form.save(commit=False)
            todo.user = request.user
            todo.save()
            return redirect('list_todo')  # Redirect to your todo list view
        else:
            # Add this to check for form validation errors
            print(form.errors)
    else:
        form = ToDoForm()

    return render(request, 'todos/add_todo.html', {'form': form})
```

## Update html

``` html
{% extends 'base.html' %}

{% block title %}Add New ToDo{% endblock %}

{% block content %}
    <h1>Add a New ToDo</h1>
    <form method="post">
        {% csrf_token %}
        {% load crispy_forms_tags %}
        {% crispy form %}
        
    </form>
{% endblock %}
```

## Template

- base.html

- nav.html

- app

  - app_specific.html

- app2

  - app2_specific.html

Back to top
