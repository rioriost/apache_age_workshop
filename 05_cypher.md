# Cypher query in Apache AGE

The representation of Cypher queries in Apache AGE differs from **standard** Cypher.

For example, a standard Cypher like:
```cypher
MATCH (n)-[r]->(m) RETURN n, r, m
```

is written in Apache AGE as:
```sql
SELECT * FROM cypher('graph_name',
  $$
  MATCH (n)-[r]->(m) RETURN n, r, m
  $$
)
AS (n agtype, r agtype, m agtype);
```

## How to convert standard Cypher queries into AGE's representation

In Apache AGE every Cypher query is embedded within an SQL statement. The primary reason for using the AS clause is to define the structure of the result set — essentially, the column names and their corresponding PostgreSQL types — returned by the `cypher()` function.

[AGEFreighter](https://github.com/rioriost/agefreighter) offers `convert` sub command for this purpose.

```bash
agefreighter convert -q 'MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m'
Converted Cypher queries for Apache AGE:

line 1, MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m ->
SELECT * FROM cypher('GRAPH_FOR_DRYRUN', $$ MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n, r, m $$) AS (m agtype, n agtype, r agtype);
```

AGEFreighter employs its grammatical and lexical parser to convert Cypher queries. Running the `parse` subcommand demonstrates how it works:

```bash
agefreighter parse 'MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m'
[('MATCH', [('chain', ('node', 'n', ['Person'], None), [(('directed', ('relationship', [{'variable': 'r', 'type': 'WORK_AT'}], None, None)), ('node', 'm', ['Company'], None))])], None), ('RETURN', ['n', 'r', 'm'])]
```

To add an `AS` clause to an AGE query, the presence of a `RETURN` token group is essential.

Unfortunately, AGEFreighter does not support every Cypher query pattern. The following guides will be useful in such cases.

### Purpose of the AS Clause

The AS clause tells PostgreSQL what to expect in the output of the query:
  - **Column Name**: In the examples, `n`, `r`, and `m` are used as the column name.
  - **Data Type**: `agtype` is used to denote that the column contains graph elements (nodes, edges, and so on) in AGE’s JSON-like format.

### Matching the Cypher RETURN:

The count of columns declared in the AS clause must exactly match the items returned in the Cypher query’s RETURN clause. For example, if your Cypher query returned multiple elements:

```cypher
MATCH (a)-[r:KNOWS]->(b) RETURN a, r, b
```

you’d need to declare three columns:
```sql
SELECT * FROM cypher('graph_name', $$MATCH (a)-[r:KNOWS]->(b) RETURN a, r, b$$) AS (a agtype, r agtype, b agtype);
```

### Data Type Flexibility:
While `agtype` is the default for nodes and relationships, if you are returning simple properties (like strings or integers), you can specify native SQL types such as text or integer. For example:

```sql
SELECT * FROM cypher('graph_name', $$MATCH (p:Person) RETURN p.name$$) AS (name text);
```

This way, the property is directly available as a SQL text value.

## Limitation of Apache AGE

Apache AGE has several limitations compared to Neo4j.

### APOC

APOC is commonly used with Neo4j to extend Cypher’s capabilities. However, Apache AGE does not support APOC. If your Cypher queries rely on APOC, you’ll need to find alternatives—in other words, **workarounds**.

1. Converting / Processing
  - apoc.convert.*: Use `CAST()` in PostgreSQL
  - apov.date.*: Use `to_date()` in PostgreSQL

2. Graph Operation
  - apoc.create.*: Use `create_graph()`, `create_vlabel()`, `create_elabel()`
  - apoc.refactor.*: Requires custom programming (e.g., Python, C#)

3. Searching
  - apoc.path.*: Use multi-step Cypher queries or external scripts

4. Integration
  - apoc.load.*: Use tools like [AGEFreighter](https://github.com/rioriost/agefreighter)
  - apoc.export.*: Combine `\COPY` with Cypher queries
  - apoc.http.*: Requires custom programming (e.g., Python, C#)

5. Utility
  - apoc.meta.*: Use `psql` or `pgdump`
  - apoc.periodic.*: Use the `pg_cron` extension
  - apoc.trigger.*: Use PostgreSQL's `TRIGGER` functionality

### Tab characters

As mentioned in [Objects](04_objects.md), `agtype` is a superset of `jsonb`. This also means that the **tab character**, `chr(9)` must be escaped when the `properties` column of nodes or edges contains a tab.

### Multiple labels

As of Apache AGE 1.5, AGE does not support assigning multiple labels to a single node.

In **Neo4j**, creating a node with multiple labels (e.g., `Person` and `Employee`) is valid:
```cypher
CREATE (:Person:Employee {name: 'Alice', age: 30})
```

In **Apache AGE**, attempting the same syntax will result in a syntax error:
```sql
SELECT * FROM cypher('graph_name', $$CREATE (n:Person:Employee) RETURN n$$) AS (n agtype);
```

Error:
```sql
ERROR:  syntax error at or near ":"
LINE 1: ...ELECT * from cypher('graph_name', $$CREATE (n:Person:Employee)...
                                                               ^
```

[Prev: Objects](04_objects.md) | [Next: Indexing](06_indexing.md)
