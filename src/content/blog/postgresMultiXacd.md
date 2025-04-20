---
title: "Journey through Multixacd in Postgres"
description:
  'Photo By <a class="underline" href="https://unsplash.com/@impatrickt?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Patrick Tomasso</a> via <a class="underline" href="https://unsplash.com/es/fotos/una-pintura-en-el-techo-de-un-edificio-1rBg5YSi00c?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
  '
icon: "3"
pubDate: "Jun 19 2024"
heroImage: "/src/assets/multixacd_postgres.jpg"
---

## The Problem
Database corruptions are a dime a dozen, they happen in my local dev environment more than I could  admit, However some of them elude them, primarily:
```
db  | 2025-04-18 11:48:45.483 UTC [33] ERROR:  MultiXactId 197 has not been created yet -- apparent wraparound
db  | 2025-04-18 11:48:45.483 UTC [33] STATEMENT:  select * from sample;
```
Here `sample` denotes the table name that is affected.
MultiXactId is a secondary data structure that tracks multiple transactions holding locks on the same row.
This error in a nuteshell means that the database is expecting a lock on a row held by a transaction somewhere which has basically corrupted the table, thereby corrupting a part of the database as well.
More details on the [problem](https://medium.com/in-the-weeds/postgresql-multixactid-error-in-vacuum-106af8fbf022)
### **Debugging**
Usually the solutions to this include locating the courrpted row,deleting it and vaccuming the table.
But in my case these solutions would not work which usually points towards the last resort in any data corruption, which is to restore the data. 
But in my case I did not want to restore the data from scratch as I had some pending work in my databse.
## **The Solution**
Instead of restoring the whole db from scratch, I wanted to restore the contents of the affected table only so off I went to explore this. Before moving ahead a snapshot of the db or of the table would help,
Lets assume, the name of the dump is `db_dump.db` which is a compressed format of the dump
the name of the affected table is `sample`
### **Understanding PostgreSQL Dump Formats**
Before we begin, it's important to understand that PostgreSQL dumps come in two main formats:
- Plain text format (created with `pg_dump -Fp`): Contains SQL commands
- Custom format (created with `pg_dump -Fc`): A compressed, non-text format that allows selective restoration

### **Restoring a Corrupted Table from a PostgreSQL Dump**
When you're facing a table corrupted by MultiXactId issues, the selective restore approach is perfect for targeted recovery. Let me guide you through restoring just your corrupted "sample" table from your "dump.db" backup file.
### **Step 1: Verify the Dump Format and Content**
First, let's verify that your dump file is in the custom format and identify the table components:
```bash

    pg_restore -l dump.db | grep -i sample


```
This will show you all objects related to your "sample" table, including:
 1. Table definition
 2. Table data
 3. Indexes
 4. Constraints
 5. Triggers
 6. Any sequences owned by the table

You should see output similar to:
```

    123; 1259 12345 TABLE sample postgres
    456; 0 12345 TABLE DATA sample postgres
    789; 1259 12346 SEQUENCE sample_id_seq postgres
    ...

```
### **Step 2: Prepare Your Database Environment**
Since the table is corrupted, we'll need to remove it before restoring. Connect to your database and:
```sql

    -- First, disable any foreign key constraints pointing to this table
    SELECT 'ALTER TABLE ' || relname || ' DROP CONSTRAINT ' || conname || ';'
    FROM pg_constraint
    JOIN pg_class ON pg_constraint.conrelid = pg_class.oid
    WHERE confrelid = (SELECT oid FROM pg_class WHERE relname = 'sample');

    -- Execute the generated statements to remove the constraints

    -- Then drop the corrupted table
    DROP TABLE sample;

```
### **Step 3: Create a Restoration List File**
Extract just the components needed for the "sample" table:
```bash
    # Create a file containing all components related to the sample table
    pg_restore -l dump.db | grep -E "(TABLE.*sample|TABLE DATA.*sample|INDEX.*sample|CONSTRAINT.*sample|TRIGGER.*sample|SEQUENCE.*sample)" > sample_restore.list

```
### **Step 4: Restore in Three Phases**
Now we'll restore the table in three distinct phases to ensure everything is properly created:
### **Phase 1: Restore Table Structure and Related Objects**
```bash
    # Restore the pre-data section (table definitions, sequences, etc.)
    pg_restore --section=pre-data --use-list=sample_restore.list --dbname=your_database dump.db

```
This command recreates the table structure, sequences, and other schema objects before we load any data.
### **Phase 2: Restore Table Data**
```bash
    # Restore just the data
    pg_restore --section=data --use-list=sample_restore.list --dbname=your_database dump.db

```
This loads all the rows from your backup into the newly created table structure.
### **Phase 3: Restore Indexes, Constraints, and Triggers**
```bash

    # Restore post-data objects (indexes, constraints, triggers)
    pg_restore --section=post-data --use-list=sample_restore.list --dbname=your_database dump.db

```

Creating indexes and constraints after loading data is much more efficient, especially for larger tables.

### **Step 5: Verify the Restoration**
After restoration, verify that your table is working correctly:
```sql

    -- Check that all data is present
    SELECT COUNT(*) FROM sample;

    -- Check table structure
    \d sample

    -- Run a simple query to verify data access
    SELECT * FROM sample LIMIT 10;

    -- Check for any unusual performance or lock issues
    SELECT relation::regclass, locktype, mode, granted
    FROM pg_locks JOIN pg_class ON pg_locks.relation = pg_class.oid
    WHERE relname = 'sample';

```



With these steps we can restore only the corrupted data from the db.