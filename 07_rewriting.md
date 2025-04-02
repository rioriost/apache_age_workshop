# Rewriting Neo4j's Cypher

As described in [Cypher](05_cypher.md), Apache AGE has certain limitations.
However, the main challenge when migrating from Neo4j lies in the differences from the [openCypher specification](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf).

## Pattern Matching Extensions

- **Label Union/Intersection/Negation in Patterns** – Neo4j 5 allows logical combinations of node labels using `|` (OR), `&` (AND), and `!` (NOT) within a node pattern. For example, `MATCH (c:Character|Harkonnen)` finds nodes with **either** the `Character` or `Harkonnen` label, while `MATCH (c:Character&!Harkonnen)` finds nodes with the `Character` label that do not have the `Harkonnen` label. OpenCypher 9 does not support inline label boolean expressions – in openCypher one must use separate patterns or a `WHERE` clause to filter by labels.

```cypher
-- Neo4j 5.x example:
MATCH (p:Person|Employee)        // `p` has label Person OR Employee
RETURN p

MATCH (c:Character&!Harkonnen)   // `c` has label Character AND NOT Harkonnen
RETURN c.name
```

In openCypher, the above would require `MATCH (p) WHERE p:Person OR p:Employee` and a separate `WHERE NOT c:Harkonnen` filter, as the `|`, `&`, `!` label syntax is Neo4j-specific.

- **Relationship Type Negation** – Similarly, Neo4j extends relationship patterns with the `!` operator to exclude certain relationship types. For example:

```cypher
MATCH (p:Person)-[r:!FRIEND]-() RETURN p, type(r)
```

matches any relationship from a `Person` that is not of type `FRIEND`. This syntax (using `:!Type`) is not in openCypher 9 – in openCypher, one must filter after the match (e.g. `WHERE type(r) <> "FRIEND"`). Neo4j 5 also permits combining negation with `&/|` for multiple types (e.g. `[r:!ALLIES&!FAMILY]` to exclude both types).

- **Fixed-Length Path Shorthand** – Neo4j supports an alternative pattern syntax using curly braces to specify an exact path length. For example, `MATCH (a)--{2}(b)` finds paths of exactly length 2 between nodes `a` and `b`. This is equivalent to the standard `MATCH (a)-[*2]-(b)` in Cypher. The `{n}` or `{m..n}` curly-brace notation for relationship hops is a Neo4j extension (inspired by SPARQL/GQL), not defined in openCypher 9 (which uses the `[*min..max]` syntax). OpenCypher requires `-[*2]-` for a 2-hop pattern.

- **Inline Property Filters in `MATCH`** – Neo4j 5 introduced the ability to put a `WHERE` filter directly inside a pattern, immediately after a node. For example:

```cypher
MATCH (c:Character **WHERE** c.Culture STARTS WITH "Z" AND c.Died IS NOT NULL)
RETURN c.name
```

This matches `Character` nodes satisfying the given property predicates. In openCypher, such property conditions cannot reside in the `MATCH` pattern and must be placed in a separate `WHERE` clause after the pattern. The inline `WHERE` in Neo4j’s pattern syntax is a convenience not found in openCypher 9.

## Subquery and Filtering Extensions

- **Existential Subqueries (`EXISTS {...}`)** – Neo4j extends the `WHERE` clause with existential subqueries that wrap a pattern. Instead of using the (now deprecated) `exists((n)-->(m))` function, Neo4j 5 requires a subquery block with curly braces. For example:

```cypher
MATCH (c:Character)
WHERE **exists { (c)-[:FAMILY]-() }**
RETURN c
```

This filters characters that have at least one `FAMILY` relationship. OpenCypher 9 did not include this subquery syntax – in Neo4j it’s a syntax change where the pattern to check is placed inside `exists { ... }`. You can even include an inner `WHERE` and introduce new variables inside this block. In openCypher, one had to use a different approach (e.g. an `OPTIONAL MATCH + WHERE` or the old `exists()` predicate with pattern, which wasn’t part of the openCypher spec).

- **Pattern Count Subquery (`count{...}`)** – Neo4j 5 adds a concise syntax for counting pattern occurrences without affecting query cardinality. Using `count{ (n)--() }` in a return or predicate will yield the number of matches of that pattern for each row. For example, to get each person’s degree (number of connections):

```cypher
MATCH (p:Person)
RETURN p.name, **count{ (p)--() } AS degree**
```

