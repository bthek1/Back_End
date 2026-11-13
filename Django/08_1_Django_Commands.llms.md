# Djando and Django-extension Commands

Django and Django Extensions provide several commands to manage and operate Django applications effectively. Here’s a categorized explanation of the **core Django commands** and **Django Extensions commands**:

Author

Benedict Thekkel

## **Core Django Commands**

> These commands are provided by the Django framework itself.

------------------------------------------------------------------------

### **1. Project Management Commands**

- **`startproject <project_name>`**: Creates a new Django project with the specified name.

  ``` bash
  django-admin startproject my_project
  ```

  - Structure:
    - `manage.py`: Entry point for project commands.
    - Project folder with settings, URLs, and ASGI/WSGI files.

### **2. App Management Commands**

- **`startapp <app_name>`**: Creates a new Django app within your project.

  ``` bash
  python manage.py startapp my_app
  ```

### **3. Database Management Commands**

- **`migrate`**: Applies database migrations for installed apps.

  ``` bash
  python manage.py migrate
  ```

- **`makemigrations`**: Creates migration files for model changes.

  ``` bash
  python manage.py makemigrations
  ```

- **`sqlmigrate <app_name> <migration_number>`**: Displays the SQL commands Django would run for a migration.

  ``` bash
  python manage.py sqlmigrate my_app 0001
  ```

### **4. Server and Debugging**

- **`runserver`**: Starts the development server.

  ``` bash
  python manage.py runserver
  ```

  - Default: Runs on `localhost:8000`.
  - Specify IP/port: `python manage.py runserver 0.0.0.0:8080`.

- **`check`**: Validates the project for issues.

  ``` bash
  python manage.py check
  ```

### **5. User Management**

- **`createsuperuser`**: Creates an admin user for the project.

  ``` bash
  python manage.py createsuperuser
  ```

- **`changepassword <username>`**: Changes the password for a user.

  ``` bash
  python manage.py changepassword admin
  ```

### **6. Testing and Debugging**

- **`test`**: Runs all test cases in the project.

  ``` bash
  python manage.py test
  ```

### **7. Other Commands**

- **`showmigrations`**: Displays migrations and their status.

  ``` bash
  python manage.py showmigrations
  ```

------------------------------------------------------------------------

## **Django Extensions Commands**

> Django Extensions adds powerful custom commands to extend functionality beyond the core Django commands.

------------------------------------------------------------------------

### **1. Shell and Debugging**

- **`shell_plus`**: An enhanced shell command that auto-imports models and settings.

  ``` bash
  python manage.py shell_plus
  ```

- **`shell_plus --notebook`**: Integrates with Jupyter Notebook for a better debugging interface.

### **2. Model Visualization**

- **`graph_models`**: Generates a diagram of your models and relationships.

``` sh
python manage.py graph_models my_app -o models.png
python manage.py graph_models budget --dot -o dot.txt
```

- Requires Graphviz.

### **3. URL Management**

- **`show_urls`**: Lists all registered URLs in your project.

  ``` bash
  python manage.py show_urls
  ```

### **4. Profiler and Performance**

- **`runprofileserver`**: Profiles your project to detect performance bottlenecks.

  ``` bash
  python manage.py runprofileserver
  ```

### **5. Jobs and Background Tasks**

- **`create_jobs`**: Creates job templates for background tasks.

  ``` bash
  python manage.py create_jobs my_job
  ```

### **6. Database Commands**

- **`sqlcreate`**: Outputs SQL for creating a database.

  ``` bash
  python manage.py sqlcreate
  ```

### **7. Random Data Generation**

- **`generate_password`**: Generates a random secure password.

  ``` bash
  python manage.py generate_password
  ```

### **8. Command Enhancements**

- **`print_settings`**: Displays project settings for debugging.

  ``` bash
  python manage.py print_settings
  ```

### **9. Logging and Testing**

- **`test_with_coverage`**: Runs tests with coverage analysis.

  ``` bash
  python manage.py test_with_coverage
  ```

------------------------------------------------------------------------

### **Comparison**

| **Category** | **Django Commands** | **Django Extensions Commands** |
|----|----|----|
| **Debugging** | `runserver`, `check` | `shell_plus`, `runprofileserver` |
| **Database** | `migrate`, `makemigrations` | `sqlcreate`, `graph_models` |
| **Model Management** | `showmigrations`, `sqlmigrate` | `graph_models` |
| **Performance** | Not Available | `runprofileserver` |
| **URL Insights** | Not Available | `show_urls` |
| **Testing** | `test` | `test_with_coverage` |

------------------------------------------------------------------------

### **How to Explore All Commands**

To list all available commands in your project:

``` bash
python manage.py help
```

For Django Extensions commands:

``` bash
python manage.py help | grep django_extensions
```

------------------------------------------------------------------------

### **Tips for Effective Use**

- Use **`shell_plus`** for fast prototyping and debugging.
- Leverage **`graph_models`** to understand model relationships visually.
- Combine **`runprofileserver`** with test cases for performance tuning.
- Use **`show_urls`** for insights into the URL structure.

Both Django and Django Extensions commands are designed to make development efficient and organized!

Back to top
