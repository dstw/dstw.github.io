---
layout: post
title: "Using pg_partman in Postgresql for Table Partitioning"
date: 2024-06-01 22:22:00 +0700
comments: true
categories: database postgresql partitioning
---

`pg_partman` is a PostgreSQL extension that simplifies and automates the
management of partitioned tables. It is especially useful for managing
time-based and serial-based partitioning. `pg_partman` handles the creation,
maintenance, and removal of partitions, making it easier to manage partitioned
tables over time.

### Benefits of Using `pg_partman`

1. **Automated Partition Management**: Automatically creates new partitions as
needed based on your data growth patterns.
2. **Maintenance Tasks**: Automates tasks like dropping old partitions,
reindexing, and more.
3. **Ease of Use**: Simplifies the management of partitioned tables with
user-friendly functions and procedures.
4. **Flexibility**: Supports a wide range of partitioning strategies, including
time-based and serial-based partitioning.

### Installation

To use `pg_partman`, you need to install it. Here are the general steps:

1. **Install PostgreSQL Contrib Package**:
    ```bash
    sudo apt-get install postgresql-contrib
    ```

2. **Install `pg_partman`**:
    ```bash
    git clone https://github.com/pgpartman/pg_partman.git
    cd pg_partman
    make
    sudo make install
    ```

3. **Create the Extension**:
    ```sql
    CREATE EXTENSION pg_partman;
    ```

### Configuration

Once installed, you can configure `pg_partman` to manage your partitioned
tables. Here’s an example of setting up a time-based partitioning on a table:

1. **Create the Parent Table**:
    ```sql
    CREATE TABLE sales (
        id SERIAL PRIMARY KEY,
        sale_date TIMESTAMP NOT NULL,
        amount DECIMAL NOT NULL
    );
    ```

2. **Configure `pg_partman`**:
    ```sql
    -- Add the parent table to `pg_partman`
    SELECT partman.create_parent('public.sales', 'sale_date', 'time', 'monthly');
    ```

This command will configure `pg_partman` to manage the `sales` table, creating
monthly partitions based on the `sale_date` column.

### Managing Partitions

`pg_partman` provides several functions to manage partitions:

- **Creating New Partitions**:
    ```sql
    -- Manually run partition maintenance (usually done via cron job)
    SELECT partman.run_maintenance('public.sales');
    ```

- **Dropping Old Partitions**:
    ```sql
    -- Set retention policy to keep only the last 12 months of data
    SELECT partman.set_retention('public.sales', '12 months');
    ```

- **Viewing Partition Setups**:
    ```sql
    -- View configuration for the partition set
    SELECT * FROM partman.part_config WHERE parent_table = 'public.sales';
    ```

### Automating Maintenance

To ensure partitions are created and maintained automatically, set up a cron job
or a scheduled task to run `pg_partman` maintenance regularly.

Example Cron Job:
```bash
# Edit the crontab file
crontab -e

# Add a job to run maintenance every hour
0 * * * * psql -U postgres -d your_database -c "SELECT partman.run_maintenance('public.sales');"
```

### Example Usage

Here’s a complete example from setting up to querying a partitioned table using `pg_partman`:

1. **Setup**:
    ```sql
    CREATE TABLE sales (
        id SERIAL PRIMARY KEY,
        sale_date TIMESTAMP NOT NULL,
        amount DECIMAL NOT NULL
    );

    SELECT partman.create_parent('public.sales', 'sale_date', 'time', 'monthly');
    ```

2. **Insert Data**:
    ```sql
    INSERT INTO sales (sale_date, amount) VALUES ('2024-06-15', 100.00);
    INSERT INTO sales (sale_date, amount) VALUES ('2024-07-20', 150.00);
    ```

3. **Query Data**:
    ```sql
    SELECT * FROM sales WHERE sale_date BETWEEN '2024-06-01' AND '2024-06-30';
    ```

### Minimum PostgreSQL Version for `pg_partman`

The `pg_partman` extension is compatible with multiple versions of PostgreSQL.
However, the specific features and functionalities available can depend on the
PostgreSQL version you're using. As of the latest information, `pg_partman`
generally requires PostgreSQL version 10 or higher to function effectively. This
is because PostgreSQL 10 introduced declarative partitioning, which `pg_partman`
leverages for its operations.

- **PostgreSQL 10**: The minimum required version for `pg_partman`. This version
  introduced native support for declarative partitioning, which is essential for
`pg_partman` to manage partitions effectively.

### Features Across Versions

- **PostgreSQL 10**: Basic support for declarative partitioning. `pg_partman`
  can manage range and list partitioning.
- **PostgreSQL 11**: Improved performance with partition pruning, and support
  for primary keys and foreign keys on partitioned tables.
- **PostgreSQL 12**: Enhanced features like automatic creation of default
  partitions and better performance.
- **PostgreSQL 13 and later**: Further optimizations and support for more
  complex partitioning strategies and maintenance tasks.

### Installation and Setup

Here is a summary of how to install and configure `pg_partman` on a PostgreSQL
10 or later instance.

#### Step-by-Step Guide

1. **Install PostgreSQL Contrib Package**:
    ```bash
    sudo apt-get install postgresql-contrib
    ```

2. **Download and Install `pg_partman`**:
    ```bash
    git clone https://github.com/pgpartman/pg_partman.git
    cd pg_partman
    make
    sudo make install
    ```

3. **Create the `pg_partman` Extension in Your Database**:
    ```sql
    CREATE EXTENSION pg_partman;
    ```

#### Configure `pg_partman`

1. **Create a Parent Table**:
    ```sql
    CREATE TABLE sales (
        id SERIAL PRIMARY KEY,
        sale_date TIMESTAMP NOT NULL,
        amount DECIMAL NOT NULL
    );
    ```

2. **Add the Table to `pg_partman`**:
    ```sql
    SELECT partman.create_parent('public.sales', 'sale_date', 'time', 'monthly');
    ```

#### Automated Maintenance

To ensure partitions are created and managed automatically:

1. **Set Up a Cron Job**:
    ```bash
    # Edit the crontab file
    crontab -e

    # Add a job to run maintenance every hour
    0 * * * * psql -U postgres -d your_database -c "SELECT partman.run_maintenance('public.sales');"
    ```

### Summary

`pg_partman` is a powerful extension for managing partitioned tables in
PostgreSQL, especially for time-series data or large datasets that require
regular partition management. It automates the creation and maintenance of
partitions, simplifying database administration and improving performance. Using
`pg_partman` with PostgreSQL’s native partitioning features can greatly enhance
your database's efficiency and manageability.
`pg_partman` requires PostgreSQL 10 or higher due to the reliance on the
declarative partitioning features introduced in PostgreSQL 10. For optimal use,
particularly for advanced partition management and improved performance, using
PostgreSQL 11 or later is recommended.
