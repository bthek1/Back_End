# MariaDB and MySQL

MariaDB is a popular open-source relational database management system (RDBMS) that is a fork of MySQL. It is designed for high performance, scalability, and reliability.

Author

Benedict Thekkel

## Introduction to MySQL

- Origin: Developed by MySQL AB in 1995, acquired by Oracle Corporation in 2010.
- License: Open-source under the GPL license, with a commercial version available.
- Compatibility: Supports multiple platforms including Linux, Windows, macOS.

## Installation

``` sh
sudo apt update
sudo apt install mysql-server
```

## Introduction to MariaDB

- Origin: Created by the original developers of MySQL after Oracle acquired MySQL.
- Compatibility: Highly compatible with MySQL, allowing easy migration of databases.
- License: Open-source under the GPL license.

## Installation

``` sh
sudo apt update
sudo apt install mariadb-server
```

## Configuration

- Configuration File: The primary configuration file is my.cnf or my.ini.
- Location: Typically found in /etc/mysql/ or /etc/ on Linux and in the MariaDB installation directory on Windows.

## Basic Usage

``` sh
sudo systemctl start mariadb
sudo systemctl stop mariadb
sudo systemctl enable mariadb
```

### Accessing the MariaDB Shell

``` sh
mysql -u root -p
```

### Creating a Database

``` sql
CREATE DATABASE mydatabase;
```

### Creating a User and Granting Permissions

``` sql
CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'mypassword';
GRANT ALL PRIVILEGES ON mydatabase.* TO 'myuser'@'localhost';
FLUSH PRIVILEGES;
```

## Database Operations

### Creating Tables

``` sql
USE mydatabase;
CREATE TABLE mytable (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    age INT
);
```

### Inserting Data

``` sql
INSERT INTO mytable (name, age) VALUES ('Alice', 30);
```

### Querying Data

``` sql
SELECT * FROM mytable;
```

### Updating Data

``` sql
UPDATE mytable SET age = 31 WHERE name = 'Alice';
```

### Deleting Data

``` sql
DELETE FROM mytable WHERE name = 'Alice';
```

## Advanced Features

### Stored Procedures

``` sql
DELIMITER //
CREATE PROCEDURE myprocedure()
BEGIN
    SELECT * FROM mytable;
END //
DELIMITER ;
CALL myprocedure();
```

### Triggers

``` sql
CREATE TRIGGER before_insert_mytable
BEFORE INSERT ON mytable
FOR EACH ROW
BEGIN
    SET NEW.age = IF(NEW.age IS NULL, 0, NEW.age);
END;
```

### Views

``` sql
CREATE VIEW myview AS SELECT name, age FROM mytable WHERE age > 25;
SELECT * FROM myview;
```

## Security

### Secure Installation

``` sh
sudo mysql_secure_installation
```

### User Privileges

> Grant specific privileges to users instead of using GRANT ALL PRIVILEGES.

## Performance Tuning

### Indexes

> Use indexes to speed up query performance.

``` sql
CREATE INDEX idx_name ON mytable (name);
```

### Query Optimization

> Use the EXPLAIN statement to analyze query performance.

``` sql
EXPLAIN SELECT * FROM mytable WHERE name = 'Alice';
```

### Caching

> Enable query caching in the configuration file.

## Backup and Restore

### Backup

``` sh
mysqldump -u root -p mydatabase > mydatabase.sql
```

### Restore

``` sh
mysql -u root -p mydatabase < mydatabase.sql
```

## Replication

- Master-Slave Replication: Set up replication for high availability and scalability.
- Configuration: Configure my.cnf on both master and slave servers.

## High Availability

- Galera Cluster: Use Galera Cluster for synchronous multi-master replication.

## Monitoring and Maintenance

- Monitoring Tools: Use tools like MySQLTuner and Percona Monitoring and Management (PMM) to monitor performance.
- Log Files: Monitor log files (error.log, slow-query.log) for troubleshooting.

Back to top
