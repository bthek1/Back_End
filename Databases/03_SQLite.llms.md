# SQLite

SQLite is a lightweight, self-contained, serverless, zero-configuration, transactional SQL database engine. It is widely used for embedded databases, prototyping, and development purposes.

Author

Benedict Thekkel

## Introduction to SQLite

- Origin: Created by D. Richard Hipp in 2000.
- License: Public domain.
- Characteristics:
  - Serverless: No need for a separate server process.
  - Zero Configuration: No setup or administration required.
  - Self-contained: A single library file.
  - Cross-platform: Works on many operating systems.

## Installation

``` sh
sudo apt update
sudo apt install sqlite3
```

## Basic Usage

### Creating a Database

``` sh
sqlite3 mydatabase.db
```

### Basic Commands

#### Open an existing database

``` sh
sqlite3 mydatabase.db
```

#### Creating a Table

``` sql
CREATE TABLE mytable (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    age INTEGER
);
```

#### Inserting Data

``` sql
INSERT INTO mytable (name, age) VALUES ('Alice', 30);
```

#### Querying Data

``` sql
SELECT * FROM mytable;
```

#### Updating Data

``` sql
UPDATE mytable SET age = 31 WHERE name = 'Alice';
```

#### Deleting Data

``` sql
DELETE FROM mytable WHERE name = 'Alice';
```

## Features

### Data Types

> SQLite uses dynamic typing. The data types it supports are:

- NULL
- INTEGER
- REAL
- TEXT
- BLOB

### Transactions

``` sql
BEGIN TRANSACTION;
INSERT INTO mytable (name, age) VALUES ('Bob', 25);
COMMIT;
```

### Indexes

``` sql
CREATE INDEX idx_name ON mytable (name);
```

### Views

``` sql
CREATE VIEW myview AS SELECT name, age FROM mytable WHERE age > 25;
SELECT * FROM myview;
```

### Triggers

``` sql
CREATE TRIGGER before_insert_mytable
BEFORE INSERT ON mytable
FOR EACH ROW
BEGIN
    SELECT RAISE(FAIL, "Duplicate name") WHERE EXISTS (SELECT 1 FROM mytable WHERE name = NEW.name);
END;
```

### Stored Procedures

> SQLite does not support stored procedures like other RDBMSs. However, you can use triggers and user-defined functions to achieve similar functionality.

## Security

- Database Encryption: Use SQLCipher for database encryption.
- Access Control: SQLite does not provide built-in user management and access control.

## Performance Tuning

- Indexes: Proper indexing can significantly improve query performance.
- PRAGMA Statements: Use PRAGMA commands to modify the database behavior.

``` sql
PRAGMA synchronous = OFF;
PRAGMA journal_mode = WAL;
```

## Backup and Restore

### Backup

``` sql
sqlite3 mydatabase.db ".backup 'backup.db'"
```

### Restore

``` sql
sqlite3 backup.db ".restore 'mydatabase.db'"
```

## Integration with Programming Languages

> Python

``` python
import sqlite3

# Connect to the database
conn = sqlite3.connect('mydatabase.db')

# Create a cursor object
cursor = conn.cursor()

# Execute SQL commands
cursor.execute("CREATE TABLE IF NOT EXISTS mytable (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)")
cursor.execute("INSERT INTO mytable (name, age) VALUES ('Alice', 30)")
conn.commit()

# Query the database
cursor.execute("SELECT * FROM mytable")
rows = cursor.fetchall()
for row in rows:
    print(row)

# Close the connection
conn.close()
```

> JavaScript

``` java
const sqlite3 = require('sqlite3').verbose();

// Connect to the database
let db = new sqlite3.Database('mydatabase.db');

// Create a table
db.run('CREATE TABLE IF NOT EXISTS mytable (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)');

// Insert data
db.run('INSERT INTO mytable (name, age) VALUES (?, ?)', ['Alice', 30]);

// Query the database
db.all('SELECT * FROM mytable', [], (err, rows) => {
  if (err) {
    throw err;
  }
  rows.forEach((row) => {
    console.log(row);
  });
});

// Close the database connection
db.close();
```

## Tools and Utilities

- SQLite Command Line: The primary tool for interacting with SQLite databases.
- DB Browser for SQLite: A visual tool to manage SQLite databases.
- SQLiteStudio: Another powerful SQLite database manager.

## Best Practices

- Normalization: Properly normalize your database schema.
- Indexes: Use indexes to optimize query performance but avoid over-indexing.
- Backups: Regularly back up your database.
- Testing: Thoroughly test database schema changes.

Back to top
