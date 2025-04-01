# Introduction of Apache AGE

## What's Apache AGE

[Apache AGE](https://age.apache.org) is an extension for PostgreSQL designed to handle graph data, as its name — Apache Graph Extension — suggests.
Graph data is represented as a combination of nodes and edges: nodes represent entities or elements of data, while edges represent the relationships between them.
Both nodes and edges can have properties (attributes) associated with them. AGE allows you to manipulate graph data using a query language called Cypher.
The most well-known graph database that supports Cypher is Neo4j, but there are other query languages as well, such as [Gremlin](https://tinkerpop.apache.org/gremlin.html) implemented in a graph framework, Apache TinkerPop.

## Why graph database?
As I desribed above, AGE can handle graph data. But, why do we need to think about graph data?

Unlike relational or document data, graph data excels at representing complex and highly interconnected relationships. One of the key benefits of using graph data is its ability to model and query relationships efficiently and intuitively.

In relational databases, joining multiple tables to explore connections can become complicated and performance-intensive, especially as the number of relationships grows. Document databases, while flexible, are not optimized for traversing deep relationships between entities.

Graph data, on the other hand, naturally represents entities (nodes) and their relationships (edges), making it ideal for use cases like social networks, recommendation systems, fraud detection, network analysis, and knowledge graphs. Queries that involve multiple levels of relationships—such as “friends of friends” or “shortest path between two points”—are much more efficient and expressive in a graph database.

Let’s compare the case of retrieving a user’s “friends of friends.”

With standard SQL query:

```sql
SELECT DISTINCT fof.name
FROM users u
JOIN friendships f1 ON u.id = f1.user_id
JOIN users f ON f1.friend_id = f.id
JOIN friendships f2 ON f.id = f2.user_id
JOIN users fof ON f2.friend_id = fof.id
WHERE u.name = 'Alice';
```

With Cypher query:
```
MATCH (u:User {name: 'Alice'})-[:FRIEND]->(:User)-[:FRIEND]->(fof)
RETURN DISTINCT fof.name
```

As you can see, the Cypher query is more concise and expressive, making it easier to understand and maintain.

## Graph data with AI

And as is the case in many other areas, the wave of generative AI has also reached the world of graph data.
As the limitations of retrieval and generation accuracy in RAG (Retrieval-Augmented Generation) become more widely recognized, Knowledge Graphs are gaining attention as one of the next promising approaches.

A Knowledge Graph represents facts and concepts as nodes, and the relationships between them as edges.
For example, the fact that “Elizabeth I ruled England and Ireland and fought against Spain” can be represented in a graph as nodes like “Elizabeth I,” “England,” “Ireland,” and “Spain,” connected by edges such as “ruled” and “fought.”

By leveraging this kind of network structure, it is expected that AI-driven search and generation results can be significantly improved.

[Next: How to enable AGE](02_enablement.md)
