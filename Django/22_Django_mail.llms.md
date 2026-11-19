# Mail Configuration

Using mailing in Django involves setting up email configurations, sending emails, and potentially customizing how emails are sent. Here’s an overview of everything you need to know:

Author

Benedict Thekkel

------------------------------------------------------------------------

### 1. **Basic Email Configuration**

Django uses the `EmailMessage` class and provides a simple interface for sending emails. To start, configure your email backend in `settings.py`:

``` python
# settings.py

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'  # Default backend
EMAIL_HOST = 'smtp.gmail.com'  # Replace with your email provider's SMTP server
EMAIL_PORT = 587  # Typically 587 for TLS
EMAIL_USE_TLS = True
EMAIL_USE_SSL = False  # Use either TLS or SSL, not both
EMAIL_HOST_USER = 'your_email@example.com'
EMAIL_HOST_PASSWORD = 'your_password'
DEFAULT_FROM_EMAIL = 'your_email@example.com'
```

- **EMAIL_BACKEND**: Choose an email backend.
  - `smtp.EmailBackend`: Default option for sending real emails.
  - `django.core.mail.backends.console.EmailBackend`: Logs emails to the console (useful for testing).
  - `django.core.mail.backends.filebased.EmailBackend`: Writes emails to files.
  - `django.core.mail.backends.dummy.EmailBackend`: Disables sending emails (useful for development).
  - Third-party options like `django-ses` for AWS SES.

------------------------------------------------------------------------

#### AWS SES

``` python
# Email backend for AWS SES
EMAIL_BACKEND = 'django_ses.SESBackend'

# AWS SES Configuration
AWS_ACCESS_KEY_ID = '<your_aws_access_key_id>'
AWS_SECRET_ACCESS_KEY = '<your_aws_secret_access_key>'
AWS_SES_REGION_NAME = '<your_region>'  # e.g., 'us-east-1'
AWS_SES_REGION_ENDPOINT = 'email.<your_region>.amazonaws.com'  # e.g., 'email.us-east-1.amazonaws.com'

# Optional settings for SES
AWS_SES_AUTO_THROTTLE = 0.5  # Adjust based on your sending rate (0.5 = 50% utilization)
AWS_SES_RETURN_PATH = 'bounce@example.com'  # Email address for bounce notifications
```

with `.env`

``` python
import environ

env = environ.Env()
environ.Env.read_env()

AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')
AWS_SES_REGION_NAME = env('AWS_SES_REGION_NAME')
AWS_SES_REGION_ENDPOINT = env('AWS_SES_REGION_ENDPOINT')
```

### 2. **Sending Emails**

Use Django’s built-in `send_mail` or `EmailMessage`:

#### Simple Email

``` python
from django.core.mail import send_mail

send_mail(
    'Subject here',
    'Here is the message.',
    'from@example.com',
    ['to@example.com'],
    fail_silently=False,
)
```

#### Advanced Email with Attachments

``` python
from django.core.mail import EmailMessage

email = EmailMessage(
    'Subject here',
    'Here is the message.',
    'from@example.com',
    ['to@example.com'],
    cc=['cc@example.com'],
    bcc=['bcc@example.com'],
)
email.attach_file('/path/to/file.pdf')
email.send()
```

------------------------------------------------------------------------

### 3. **HTML Emails**

To send emails with HTML content:

``` python
from django.core.mail import EmailMessage

email = EmailMessage(
    'Subject here',
    'Fallback plain text content.',
    'from@example.com',
    ['to@example.com']
)
email.content_subtype = "html"  # Specify the email content type as HTML
email.body = '<h1>Hello</h1><p>This is a test email.</p>'
email.send()
```

------------------------------------------------------------------------

### 4. **Using Templates for Emails**

Use Django templates for dynamic content:

#### Email Template (`templates/email.html`):

``` html
<h1>Hello {{ name }}</h1>
<p>This is your message: {{ message }}</p>
```

#### Sending the Email:

``` python
from django.template.loader import render_to_string
from django.core.mail import EmailMessage

context = {'name': 'Ben', 'message': 'Welcome to Django mailing!'}
html_content = render_to_string('email.html', context)

email = EmailMessage(
    'Welcome Email',
    html_content,
    'from@example.com',
    ['to@example.com']
)
email.content_subtype = "html"
email.send()
```

------------------------------------------------------------------------

### 5. **Testing Emails**

Django offers tools for testing email functionality:

#### Console Email Backend:

``` python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

Emails are printed to the console instead of being sent.

#### File-Based Email Backend:

``` python
EMAIL_BACKEND = 'django.core.mail.backends.filebased.EmailBackend'
EMAIL_FILE_PATH = '/tmp/app-messages'  # Directory where emails are saved
```

------------------------------------------------------------------------

### 6. **Email Queues**

If you have high email volume, use a task queue like Celery:

#### Install Celery and a Broker:

``` bash
pip install celery[redis]
```

#### Celery Configuration:

- Configure Celery with Redis as the broker.
- Use `delay()` to send emails asynchronously.

------------------------------------------------------------------------

### 7. **Third-Party Libraries**

- **django-anymail**: Integrates with email services like Mailgun, SendGrid, and Amazon SES.
- **django-ses**: For Amazon SES.
- **django-email-multi-alternatives**: Makes it easier to send multi-part emails.

------------------------------------------------------------------------

### 8. **Email Signal Example**

You can use Django signals to send emails on specific actions:

``` python
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.core.mail import send_mail
from .models import UserProfile

@receiver(post_save, sender=UserProfile)
def send_welcome_email(sender, instance, created, **kwargs):
    if created:
        send_mail(
            'Welcome!',
            f'Hi {instance.user.username}, welcome to our platform!',
            'from@example.com',
            [instance.user.email],
        )
```

------------------------------------------------------------------------

### 9. **Best Practices**

1.  **Environment Variables**: Store sensitive credentials (e.g., `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`) in environment variables using libraries like `python-decouple` or `django-environ`.
2.  **Error Handling**: Use `try...except` blocks for email failures.
3.  **Rate Limits**: Be aware of your email provider’s rate limits.
4.  **SPF, DKIM, DMARC**: Configure these DNS records to avoid being flagged as spam.
5.  **Use an Email Service**: For production, consider services like Mailgun, SendGrid, or AWS SES for reliability.

------------------------------------------------------------------------

Let me know if you need examples or help configuring specific email features in Django!

Back to top
