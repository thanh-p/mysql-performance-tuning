# mysql-performance-tuning
MySQL flow: Query => Parser => Optimizer => Executor => Storage Engine

# Why query slow?
* Query is a task that contains a lot of subtasks => need optimize each subtask.
* Execution subtask: call to storage engine, post retrieval operation (sorting, grouping).
* The query spends time in operations such as statistics, planning, locking, call to storage engine to retrieve row  
=> consume memory, CPU, IO 
* Query is slow? perform too many times or too slow
* Goal of optimization: eliminate or make operations faster.

# Spotting query problem:
## Where should we look?
* Statement Views: events_statements_summary_by_digest, sys.statement_analysis, sys.statements_with_full_table_scans
* Table & File I/O Ciews: table_io_waits_summary_by_index_usage, table_io_waits_summary_by_table
* Error Summary Views: events_errors_summary_by_account_by_error

## Create DB
```
CREATE DATABASE world;
CREATE DATABASE sakila;
```

## Import sample data
```
mysql -u root -p < world.sql
mysql -u root -p sakila < sakila-schema.sql
mysql -u root -p sakila < sakila-data.sql
```

## Use performance schema to analyze
```
USE performance_schema;
SHOW tables; 
```
## Check events_statements_summary_by_digest
```
SELECT   *
FROM     performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC limit 10\G
```
## Check cooked events_statements_summary_by_digest
```
SELECT * FROM   sys.statement_analysis LIMIT 10\G
```

## Important factors in events_statements_summary_by_digest
* DIGEST_TEXT: the query.
* COUNT_STAR: the number of time the query executed.
* SUM_TIMER_WAIT: the total time executing the query.
* SUM_LOCK_TIME: the total time waiting for table lock.  
Other analysis:
* SUM_ROWS_EXAMINED > SUM_ROWS_SENT: poor index usage
* SUM_SELECT_FULL_JOIN: join condition missing or need index.
* SUM_SELECT_RANGE_CHECK: need change index
* SUM_SORT_MERGE_PASSES: need larger sort buffer.

## Top 10 time consuming query
```
SELECT (100 * SUM_TIMER_WAIT / sum(SUM_TIMER_WAIT)
            OVER ()) percent,
            SUM_TIMER_WAIT AS total,
            COUNT_STAR AS calls,
            AVG_TIMER_WAIT AS mean,
            substring(DIGEST_TEXT, 1, 75)
  FROM  performance_schema.events_statements_summary_by_digest
            ORDER BY SUM_TIMER_WAIT DESC
            LIMIT 10;

SELECT *
FROM   sys.statement_analysis LIMIT 10\G
```

## Check for full scan problem
```
SELECT   *
FROM     sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC\G
```
no_index_used_count: full table scan count

## Check IO usage
```
SELECT OBJECT_TYPE, OBJECT_SCHEMA,
              OBJECT_NAME, INDEX_NAME,
              COUNT_STAR
         FROM performance_schema.table_io_waits_summary_by_index_usage
        WHERE OBJECT_SCHEMA = 'world'
              AND OBJECT_NAME = 'city'\G
```
no index usage results in large COUNT_STAR

## Check COUNT READ of IO of UPDATE statement
```
USE world;
describe city; 

SELECT *
FROM   city
WHERE  id = 5; 

SELECT *
FROM   city
WHERE  countrycode = 'nld';

SELECT *
FROM   city
WHERE  NAME = 'amsterdam';

SELECT *
FROM   performance_schema.table_io_waits_summary_by_table
WHERE  object_name = 'city' \G

UPDATE city
SET    NAME = 'Amsterdam1'
WHERE  NAME = 'Amsterdam'; 
```
=> COUNT_READ is large even for update if there is no index.

## Check Error and how many time dead lock occurs
```
SELECT *
FROM   performance_schema.events_errors_summary_by_account_by_error
WHERE  error_name = 'ER_LOCK_DEADLOCK' \G
```

# Analyze Queries

## EXPLAIN
```
explain format=json select * from city where name = 'London'\G
explain analyze select * from city where name = 'London'\G
explain analyze select * from city where countrycode = 'FRA';
explain analyze select * from countrylanguage where CountryCode = 'CHN';
```
* "access_type": "ALL" => scan all table
* explain analyze return tree of operations, read inside out.

```
EXPLAIN ANALYZE
    SELECT ci.ID, ci.Name, ci.District, co.Name AS Country, ci.Population
    From world.city ci
    INNER JOIN
     (SELECT Code, Name
     FROM world.country
     WHERE Continent = 'Europe'
     ORDER BY SurfaceArea
     LIMIT 10
     ) co ON co.Code = ci.CountryCode
    ORDER BY ci.Population DESC
    LIMIT 5\G
```

## Spotting jump in rumtime
* Use explain analyze
* Spot jump run actual time
* Spot different between estimated row vs actual row examined
  * Can be overestimate and optimizer scan whole table.
  * Can be fix by adding index or not use some factors in Where clause, for example, expressions.

## Hot cache behavior
* if a query is executed 2nd time, 3nd time, the performance is staying constant because the cache.

# Clustering index and chosing primary key
* An optimal primary key with respect to the clustered index is as small (in bytes) as possible, keeps increasing monotonically, and groups the rows you query frequently and within short time of each other.

# Indexing for Performance
* Reduce the rows examined
* Sort data
* Validate data (no duplicate data)
* Avoid reading rows (it's map 2nd index to primary key)
* Find Min/Max values (first and last record)
## When to remove indexes?
* When it's unused and redundant.
```
select * from schema_unused_indexes\G
select * from schema_redundant_indexes\G
```
## Improve index statistics
* Update pages variables to improve statistics.
```
SHOW VARIABLE LIKE 'innodb_stats_persistent_sample_pages';
SHOW VARIABLE LIKE 'innodb_stats_transient_sample_pages';
```
## Covering indexes
* Design index for the whole query not just the WHERE. Covering indexes is indexes for all required columns in a query.
```
USE WORLD; 

DESCRIBE city;

EXPLAIN Analyze 
        SELECT Name, District
          FROM city
         WHERE CountryCode = 'USA'\G
         
         
ALTER TABLE city ADD INDEX Country_District_Name
                  (CountryCode, District, Name);
                  
ALTER TABLE city ALTER INDEX CountryCode INVISIBLE;
```
