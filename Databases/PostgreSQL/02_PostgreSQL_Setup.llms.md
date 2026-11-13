# PostgreSQL Setup

postgreSQL

Author

Benedict Thekkel

## Install PostgreSQL

### Update

> Update the package list before installing PostgreSQL

``` sh
sudo apt-get update
```

### Install PostgreSQL

> Install PostgreSQL and its client packages:

``` sh
sudo apt-get install postgresql postgresql-contrib
```

### Verify Installation

> Check the PostgreSQL version to verify installation:

``` sh
psql --version
```

## Initialize and Configure PostgreSQL

### Initialize PostgreSQL Cluster:

> PostgreSQL installation typically initializes a cluster automatically. If not, you can do it manually:

``` sh
sudo pg_createcluster 13 main --start
```

### Start PostgreSQL Service:

``` sh
sudo service postgresql start
```

### Configure PostgreSQL User and Password

> By default, PostgreSQL creates a user postgres with no password. Set a password for the postgres user if needed:

``` sh
sudo -u postgres psql
\password postgres
\q
```

## Access PostgreSQL

### Access PostgreSQL Shell (psql):

``` sh
sudo -u postgres psql
```

### Basic Commands

> Once in psql, you can execute SQL commands. Here are a few essential commands:

- : List all databases.
- atabase_name: Connect to a specific database.
- : List all tables in the current database.
- : Quit psql.

## Configure PostgreSQL for Use with Applications

### Edit PostgreSQL Configuration (if necessary):

> Modify PostgreSQL configuration files (postgresql.conf and pg_hba.conf) located typically in /etc/postgresql//main/ to adjust settings like authentication methods, listen addresses, etc.

### Connect PostgreSQL with Django or Other Applications

> Update your application’s database settings (settings.py for Django) to use PostgreSQL with the appropriate credentials (USER, PASSWORD, HOST, PORT).

## Remove PostgreSQL

### Stop

> If PostgreSQL is running, stop the service:

``` sh
sudo service postgresql stop
```

### Remove

> Remove PostgreSQL and its configuration files:

``` sh
sudo apt-get remove --purge postgresql\*
```

### Clean up

> Remove any remaining configuration files:

``` sh
sudo rm -r /etc/postgresql/
sudo rm -r /etc/postgresql-common/
sudo rm -r /var/lib/postgresql/
```

### Additional Tips

- Backup and Restore: Use tools like pg_dump and pg_restore for database backups and restoration.
- Security: Ensure PostgreSQL is configured securely, especially if accessible from outside WSL.

Back to top
