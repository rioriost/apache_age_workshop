# インデックス作成

デフォルトでは、Apache AGEは新しく作成されたグラフに対してインデックスを作成しません。

## WHERE句

Apache AGEでは、以下のクエリは異なる評価を受けます：
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

インデックスの利点を最大限に活用するためには、`WHERE`句を含むクエリと含まないクエリでどのタイプのインデックスが利用されるかを理解する必要があります。

## Apache AGEのEXPLAIN

標準的なSQLとは異なり、Cypherクエリの`EXPLAIN`キーワードは異なるクエリフォーマットを必要とします。

クエリ：
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

結果：
```sql
QUERY PLAN
--------------------------------------------------------------------------------------------------------------
Seq Scan on "Customer" n  (cost=0.00..418.51 rows=43 width=32)
Filter: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
```

`WHERE`句がないクエリプランの違いを見るには：
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

## 一般的なインデックス作成

[Objects](04_objects.md)で説明したように、エッジテーブルは**開始ノードの** `id`に対応する`start_id`と**終了ノードの** `id`に対応する`end_id`を保存します。`start_id`と`end_id`にインデックスを作成することは、パフォーマンス向上に非常に効果的です。

```sql
CREATE INDEX ON graph_name."VLABEL" USING BTREE (id);
CREATE INDEX ON graph_name."ELABEL" USING BTREE (start_id);
CREATE INDEX ON graph_name."ELABEL" USING BTREE (end_id);
```

さらに、典型的なクエリでは、`properties`列内の`key`による`value`の検索がよく行われるため、その列にGIN（Generalized Inverted Index）を作成することも非常に有益です：

```sql
CREATE INDEX ON graph_name."VLABEL" USING GIN (properties);
```

`properties`列にGINインデックスがあると、`WHERE`句がないクエリは次の実行プランを結果とします：

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

これは、クエリプランナーが`properties`列のインデックスを利用していることを示しています。

一方、`WHERE`句を含むクエリは次のプランを生成します：

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

これは、クエリプランナーがこの場合、インデックスを利用していないことを示しています。

## `properties`の特定のキー-バリューに対するインデックス作成

`properties`列のキー-バリューペア全体をインデックス化する必要がない場合、特定のキーに対してより小さく、より効率的な`BTREE`インデックスを作成することができます。

以下のフィルタ式は、前のクエリプランで示されており、そのようなインデックスを作成する方法についての手がかりを提供します：

```sql
Filter: (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]) = '"Alice"'::agtype)
``````
この表現を直接使用して、`Name`キーを対象とするインデックスを作成できます：

```sql
CREATE INDEX ON graph_name.label_name USING BTREE (agtype_access_operator(VARIADIC ARRAY[properties, '"Name"'::agtype]));
```

インデックスを作成した後、`WHERE`句が**ない**クエリは次のような計画を生成します：
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

これは、この場合、クエリプランナーがインデックスを**利用しない**ことを示しています。

しかし、`WHERE`句が**ある**クエリは、はるかに効率的な計画をもたらします：

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

これははるかに好ましい実行計画であり、クエリプランナーが`WHERE`句が使用されるときにカスタムインデックスを利用することを示しています。

[前：CypherクエリとAGE](05_cypher_ja.md)
