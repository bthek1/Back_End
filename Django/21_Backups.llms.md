# Back ups

Below is a comprehensive guide to **django-dbbackup**, a popular Django application that helps automate database and media backups. This covers installation, configuration, usage, storage options (local and cloud), and best practices.

Author

Benedict Thekkel

## **1. What is django-dbbackup?**

**django-dbbackup** is a Django application designed to: 1. **Back up** your database (and optionally media files). 2. **Restore** your database from these backups. 3. **Store** backups locally or remotely (e.g., Amazon S3, Dropbox, etc.).

It integrates with Django’s settings and management commands, making it straightforward to schedule or invoke backups from the command line or cron jobs.

- **GitHub Repo**: [django-dbbackup](https://github.com/jazzband/django-dbbackup)
- **PyPI**: [django-dbbackup](https://pypi.org/project/django-dbbackup/)

------------------------------------------------------------------------

## **2. Installation**

1.  **Install via pip**:

    ``` bash
    pip install django-dbbackup
    pip install django-storages
    ```

2.  **Add to Installed Apps**:

    ``` python
    INSTALLED_APPS = [
        ...
        'dbbackup',  # django-dbbackup
    ]
    ```

3.  **Migrate** (if any model changes):

    ``` bash
    python manage.py migrate
    ```

------------------------------------------------------------------------

## **3. Basic Configuration in `settings.py`**

A minimal setup might look like this:

``` python
# DBBackup S3 Credentials
DBBACKUP_S3_ACCESS_KEY_ID = os.environ.get('DBBACKUP_S3_ACCESS_KEY_ID')
DBBACKUP_S3_SECRET_ACCESS_KEY = os.environ.get('DBBACKUP_S3_SECRET_ACCESS_KEY')
DBBACKUP_S3_BUCKET_NAME = os.environ.get('DBBACKUP_S3_BUCKET_NAME')


# DBBACKUP_STORAGE = 'django.core.files.storage.FileSystemStorage'
# DBBACKUP_STORAGE_OPTIONS = {
#     'location': BASE_DIR.parent / 'backup',  # Folder called 'backup' in the parent directory
# }

# DBBACKUP_STORAGE = 'dbbackup.storage.s3_storage.S3Storage'
DBBACKUP_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

DBBACKUP_STORAGE_OPTIONS = {
    'access_key': DBBACKUP_S3_ACCESS_KEY_ID,
    'secret_key': DBBACKUP_S3_SECRET_ACCESS_KEY,
    'bucket_name': DBBACKUP_S3_BUCKET_NAME,  # ✅ Replace with your bucket name
}


# (Optional) Compress backups
DBBACKUP_COMPRESS = True

# (Optional) Encrypt backups using GPG
DBBACKUP_GPG_RECIPIENT = None
DBBACKUP_CLEANUP_KEEP = 5
```

### 3.1 Key Settings

1.  **`DBBACKUP_STORAGE`**: Defines the storage class used to store backup files.

    - Common values:
      - `django.core.files.storage.FileSystemStorage` for local backups.
      - `storages.backends.s3boto3.S3Boto3Storage` for S3 backups (requires `django-storages`).

2.  **`DBBACKUP_STORAGE_OPTIONS`**: A dictionary of options passed to your storage class.

    - For local file system: `{'location': '/path/to/backups/'}`.
    - For S3: `{'access_key': '...', 'secret_key': '...', 'bucket_name': 'my-backups'}`.

3.  **`DBBACKUP_FILENAME_TEMPLATE`** (Optional): Custom filename format for backups, e.g., `"{datetime}-{databasename}.{extension}"`.

4.  **`DBBACKUP_CLEANUP_KEEP`** and **`DBBACKUP_CLEANUP_KEEP_MEDIA`** (Optional): Number of backups to keep if you use the `cleanup` command.

5.  **`DBBACKUP_COMPRESS`** (Boolean): If `True`, compresses backup files (default is `False`).

6.  **Encryption**: If you want GPG encryption, set `DBBACKUP_GPG_RECIPIENT = 'my-gpg-id'`.

------------------------------------------------------------------------

## **4. Usage and Commands**

### 4.1 Create a Database Backup

``` bash
python manage.py dbbackup
```

- This command creates a backup of your default database and saves it to the configured storage.
- By default, it also backs up all configured databases in `settings.DATABASES` unless you specify `--database`.

**Key Options**: - `--database <alias>`: Backup a specific DB alias. - `--compress`: Override the default compress setting. - `--encrypt`: Override the default GPG encrypt setting. - `--output` or `-o`: Writes the backup to a local file (e.g., `dbbackup -o /tmp/mydbbackup`).

Example:

``` bash
python manage.py dbbackup --database default --compress --encrypt
```

### 4.2 Restore a Database Backup

``` bash
python manage.py dbrestore
```

By default, this looks for the **latest** backup file in the configured storage and restores it.

**Key Options**: - `--database <alias>`: Restore a specific DB. - `--input <filename>` or `-i`: Restore from a specific backup file. - `--noinput`: Skip “Are you sure?” prompts. - `--decrypt`: Decrypt if needed.

Example:

``` bash
python manage.py dbrestore --database default -i /path/to/backups/2023-01-20-default.gz --noinput
```

### 4.3 Backing Up Media Files

**django-dbbackup** can back up your `MEDIA_ROOT` with:

``` bash
python manage.py mediabackup
```

And restore media:

``` bash
python manage.py mediarestore
```

> Ensure you have the relevant storage configured if you want media backups stored similarly (same or different location).

### 4.4 Cleanup Old Backups

To automatically remove older backups (beyond a certain number): 1. Set `DBBACKUP_CLEANUP_KEEP = <int>` in settings. 2. Run: `bash python manage.py dbbackup_cleanup` or `bash python manage.py mediabackup_cleanup`

------------------------------------------------------------------------

## **5. Common Storage Options**

### 5.1 Local Filesystem

Set:

``` python
DBBACKUP_STORAGE = 'django.core.files.storage.FileSystemStorage'
DBBACKUP_STORAGE_OPTIONS = {
    'location': '/path/to/backups/',
}
```

Make sure `/path/to/backups/` is writable by Django.

### 5.2 Amazon S3

You can use **django-storages**: 1. Install: `bash pip install django-storages boto3` 2. Configure S3 in `settings.py`: `python DBBACKUP_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage' DBBACKUP_STORAGE_OPTIONS = { 'access_key': 'YOUR_ACCESS_KEY', 'secret_key': 'YOUR_SECRET_KEY', 'bucket_name': 'my-backups-bucket', }` 3. Alternatively, if you already have a custom storage class for S3, point `DBBACKUP_STORAGE` to that class.

### 5.3 Other Storage

- **Dropbox**: There are examples of custom Dropbox storages or `django-storages` integration.
- **FTP / SFTP**: Use a custom storage or library that integrates with ftp/sftp.

------------------------------------------------------------------------

## **6. Scheduling Backups**

Typically, you schedule backups with **cron** or a job scheduler like **celery beat** or **systemd timers**. For example, a cron job to back up the DB daily at midnight:

``` bash
0 0 * * * /path/to/venv/bin/python /path/to/project/manage.py dbbackup --compress --encrypt
```

You can similarly schedule media backups or calls to `dbbackup_cleanup`.

------------------------------------------------------------------------

## **7. Security Considerations**

1.  **Encryption (GPG)**:
    - If you store backups in a remote location or want extra security, set `DBBACKUP_GPG_RECIPIENT` so that backups are encrypted at rest.
    - You’ll need GPG installed and a matching key.
2.  **Permissions**:
    - Ensure backup directories and keys are protected so only authorized users can read them.
3.  **Handling Secrets**:
    - Do not commit your DB or storage credentials in version control. Use environment variables or a secrets manager.

------------------------------------------------------------------------

## **8. Debugging & Common Issues**

1.  **File Permissions**:
    - Make sure Django can write to `DBBACKUP_STORAGE_OPTIONS['location']` if using local storage.
2.  **Missing or Wrong Database Dumps**:
    - Check that your `DATABASES` config has the correct engine. For example, if using Postgres, ensure the `pg_dump` utility is installed and in `PATH`.
3.  **S3 Access Denied**:
    - Double-check your AWS credentials and bucket permissions.

------------------------------------------------------------------------

## **9. Additional Features & Tips**

1.  **Partial Database Backup**: Not supported out of the box, dbbackup typically dumps the entire DB.
2.  **Multiple DBs**:
    - If you have more than one database in `settings.DATABASES`, `dbbackup` will back them all up by default, naming them accordingly. Use `--database` to target one.
3.  **Versioning**: If storing backups on S3, you can enable versioning on the bucket for extra safety (though this is outside django-dbbackup’s scope).
4.  **Notifications**: You might integrate Slack or email notifications to confirm successful backups or handle errors.

------------------------------------------------------------------------

## **10. Example End-to-End Setup**

**`settings.py`:**

``` python
INSTALLED_APPS = [
    ...,
    'dbbackup',
]

DBBACKUP_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
DBBACKUP_STORAGE_OPTIONS = {
    'access_key': os.getenv('AWS_ACCESS_KEY_ID'),
    'secret_key': os.getenv('AWS_SECRET_ACCESS_KEY'),
    'bucket_name': 'myproject-dbbackups',
}
DBBACKUP_CLEANUP_KEEP = 5
DBBACKUP_COMPRESS = True
DBBACKUP_GPG_RECIPIENT = 'my_gpg_key@example.com'  # If you want encryption
```

**Crontab entry** (runs daily at 2am):

``` bash
0 2 * * * /path/to/venv/bin/python /path/to/project/manage.py dbbackup_cleanup && \
 /path/to/venv/bin/python /path/to/project/manage.py dbbackup --clean --compress --encrypt
```

- `--clean` triggers cleanup after backup, but you can also run `dbbackup_cleanup` separately.

**Command line usage** to restore the latest backup:

``` bash
python manage.py dbrestore --noinput --decrypt
```

------------------------------------------------------------------------

### **Summary**

1.  **Install** `django-dbbackup`, **configure** it in `settings.py`.
2.  **Choose storage** (local or remote).
3.  **Run commands** (`dbbackup`, `dbrestore`, `mediabackup`, etc.) from the CLI or a scheduler.
4.  **Optional enhancements**: GPG encryption, compression, and cleanup for old backups.
5.  **Test** your backup and restore procedure regularly to ensure your data can be recovered when needed.

With these steps, you’ll have a robust backup and restore process in place for your Django project using **django-dbbackup**.

Back to top
