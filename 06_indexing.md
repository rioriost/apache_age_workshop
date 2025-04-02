# Indexing

By default, Apache AGE does not create indexes for newly created graphs.

## WHERE Clause

In Apache AGE, the following queries are evaluated differently:
```sql
SELECT * FROM cypher('graph_name',
  $$
    MATCH (n:Customer {Name:'Alice'}) RETURN n
  $$)
AS (n agtype);
```

```sql
SELECT * FROM cypher('graph_name',
  $$
    MATCH (n:Customer) WHERE n.Name='Alice' RETURN n
  $$)
AS (n agtype);
```

To take full advantage of indexing, you **must** understand which types of indexes are utilized by queries with and without a `WHERE` clause.

## EXPLAIN in Apache AGE

Unlike standard SQL, the `EXPLAIN` keyword in Cypher queries requires a different query format.

Query:
```sql
SELECT * FROM cypher('graph_name',
  $$
    EXPLAIN
      MATCH (n:Customer)
      WHERE n.Name='Alice'
      RETURN n
  $$)
AS (plan text);
```

Result:
```sql
QUERY PLAN
--------------------------------------------------------------------------------------------------------------
Seq Scan on "Customer" n  (cost=0.00..418.51 rows=43 width=32)
Filter: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
```

To see the differences of query plans **without** `WHERE` clause:
```sql
SELECT * FROM cypher('graph_name',
  $$
    MATCH (n:Customer {Name:'Alice'}) RETURN n
  $$)
AS (n agtype);
```

```sql
QUERY PLAN
---------------------------------------------------------------
Seq Scan on "Customer" n  (cost=0.00..396.56 rows=9 width=32)
Filter: (properties @> '{"Name": "Alice"}'::agtype)
```

## Common indexing

As explained in [Objects](04_objects.md), an edge table stores the `start_id` corresponding to the **start node’s** `id` and the `end_id` corresponding to the **end node’s** `id`. Creating indexes on `start_id` and `end_id` is highly effective for improving performance.

```sql
CREATE INDEX ON graph_name."VLABEL" USING BTREE (id);
CREATE INDEX ON graph_name."ELABEL" USING BTREE (start_id);
CREATE INDEX ON graph_name."ELABEL" USING BTREE (end_id);
```

Additionally, since typical queries often search for a `value` by `key` within the `properties` column, creating a GIN (Generalized Inverted Index) on that column is also very beneficial:

```sql
CREATE INDEX ON graph_name."VLABEL" USING GIN (properties);
```

With a GIN index on the `properties` column, a query **without** a `WHERE` clause results in the following execution plan:

```sql
SELECT * FROM cypher('graph_name',
  $$
    EXPLAIN MATCH (n:Customer {Name:'Alice'}) RETURN n
  $$)
AS (plan text);
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Bitmap Heap Scan on "Customer" n  (cost=16.09..32.67 rows=9 width=32)
   Recheck Cond: (properties @> '{"Name": "Alice"}'::agtype)
   ->  Bitmap Index Scan on "Customer_properties_idx"  (cost=0.00..16.08 rows=9 width=0)
         Index Cond: (properties @> '{"Name": "Alice"}'::agtype)
```

This indicates that the query planner does utilize the index on the `properties` column.

On the other hand, a query **with** `WHERE` clause yields the following plan:

```sql
SELECT * FROM cypher('graph_name',
  $$
    EXPLAIN MATCH (n:Customer) WHERE n.Name='Alice' RETURN n
  $$)
AS (plan text);
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on "Customer" n  (cost=0.00..418.51 rows=43 width=32)
   Filter: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
```

This shows that the query planner **does not** utilize the index in this case.

## Indexing a Specific Key-Value in `properties`

If you don’t need to index the entire set of key-value pairs in the `properties` column, you can create a smaller and more efficient `BTREE` index for a specific key.

The following filter expression, shown in the previous query plan, gives a clue on how to create such an index:

```sql
Filter: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
```

You can use this expression directly to create an index targeting the `Name` key:

```sql
CREATE INDEX ON graph_name.label_name USING BTREE (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]));
```

After creating the index, a query **without** a `WHERE` clause will produce the following plan:
```sql
SELECT * FROM cypher('graph_name',
  $$
    EXPLAIN MATCH (n:Customer {Name:'Alice'}) RETURN n
  $$)
AS (plan text);
                          QUERY PLAN
---------------------------------------------------------------
 Seq Scan on "Customer" n  (cost=0.00..396.56 rows=9 width=32)
   Filter: (properties @> '{"Name": "Alice"}'::agtype)
```

This shows that the query planner **does not** utilize the index in this case.

However, a query **with** a `WHERE` clause results in a much more efficient plan:

```sql
SELECT * FROM cypher('graph_name',
  $$
    EXPLAIN MATCH (n:Customer) WHERE n.Name='Alice' RETURN n
  $$)
AS (plan text);
                                                   QUERY PLAN
----------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on "Customer" n  (cost=2.62..70.12 rows=43 width=32)
   Recheck Cond: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
   ->  Bitmap Index Scan on "Customer_agtype_access_operator_idx"  (cost=0.00..2.61 rows=43 width=0)
         Index Cond: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
```

This is a much more favorable execution plan, demonstrating that the query planner does utilize the custom index when the `WHERE` clause is used.

[Prev: Cypher Query with AGE](05_cypher.md) | [Next: Rewriting](07_rewriting.md)
