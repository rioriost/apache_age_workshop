# Objects

AGE offers following objects. Here, I only explain the important items.

## Tables

```sql
table ag_catalog.ag_graph
table ag_catalog.ag_label
```

### `ag_graph`

The table holds the information of graph.

```sql
\d+ ag_graph
                                          Table "ag_catalog.ag_graph"
  Column   |     Type     | Collation | Nullable | Default | Storage | Compression | Stats target | Description
-----------+--------------+-----------+----------+---------+---------+-------------+--------------+-------------
 graphid   | oid          |           | not null |         | plain   |             |              |
 name      | name         |           | not null |         | plain   |             |              |
 namespace | regnamespace |           | not null |         | plain   |             |              |
Indexes:
    "ag_graph_graphid_index" UNIQUE, btree (graphid)
    "ag_graph_name_index" UNIQUE, btree (name)
    "ag_graph_namespace_index" UNIQUE, btree (namespace)
Referenced by:
    TABLE "ag_label" CONSTRAINT "fk_graph_oid" FOREIGN KEY (graph) REFERENCES ag_graph(graphid)
Access method: heap
```

For instance, the following result of the query shows that the database has 6 graphs.

```sql
SELECT * FROM ag_graph;
 graphid |    name     |   namespace
---------+-------------+---------------
  411117 | FROM_CSV_SS | "FROM_CSV_SS"
  411163 | FROM_CSV_SM | "FROM_CSV_SM"
  411218 | FROM_CSV_MS | "FROM_CSV_MS"
  411277 | FROM_CSV_MM | "FROM_CSV_MM"
  411360 | FROM_COSMOS | "FROM_COSMOS"
  442532 | Airroute    | "Airroute"
(6 rows)
```

### `ag_label`
The table holds the meta data of each graph.

```sql
\d+ ag_label
                                         Table "ag_catalog.ag_label"
  Column  |    Type    | Collation | Nullable | Default | Storage | Compression | Stats target | Description
----------+------------+-----------+----------+---------+---------+-------------+--------------+-------------
 name     | name       |           | not null |         | plain   |             |              |
 graph    | oid        |           | not null |         | plain   |             |              |
 id       | label_id   |           |          |         | plain   |             |              |
 kind     | label_kind |           |          |         | plain   |             |              |
 relation | regclass   |           | not null |         | plain   |             |              |
 seq_name | name       |           | not null |         | plain   |             |              |
Indexes:
    "ag_label_graph_oid_index" UNIQUE, btree (graph, id)
    "ag_label_name_graph_index" UNIQUE, btree (name, graph)
    "ag_label_relation_index" UNIQUE, btree (relation)
    "ag_label_seq_name_graph_index" UNIQUE, btree (seq_name, graph)
Foreign-key constraints:
    "fk_graph_oid" FOREIGN KEY (graph) REFERENCES ag_graph(graphid)
Access method: heap
```

The following query result shows that the graph `"Airroute"` has a vertex (node) labeled `"Airport"` and an edge labeled `"ROUTE"`. The relation, `_ag_label_vertex` is the parent of the relation, `"Airroute"."Airport"` and `_ag_label_edge` is the parent of `"Airroute"."ROUTE"`. In other words, `"Airroute"."Airport"` inherits from `_ag_label_vertex`.

```sql
SELECT * FROM ag_label WHERE relation::text LIKE '"Airroute"%';
       name       | graph  | id | kind |          relation           |        seq_name
------------------+--------+----+------+-----------------------------+-------------------------
 Airport          | 442532 |  3 | v    | "Airroute"."Airport"        | Airport_id_seq
 ROUTE            | 442532 |  4 | e    | "Airroute"."ROUTE"          | ROUTE_id_seq
 _ag_label_vertex | 442532 |  1 | v    | "Airroute"._ag_label_vertex | _ag_label_vertex_id_seq
 _ag_label_edge   | 442532 |  2 | e    | "Airroute"._ag_label_edge   | _ag_label_edge_id_seq
(4 rows)
```

Actual data is stored in each relations.

For node:
```sql
ELECT * FROM "Airroute"."Airport" LIMIT 2;
       id        |                                                                                      properties
-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 844424930131969 | {"city": "Atlanta", "code": "ATL", "name": "Hartsfield-Jackson Atlanta International Airport", "country": "USA", "latitude": 33.6367, "longitude": -84.4281, "passengers": 107394029}
 844424930131970 | {"city": "Los Angeles", "code": "LAX", "name": "Los Angeles International Airport", "country": "USA", "latitude": 33.9416, "longitude": -118.4085, "passengers": 88068013}
(2 rows)
```

For edge:
```sql
SELECT * FROM "Airroute"."ROUTE" LIMIT 1;
        id        |    start_id     |     end_id      |                         properties
------------------+-----------------+-----------------+-------------------------------------------------------------
 1125899906842625 | 844424930131969 | 844424930131970 | {"distance": 3100, "flight_time": 300, "daily_flights": 35}
(1 row)
```

### Why are `id` so large?

