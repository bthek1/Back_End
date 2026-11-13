# Admin Unfold

**Django Unfold** is a third-party package that **enhances** and **modernizes** the **Django Admin** interface. It introduces a refined layout, improved user experience (UX), and greater flexibility for customizing forms and admin pages—without having to rebuild your own admin interface from scratch.

Author

Benedict Thekkel

## **1. What Is Django Unfold?**

### **Key Advantages**

1.  **Modern Look & Feel**: Provides an updated design over the default Django Admin.\
2.  **Collapsible Sections & Tabs**: Makes large or complex forms more manageable.\
3.  **Better Layout & Navigation**: Simplifies admin pages to reduce clutter.\
4.  **Configurable & Extensible**: Easily override templates or add custom CSS/JS.\
5.  **Integration with Default Admin**: Minimal disruption to your existing admin code.

------------------------------------------------------------------------

## **2. Installation & Setup**

1.  **Install the Package**:

    ``` bash
    pip install django-unfold
    ```

    Or if you use Poetry:

    ``` bash
    poetry add django-unfold
    ```

2.  **Add to `INSTALLED_APPS`**:

    ``` python
    # settings.py
    INSTALLED_APPS = [
        # ...
        'django_unfold',
    ]
    ```

3.  **Run Migrations** (if prompted, though Unfold typically doesn’t add models):

    ``` bash
    python manage.py migrate
    ```

4.  **Collect Static**:

    ``` bash
    python manage.py collectstatic
    ```

    This ensures all static files related to Django Unfold’s styling and scripts are in place.

5.  **Test**:

    ``` bash
    python manage.py runserver
    ```

    Log into your admin. You should notice a more refined UI.

------------------------------------------------------------------------

## **3. Using Unfold in Your Admin**

### **A. Subclass the Unfold Admin**

By default, you can get many enhancements just by using `UnfoldAdmin` instead of Django’s `ModelAdmin`:

``` python
# myapp/admin.py
from django_unfold.admin import UnfoldAdmin
from django.contrib import admin
from .models import MyModel

@admin.register(MyModel)
class MyModelAdmin(UnfoldAdmin):
    list_display = ('id', 'name', 'created_at')
    search_fields = ('name',)
```

This typically gives you: - A cleaner form layout - Improved inlines handling - Enhanced styling out of the box

### **B. Collapsible Fieldsets**

Similar to Django’s default `fieldsets`, you can make them collapsible for better organization:

``` python
class MyModelAdmin(UnfoldAdmin):
    fieldsets = [
        ('Basic Info', {
            'fields': ('name', 'description'),
            'classes': ('collapse',),
        }),
        ('Additional Info', {
            'fields': ('some_other_field',),
        }),
    ]
```

### **C. Using Tabs**

Unfold provides a tabbed layout option to group fields:

``` python
class MyModelAdmin(UnfoldAdmin):
    tabs = [
        ('General', {
            'fields': ('name', 'description'),
        }),
        ('Details', {
            'fields': ('some_other_field', 'another_field'),
        }),
    ]
```

When you load the change form page in the admin, the fields appear under tabbed sections instead of a single vertical form.

### **D. Custom List Displays**

Leverage Unfold’s enhanced list views (e.g., improved table responsiveness):

``` python
class MyModelAdmin(UnfoldAdmin):
    list_display = ('id', 'name', 'status')
    list_filter = ('status', 'created_at')
    search_fields = ('name',)
```

### **E. Admin-Site-Wide Settings**

If you want to **override** the admin site headers or site titles:

``` python
# myproject/admin.py
from django.contrib import admin
admin.site.site_header = "My Unfold Admin"
admin.site.site_title = "Unfold Admin Portal"
admin.site.index_title = "Welcome to My Unfold Admin"
```

------------------------------------------------------------------------

## **4. Advanced Customization**

### **A. Extending `UnfoldAdmin`**

If you want a **custom base admin** for all your models:

``` python
# myapp/admin_bases.py
from django_unfold.admin import UnfoldAdmin

class BaseUnfoldAdmin(UnfoldAdmin):
    # e.g. add custom media, override default behaviors
    class Media:
        css = {
            'all': ('css/custom_admin.css',)
        }
        js = ('js/custom_admin.js',)

    # You can override built-in methods or add custom logic here
    def some_custom_method(self, obj):
        return "Custom Display"
```

