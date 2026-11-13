# npm in Django

Using npm in a Django project to manage and integrate **Bootstrap** is an efficient way to keep front-end dependencies organized. Here’s a comprehensive guide to using npm for Bootstrap in your Django project:

Author

Benedict Thekkel

## **1. Why Use npm for Bootstrap in Django?**

- **Dependency Management**: Easily install, update, or remove Bootstrap and its dependencies.
- **Custom Builds**: Customize Bootstrap (e.g., overriding SCSS variables).
- **Modern Workflow**: Use tools like Webpack, Vite, or Gulp for advanced front-end development.
- **Consistent Versioning**: Lock versions to avoid breaking changes during updates.

------------------------------------------------------------------------

## **2. Prerequisites**

- **Django Installed**: Ensure your Django project is set up.

- **Node.js and npm Installed**: Verify installation:

  ``` bash
  node -v
  npm -v
  ```

If not installed:

``` bash
sudo apt install npm
```

------------------------------------------------------------------------

## **3. Initialize npm in Your Django Project**

Navigate to your Django project directory (where `manage.py` is located) and initialize npm:

``` bash
cd your-django-project
npm init -y
```

This creates a `package.json` file to manage your project dependencies.

------------------------------------------------------------------------

## **4. Install Bootstrap with npm**

Install Bootstrap and its peer dependencies (like Popper.js for tooltips):

``` bash
npm install bootstrap
```

This adds Bootstrap to your `node_modules` directory and updates `package.json` with the Bootstrap dependency.

------------------------------------------------------------------------

## **5. Add a `static` Directory in Your Django Project**

Set up a directory for static files if it doesn’t already exist:

### Example Structure:

    your_project/
    ├── static/
    │   ├── js/
    │   ├── css/
    │   ├── scss/

------------------------------------------------------------------------

## **6. Using Bootstrap Files**

Bootstrap provides SCSS files for customization.

1.  Copy Bootstrap SCSS files to your static directory:

``` bash
mkdir -p static/scss
cp -r node_modules/bootstrap/scss/* static/scss/
```

2.  Install a Sass compiler:

``` bash
npm install sass
```

3.  Create a custom SCSS file (`static/scss/custom.scss`) to override Bootstrap variables:

``` scss


@use "bootstrap/scss/bootstrap" with (
  $primary:       #44b878,
  $secondary:     #ff4136,
  $success:       #28a745,
  $warning:       #ffc107,
  $danger:        #dc3545,
  $light:         #f8f9fa,
  $dark:          #343a40,
  $font-family-sans-serif: 'Arial', // Customize font
  $body-bg: #f8f9fa,                // Change body background color
);
```

4.  Compile the SCSS file into CSS:

``` bash
npx sass static/scss/custom.scss static/css/custom.css
```

5.  Include the compiled CSS in your template:

``` html
{% load static %}
<link rel="stylesheet" href="{% static 'css/custom.css' %}">
```

------------------------------------------------------------------------

## **7. Automating with npm Scripts**

To streamline the SCSS compilation process, add a script to `package.json`:

``` json
  "scripts": {
    "build-css": "npx sass --load-path=node_modules --quiet-deps frontend/scss/custom.scss frontend/css/custom.css",
    "watch-css": "npx sass --watch --load-path=node_modules --quiet-deps frontend/scss/custom.scss:frontend/css/custom.css"
  },
```

Now, compile SCSS with:

``` bash
npm run build-css
```

------------------------------------------------------------------------

## **8. Serving Static Files in Django**

Ensure Django is configured to serve static files: 1. Add `STATIC_URL` and `STATICFILES_DIRS` to `settings.py`: `python STATIC_URL = '/static/' STATICFILES_DIRS = [BASE_DIR / 'static']`

2.  During development, Django will automatically serve static files. For production, use `collectstatic` to gather all static files in one directory:

    ``` bash
    python manage.py collectstatic
    ```

------------------------------------------------------------------------

## **10. Real-World Example**

### SCSS Customization:

1.  Create `static/scss/custom.scss`:

    ``` scss
    $primary: #007bff;
    $secondary: #6c757d;
    @import "bootstrap";
    ```

2.  Compile the SCSS:

    ``` bash
    npx sass static/scss/custom.scss static/css/custom.css
    ```

3.  Include in your Django template:

    ``` html
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/custom.css' %}">
    ```

------------------------------------------------------------------------

## **11. Debugging and Maintenance**

1.  **Upgrade Dependencies**: To update Bootstrap or other dependencies:

    ``` bash
    npm update
    ```

2.  **Clear npm Cache**: If you face issues with npm:

    ``` bash
    npm cache clean --force
    ```

3.  **Handle Production Builds**: For optimized builds, use:

    ``` bash
    npx sass static/scss/custom.scss static/css/custom.css --style=compressed
    ```

------------------------------------------------------------------------

## **12. Tools and Alternatives**

- **Parcel or Vite**: Alternative to Webpack for bundling.
- **Django-tailwind**: For integrating Tailwind CSS with Django.

------------------------------------------------------------------------

Back to top
