# experiment-presto-trino

A few experiments to get more familiar with Trino ([renamed from
PrestoSQL](https://trino.io/blog/2020/12/27/announcing-trino.html), which is a
fork of [PrestoDB](https://prestodb.io/)).

## Resources
- [Trino Docs](https://trino.io/docs/current/index.html)
- [Trino Concepts](https://trino.io/docs/current/overview/concepts.html)
- [Presto - The Definitive
  Guide](https://trino.io/presto-the-definitive-guide.html) (book)

## Hands on example
Start Trino with Docker
```sh
docker run --rm --name trino-experiment trinodb/trino
```

Connect to the container and run the Trino CLI
```sh
docker exec -it trino-experiment trino
```

There are some sample datasets already wired up in this docker image.

Execute a query on a table of the `tpch` benchmark data:
```sql
SELECT * FROM tpch.sf1.nation LIMIT 5;

--  nationkey |   name    | regionkey |                                                   comment
-- -----------+-----------+-----------+-------------------------------------------------------------------------------------------------------------
--          0 | ALGERIA   |         0 |  haggle. carefully final deposits detect slyly agai
--          1 | ARGENTINA |         1 | al foxes promise slyly according to the regular accounts. bold requests alon
--          2 | BRAZIL    |         1 | y alongside of the pending deposits. carefully special packages are about the ironic forges. slyly special
--          3 | CANADA    |         1 | eas hang ironic, silent packages. slyly regular packages are furiously over the tithes. fluffily bold
--          4 | EGYPT     |         4 | y above the carefully unusual theodolites. final dugouts are quickly across the furiously regular d
-- (5 rows)

-- Query 20210208_150730_00002_eah5z, FINISHED, 1 node
-- Splits: 21 total, 21 done (100.00%)
-- 0.28 [25 rows, 0B] [89 rows/s, 0B/s]
```

See what operations are available. Also, see
[here](https://trino.io/docs/current/sql.html).
```
trino> help

Supported commands:
QUIT
EXPLAIN [ ( option [, ...] ) ] <query>
    options: FORMAT { TEXT | GRAPHVIZ | JSON }
             TYPE { LOGICAL | DISTRIBUTED | VALIDATE | IO }
DESCRIBE <table>
SHOW COLUMNS FROM <table>
SHOW FUNCTIONS
SHOW CATALOGS [LIKE <pattern>]
SHOW SCHEMAS [FROM <catalog>] [LIKE <pattern>]
SHOW TABLES [FROM <schema>] [LIKE <pattern>]
USE [<catalog>.]<schema>
```

List [catalogs](https://trino.io/docs/current/overview/concepts.html#catalog)
```sql
SHOW CATALOGS;

--  Catalog
-- ---------
--  jmx
--  memory
--  system
--  tpcds
--  tpch
-- (5 rows)
```

List [schemas](https://trino.io/docs/current/overview/concepts.html#schema) in a
catalog
```sql
SHOW SCHEMAS FROM tpch;

--        Schema
-- --------------------
--  information_schema
--  sf1
--  sf100
--  sf1000
--  sf10000
--  sf100000
--  sf300
--  sf3000
--  sf30000
--  tiny
-- (10 rows)
```

List [tables](https://trino.io/docs/current/overview/concepts.html#table) in a
schema
```sql
SHOW TABLES FROM tpch.sf1;

--   Table
-- ----------
--  customer
--  lineitem
--  nation
--  orders
--  part
--  partsupp
--  region
--  supplier
-- (8 rows)
```

One useful capability of Trino is the ability to perform joins across catalogs -
(i.e. different data stores) - without requiring ETL's to physically move the
data prior.
```sql
SELECT n.nationkey, n.name, c.c_customer_sk, c.c_customer_id
FROM tpch.sf1.nation n
  JOIN tpcds.sf1.customer c
  ON n.nationkey = c.c_customer_sk
LIMIT 5;

--  nationkey |   name    | c_customer_sk |  c_customer_id
-- -----------+-----------+---------------+------------------
--          1 | ARGENTINA |             1 | AAAAAAAABAAAAAAA
--          2 | BRAZIL    |             2 | AAAAAAAACAAAAAAA
--          3 | CANADA    |             3 | AAAAAAAADAAAAAAA
--          4 | EGYPT     |             4 | AAAAAAAAEAAAAAAA
--          5 | ETHIOPIA  |             5 | AAAAAAAAFAAAAAAA
-- (5 rows)

-- Query 20210208_151420_00007_eah5z, FINISHED, 1 node
-- Splits: 57 total, 57 done (100.00%)
-- 0.78 [26.9K rows, 746B] [34.8K rows/s, 963B/s

```

The results of a query can be exported as a CSV via a non-interactive Trino CLI
call
```sql
docker exec -it trino-experiment trino --execute "SELECT * FROM tpch.sf1.nation n join tpcds.sf1.customer c on n.nationkey = c.c_customer_sk" > results.csv
```

Get metadata
```sql
SHOW TABLES FROM tpch.information_schema;

--              Table
-- --------------------------------
--  applicable_roles
--  columns
--  enabled_roles
--  role_authorization_descriptors
--  roles
--  schemata
--  table_privileges
--  tables
--  views
-- (9 rows)
```

For example, getting all schemas in the `tpch` catalog with a `customer` table
```sql
SELECT * FROM tpch.information_schema.tables WHERE table_name = 'customer';

--  table_catalog | table_schema | table_name | table_type
-- ---------------+--------------+------------+------------
--  tpch          | sf300        | customer   | BASE TABLE
--  tpch          | sf1          | customer   | BASE TABLE
--  tpch          | sf3000       | customer   | BASE TABLE
--  tpch          | sf10000      | customer   | BASE TABLE
--  tpch          | sf100000     | customer   | BASE TABLE
--  tpch          | sf1000       | customer   | BASE TABLE
--  tpch          | tiny         | customer   | BASE TABLE
--  tpch          | sf100        | customer   | BASE TABLE
--  tpch          | sf30000      | customer   | BASE TABLE
-- (9 rows)
```

What if data is partitioned across schemas? How do we perform analysis across
more than one schema?

Trying to perform analysis across schemas by union-ing schemas together via a
CTE like this seems to overwhelm Trino. Not to mention manual and not pretty.

TODO: take a closer look at the `EXPLAIN` output for the below.
```sql
WITH customers AS (
  SELECT *, 'sf1' AS account FROM tpcds.sf1.customer
  UNION
  SELECT *, 'sf100' AS account FROM tpcds.sf100.customer
)
SELECT c_salutation, count(1) FROM customers GROUP BY c_salutation;

-- Query 20210205_194939_00067_chsyd, FAILED, 1 node
-- Splits: 72 total, 20 done (27.78%)
-- 5.60 [357K rows, 0B] [63.7K rows/s, 0B/s]

-- Query 20210205_194939_00067_chsyd failed: Query exceeded per-node user memory limit of 102.40MB [Allocated: 101.39MB, Delta: 1.25MB, Top Consumers: {HashAggregationOperator=161.67MB, PartitionedOutputOperator=1.78MB, FilterAndProjectOperator=70.13kB}]
```

We can resort to manually aggregating across schemas - which works, but is kind
of cumbersome to write.

TODO: compare the `EXPLAIN` output of this to the one above.
```sql
SELECT
  *,
  a_count + b_count AS total_count
FROM
  (
    SELECT c_salutation, count(1) a_count
    FROM tpcds.sf1.customer
    GROUP BY c_salutation
  ) a,
  (
    SELECT c_salutation, count(1) b_count
    FROM tpcds.sf100.customer
    GROUP BY c_salutation
  ) b
WHERE a.c_salutation = b.c_salutation;

--  c_salutation | a_count | c_salutation | b_count | total_count
-- --------------+---------+--------------+---------+-------------
--  Mr.          |   16731 | Mr.          |  331964 |      348695
--  Miss         |   11532 | Miss         |  233683 |      245215
--  Mrs.         |   11788 | Mrs.         |  233719 |      245507
--  Ms.          |   11730 | Ms.          |  234135 |      245865
--  Dr.          |   28346 | Dr.          |  565169 |      593515
--  Sir          |   16463 | Sir          |  331490 |      347953
-- (6 rows)
```

## TODO
- connect to Postgres (p.87 Presto Definitive Guide,
  https://trino.io/docs/current/connector/postgresql.html)
- explore with separate Postgres clusters. Representing different datastores
  for distinct services. Also representing different clusters for the same
  service. What's it like to work across them. Join and query across them.
- explore business domain boundaries within a single schema (related to
  ownership). Can a single schema be subdivided to align with business domains?
  Can a catalog only include and show one of these subsets?
- explore Postgres schemas. Querying across more than one, all of them. Joining
  across datastores, which both have schemas.
- explore effect of underlying datastore table schema changes
- explore Postgres views. This might be a low cost (vs ETL, CDC, warehouse,
  lake, etc.) option for teams to create their data interface for others (vs
  directly exposing transactional tables).
- explore Postgres access controls. Can tables be hidden?
- explore Trino views
- explore performance of cross catalog joins on large datasets
- clarify limitations and next id potential steps. Connecting to OLTP read
  replicas is only a potential first step forward. What next steps might look
  like?
- think more about ownership. [data mesh
  principles](https://martinfowler.com/articles/data-mesh-principles.html)
- explore access controls
- explore integration with data catalogs
- look into cloud hosting options. Trino (PrestoSQL) vs PrestoDB support.
