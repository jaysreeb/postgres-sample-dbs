# PostgreSQL Learning Guide: Chinook Dataset

A hands-on SQL guide for beginners using the Chinook music store dataset.  

> **Dataset credit:** [Neon Database Labs](https://github.com/neondatabase-labs/postgres-sample-dbs)

---

## What is Chinook?

Chinook is a sample music store database. It has artists, albums, tracks, customers, invoices, and employees, enough tables to write real, meaningful queries without getting overwhelmed.

**Tables you'll use most:**

| Table | What it contains |
|---|---|
| `Artist` | Artist names |
| `Album` | Albums, linked to artists |
| `Track` | Individual songs, linked to albums |
| `Customer` | Customer name, country, email |
| `Invoice` | Purchase records, linked to customers |
| `InvoiceLine` | Individual items on each invoice |
| `Genre` | Music genres |

---

## Setup

### 1. Clone this repo

```bash
git clone https://github.com/jaysreeb/postgres-sample-dbs.git
cd postgres-sample-dbs
```

### 2. Create a sandbox database

Never load practice data into your main postgres database. Create a safe sandbox:

```bash
psql -U your_username -c "CREATE DATABASE sandbox;"
```

Replace `your_username` with your PostgreSQL user (e.g. `meeows`).

### 3. Load chinook into your sandbox

```bash
psql -U your_username -d sandbox -f chinook.sql
```

You'll see a stream of `CREATE TABLE` and `INSERT 0 1` lines. That's normal, it means it's working.

### 4. Fix the search path (do this once)

The chinook dump blanks out the default search path. Fix it permanently so you don't have to type `public.` before every table name:

```bash
psql -U your_username -d sandbox
```

Then inside psql:

```sql
ALTER ROLE your_username SET search_path TO public;
```

Reconnect after this for it to take effect.

### 5. Verify everything loaded

```sql
\dt
```

You should see all the tables listed. Then:

```sql
SELECT COUNT(*) FROM "Artist";   -- should return 275
SELECT COUNT(*) FROM "Track";    -- should return 3503
```

---

## Things that will confuse you (and why)

### Why do table names need double quotes?

PostgreSQL converts all unquoted identifiers to lowercase. Chinook was created with capital letters-`"Artist"`, `"Album"`, `"Track"` so you must use double quotes to match the exact case.

```sql
SELECT * FROM artist;     -- error: relation "artist" does not exist
SELECT * FROM "Artist";   -- works
```

**Lesson:** In your own projects, always use `snake_case` - `artist`, `track_id`, `invoice_date`. You'll never need double quotes and life gets much simpler.

### Why did I need `public.` at first?

The chinook dump contains this line at the top:

```sql
SELECT pg_catalog.set_config('search_path', '', false);
```

This blanked out the search path, so Postgres didn't know where to look for your tables. The `ALTER ROLE` fix in Step 4 solves this permanently.

---

## Before writing any query - inspect your tables

This habit saves you from unnecessary JOINs and wrong column names:

```sql
\dt                  -- list all tables
\d "Invoice"         -- see all columns on the Invoice table
\d "Customer"        -- see all columns on the Customer table
```

---

## Level 1 (Aggregations, JOINs, GROUP BY, HAVING)

### Query 1: Revenue by country

**Question:** How much total revenue did each country generate? Show top 10, highest first.

Before writing the query, check `\d "Invoice"`. You'll see `"BillingCountry"` and `"Total"` are both right there - no JOIN needed.

```sql
SELECT "BillingCountry", SUM("Total") AS total_revenue
FROM "Invoice"
GROUP BY "BillingCountry"
ORDER BY total_revenue DESC
LIMIT 10;
```

**Result:**
```
 BillingCountry | total_revenue
----------------+---------------
 USA            |        523.06
 Canada         |        303.96
 France         |        195.10
 Brazil         |        190.10
 Germany        |        156.48
```

**Lesson:** Always inspect your tables before assuming you need a JOIN. You only JOIN when the data you need lives in a *different* table.

---

### Query 2: Top customers by total spend (your first JOIN)

**Question:** Who are the top 5 customers by total amount spent? Show their full name and country.

The `Invoice` table has `CustomerId` but no customer name. The `Customer` table has the name. You need both — this is when you JOIN.

#### Common mistake first

```sql
SELECT "FirstName", "LastName", "Country", "Total"
FROM "Customer"
INNER JOIN "Invoice" ON "Customer"."CustomerId" = "Invoice"."CustomerId"
ORDER BY "Total" DESC
LIMIT 5;
```

This looks reasonable but gives you the **single largest invoice** per customer, not their total lifetime spend. A customer with 10 small invoices would rank lower than someone with one big invoice.

#### Correct query

```sql
SELECT
    "FirstName",
    "LastName",
    "Country",
    SUM("Total") AS total_spent
FROM "Customer"
INNER JOIN "Invoice" ON "Customer"."CustomerId" = "Invoice"."CustomerId"
GROUP BY "Customer"."CustomerId", "FirstName", "LastName", "Country"
ORDER BY total_spent DESC
LIMIT 5;
```

**Result:**
```
 FirstName |  LastName  |    Country     | total_spent
-----------+------------+----------------+-------------
 Helena    | Holý       | Czech Republic |       49.62
 Richard   | Cunningham | USA            |       47.62
 Luis      | Rojas      | Chile          |       46.62
```

**Two things to notice:**
- `SUM("Total")` adds up all invoices per customer, not just the biggest one
- `GROUP BY "Customer"."CustomerId"` - always group by the primary key first, then include the other columns you're selecting. Postgres requires every non-aggregated column in SELECT to also be in GROUP BY.

---

### Query 3: HAVING (filtering after aggregation)

**Question:** Which customers have spent more than $40 in total?

You cannot use `WHERE` here because the `SUM` doesn't exist yet when `WHERE` runs. Use `HAVING` instead, it filters *after* the grouping is done.

```sql
SELECT
    "FirstName",
    "LastName",
    "Country",
    SUM("Total") AS total_spent
FROM "Customer"
INNER JOIN "Invoice" ON "Customer"."CustomerId" = "Invoice"."CustomerId"
GROUP BY "Customer"."CustomerId", "FirstName", "LastName", "Country"
HAVING SUM("Total") > 40
ORDER BY total_spent DESC;
```

**The rule to memorize:**

| Clause | When it runs | Use it for |
|---|---|---|
| `WHERE` | Before grouping | Filtering raw rows and columns |
| `HAVING` | After grouping | Filtering aggregated values like SUM, COUNT |

```sql
WHERE SUM("Total") > 40    -- SUM doesn't exist yet at this point
HAVING "Country" = 'USA'   -- technically works but use WHERE for non-aggregates
```

---

### Query 4: Two JOINs (chaining three tables)

**Question:** Which artists have more than 10 tracks in the database?

The chain is: `Artist` → `Album` → `Track`  
You need two JOINs to walk that chain.

```sql
SELECT
    "Artist"."Name",
    COUNT("Track"."TrackId") AS total_tracks
FROM "Artist"
INNER JOIN "Album" ON "Artist"."ArtistId" = "Album"."ArtistId"
INNER JOIN "Track" ON "Album"."AlbumId" = "Track"."AlbumId"
GROUP BY "Artist"."ArtistId", "Artist"."Name"
HAVING COUNT("Track"."TrackId") > 10
ORDER BY total_tracks DESC;
```

**Result:**
```
     Name      | total_tracks
---------------+--------------
 Iron Maiden   |          213
 U2            |          135
 Led Zeppelin  |          114
 Metallica     |          112
 Deep Purple   |           92
```

---

## Level 1 (What you now know)

| Concept | Query where you used it |
|---|---|
| `SUM` + `GROUP BY` | Total revenue by country, total spend per customer |
| `INNER JOIN` | Linking Customer → Invoice |
| Two `INNER JOIN`s | Chaining Artist → Album → Track |
| `HAVING` | Filtering customers above $40, artists above 10 tracks |
| `COUNT` | Counting tracks per artist |
| `WHERE` vs `HAVING` | Understood conceptually and practically |

---

## What's next

- **Level 2** — Subqueries and CTEs (breaking complex problems into readable steps)
- **Level 3** — Window functions (`RANK`, `LAG`, `SUM OVER`) — where SQL gets genuinely powerful
- **Level 4** — `EXPLAIN ANALYZE` and indexes — understanding *why* a query is slow

---

## Author

**Jaysree B**

Follow me on: [Medium](https://medium.com/@jaysreeb)

GitHub: [github.com/jaysreeb](https://github.com/jaysreeb)