`graphid` type is [defined as int64](https://github.com/apache/age/blob/c75d9e477e2f83016a7a144291d2c15038cf229f/src/include/utils/graphid.h#L29).

`graphid` is made by `make_graphid()` function in [graphid.c](https://github.com/apache/age/blob/c75d9e477e2f83016a7a144291d2c15038cf229f/src/backend/utils/adt/graphid.c#L200)

```c
graphid make_graphid(const int32 label_id, const int64 entry_id)
{
    uint64 tmp;

    if (!label_id_is_valid(label_id))
    {
        ereport(ERROR, (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                        errmsg("label_id must be %d .. %d",
                               LABEL_ID_MIN, LABEL_ID_MAX)));
    }
    if (!entry_id_is_valid(entry_id))
    {
        ereport(ERROR,
                (errcode(ERRCODE_INVALID_PARAMETER_VALUE),
                 errmsg("entry_id must be " INT64_FORMAT " .. " INT64_FORMAT,
                        ENTRY_ID_MIN, ENTRY_ID_MAX)));
    }

    tmp = (((uint64)label_id) << ENTRY_ID_BITS) |
          (((uint64)entry_id) & ENTRY_ID_MASK);

    return (graphid)tmp;
}
```

It consists of the upper 16 bits to store `label_id` and the lower 48 bits to store `entry_id`.

For example, if the `id` of a node table - such as "Airroute"."Airport" - is 3 (an OID within the namespace), then the first ID of a node labeled "Airport" is always 844424930131969 (calculated as 3 << 48 + 1).

## Types

```sql
type ag_catalog.agtype
type ag_catalog.graphid
type ag_catalog.label_id
type ag_catalog.label_kind
```

### Overview of agtype

Apache AGE introduces **agtype** as a custom data type for representing graph query results. It is essentially a superset of JSON, implemented with a structure very similar to PostgreSQL’s `jsonb` format. In practice, `agtype` extends `jsonb` to support not only basic JSON values (null, boolean, numeric, string, arrays, objects) but also **graph-specific types** such as vertices (nodes), edges, and paths. All query results in AGE are returned as `agtype`, making it the central data type for the extension.

Internally, `agtype` is stored as a **varlena (variable-length)** datum, meaning the first 4 bytes indicate the total size. The content that follows uses a binary JSON-like tree structure. Just like `jsonb`, `agtype` organizes its content in a single contiguous structure optimized for quick access and compression. Each value is stored as a node consisting of a small fixed-size header (called an **agtentry**) and its actual data payload. For complex values like objects or arrays, a container header at the top describes the number of elements and the type (object vs array). This design allows efficient traversal and lookup, similar to `jsonb`. The key difference is that `agtype` extends this structure to handle additional Cypher/graph value types while reusing the proven `jsonb` layout.

### Core Structures in `agtype.h`

Several C structures in [agtype.h](https://github.com/apache/age/blob/master/src/include/utils/agtype.h) define the on-disk and in-memory representation of `agtype`. Below is a breakdown of the main structures and their roles:

#### agtype_container – On-disk Container for Arrays/Objects

An `agtype_container` represents a JSON array or object in binary form (it corresponds closely to PostgreSQL’s `JsonbContainer`). It contains a header and an array of entries for child elements:
  - **Header (uint32 header)** – Stores the count of elements (or key-value pairs) in the container and flag bits indicating container type. The lower 28 bits hold the count, and the upper 4 bits are flags. For example, a flag marks whether this container is an object or an array. Specifically, `AGT_FOBJECT` and `AGT_FARRAY` are set in the header for objects and arrays respectively. There is also an `AGT_FSCALAR` flag to denote a scalar container – a special case used when a lone scalar is stored as a container (this mirrors how `jsonb` represents top-level scalars).
	-	**Children (agtentry children[])** – A flexible array of **entry headers** (of type `agtentry`) for each child element in the container. Each `agtentry` is a 32-bit metadata value describing one child element’s type and length/offset within the container. The actual data for all children (e.g. string content, numeric bytes, sub-containers, etc.) is stored sequentially after this array of `agtentry` headers. This separation of metadata and data is done for space efficiency and to improve cache locality when scanning keys or elements.

Following the `children` array in memory is the **data region** containing the variable-length content of each child node. If the container is an object, the children entries are ordered such that all keys come first (sorted in lexical order), followed by all values. This is inherited from `jsonb` and makes key lookup faster (since keys are contiguous and sorted) at the cost of losing original insertion order in storage. However, `agtype` tracks original insertion order for object fields in a separate structure when building (as we will see with `agtype_pair`). If the container is an array, the entries correspond to elements in order. The container’s header flags distinguish an object from an array.

**Flags and macros**: The `agtype_container` flags can be accessed with macros. For example, `AGTYPE_CONTAINER_IS_OBJECT(container)` checks an object flag. Similarly, `AGTYPE_CONTAINER_SIZE(container)` gives the number of children (keys for objects count as one each in this total). The design ensures the root of an `agtype` value is always a container or is treated as a scalar container – no separate entry header is stored for the root value, since the container header itself tells us if it’s an object or array.

#### agtentry – Entry Metadata for Values

Each element or field within a container is described by an `agtentry` (analogous to `jsonb`’s `JEntry`). An `agtentry` is a 4-byte value encoding both the type of the value and either its length or offset. The layout and constants for `agtentry` are as follows:
  - **Type bits (upper 3 bits)** – Indicate what kind of data the entry represents. `agtype` defines several types: string, numeric, boolean (true/false have distinct codes), null, container (for nested array/object), or a special **“extended agtype”** type for graph-specific or extended types. For example, `AGTENTRY_IS_STRING (0x00000000)`, `AGTENTRY_IS_NUMERIC (0x10000000)`, `AGTENTRY_IS_BOOL_FALSE (0x20000000)`, `AGTENTRY_IS_BOOL_TRUE (0x30000000)`, `AGTENTRY_IS_NULL (0x40000000)`, and `AGTENTRY_IS_CONTAINER (0x50000000)` cover the basic JSON types. In addition, `AGTENTRY_IS_AGTYPE (0x70000000)` is a type designator for extended types introduced by AGE.
  -	**HAS_OFF flag (1 bit)** – The next bit (the 4th highest bit) is a flag indicating how the remaining 28 bits should be interpreted. If `AGTENTRY_HAS_OFF (0x80000000)` is set, then the lower 28 bits store the **offset** from the start of the data region to the beginning of this value’s content. If the flag is not set, the lower 28 bits store the **length** of the value’s content. This scheme is used to balance direct addressing versus compression: typically most entries store lengths for better compression, but periodically an entry will store an absolute offset to allow computing positions (this follows PostgreSQL’s JSONB strategy for TOAST compression efficiency).
	-	**Length/Offset (lower 28 bits)** – Depending on the flag above, this field is either the byte length of the value’s data, or the end position of the value’s data relative to the container’s data start (offset+1). In either case, it allows retrieving the value bytes. Large values or values occurring at interval positions will carry an offset, others a length, to optimize access patterns.

For extended types (`AGTENTRY_IS_AGTYPE`), the entry’s data is prefixed with an **AG type header** to identify the specific subtype of value. AGE defines several `AGT_HEADER_*` constants for this purpose. The first 4 bytes of the data for an extended entry are one of these headers, which then indicates how to interpret the subsequent bytes. The defined headers include: `AGT_HEADER_INTEGER (0x00000000)`, `AGT_HEADER_FLOAT (0x00000001)`, `AGT_HEADER_VERTEX (0x00000002)`, `AGT_HEADER_EDGE (0x00000003)`, and `AGT_HEADER_PATH (0x00000004)`. In other words, when an `agtentry`’s type is “AGTYPE”, the content starts with a 4-byte tag saying “this is a 64-bit integer” (or float, vertex, etc.), followed by the actual value data (e.g. 8 bytes of integer data, or a nested container for a vertex). This mechanism allows `agtype` to encode graph entities and new scalar types in the same container framework as JSON.

#### agtype – Top-Level Datum Structure

The `agtype` struct itself represents the top-level varlena object stored in a PostgreSQL tuple. In memory it is defined as:

```c
typedef struct {
    int32 vl_len_;         /* varlena header (total size) */
    agtype_container root;
} agtype;
```

This simply embeds an `agtype_container` (the root container) after the 4-byte length header. The root may be an object or array container, possibly marked as a scalar container if it actually holds a single scalar. Utility macros like `AGT_ROOT_IS_OBJECT(ag)` and `AGT_ROOT_COUNT(ag)` are provided to inspect the root’s header flags and count. In essence, the bytes of an `agtype` datum on disk or in memory after the length header correspond directly to an `agtype_container` structure (or an extended entry, if a scalar is stored). This mirrors the layout of PostgreSQL’s `jsonb` (which has a `Jsonb` struct containing a `JsonbContainer` root).

#### agtype_value – In-memory Deserialized Value

While `agtype_container` and `agtentry` manage the compact binary format, `agtype_value` is used for convenient in-memory manipulation of `agtype` content (similar to `JsonbValue` in PostgreSQL). An `agtype_value` is a C struct that can represent any `agtype` value in a decomposed form, making it easier to build or examine values without manual byte offsets. It is defined as follows:

```c
struct agtype_value {
    enum agtype_value_type type;
    union {
        int64        int_value;      /* 64-bit Integer value */
        float8       float_value;    /* 64-bit Float value */
        Numeric      numeric;        /* Arbitrary precision numeric (Postgres Numeric) */
        bool         boolean;        /* Boolean value */
        struct { int len; char *val; } string;  /* String (not null-terminated) */
        struct { int num_elems; struct agtype_value *elems; bool raw_scalar; } array;
        struct { int num_pairs; struct agtype_pair *pairs; } object;
        struct { int len; struct agtype_container *data; } binary;
    } val;
};
```

In this structure, the type field is an **enum agtype_value_type** that tags which member of the union is valid. The possible type values include all JSON types and the extended AGE types:
  -	**Scalars**: `AGTV_NULL`, `AGTV_BOOL`, `AGTV_INTEGER`, `AGTV_FLOAT`, `AGTV_NUMERIC`, `AGTV_STRING` cover the primitive types. (Here `AGTV_BOOL` is used for both true/false, with the boolean field storing the actual value.) Notably, `AGTV_INTEGER` and `AGTV_FLOAT` are new scalar types for AGE (whereas vanilla JSONB would use `AGTV_NUMERIC` for any number). These correspond to Cypher’s 64-bit integer and double types. The enum values are ordered such that all scalar types (including these new ones) are in a contiguous range, which is important for sorting and type-check macros.
  - **Graph entity types**: `AGTV_VERTEX`, `AGTV_EDGE`, `AGTV_PATH` are additional types representing node, edge, and path values. Internally, values of these types are typically stored as composite structures (for example, a vertex contains an id, a label, and a properties map). In the in-memory `agtype_value`, a vertex or edge would likely be represented using the object union member (since a node or edge can be viewed as an object of properties) or the binary member pointing to an on-disk format. The type tag `AGTV_VERTEX` (or EDGE/PATH) distinguishes it from a generic object. A path might be represented via the array member (as it can be seen as an ordered list of vertices and edges) or a dedicated path structure. Regardless, the presence of these enum values means the system can handle these as first-class types in comparisons and processing.
  - **Composite types**: `AGTV_ARRAY` and `AGTV_OBJECT` indicate that the `agtype_value` holds an array or object by value (deserialized). In these cases the array or object union member is used, containing pointers to sub-values. For example, if `type == AGTV_ARRAY`, then `val.array.num_elems` and `val.array.elems` describe the elements. The `raw_scalar` flag in the array struct is true if this array represents a “raw scalar” (a special case for representing a lone scalar as an array). If `type == AGTV_OBJECT`, then `val.object.num_pairs` and `val.object.pairs` are set, where `agtype_pair` (described below) holds each key and value.
  - **Binary container**: `AGTV_BINARY` is a special type used when the value is already in its binary form (an `agtype_container`) and we want to treat it as a value without fully deserializing. In this case, the `val.binary` member points to an `agtype_container` and its length. This is useful for deferring parsing – for example, if a function retrieves a sub-object and only needs to pass it around or output it, it can use the binary representation directly. The `AGTV_BINARY` type essentially wraps an on-disk format array or object within an `agtype_value`. (This is analogous to `jbvBinary` in `JsonbValue` for JSONB.)

The `agtype_value` struct makes it easy to build complex values programmatically. For instance, when constructing a new vertex, one might allocate an `agtype_value` of type `AGTV_OBJECT` for its properties map, or use `AGTV_BINARY` if the properties are already in a container form, then wrap it in an `AGTV_VERTEX` higher-level `agtype_value`. Eventually, functions like `agtype_value_to_agtype()` can convert the in-memory representation back into the compact `agtype` container form for storage or output.

#### agtype_pair – Key-Value Pair for Building Objects

When constructing an object (map) value in memory, AGE uses the `agtype_pair` structure to hold each key and value pair before serialization. It is defined as:

```c
struct agtype_pair {
    agtype_value key;    /* must be AGTV_STRING type */
    agtype_value value;  /* can be any type */
    uint32      order;   /* original insertion order index */
};
```

This is not an on-disk structure; it’s used transiently while forming an `AGTV_OBJECT` value. The key is an `agtype_value` expected to be a string, and the value is an `agtype_value` of any type. The order field records the pair’s index in the original input sequence. This is important because if the same key appears multiple times, the system can choose which occurrence to keep. In AGE (and JSONB), the rule is “last observed value wins” for duplicate keys. During serialization, keys are sorted for storage, but this order helps to decide duplicates by remembering how the input was ordered. After processing, duplicate keys are removed as needed, and only the last occurrence is stored.

#### agtype_parse_state – Parsing/Building State (Internal)

The `agtype_parse_state` is an internal stack frame used when parsing JSON or constructing `agtype` values incrementally (similar to `JsonbParseState`). It holds a partial `cont_val` (container value under construction), the current count of children, and a pointer to the parent state. It also tracks a pointer to the last value that was updated, which is used in certain casting or mutation scenarios while building nested structures. While important for the implementation, this structure is not directly exposed to users; it just enables the step-by-step assembly of `agtype` values via functions like `push_agtype_value()`.

#### agtype_iterator – Reading an agtype Sequentially

To traverse an `agtype` value (for example, to output it as text or to evaluate a containment query), AGE provides an iterator interface. The `agtype_iterator` structure allows a client to walk through the complex `agtype` structure without exposing its raw layout. Internally, `agtype_iterator` contains:
  - A pointer to the current `agtype_container` being iterated (container),
  - The number of elements in it (`num_elems`), and a current index,
  - A pointer to the container’s `agtentry` array (`children`) and a pointer into the data region (`data_proper`),
  - Flags like `is_scalar` to handle the scalar-container case,
  - Tracking of the current element’s data offsets (`curr_data_offset` and `curr_value_offset` for object values),
  - A state enum `agt_iterator_state` indicating whether we are at an array element, object key, object value, etc.,
  - A link to a parent iterator node for nested structures (so the iterator can unwind after finishing a sub-container).

The iterator is advanced by calling `agtype_iterator_next()`, which returns an `agtype_iterator_token` indicating what type of item was encountered (start of an object, a key, a value, end of array, etc.). These tokens (e.g., `WAGT_BEGIN_OBJECT`, `WAGT_KEY`, `WAGT_VALUE`, `WAGT_END_OBJECT`, etc.) allow the caller to reconstruct the hierarchical structure. This design is directly modeled on the JSONB iterator and makes it easy to output `agtype` as JSON text or to navigate it for query operations. The iterator understands the extended types as well – for example, if it encounters an `AGTENTRY_IS_AGTYPE` entry with a vertex, it would likely return it as a value token and one could inspect the `agtype_value` yielded to see the type `AGTV_VERTEX`.

### Comparison with PostgreSQL jsonb

Because `agtype` is built as an extension of `jsonb`, it shares many internal characteristics with PostgreSQL’s JSONB type, but there are important differences introduced to support graph data. Below is a comparison of their structure and functionality:
  - **Overall Structure (Varlena and Container)**: In both `jsonb` and `agtype`, data is stored in a single varlena blob with a container header and a series of entries followed by data. The mechanism of a header with count and type flags (object/array/scalar) and an array of fixed-size metadata entries is common to both. In fact, `agtype_container` and `JsonbContainer` have the same layout and purpose. This means `agtype` benefits from the same efficient parsing and indexed access techniques that `jsonb` uses (binary search on keys, fast skip of array elements, etc.). The storage is also toast-compressible thanks to the similar offset/length strategy for entries. Where they differ is in the types of values that can be encoded and some details of those encodings.
  - **Type Coverage – JSON Primitives vs Graph Entities**: PostgreSQL’s `jsonb` supports the JSON primitive types: null, Boolean, text strings, and numeric values (which in JSONB are all stored using the PostgreSQL Numeric type for uniformity), as well as composite types: arrays and objects. Apache AGE’s `agtype` includes all of those, but extends the type system to cover graph-specific data:
    - **Vertices (Nodes)**: Represented in `agtype` as `AGTV_VERTEX`. A vertex value includes a graph identifier (often a 64-bit id), a label, and a set of properties (key/value map). In JSONB, one could only represent a vertex as a generic JSON object (e.g., `{"id": ..., "label": ..., "properties": {...}}`), with no special significance. In `agtype`, vertices are first-class: they carry a distinct type tag internally. This allows AGE to recognize a value as a vertex and, for example, print it in a specific format or handle equality in terms of graph identity. Internally, the vertex likely uses an object container to store its fields, but it is prefixed by a `AGT_HEADER_VERTEX` tag in the binary format so that the system knows that object represents a graph vertex.
    - **Edges**: Represented as `AGTV_EDGE` in `agtype`. An edge contains an id, a label (type of relationship), a start vertex id and end vertex id, and possibly properties. Like vertices, an edge in `agtype` is distinguished by type and stored with a `AGT_HEADER_EDGE` marker. In raw JSONB, an edge would again just be a plain object with those fields, with no built-in awareness.
    - **Paths**: Represented as `AGTV_PATH`. A path is essentially an ordered sequence of connected vertices and edges (for example, the result of a variable-length path match in a Cypher query). AGE can encode a path as a single `agtype` value with its own type tag. Likely, the path is stored as a list/array of vertex and edge components, or as a special structure internally, but in any case `jsonb` has no direct equivalent (it would probably be an array of objects in JSONB).
    - **Numeric Types**: `jsonb` does not distinguish between integers, floats, or decimals – any JSON numeric value is stored as a Numeric. `agtype` diverges here to align with Cypher’s type system. It provides:
      - `AGTV_INTEGE`R for 64-bit integer values ￼,
      - `AGTV_FLOAT` for double precision floating-point values ￼,
      - `AGTV_NUMERIC` for arbitrary precision numbers (using PostgreSQL’s Numeric type).
This means `agtype` can store small integers and floats more directly and efficiently. For example, the integer value 42 is stored with an `AGT_HEADER_INTEGER` tag and an 8-byte two’s complement representation, instead of being converted to a Numeric varlena. This yields performance benefits and preserves type fidelity (no unintentional conversion between int and float). JSONB, by contrast, would convert 42 or 4.2 into its Numeric form (which could be larger and slower to process). The trade-off is that `agtype` must maintain logic to handle three number types, whereas JSONB has one. However, from a user perspective, this matches Cypher expectations: an integer stays an integer, a float stays a float, and very large or high-precision numbers can fall back to Numeric. All three are still serializable to text JSON (the output will look the same) – this is purely an internal distinction.
  - **Internal Representation and Encoding**: For standard JSON data, `agtype` uses the same binary encoding as `jsonb`. A string in `agtype` is stored with the same format as in `jsonb` (type code and length, followed by its characters, not null-terminated). Booleans and nulls are stored as special `agtentry` codes just like in `jsonb` (with no additional payload bytes besides the header). Arrays and objects in `agtype` have the same container structure and ordering rules as in `jsonb`. This commonality means that many `jsonb` operations (like containment checks, hashing, iteration) could be adapted to `agtype` with minimal changes. In fact, AGE has reused a lot of PostgreSQL’s `jsonb` internals: for example, the logic for iterating over an array or object, or for creating GIN index entries, is largely the same, just extended to account for new types. The `agtype_iterator` and `agtype_parse_state` closely parallel their JSONB counterparts, and much of the serialization/deserialization code in `agtype.c` is based on PostgreSQL’s implementation.
  - **Graph-Specific Additions and Extensibility**: The primary extensions in `agtype` serve the graph model, but the way they were implemented leaves room for further types if needed. By using the `AGTENTRY_IS_AGTYPE` entry type and a small header to delineate new subtypes, the designers made `agtype` extensible. The current set (integer, float, vertex, edge, path) covers what openCypher requires. If future graph features required new kinds of values (for example, a spatial type or a custom graph value), they could theoretically be added using another header code without redesigning the whole format. JSONB alone has no concept of such extra types – it is limited to JSON’s domain. In terms of extensibility, `agtype` shows how a JSONB foundation can be expanded to a new domain (graph data) while preserving the general model of the data format.
  - **Compatibility with JSONB**: On a functional level, `agtype` can be viewed as a superset of `jsonb` – any valid JSONB data can be represented as `agtype` (indeed with the same binary format for common parts). However, the presence of graph types means that not all `agtype` data is valid `jsonb`. For instance, a pure JSON object or array stored in `agtype` uses identical flags and structures, so an AGE value like ['hello', 123] (an array of string and integer) is binary-compatible with how `jsonb` would store ["hello", 123] (except that 123 might be stored as an `AGTV_INTEGER` instead of Numeric). But a value containing a vertex or edge has no equivalent in `jsonb` – if you tried to interpret it as `jsonb`, the `AGTENTRY_IS_AGTYPE` type code would be unrecognized. Therefore, direct casts between `jsonb` and `agtype` need to serialize/deserialize through text or handle those extended types explicitly. Internally, AGE reuses the `JsonbContainer` logic but forks the storage format by allocating unused type code space to new purposes. This means `agtype` is not binary-compatible with `jsonb` in general; it’s a parallel format. That said, much of the index support and operator logic (e.g. existence of keys, containment of structures) is borrowed from `jsonb`. AGE’s GIN indexing for `agtype` indexes keys and values similarly to JSONB (including a trick where string array elements are indexed as if they were keys to support the exists operator). The `agtype` operators like `->` (field access), `@>` (contains), etc., are implemented in a manner analogous to their JSONB counterparts, extended to handle the new types appropriately.

## Casts

```sql
cast from ag_catalog.agtype to ag_catalog.graphid
cast from ag_catalog.agtype to bigint
cast from ag_catalog.agtype to boolean
cast from ag_catalog.agtype to double precision
cast from ag_catalog.agtype to integer
cast from ag_catalog.agtype to integer[]
cast from ag_catalog.agtype to smallint
cast from ag_catalog.agtype to text
cast from ag_catalog.graphid to ag_catalog.agtype
cast from bigint to ag_catalog.agtype
cast from boolean to ag_catalog.agtype
cast from double precision to ag_catalog.agtype
```

## Functions

```sql
function ag_catalog.age_abs("any")
function ag_catalog.age_acos("any")
function ag_catalog.age_agtype_float8_accum(double precision[],ag_catalog.agtype)
function ag_catalog.age_agtype_larger_aggtransfn(ag_catalog.agtype,"any")
function ag_catalog.age_agtype_smaller_aggtransfn(ag_catalog.agtype,"any")
function ag_catalog.age_agtype_sum(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_asin("any")
function ag_catalog.age_atan2("any")
function ag_catalog.age_atan("any")
function ag_catalog.age_avg(ag_catalog.agtype)
function ag_catalog.age_build_vle_match_edge(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_ceil("any")
function ag_catalog.age_collect_aggfinalfn(internal)
function ag_catalog.age_collect_aggtransfn(internal,"any")
function ag_catalog.age_collect("any")
function ag_catalog.age_cos("any")
function ag_catalog.age_cot("any")
function ag_catalog.age_create_barbell_graph(name,integer,integer,name,ag_catalog.agtype,name,ag_catalog.agtype)
function ag_catalog.age_degrees("any")
function ag_catalog.age_delete_global_graphs(ag_catalog.agtype)
function ag_catalog.age_e()
function ag_catalog.age_end_id(ag_catalog.agtype)
function ag_catalog.age_endnode(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_eq_tilde(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_exists(ag_catalog.agtype)
function ag_catalog.age_exp("any")
function ag_catalog.age_float8_stddev_pop_aggfinalfn(double precision[])
function ag_catalog.age_float8_stddev_samp_aggfinalfn(double precision[])
function ag_catalog.age_floor("any")
function ag_catalog.age_head(ag_catalog.agtype)
function ag_catalog.age_id(ag_catalog.agtype)
function ag_catalog.age_isempty(ag_catalog.agtype)
unction ag_catalog.age_keys(ag_catalog.agtype)
function ag_catalog.age_label(ag_catalog.agtype)
function ag_catalog.age_labels(ag_catalog.agtype)
function ag_catalog.age_last(ag_catalog.agtype)
function ag_catalog.age_left("any")
function ag_catalog.age_length(ag_catalog.agtype)
function ag_catalog.age_log10("any")
function ag_catalog.age_log("any")
function ag_catalog.age_ltrim("any")
function ag_catalog.age_match_two_vle_edges(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_match_vle_edge_to_id_qual("any")
function ag_catalog.age_match_vle_terminal_edge("any")
function ag_catalog.age_materialize_vle_edges(ag_catalog.agtype)
function ag_catalog.age_materialize_vle_path(ag_catalog.agtype)
function ag_catalog.age_max("any")
function ag_catalog.age_min("any")
function ag_catalog._ag_enforce_edge_uniqueness("any")
function ag_catalog.age_nodes(ag_catalog.agtype)
function ag_catalog.age_percentile_aggtransfn(internal,ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_percentilecont(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_percentile_cont_aggfinalfn(internal)
function ag_catalog.age_percentiledisc(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_percentile_disc_aggfinalfn(internal)
function ag_catalog.age_pi()
function ag_catalog.age_prepare_cypher(cstring,cstring)
function ag_catalog.age_properties(ag_catalog.agtype)
function ag_catalog.age_radians("any")
function ag_catalog.age_rand()
function ag_catalog.age_range("any")
function ag_catalog.age_relationships(ag_catalog.agtype)
function ag_catalog.age_replace("any")
function ag_catalog.age_reverse("any")
function ag_catalog.age_right("any")
function ag_catalog.age_round("any")
function ag_catalog.age_rtrim("any")
function ag_catalog.age_sign("any")
function ag_catalog.age_sin("any")
function ag_catalog.age_size("any")
function ag_catalog.age_split("any")
function ag_catalog.age_sqrt("any")
function ag_catalog.age_start_id(ag_catalog.agtype)
function ag_catalog.age_startnode(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_stdev(ag_catalog.agtype)
function ag_catalog.age_stdevp(ag_catalog.agtype)
function ag_catalog.age_substring("any")
function ag_catalog.age_sum(ag_catalog.agtype)
function ag_catalog.age_tail("any")
function ag_catalog.age_tan("any")
function ag_catalog.age_timestamp()
function ag_catalog.age_toboolean("any")
function ag_catalog.age_tobooleanlist("any")
function ag_catalog.age_tofloat("any")
function ag_catalog.age_tofloatlist("any")
function ag_catalog.age_tointeger("any")
function ag_catalog.age_tointegerlist("any")
function ag_catalog.age_tolower("any")
function ag_catalog.age_tostring("any")
function ag_catalog.age_tostringlist("any")
function ag_catalog.age_toupper("any")
function ag_catalog.age_trim("any")
function ag_catalog.age_type(ag_catalog.agtype)
function ag_catalog.age_unnest(ag_catalog.agtype)
function ag_catalog.age_vertex_stats(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_vle(ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.age_vle(ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_access_operator(ag_catalog.agtype[])
function ag_catalog.agtype_access_slice(ag_catalog.agtype,ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_add(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_any_add(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_add(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_add(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_add(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_add(ag_catalog.agtype,real)
function ag_catalog.agtype_any_add(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_add(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_add(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_add(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_add(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_add(real,ag_catalog.agtype)
function ag_catalog.agtype_any_add(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_div(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_div(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_div(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_div(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_div(ag_catalog.agtype,real)
function ag_catalog.agtype_any_div(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_div(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_div(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_div(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_div(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_div(real,ag_catalog.agtype)
function ag_catalog.agtype_any_div(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,real)
function ag_catalog.agtype_any_eq(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_eq(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(real,ag_catalog.agtype)
function ag_catalog.agtype_any_eq(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,real)
function ag_catalog.agtype_any_ge(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_ge(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(real,ag_catalog.agtype)
function ag_catalog.agtype_any_ge(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,real)
function ag_catalog.agtype_any_gt(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_gt(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(real,ag_catalog.agtype)
function ag_catalog.agtype_any_gt(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_le(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_le(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_le(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_le(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_le(ag_catalog.agtype,real)
function ag_catalog.agtype_any_le(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_le(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_le(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_le(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_le(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_le(real,ag_catalog.agtype)
function ag_catalog.agtype_any_le(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,real)
function ag_catalog.agtype_any_lt(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_lt(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(real,ag_catalog.agtype)
function ag_catalog.agtype_any_lt(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,real)
function ag_catalog.agtype_any_mod(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_mod(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(real,ag_catalog.agtype)
function ag_catalog.agtype_any_mod(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,real)
function ag_catalog.agtype_any_mul(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_mul(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(real,ag_catalog.agtype)
function ag_catalog.agtype_any_mul(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,real)
function ag_catalog.agtype_any_ne(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_ne(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(real,ag_catalog.agtype)
function ag_catalog.agtype_any_ne(smallint,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,bigint)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,double precision)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,integer)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,numeric)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,real)
function ag_catalog.agtype_any_sub(ag_catalog.agtype,smallint)
function ag_catalog.agtype_any_sub(bigint,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(double precision,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(integer,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(numeric,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(real,ag_catalog.agtype)
function ag_catalog.agtype_any_sub(smallint,ag_catalog.agtype)
function ag_catalog.agtype_array_element(ag_catalog.agtype,integer)
function ag_catalog.agtype_array_element_text(ag_catalog.agtype,integer)
function ag_catalog.agtype_btree_cmp(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog._agtype_build_edge(ag_catalog.graphid,ag_catalog.graphid,ag_catalog.graphid,cstring,ag_catalog.agtype)
function ag_catalog.agtype_build_list()
function ag_catalog.agtype_build_list("any")
function ag_catalog.agtype_build_map()
function ag_catalog.agtype_build_map("any")
function ag_catalog.agtype_build_map_nonull("any")
function ag_catalog._agtype_build_path("any")
function ag_catalog._agtype_build_vertex(ag_catalog.graphid,cstring,ag_catalog.agtype)
function ag_catalog.agtype_concat(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_contained_by(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_contains(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_div(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_eq(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_exists(ag_catalog.agtype,text)
function ag_catalog.agtype_exists_agtype(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_exists_all(ag_catalog.agtype,text[])
function ag_catalog.agtype_exists_all_agtype(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_exists_any(ag_catalog.agtype,text[])
function ag_catalog.agtype_exists_any_agtype(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_extract_path(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_extract_path_text(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_ge(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_gt(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_hash_cmp(ag_catalog.agtype)
function ag_catalog.agtype_in(cstring)
function ag_catalog.agtype_in_operator(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_le(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_lt(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_mod(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_mul(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_ne(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_neg(ag_catalog.agtype)
function ag_catalog.agtype_object_field(ag_catalog.agtype,text)
function ag_catalog.agtype_object_field_agtype(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_object_field_text(ag_catalog.agtype,text)
function ag_catalog.agtype_object_field_text_agtype(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_out(ag_catalog.agtype)
function ag_catalog.agtype_pow(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_recv(internal)
function ag_catalog.agtype_send(ag_catalog.agtype)
function ag_catalog.agtype_string_match_contains(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_string_match_ends_with(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_string_match_starts_with(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_sub(ag_catalog.agtype,ag_catalog.agtype)
function ag_catalog.agtype_to_bool(ag_catalog.agtype)
function ag_catalog.agtype_to_float8(ag_catalog.agtype)
function ag_catalog.agtype_to_graphid(ag_catalog.agtype)
function ag_catalog.agtype_to_int2("any")
function ag_catalog.agtype_to_int4("any")
function ag_catalog.agtype_to_int4_array("any")
function ag_catalog.agtype_to_int8("any")
function ag_catalog.agtype_to_text(ag_catalog.agtype)
function ag_catalog.agtype_typecast_bool("any")
function ag_catalog.agtype_typecast_edge("any")
function ag_catalog.agtype_typecast_float("any")
function ag_catalog.agtype_typecast_int("any")
function ag_catalog.agtype_typecast_numeric("any")
function ag_catalog.agtype_typecast_path("any")
function ag_catalog.agtype_typecast_vertex("any")
function ag_catalog.agtype_volatile_wrapper("any")
function ag_catalog.alter_graph(name,cstring,name)
function ag_catalog.bool_to_agtype(boolean)
function ag_catalog.create_complete_graph(name,integer,name,name)
function ag_catalog.create_elabel(name,name)
function ag_catalog.create_graph(name)
function ag_catalog.create_vlabel(name,name)
function ag_catalog._cypher_create_clause(internal)
function ag_catalog._cypher_delete_clause(internal)
function ag_catalog._cypher_merge_clause(internal)
function ag_catalog.cypher(name,cstring,ag_catalog.agtype)
function ag_catalog._cypher_set_clause(internal)
function ag_catalog.drop_graph(name,boolean)
function ag_catalog.drop_label(name,name,boolean)
function ag_catalog._extract_label_id(ag_catalog.graphid)
function ag_catalog.float8_to_agtype(double precision)
function ag_catalog.get_cypher_keywords()
function ag_catalog.gin_compare_agtype(text,text)
function ag_catalog.gin_consistent_agtype(internal,smallint,ag_catalog.agtype,integer,internal,internal)
function ag_catalog.gin_extract_agtype(ag_catalog.agtype,internal)
function ag_catalog.gin_extract_agtype_query(ag_catalog.agtype,internal,smallint,internal,internal)
function ag_catalog.gin_triconsistent_agtype(internal,smallint,ag_catalog.agtype,integer,internal,internal,internal)
function ag_catalog.graphid_btree_cmp(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_btree_sort(internal)
function ag_catalog.graphid_eq(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_ge(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_gt(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_hash_cmp(ag_catalog.graphid)
function ag_catalog.graphid_in(cstring)
function ag_catalog._graphid(integer,bigint)
function ag_catalog.graphid_le(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_lt(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_ne(ag_catalog.graphid,ag_catalog.graphid)
function ag_catalog.graphid_out(ag_catalog.graphid)
function ag_catalog.graphid_recv(internal)
function ag_catalog.graphid_send(ag_catalog.graphid)
function ag_catalog.graphid_to_agtype(ag_catalog.graphid)
function ag_catalog.int8_to_agtype(bigint)
function ag_catalog._label_id(name,name)
function ag_catalog._label_name(oid,ag_catalog.graphid)
function ag_catalog.load_edges_from_file(name,name,text)
function ag_catalog.load_labels_from_file(name,name,text,boolean)
```

### create_graph(name) / drop_graph(name,boolean)

`create_graph` is the initial function used to create a graph with Apache AGE. It takes a single argument, `name`. In contrast, `drop_graph` takes a second argument — a flag indicating whether to cascade the deletion. As mentioned above, a graph consists of an entry in `ag_graph`, a `SERIAL` sequence, and its associated nodes, edges, and their sequences. Dropping a graph requires **cascading** deletion of these metadata elements.

### create_vlabel(name,name) / create_elabel(name,name) / drop_label(name,name,boolean)

These functions are not strictly necessary, as nodes and edges can be created and dropped using Cypher queries.

### cypher(name,cstring,ag_catalog.agtype)

`cypher` function is the core of Apache AGE. As mentioned in [Implementation of AGE](03_implementation.md), it's an **entrance** of Cypher query.

### load_labels_from_file(name,name,text,boolean) / load_edges_from_file(name,name,text)

These functions are used to load labels (nodes) and edges from files. The arguments are: the graph name, the label name, and the file path. For `load_labels_from_file`, there is an additional fourth argument — a boolean flag indicating whether to exclude the `id` field from the file.

Unfortunately, the arguments these two functions can accept as file paths are limited to **local** files, and therefore cannot be used with Azure Database for PostgreSQL.

## Operators

```sql
operator ag_catalog.^(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.<=(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.<>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.<(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.<@(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.=(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.>=(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.||(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.->>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.->(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.-(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.?|(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.?(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.?&(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog./(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.@>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.*(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.#>>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.#>(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.%(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.+(ag_catalog.agtype,ag_catalog.agtype)
operator ag_catalog.<=(ag_catalog.agtype,bigint)
operator ag_catalog.<>(ag_catalog.agtype,bigint)
operator ag_catalog.<(ag_catalog.agtype,bigint)
operator ag_catalog.=(ag_catalog.agtype,bigint)
operator ag_catalog.>=(ag_catalog.agtype,bigint)
operator ag_catalog.>(ag_catalog.agtype,bigint)
operator ag_catalog.-(ag_catalog.agtype,bigint)
operator ag_catalog./(ag_catalog.agtype,bigint)
operator ag_catalog.*(ag_catalog.agtype,bigint)
operator ag_catalog.%(ag_catalog.agtype,bigint)
operator ag_catalog.+(ag_catalog.agtype,bigint)
operator ag_catalog.<=(ag_catalog.agtype,double precision)
operator ag_catalog.<>(ag_catalog.agtype,double precision)
operator ag_catalog.<(ag_catalog.agtype,double precision)
operator ag_catalog.=(ag_catalog.agtype,double precision)
operator ag_catalog.>=(ag_catalog.agtype,double precision)
operator ag_catalog.<=(ag_catalog.agtype,integer)
operator ag_catalog.<>(ag_catalog.agtype,integer)
operator ag_catalog.<(ag_catalog.agtype,integer)
operator ag_catalog.=(ag_catalog.agtype,integer)
operator ag_catalog.>=(ag_catalog.agtype,integer)
operator ag_catalog.>(ag_catalog.agtype,integer)
operator ag_catalog.->>(ag_catalog.agtype,integer)
operator ag_catalog.->(ag_catalog.agtype,integer)
operator ag_catalog.-(ag_catalog.agtype,integer)
operator ag_catalog./(ag_catalog.agtype,integer)
operator ag_catalog.*(ag_catalog.agtype,integer)
operator ag_catalog.%(ag_catalog.agtype,integer)
operator ag_catalog.+(ag_catalog.agtype,integer)
operator ag_catalog.<=(ag_catalog.agtype,numeric)
operator ag_catalog.<>(ag_catalog.agtype,numeric)
operator ag_catalog.<(ag_catalog.agtype,numeric)
operator ag_catalog.=(ag_catalog.agtype,numeric)
operator ag_catalog.>=(ag_catalog.agtype,numeric)
operator ag_catalog.>(ag_catalog.agtype,numeric)
operator ag_catalog.-(ag_catalog.agtype,numeric)
operator ag_catalog./(ag_catalog.agtype,numeric)
operator ag_catalog.*(ag_catalog.agtype,numeric)
operator ag_catalog.%(ag_catalog.agtype,numeric)
operator ag_catalog.+(ag_catalog.agtype,numeric)
operator ag_catalog.<=(ag_catalog.agtype,real)
operator ag_catalog.<>(ag_catalog.agtype,real)
operator ag_catalog.<(ag_catalog.agtype,real)
operator ag_catalog.=(ag_catalog.agtype,real)
operator ag_catalog.>=(ag_catalog.agtype,real)
operator ag_catalog.>(ag_catalog.agtype,real)
operator ag_catalog.-(ag_catalog.agtype,real)
operator ag_catalog./(ag_catalog.agtype,real)
operator ag_catalog.*(ag_catalog.agtype,real)
operator ag_catalog.%(ag_catalog.agtype,real)
operator ag_catalog.+(ag_catalog.agtype,real)
operator ag_catalog.<=(ag_catalog.agtype,smallint)
operator ag_catalog.<>(ag_catalog.agtype,smallint)
operator ag_catalog.<(ag_catalog.agtype,smallint)
operator ag_catalog.=(ag_catalog.agtype,smallint)
operator ag_catalog.>=(ag_catalog.agtype,smallint)
operator ag_catalog.>(ag_catalog.agtype,smallint)
operator ag_catalog.-(ag_catalog.agtype,smallint)
operator ag_catalog./(ag_catalog.agtype,smallint)
operator ag_catalog.?(ag_catalog.agtype,text)
operator ag_catalog.?&(ag_catalog.agtype,text[])
operator ag_catalog.<=(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.<>(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.<(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.=(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.>=(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.>(ag_catalog.graphid,ag_catalog.graphid)
operator ag_catalog.<=(bigint,ag_catalog.agtype)
operator ag_catalog.<>(bigint,ag_catalog.agtype)
operator ag_catalog.<(bigint,ag_catalog.agtype)
operator ag_catalog.=(bigint,ag_catalog.agtype)
operator ag_catalog.>=(bigint,ag_catalog.agtype)
operator ag_catalog.>(bigint,ag_catalog.agtype)
operator ag_catalog.-(bigint,ag_catalog.agtype)
operator ag_catalog./(bigint,ag_catalog.agtype)
operator ag_catalog.*(bigint,ag_catalog.agtype)
operator ag_catalog.%(bigint,ag_catalog.agtype)
operator ag_catalog.+(bigint,ag_catalog.agtype)
operator ag_catalog.<=(double precision,ag_catalog.agtype)
operator ag_catalog.<>(double precision,ag_catalog.agtype)
operator ag_catalog.<(double precision,ag_catalog.agtype)
operator ag_catalog.=(double precision,ag_catalog.agtype)
operator ag_catalog.>=(double precision,ag_catalog.agtype)
operator ag_catalog.>(double precision,ag_catalog.agtype)
operator ag_catalog.-(double precision,ag_catalog.agtype)
operator ag_catalog./(double precision,ag_catalog.agtype)
operator ag_catalog.*(double precision,ag_catalog.agtype)
operator ag_catalog.%(double precision,ag_catalog.agtype)
operator ag_catalog.+(double precision,ag_catalog.agtype)
operator ag_catalog.<=(integer,ag_catalog.agtype)
operator ag_catalog.<>(integer,ag_catalog.agtype)
operator ag_catalog.<(integer,ag_catalog.agtype)
operator ag_catalog.=(integer,ag_catalog.agtype)
operator ag_catalog.>=(integer,ag_catalog.agtype)
operator ag_catalog.>(integer,ag_catalog.agtype)
operator ag_catalog.-(integer,ag_catalog.agtype)
operator ag_catalog./(integer,ag_catalog.agtype)
operator ag_catalog.*(integer,ag_catalog.agtype)
operator ag_catalog.%(integer,ag_catalog.agtype)
operator ag_catalog.+(integer,ag_catalog.agtype)
operator ag_catalog.-(NONE,ag_catalog.agtype)
operator ag_catalog.<=(numeric,ag_catalog.agtype)
operator ag_catalog./(numeric,ag_catalog.agtype)
operator ag_catalog.*(numeric,ag_catalog.agtype)
operator ag_catalog.%(numeric,ag_catalog.agtype)
operator ag_catalog.+(numeric,ag_catalog.agtype)
operator ag_catalog.<=(real,ag_catalog.agtype)
operator ag_catalog.<>(real,ag_catalog.agtype)
operator ag_catalog.<(real,ag_catalog.agtype)
operator ag_catalog.=(real,ag_catalog.agtype)
operator ag_catalog.>=(real,ag_catalog.agtype)
operator ag_catalog.>(real,ag_catalog.agtype)
operator ag_catalog.-(real,ag_catalog.agtype)
operator ag_catalog./(real,ag_catalog.agtype)
operator ag_catalog.*(real,ag_catalog.agtype)
operator ag_catalog.%(real,ag_catalog.agtype)
operator ag_catalog.+(real,ag_catalog.agtype)
operator ag_catalog.<=(smallint,ag_catalog.agtype)
operator ag_catalog.<>(smallint,ag_catalog.agtype)
operator ag_catalog.<(smallint,ag_catalog.agtype)
operator ag_catalog.=(smallint,ag_catalog.agtype)
operator ag_catalog.>=(smallint,ag_catalog.agtype)
operator ag_catalog.>(smallint,ag_catalog.agtype)
operator ag_catalog.-(smallint,ag_catalog.agtype)
operator ag_catalog./(smallint,ag_catalog.agtype)
operator ag_catalog.*(smallint,ag_catalog.agtype)
operator ag_catalog.%(smallint,ag_catalog.agtype)
operator ag_catalog.+(smallint,ag_catalog.agtype)
operator ag_catalog.?(text,ag_catalog.agtype)
operator class ag_catalog.agtype_ops_btree for access method btree
operator class ag_catalog.agtype_ops_hash for access method hash
operator class ag_catalog.gin_agtype_ops for access method gin
operator class ag_catalog.graphid_ops for access method btree
operator class ag_catalog.graphid_ops_hash for access method hash
operator family ag_catalog.agtype_ops_btree for access method btree
operator family ag_catalog.agtype_ops_hash for access method hash
operator family ag_catalog.gin_agtype_ops for access method gin
operator family ag_catalog.graphid_ops for access method btree
operator family ag_catalog.graphid_ops_hash for access method hash
```

[Prev: Implementation](03_implementation.md) | [Next: Cypher Query with AGE](05_cypher.md)
