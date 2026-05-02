# PostgreSQL Learning Guide - Chinook Dataset

A hands-on SQL guide for beginners using the Chinook music store dataset.


## Setup

### 1. Clone this repo
git clone https://github.com/jaysreeb/postgres-sample-dbs.git

### 2. Create a sandbox database
psql -U your_username -c "CREATE DATABASE sandbox;"

### 3. Load chinook into your sandbox
psql -U your_username -d sandbox -f chinook.sql

### 4. Fix the search path (do this once)
psql -U your_username -d sandbox
ALTER ROLE your_username SET search_path TO public;

---

## Level 1 — JOINs, Aggregations, GROUP BY, HAVING

### Before writing any query — inspect your tables
\dt                    -- list all tables
\d "Invoice"           -- see columns of a specific table

### Query 1 — Revenue by country (no JOIN needed)
...

### Query 2 — Top customers by spend (your first JOIN)
...

### Common mistake I made
...

### WHERE vs HAVING
...
