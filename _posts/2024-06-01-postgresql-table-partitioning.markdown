---
layout: post
title: "Postgresql Table Partitioning"
date: 2024-06-01 22:22:00 +0700
comments: true
categories: database postgresql partitioning
---

On PostgreSQL, table partitioning can be a powerful tool to improve performance.
Here are the steps and best practices for implementing and managing table
partitioning in PostgreSQL:

### 1. Understanding PostgreSQL Partitioning Types

PostgreSQL supports several partitioning methods:

- **Range Partitioning**: Divides a table into ranges based on a column value,
  like dates or numeric ranges.
- **List Partitioning**: Divides a table based on a list of discrete values,
  such as categories or regions.
- **Hash Partitioning**: Distributes data evenly using a hash function, useful
  for even data distribution when no clear range or list exists.
- **Composite Partitioning**: Combines multiple partitioning strategies, such as
  range partitioning within a list partition.

### 2. Creating Partitioned Tables

#### Example: Range Partitioning by Date

Let's create a table to store sales data, partitioned by year.

```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    amount DECIMAL NOT NULL
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2022 PARTITION OF sales
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE sales_2023 PARTITION OF sales
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
```

### 3. Adding Indexes and Constraints

Each partition can have its own indexes and constraints to optimize queries and
enforce data integrity.

```sql
CREATE INDEX ON sales_2022 (sale_date);
CREATE INDEX ON sales_2023 (sale_date);
```

### 4. Query Optimization

Ensure queries include the partition key to enable partition pruning. This
ensures only the relevant partitions are scanned.

```sql
SELECT * FROM sales WHERE sale_date BETWEEN '2023-01-01' AND '2023-06-30';
```

### 5. Managing Partitions

Regular maintenance and management of partitions are crucial:

#### Adding New Partitions

Add new partitions as needed, based on your data growth patterns.

```sql
CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

#### Removing Old Partitions

Remove or archive old partitions to free up space and improve performance.

```sql
DROP TABLE sales_2022;
```

### 6. Monitoring Performance

Use PostgreSQL’s built-in tools to monitor performance and ensure partitions are
balanced:

- **`pg_stat_user_tables`**: Provides statistics on table usage.
- **`EXPLAIN`**: Analyzes query execution plans to ensure partition pruning is
  working correctly.

### 7. Example Scenarios

**Scenario 1: Time-Series Data**

For log data with timestamped entries, partitioning by month:

```sql
CREATE TABLE logs (
    id SERIAL,
    log_date TIMESTAMP,
    log_message TEXT
) PARTITION BY RANGE (log_date);

CREATE TABLE logs_2023_01 PARTITION OF logs
    FOR VALUES FROM ('2023-01-01') TO ('2023-02-01');

CREATE TABLE logs_2023_02 PARTITION OF logs
    FOR VALUES FROM ('2023-02-01') TO ('2023-03-01');
```

**Scenario 2: Regional Data**

For sales data partitioned by region:

```sql
CREATE TABLE sales (
    id SERIAL,
    region TEXT,
    sale_amount DECIMAL
) PARTITION BY LIST (region);

CREATE TABLE sales_north PARTITION OF sales
    FOR VALUES IN ('North');

CREATE TABLE sales_south PARTITION OF sales
    FOR VALUES IN ('South');
```

### 8. Best Practices

- **Partition Key Choice**: Ensure the partition key is chosen based on query
  patterns and data distribution.
- **Balanced Partitions**: Avoid partitions that are too large or too small;
  they should be balanced for optimal performance.
- **Automate Partition Management**: Use scripts or tools to automate the
  creation, maintenance, and removal of partitions.
- **Regularly Monitor**: Continuously monitor performance metrics and adjust the
  partitioning strategy as needed.

### 9. Minimum PostgreSQL Version

Table partitioning in PostgreSQL has been available for a long time, but the
implementation and features have significantly improved over different
versions. Here's a brief overview of the capabilities in various versions:

#### PostgreSQL 10

- **Declarative Partitioning**: PostgreSQL 10 introduced declarative table
partitioning, making it much easier to create and manage partitioned tables.
- **Range and List Partitioning**: Basic support for range and list
partitioning was introduced.

Example:
```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    amount DECIMAL NOT NULL
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2022 PARTITION OF sales
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
```

#### PostgreSQL 11

- **Partition Pruning**: Improved partition pruning at execution time.
- **Primary Keys and Foreign Keys**: Support for primary keys, unique indexes,
and foreign keys on partitioned tables.

#### PostgreSQL 12

- **Automatic Partition Creation**: Added support for default partitions.
- **Improved Performance**: Enhancements in partition pruning during query
planning.

#### PostgreSQL 13

- **Partition-wise Joins and Aggregation**: Optimizations for partition-wise
joins and aggregation.
- **Incremental Sorting**: Improved performance for ordered queries on
partitioned tables.

### PostgreSQL 14 and Later

- **Improved Index Support**: Further enhancements in index management and
performance.
- **Improved Foreign Key Support**: Enhanced foreign key functionality on
partitioned tables.

### Minimum Version Requirement

To use declarative partitioning with basic range and list partitioning
capabilities, you need at least **PostgreSQL 10**. However, for more advanced
features and improved performance, consider using **PostgreSQL 11 or later**.
Each subsequent version has introduced significant enhancements to partitioning
features and performance optimizations, so using the latest stable version is
recommended for the best results.

### Example with PostgreSQL 10

Here’s a quick example of creating and using partitioned tables in PostgreSQL
10:

1. **Create Partitioned Table**:
    ```sql
    CREATE TABLE sales (
        id SERIAL PRIMARY KEY,
        sale_date DATE NOT NULL,
        amount DECIMAL NOT NULL
    ) PARTITION BY RANGE (sale_date);
    ```

2. **Create Partitions**:
    ```sql
    CREATE TABLE sales_2022 PARTITION OF sales
        FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

    CREATE TABLE sales_2023 PARTITION OF sales
        FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
    ```

3. **Insert Data**:
    ```sql
    INSERT INTO sales (sale_date, amount) VALUES ('2022-06-15', 100.00);
    INSERT INTO sales (sale_date, amount) VALUES ('2023-03-20', 150.00);
    ```

4. **Query Data**:
    ```sql
    SELECT * FROM sales WHERE sale_date BETWEEN '2023-01-01' AND '2023-06-30';
    ```

By using PostgreSQL 10 or later, you can take advantage of these partitioning
features to improve the performance and manageability of your large tables.

### Conclusion

Partitioning in PostgreSQL can significantly enhance performance by reducing
query response times and improving data management. By carefully selecting
partition keys, regularly maintaining partitions, and monitoring performance,
you can ensure your database operates efficiently.