Then in `admin.py`:

``` python
from .admin_bases import BaseUnfoldAdmin
from .models import MyModel

@admin.register(MyModel)
class MyModelAdmin(BaseUnfoldAdmin):
    pass
```

### **B. Template Overrides**

Django Unfold references certain admin templates like `admin/base.html` or `admin/change_form.html`. You can override these by creating templates in your project:

    myproject/
        templates/
            admin/
                base.html
                change_form.html
                ...

If you want to specifically override Unfold’s template logic, check the Unfold documentation or source to see which template names it’s using. Usually, it’s the same structure as Django’s admin templates, so standard overrides typically apply.

### **C. Enhanced Inlines**

Inline forms for related models can also benefit from Unfold’s layout. Simply swap `admin.TabularInline` or `admin.StackedInline` with the equivalents from Unfold if provided, or keep them as is—Unfold’s CSS typically styles them automatically.

------------------------------------------------------------------------

## **5. Configuration & Settings**

Some versions of Unfold might expose settings in `settings.py` to enable or disable specific features. For example:

``` python
# settings.py
UNFOLD_SETTINGS = {
    'ENABLE_RESPONSIVE_TABLES': True,
    'USE_TABS': True,
    'THEME': 'default',  # or 'dark', etc., if multiple themes are available
}
```

Check the project’s documentation for the exact keys and possible values. (Not all releases of Unfold have these settings, so it depends on your version.)

------------------------------------------------------------------------

## **6. Combining Unfold with Other Admin Tools**

Django Unfold can coexist with other admin enhancements like: - **django-extensions**: Greatly expands your management commands. - **django-import-export**: Enables data import/export from the admin. - **django-grappelli** or **django-suit**: *However*, mixing admin UI frameworks often causes conflicts in CSS or JavaScript. Generally, you pick **one** UI library (Unfold, Grappelli, Suit, etc.) to avoid collisions.

If you do combine them, be sure to test thoroughly and override CSS if needed to prevent style conflicts.

------------------------------------------------------------------------

## **7. Potential Limitations**

1.  **Less Common**: It’s not as widely used as Grappelli or Django Suit, so there might be fewer community snippets.
2.  **Version Compatibility**: Always check if the Unfold version supports your Django version (especially if you’re on newer Django releases).
3.  **Template Collisions**: If you have heavy template customizations for the admin, you may need to reconcile them with Unfold’s approach.
4.  **Development Activity**: Check the repository’s activity for updates or potential issues with stale code.

------------------------------------------------------------------------

## **8. Quick Reference Table**

| **Feature** | **Usage** |
|----|----|
| **Installation** | `pip install django-unfold` |
| **Admin Import** | `from django_unfold.admin import UnfoldAdmin` |
| **Basic Usage** | `class MyModelAdmin(UnfoldAdmin): ...` |
| **Collapsible Fieldsets** | `fieldsets = [(... {'classes': ('collapse',)})]` |
| **Tabbed Layout** | `tabs = [("Tab1", {...}), ("Tab2", {...})]` |
| **Template Overrides** | Place overrides in `templates/admin/` (same as default Django admin) |
| **Custom Theme** | Adjust `UNFOLD_SETTINGS` in `settings.py` or override CSS via `Media` class |
| **Registration** | `@admin.register(MyModel)` or `admin.site.register(MyModel, MyModelAdmin)` |

------------------------------------------------------------------------

## **9. Conclusion & Best Practices**

1.  **Start Simple**
    - Swap in `UnfoldAdmin` for your existing `ModelAdmin` to see immediate UI improvements.
2.  **Organize Fields**
    - Use collapsible fieldsets or tabs to manage complex models with many fields.
3.  **Override Responsibly**
    - Use the standard Django approach to override admin templates only if you need deep layout changes.
4.  **Maintain Code Consistency**
    - If you adopt Unfold, consider making it the default for all your `ModelAdmin`s to have a consistent admin UI.
5.  **Check Documentation**
    - If you run into styling issues or need advanced features, consult the official GitHub/Docs for up-to-date instructions.

By following these steps and guidelines, you can **transform** your Django Admin into a more **modern**, **user-friendly**, and **efficient** environment using Django Unfold.

------------------------------------------------------------------------

Back to top