And to filter nodes with more than 2 relationships: `WHERE count{ (p)--() } > 2`. This is a Neo4j-specific extension replacing the older `size((p)--())` construct (itself not in openCypher). OpenCypher 9 has no equivalent pattern-counting syntax; the typical workaround is to expand and use `COUNT` in a subquery or use size of a list of matches.

- **List Predicate Functions (`ANY`, `ALL`, `NONE`, `SINGLE`)** – Neo4j Cypher provides predicate functions to evaluate conditions over list elements, such as `any(x IN list WHERE ...)`, `all(...)`, `none(...)`, and `single(...)`. For example:

```cypher
MATCH p=(a:Start)-[:HOP*1..]->(z:End)
WHERE **none(node IN nodes(p) WHERE node.class = 'D')**
RETURN p
```

finds paths from `Start` to `End` with no node having class `"D"`. These convenient functions are not part of the openCypher 9 spec. OpenCypher-compliant systems (like Neptune) require rewriting such logic using list comprehensions and size, e.g. `WHERE size([node IN nodes(p) WHERE node.class='D']) = 0` to mimic `NONE`. In Neo4j, `ANY`, `ALL`, etc. are built-in syntactic constructs; openCypher 9 lacks them.

- **`reduce()` List Reduction** – Neo4j’s Cypher supports the `reduce(accumulator = initial, elem IN list | expression)` function to fold a list into a single value (e.g. summing a list). For example:

```cypher
MATCH p=(a:Airport {code:'ANC'})-[r:ROUTE*1..3]->(b:Airport {code:'AUS'})
RETURN reduce(totalDist = 0, r IN relationships(p) | totalDist + r.dist) AS totalDist
```

returns the total distance of each path. This functional syntax is absent from openCypher 9, which means openCypher implementations must use workarounds (like unwinding the list and summing, as shown in the Neptune example). Thus, `reduce()` is a Neo4j-specific extension to the query language’s expressive power.

## Write and Load Clause Extensions

- **`FOREACH` Update Clause** – Neo4j’s Cypher includes a `FOREACH` clause for performing updates on each element of a list within a query (usually in the middle of a write query). For example, to set a visited flag on all nodes along a path:

```cypher
MATCH p=(:Airport {code:'ANC'})-[*1..2]->(:Airport {code:'AUS'})
**FOREACH** (n IN nodes(p) | SET n.visited = true)
```

This clause, which iterates over a list (often derived from a path or aggregation) and executes sub-commands, is not part of the openCypher spec. OpenCypher-compliant languages must instead unwind the list and perform the update in a regular `SET` (as shown in the Neptune rewrite). Neo4j’s `FOREACH` is a purely syntax-level convenience for batch updates within a query.

- **`MERGE` with `ON CREATE/ON MATCH`** – Neo4j extends the `MERGE` clause with optional sub-clauses to conditionally execute updates depending on whether the pattern was created or matched. For example:

```cypher
MERGE (u:User {id: $userId})
  ON CREATE SET u.createdAt = timestamp()
  ON MATCH  SET u.lastSeen = timestamp();
```

This syntax lets you set properties differently on node creation vs when it already exists. OpenCypher 9’s specification did not include these `ON CREATE/ON MATCH` modifiers on `MERGE` (they were introduced in Neo4j 3.5+). In standard openCypher, achieving the same effect requires more verbose patterns (e.g. using separate `MERGE` or `CASE` logic). The conditional merge syntax is a Neo4j-specific enhancement.

- **`LOAD CSV` Clause** – Neo4j supports importing data from CSV files via the `LOAD CSV` clause (with options like `WITH HEADERS` and field terminators). This clause can be used to read external CSV data into a query, often combined with `FOREACH` or `UNWIND` for bulk inserts. Example:

```cypher
LOAD CSV WITH HEADERS FROM "file:///people.csv" AS line
CREATE (:Person { name: line.Name, age: toInteger(line.Age) });
```

`LOAD CSV` is not part of openCypher 9’s standard. It is a Neo4j-specific clause for bulk data loading. Other openCypher implementations (like AWS Neptune) do not support it and must load data by other means.

- **Procedure Calls (`CALL ... YIELD`)** – Neo4j’s Cypher allows calling stored procedures and user-defined functions from within queries. For example, one can `CALL db.index.fulltext.search("Movies", "matrix") YIELD node, score RETURN node.title` to call a fulltext search procedure. The Cypher syntax for invoking procedures is `CALL procedureName(args) YIELD resultVars`. This mechanism (and the `CALL` keyword) is not defined in openCypher 9. In openCypher, there is no standard way to call procedures – Neo4j added this as an extension. (Note: APOC procedures are an example library using this, but APOC itself is outside this scope.) The `CALL ... YIELD` clause in Neo4j is an extension enabling imperative sub-calls in what is otherwise a declarative language.

