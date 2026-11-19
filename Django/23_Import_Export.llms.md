# Import Export

The **Django Import-Export** library is a powerful tool for handling **importing** and **exporting** data in Django models using a variety of file formats such as CSV, Excel, JSON, and others. It integrates seamlessly into Django Admin and provides a clean API for programmatic imports and exports.

Author

Benedict Thekkel

## **Key Features**

1.  **Supports Multiple Formats**:
    - File formats: **CSV, Excel (XLSX), JSON, YAML**, etc.
    - Custom formats can also be implemented.
2.  **Seamless Integration with Django Admin**:
    - Import and export directly from the Django admin interface.
    - Buttons are added to the change list page for importing and exporting data.
3.  **Customizable Import/Export Logic**:
    - Allows you to customize how data is imported/exported by overriding specific methods.
    - Supports pre-processing and post-processing of data.
4.  **Error Handling**:
    - Detailed error reports are provided for failed imports, allowing you to identify problematic rows.
5.  **Queryset Exports**:
    - Export only specific rows or data subsets using querysets.

------------------------------------------------------------------------

## **Installation**

1.  Install the package:

    ``` bash
    pip install django-import-export
    ```

2.  Add it to your `INSTALLED_APPS` in `settings.py`:

    ``` python
    INSTALLED_APPS = [
        ...,
        'import_export',
    ]
    ```

3.  Add the `ImportExportModelAdmin` to your admin classes.

------------------------------------------------------------------------

## **Basic Usage**

### **1. Add to Django Admin**

Import `ImportExportModelAdmin` and apply it to your model admin:

``` python
from import_export.admin import ImportExportModelAdmin
from django.contrib import admin
from .models import MyModel

@admin.register(MyModel)
class MyModelAdmin(ImportExportModelAdmin):
    pass
```

This will automatically add **Import** and **Export** buttons in the Django admin.

------------------------------------------------------------------------

### **2. Use Resources for Custom Import/Export Behavior**

A **`Resource`** class lets you define the fields and logic for import/export. For example:

#### Create a `resources.py` file:

``` python
from import_export import resources
from .models import MyModel

class MyModelResource(resources.ModelResource):
    class Meta:
        model = MyModel
        fields = ('id', 'name', 'email')  # Specify fields to include
        export_order = ('id', 'name', 'email')  # Order of fields
```

#### Attach the Resource to the Admin:

``` python
from import_export.admin import ImportExportModelAdmin
from .models import MyModel
from .resources import MyModelResource

@admin.register(MyModel)
class MyModelAdmin(ImportExportModelAdmin):
    resource_class = MyModelResource
```

------------------------------------------------------------------------

### **3. Import and Export in Django Admin**

- **Import**: Upload a file (CSV, XLSX, etc.) to create or update records.
- **Export**: Download the data from the database in the selected format.

------------------------------------------------------------------------

## **File Format Support**

By default, Django Import-Export supports: - **CSV** - **JSON** - **Excel (XLSX)** (requires `openpyxl` or `xlwt`) - **YAML** - Others (depending on installed libraries).

Install additional libraries for specific formats:

- **Excel Support**: `bash pip install openpyxl`

------------------------------------------------------------------------

## **Customizing Import/Export**

### **Customizing Fields**

Exclude or include specific fields using the `fields` or `exclude` options in the `Resource` class:

``` python
class MyModelResource(resources.ModelResource):
    class Meta:
        model = MyModel
        exclude = ('password', 'last_login')  # Exclude sensitive fields
```

### **Custom Import Logic**

You can override the `before_import`, `after_import`, or `import_row` methods for custom behavior.

#### Example: Validate Data Before Import

``` python
class MyModelResource(resources.ModelResource):
    def before_import_row(self, row, **kwargs):
        if not row['email']:
            raise ValueError("Email is required.")
```

------------------------------------------------------------------------

### **Custom Export Logic**

You can override the `export_queryset` method to export specific subsets of data.

#### Example: Export Only Active Users

``` python
class MyModelResource(resources.ModelResource):
    def export_queryset(self, queryset, *args, **kwargs):
        return queryset.filter(is_active=True)
```

------------------------------------------------------------------------

## **Advanced Features**

### **1. Import/Export Specific Columns**

Customize the order of fields or include derived fields:

``` python
class MyModelResource(resources.ModelResource):
    full_name = fields.Field()

    def dehydrate_full_name(self, obj):
        return f"{obj.first_name} {obj.last_name}"

    class Meta:
        model = MyModel
        fields = ('id', 'email', 'full_name')  # Include custom fields
```

### **2. Skip Rows with Errors**

Handle errors gracefully without stopping the import:

``` python
class MyModelResource(resources.ModelResource):
    def import_row(self, row, instance_loader, **kwargs):
        try:
            return super().import_row(row, instance_loader, **kwargs)
        except Exception as e:
            print(f"Error importing row {row}: {e}")
            return None
```

------------------------------------------------------------------------

### **3. Handling Relationships**

Import/export related fields (e.g., ForeignKey, ManyToMany) by using custom logic.

#### Example: Import Foreign Keys

``` python
class MyModelResource(resources.ModelResource):
    related_field = fields.Field(attribute='related_field', column_name='Related Field')

    def before_import_row(self, row, **kwargs):
        # Ensure the related object exists
        row['related_field'] = RelatedModel.objects.get_or_create(name=row['related_field'])[0].id

    class Meta:
        model = MyModel
```

------------------------------------------------------------------------

### **4. Export Querysets in Views**

Use Django Import-Export to export querysets in views programmatically:

``` python
from django.http import HttpResponse
from .resources import MyModelResource

def export_data(request):
    resource = MyModelResource()
    dataset = resource.export()
    response = HttpResponse(dataset.csv, content_type="text/csv")
    response['Content-Disposition'] = 'attachment; filename="export.csv"'
    return response
```

------------------------------------------------------------------------

## **Error Reporting**

### Common Errors

1.  **Field Does Not Exist**: Ensure the fields listed in `fields` or `export_order` match the model fields.

2.  **Invalid File Format**: Make sure the file format matches the library used for import/export.

3.  **Foreign Key Issues**: Ensure related objects exist before importing data.

------------------------------------------------------------------------

## **Best Practices**

1.  **Validate Data Before Import**: Use the `before_import_row` or `clean` method in the model to ensure valid data.

2.  **Restrict Admin Access**: Only allow staff or superusers to use the import/export functionality.

3.  **Back Up Data**: Always back up the database before performing bulk imports or updates.

4.  **Test in Development**: Test your import/export process on a development environment before applying to production.

------------------------------------------------------------------------

### **Use Cases**

1.  Importing large datasets (e.g., user data, product catalogs).
2.  Exporting data for analysis or backup.
3.  Programmatically updating related models.
4.  Migrating data between environments or projects.

------------------------------------------------------------------------

Back to top
