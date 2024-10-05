## Analysis 
1. Create indexes if often less than 15% of the table's rows are going to be retrieved.
2. Get the suggestion what tables need an index by looking at `seq_scan` and `seq_tup_read`. These columns show what tables were used in sequential scans
    ```sql
    SELECT schemaname,
           relname as table_name,
           seq_scan, -- Number of sequential scans initiated on this table
           seq_tup_read, -- Number of live rows fetched by sequential scans
           idx_scan, -- Number of index scans initiated on this table
           idx_tup_fetch -- Number of live rows fetched by index scans
      FROM pg_stat_user_tables
     WHERE seq_scan > 0
     ORDER BY seq_tup_read DESC;
    ```
### Index selectivity

The number of distinct values in the indexed column divided by the number of records in the table is called a selectivity of an index.

Selectivity is one of factors influencing type of scanning the planner/optimezer will use for a query (Index Scan or Bitmap Index Scan).

1. Calculate the selectivity of the single column indexes you want to create
    ```sql
    SELECT ROUND(COUNT(DISTINCT rating)::NUMERIC / COUNT(*), 2) AS selectivity
      FROM BOOK;
    ```
    ```
    selectivity
    -----------
    0.04
    ```
    ```sql
    SELECT ROUND(COUNT(DISTINCT publication_date)::NUMERIC / count(*), 2) AS selectivity
      FROM BOOK;
    ```
    ```
    selectivity
    -----------
    0.04
    ```
2. The best selectivity is 1. Only unique indexes on NOT NULL columns *guaranteed* to have such selectivity.
    ```sql
    SELECT ROUND(COUNT(DISTINCT isbn)::NUMERIC / COUNT(*), 2) AS idx_selectivity
      FROM BOOK;
    ```
    ```
    selectivity
    -----------
    1
    ```

Prefer indexing columns with selectivity greater than > 0.85.


## B-Tree indexes
### Index Scan

Index Scan uses an index to find rows matching a predicate.
It finds each row in the index and then reads the actual data from the table.

### Bitmap Index Scan

Bitmap Index Scan is used when the planner decides that it is faster to combine multiple indexes to find the rows that match a predicate.
If index selectivity is bad (approximately less than < 0.85), the planner may decide to use Bitmap Index Scan instead of Index Scan.

## GIN indexes

GIN indexes are "inverted indexes".
An inverted index contains a separate entry for each component value, and can efficiently handle queries that test for the presence of specific component values.

1. B-Tree index is not used in queries with `LIKE`
    ```sql
    EXPLAIN ANALYZE
    SELECT b.isbn, 
           b.title,
           b.publication_date,
           b.rating
      FROM book b
     WHERE b.title LIKE 'Patterns%';
    ```
    ```
    QUERY PLAN
    ----------
    "Seq Scan on book b  (cost=0.00..3917.09 rows=10 width=57) (actual time=0.017..18.060 rows=1 loops=1)"
    "  Filter: ((title)::text ~~ 'Patterns%'::text)"
    "  Rows Removed by Filter: 100006"
    "Planning Time: 0.733 ms"
    "Execution Time: 18.096 ms"
    ```

2. To use a trigram GIN indexes, create an extension
    ```sql
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    ```

3. Create the trigtram GIN index
    ```sql
    CREATE INDEX idx_book_title_trgm ON book USING gin (title gin_trgm_ops);
    ```

4. Repeat the `SELECT` query
    ```
    QUERY PLAN
    ----------
    "Bitmap Heap Scan on book b  (cost=100.08..138.37 rows=10 width=57) (actual time=0.123..0.125 rows=1 loops=1)"
    "  Recheck Cond: ((title)::text ~~ 'Patterns%'::text)"
    "  Rows Removed by Index Recheck: 3"
    "  Heap Blocks: exact=1"
    "  ->  Bitmap Index Scan on idx_book_title_trgm  (cost=0.00..100.08 rows=10 width=0) (actual time=0.112..0.113 rows=4 loops=1)"
    "        Index Cond: ((title)::text ~~ 'Patterns%'::text)"
    "Planning Time: 0.509 ms"
    "Execution Time: 0.167 ms"
    ```

## Hash indexes

Hash index is a flat structure unlike B-Tree.

Hash indexes can only handle simple equality comparisons (using the `=` operator)

The main advantage of Hash indexes over B-Tree is space.
On a very large data sets Hash indexes takes less space compared to B-Tree and allow pretty fast lookups.

Only B-Tree indexes can be unique, hash indexes doesn't support this feature.

To use Hash indexes PostgreSQL 10+ is required.


## Find indexes that have never been used
```sql
SELECT s.schemaname,
       s.relname AS table_name,
       s.indexrelname AS index_name,
       s.idx_scan AS times_used,
       pg_size_pretty(pg_relation_size(t.relid)) AS table_size,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size,
       idx.indexdef AS index_ddl
  FROM pg_stat_user_indexes s
  JOIN pg_stat_user_tables t 
    ON s.relname = t.relname
  JOIN pg_index i
    ON s.indexrelid = i.indexrelid
  JOIN pg_indexes AS idx 
    ON s.indexrelname = idx.indexname
   AND s.schemaname = idx.schemaname
 WHERE s.idx_scan = 0 -- no scans
   AND 0 <> ALL(i.indkey) -- 0 in the array means this is an expression index
   AND NOT i.indisunique -- no unique index
 ORDER BY pg_relation_size(s.indexrelid) DESC;
```

## <a name="2b1d46eeefdef5de7e4b4d5ce62849c4"></a>Keep statistics updated

The planner/optimizer relies on statistics about the contents of tables in order to generate good plans for queries.

Make sure to run `VACUUM ANALYZE` to keep statistics up to date and recover or reuse disk space occupied by updated or deleted rows.

PostgreSQL has *autovacuum* daemon automating the execution of `VACUUM` and `ANALYZE` commands.
It is highly recommended to enable autovacuum feature.

Check when the tables was last vacuumed and analyzed
```sql
SELECT schemaname, 
       relname, 
       last_vacuum, 
       vacuum_count, 
       last_analyze, 
       analyze_count 
  FROM pg_stat_user_tables;
```

---

* https://www.postgresql.org/docs/13/routine-vacuuming.html

## <a name="8f264ba3b501b66608e02c8c5efc2c58"></a>Detect slow queries

To detect slow queries `pg_stat_statements` extension is required.
It has to be preloaded using `shared_preload_libraries=pg_stat_statements`.

1. Check that `pg_stat_statements` was preloaded
    ```sql
    SHOW shared_preload_libraries;
    ```
    ```
    shared_preload_libraries
    ------------------------
    "pg_stat_statements"
    ```

2. Create extension for tracking planning and execution statistics of all executed SQL statements
    ```sql
    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    ```

3. Get 20 slowest SQL queries
    ```sql
    SELECT SUBSTRING(query, 1, 40) AS short_query,
           ROUND(( 100 * total_exec_time / SUM(total_exec_time) OVER ())::NUMERIC, 2) AS percent,
           ROUND(total_exec_time::numeric, 2) AS total_exec_time,
           calls,
           ROUND(mean_exec_time::numeric, 2) AS mean,
           query
      FROM pg_stat_statements
     ORDER BY total_exec_time DESC
     LIMIT 20;
    ```

---

* https://www.postgresql.org/docs/13/runtime-config-client.html#RUNTIME-CONFIG-CLIENT-PRELOAD