## Other Query Extensions and Modifiers

- **Nested Subqueries with `CALL { ... }`** – Neo4j (since 4.0) supports embedding an arbitrary subquery in the middle of a larger query using `CALL { ... }` syntax. This subquery runs in isolation and can return results to the enclosing query. For example:

```cypher
MATCH (q:Question)
CALL {
  MATCH (q)<-[:ANSWERED]-(:Answer)
  RETURN count(*) AS answerCount
}
RETURN q.text, answerCount
```

This is used for post-union processing, complex filtering, etc. openCypher 9 has no provision for inner subqueries. Neo4j’s addition of `CALL {}` (and its accompanying scoping rules) is a significant syntax extension beyond openCypher. Any variables from the outer scope must be passed in explicitly – originally via a `WITH`, but Neo4j 5 introduced a more direct way to import variables as described next.

- **Passing Variables into Subqueries (Scoped `CALL`)** – In Neo4j 5, you can directly pass external variables into a subquery by writing them in parentheses after `CALL`. For example:

```cypher
MATCH (p:Person)
OPTIONAL CALL **(p)** {
  MATCH (p)-[:OWNS]->(d:Dog)
  RETURN d.name AS dogName
}
RETURN p.name, dogName
```

Here `p` from the outer query is made available inside the subquery without an extra `WITH`. This scoped `CALL` syntax (`CALL(var1, var2) { ... }`) and even the ability to prefix `CALL` with `OPTIONAL` are Neo4j-specific. openCypher does not define a way to parameterize subqueries, nor an `OPTIONAL CALL`. These enhancements give Cypher compositional capabilities similar to optional joins. (Neptune’s openCypher, for instance, does not support the `CALL(n) {}` syntax at all.)

- **Executing Subqueries in Batches** – Neo4j allows a subquery to be executed in a batching mode with periodic commits using the syntax `CALL { ... } **IN TRANSACTIONS [OF N ROWS]**`. This lets large writes be split into multiple transactions. For example:

```cypher
MATCH (n:OldLabel)
CALL {
  WITH n
  DELETE n
} **IN TRANSACTIONS OF 1000 ROWS**
```

will commit every 1000 deletions. The `IN TRANSACTIONS` clause for subqueries is not part of openCypher and is unique to Neo4j (added in 4.x). It replaces the older `USING PERIODIC COMMIT` (which was specific to `LOAD CSV`). OpenCypher 9 does not have any analogous construct for batching transactions within a single query.

- **Planner Hints (`USING INDEX/SCAN`)** – Neo4j’s Cypher supports query hints that are specified in the query using the `USING` keyword (e.g., `USING INDEX n:Person(name)` or `USING JOIN` or `USING SCAN`). These are directives to the query planner about how to use indexes or other planning decisions. For example:

```cypher
MATCH (n:Person) **USING INDEX n:Person(name)**
WHERE n.name = "Alice"
RETURN n
```

Such hints are not part of the openCypher 9 specification (which leaves query planning to the implementation). They are Neo4j-specific syntax additions. Other databases implementing openCypher may ignore or not support these hints; for instance, Neptune only added partial support for `USING INDEX` in a specific engine version.

- **`EXPLAIN/PROFILE`** – In Neo4j, queries can be prefixed with the keywords `EXPLAIN` or `PROFILE` to inspect the execution plan. While not a clause within the Cypher `MATCH…​WHERE…​RETURN` sequence, these keywords are part of the query language interface in Neo4j. They are not described in openCypher 9 (which focuses on the declarative querying itself). For example, `EXPLAIN MATCH (n:Person) RETURN count(n)` will produce the query plan but not actually execute the query. This is a Neo4j-specific capability for debugging and optimization; openCypher spec does not include it as it’s outside pure query semantics.

Each of the above syntax extensions makes Neo4j’s Cypher more powerful or convenient, but also means that queries using them may need rewriting to run on a pure openCypher-9 platform. These extensions (from new pattern syntax to custom clauses) highlight how Neo4j’s Cypher has evolved beyond the openCypher v9 standard. All examples shown are valid in Neo4j 5.x and illustrate features absent in the openCypher reference implementation.

[Prev: Indexing](06_indexing.md)
