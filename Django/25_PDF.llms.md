# PDF Generation

WeasyPrint is a **powerful and easy-to-use library** for generating PDFs from HTML and CSS in Django. It provides **better styling control** than reportlab and is well-suited for **dynamic PDF generation** (e.g., invoices, reports, and receipts).

Author

Benedict Thekkel

## **🚀 Step-by-Step Guide to Using WeasyPrint in Django**

### **1️⃣ Install WeasyPrint**

Before using WeasyPrint, install it using `pip`:

``` bash
pip install weasyprint
```

WeasyPrint has some system dependencies. If you face installation issues, install the required system packages:

For **Ubuntu/Debian**:

``` bash
sudo apt install libpango-1.0-0 libpangocairo-1.0-0 libcairo2 libffi-dev
```

For **MacOS**:

``` bash
brew install pango cairo gdk-pixbuf
```

For **Windows**: - Install GTK+ from [WeasyPrint’s documentation](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html).

------------------------------------------------------------------------

### **2️⃣ Create a Django View to Generate PDFs**

We will create a **class-based view** (`WeasyTemplateResponseMixin`) to render a Django template into a PDF.

#### **📌 Example: Generating a Simple PDF**

``` python
# views.py
from django.http import HttpResponse
from django.template.loader import get_template
from weasyprint import HTML
import tempfile

def generate_pdf(request):
    """
    Generate a simple PDF from an HTML template.
    """
    template = get_template("pdf_template.html")
    context = {"title": "My PDF Report", "content": "This is a dynamically generated PDF."}
    html_content = template.render(context)

    # Create a temporary file
    with tempfile.NamedTemporaryFile(delete=True) as temp_file:
        HTML(string=html_content).write_pdf(temp_file.name)
        temp_file.seek(0)
        pdf_data = temp_file.read()

    response = HttpResponse(pdf_data, content_type="application/pdf")
    response["Content-Disposition"] = 'inline; filename="report.pdf"'
    return response
```

------------------------------------------------------------------------

### **3️⃣ Create an HTML Template for the PDF**

WeasyPrint **renders HTML and CSS**, so we need to define a well-structured HTML template.

#### **📌 Example: `pdf_template.html`**

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ title }}</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
        }
        h1 {
            color: #333;
        }
        .content {
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>{{ title }}</h1>
    <div class="content">
        <p>{{ content }}</p>
    </div>
</body>
</html>
```

------------------------------------------------------------------------

### **4️⃣ Add URL Pattern**

Map the PDF generation view in `urls.py`:

``` python
# urls.py
from django.urls import path
from .views import generate_pdf

urlpatterns = [
    path("generate-pdf/", generate_pdf, name="generate_pdf"),
]
```

Now, visiting **`/generate-pdf/`** will return a dynamically generated PDF.

------------------------------------------------------------------------

## **🖋️ Advanced WeasyPrint Usage**

### **1️⃣ Generating PDFs with Django Querysets**

You can **pass database query results** into the PDF template.

#### **📌 Example: Generating a PDF Invoice**

``` python
# views.py
from django.http import HttpResponse
from django.template.loader import render_to_string
from weasyprint import HTML
from .models import Order

def generate_invoice(request, order_id):
    """
    Generate an invoice PDF for a specific order.
    """
    order = Order.objects.get(id=order_id)
    context = {"order": order}
    html_string = render_to_string("invoice_template.html", context)

    pdf_file = HTML(string=html_string).write_pdf()
    response = HttpResponse(pdf_file, content_type="application/pdf")
    response["Content-Disposition"] = f'filename="invoice_{order.id}.pdf"'
    return response
```

#### **📌 Example: `invoice_template.html`**

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Invoice</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            padding: 20px;
        }
        h1 {
            color: #333;
            border-bottom: 2px solid #000;
            padding-bottom: 10px;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 10px;
            text-align: left;
        }
        .total {
            font-weight: bold;
            font-size: 1.2em;
        }
    </style>
</head>
<body>
    <h1>Invoice for Order #{{ order.id }}</h1>
    <p>Customer: {{ order.customer_name }}</p>
    <p>Date: {{ order.date }}</p>

    <table>
        <thead>
            <tr>
                <th>Product</th>
                <th>Quantity</th>
                <th>Price</th>
            </tr>
        </thead>
        <tbody>
            {% for item in order.items.all %}
            <tr>
                <td>{{ item.product_name }}</td>
                <td>{{ item.quantity }}</td>
                <td>${{ item.price }}</td>
            </tr>
            {% endfor %}
        </tbody>
    </table>

    <p class="total">Total Amount: ${{ order.total_price }}</p>
</body>
</html>
```

------------------------------------------------------------------------

### **2️⃣ Saving PDFs to Django Models**

You can **store generated PDFs** as files inside a Django model.

#### **📌 Example: Save Invoice as a File**

``` python
from django.core.files.base import ContentFile
from weasyprint import HTML
from .models import Order

def save_invoice_pdf(order_id):
    """
    Generate and save a PDF invoice to a model field.
    """
    order = Order.objects.get(id=order_id)
    html_string = render_to_string("invoice_template.html", {"order": order})
    
    pdf_file = HTML(string=html_string).write_pdf()
    
    # Save to model
    order.invoice.save(f"invoice_{order.id}.pdf", ContentFile(pdf_file), save=True)
```

#### **📌 Update `Order` Model**

``` python
from django.db import models

class Order(models.Model):
    customer_name = models.CharField(max_length=100)
    date = models.DateField(auto_now_add=True)
    total_price = models.DecimalField(max_digits=10, decimal_places=2)
    invoice = models.FileField(upload_to="invoices/", blank=True, null=True)  # Store PDF
```

------------------------------------------------------------------------

## **🎨 Styling PDFs with CSS**

WeasyPrint **fully supports CSS**, including: ✅ Fonts\
✅ Page Breaks\
✅ Headers & Footers\
✅ Images

#### **📌 Example: Adding Page Breaks**

``` css
@page {
    size: A4;
    margin: 20mm;
}
.page-break {
    page-break-before: always;
}
```

#### **📌 Usage in HTML**

``` html
<p>Page 1 content</p>
<div class="page-break"></div>
<p>Page 2 content</p>
```

------------------------------------------------------------------------

## **📌 Common WeasyPrint Issues & Fixes**

| **Issue** | **Solution** |
|----|----|
| `no such file or directory: cairo` | Install dependencies: `sudo apt install libcairo2` |
| `No module named weasyprint` | Run `pip install weasyprint` |
| Fonts not loading | Use absolute file paths for fonts (`file://` URLs) |

------------------------------------------------------------------------

## **✅ Summary: Key Takeaways**

| **Feature**            | **Django Implementation**                    |
|------------------------|----------------------------------------------|
| ✅ Generate PDFs       | `HTML(string).write_pdf()`                   |
| ✅ Use Templates       | `render_to_string("template.html", context)` |
| ✅ Save PDFs to Models | `order.invoice.save(...)`                    |
| ✅ Style with CSS      | Full CSS support, including `@page`          |

🚀 **Now you can generate beautifully styled PDFs in Django using WeasyPrint!** 🚀

------------------------------------------------------------------------

Back to top
