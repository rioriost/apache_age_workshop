# Apache AGEのCypherクエリ

Apache AGEでのCypherクエリの表現は、**標準**のCypherとは異なります。

例えば、標準的なCypher：
```cypher
MATCH (n)-[r]->(m) RETURN n, r, m
```

はApache AGEでは以下のように書かれます：
```sql
SELECT * FROM cypher('graph_name',
  $$
  MATCH (n)-[r]->(m) RETURN n, r, m
  $$
)
AS (n agtype, r agtype, m agtype);
```

## 標準的なCypherクエリをAGEの表現に変換する方法

Apache AGEでは、すべてのCypherクエリがSQLステートメント内に埋め込まれています。AS句を使用する主な理由は、`cypher()`関数によって返される結果セットの構造を定義することです。つまり、列名とそれに対応するPostgreSQLの型を定義します。

[AGEFreighter](https://github.com/rioriost/agefreighter)は、この目的のために`convert`サブコマンドを提供しています。

```bash
agefreighter convert -q 'MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m'
Converted Cypher queries for Apache AGE:

line 1, MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m ->
SELECT * FROM cypher('GRAPH_FOR_DRYRUN', $$ MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n, r, m $$) AS (m agtype, n agtype, r agtype);
```

AGEFreighterは、その文法的および語彙的なパーサーを使用してCypherクエリを変換します。`parse`サブコマンドを実行すると、その動作が示されます：

```bash
agefreighter parse 'MATCH (n:Person)-[r:WORK_AT]->(m:Company) RETURN n,r,m'
[('MATCH', [('chain', ('node', 'n', ['Person'], None), [(('directed', ('relationship', [{'variable': 'r', 'type': 'WORK_AT'}], None, None)), ('node', 'm', ['Company'], None))])], None), ('RETURN', ['n', 'r', 'm'])]
```

AGEクエリに`AS`句を追加するためには、`RETURN`トークングループの存在が必要です。

残念ながら、AGEFreighterはすべてのCypherクエリパターンをサポートしていません。そのような場合には、以下のガイドが役立つでしょう。

### AS句の目的

AS句は、PostgreSQLにクエリの出力で何を期待するかを伝えます：
  - **列名**：例では、`n`、`r`、`m`が列名として使用されています。
  - **データ型**：`agtype`は、列がAGEのJSONライクな形式でグラフ要素（ノード、エッジなど）を含むことを示すために使用されます。

### CypherのRETURNとのマッチング：

AS句で宣言された列の数は、CypherクエリのRETURN句で返される項目の数と完全に一致しなければなりません。例えば、あなたのCypherクエリが複数の要素を返す場合：

```cypher
MATCH (a)-[r:KNOWS]->(b) RETURN a, r, b
```

あなたは3つの列を宣言する必要があります：
```sql
SELECT * FROM cypher('graph_name', $$MATCH (a)-[r:KNOWS]->(b) RETURN a, r, b$$) AS (a agtype, r agtype, b agtype);
```

### データ型の柔軟性：
`agtype`はノードとエッジのデフォルトのデータ型ですが、単純なプロパティ（文字列や整数など）を返す場合は、テキストや整数などのネイティブSQL型を指定することができます。例えば：

```sql
SELECT * FROM cypher('graph_name', $$MATCH (p:Person) RETURN p.name$$) AS (name text);
```

このようにすると、プロパティは直接SQLテキスト値として利用可能になります。

## Apache AGEの制限

Apache AGEはNeo4jと比較していくつかの制限があります。

### APOC

APOCは一般的にNeo4jと一緒に使用され、Cypherの機能を拡張します。しかし、Apache AGEはAPOCをサポートしていません。あなたのCypherクエリがAPOCに依存している場合、代替手段を見つける必要があります。つまり、**回避策**を見つける必要があります。

1. 変換 / 処理
  - apoc.convert.*: PostgreSQLの`CAST()`を使用します
  - apov.date.*: PostgreSQLの`to_date()`を使用します

2. グラフ操作
  - apoc.create.*: `create_graph()`、`create_vlabel()`、`create_elabel()`を使用します
  - apoc.refactor.*: カスタムプログラミングが必要です（例：Python、C#）

3. 検索
  - apoc.path.*: マルチステップのCypherクエリや外部スクリプトを使用します

4. 統合
  - apoc.load.*: [AGEFreighter](https://github.com/rioriost/agefreighter)のようなツールを使用します
  - apoc.export.*: `\COPY`とCypherクエリを組み合わせます
  - apoc.http.*: カスタムプログラミングが必要です（例：Python、C#）

5. ユーティリティ
  - apoc.meta.*: `psql`や`pgdump`を使用します
  - apoc.periodic.*: `pg_cron`拡張を使用します
  - apoc.trigger.*: PostgreSQLの`TRIGGER`機能を使用します

### タブ文字

[Objects](04_objects.md)で述べたように、`agtype`は`jsonb`のスーパーセットです。これは、ノードやエッジの`properties`列にタブが含まれている場合、**タブ文字**、`chr(9)`をエスケープする必要があることも意味します。

### 複数のラベル

Apache AGE 1.5の時点では、AGEは単一のノードに複数のラベルを割り当てることをサポートしていません。

**Neo4j**では、複数のラベル（例えば、`Person`と`Employee`）を持つノードを作成することが可能です：
```cypher
CREATE (:Person:Employee {name: 'Alice', age: 30})
```

しかし、**Apache AGE**では、同じ構文を試すと構文エラーが発生します：
```sql
SELECT * FROM cypher('graph_name', $$CREATE (n:Person:Employee) RETURN n$$) AS (n agtype);
```

エラー：
```sql
ERROR:  syntax error at or near ":"
LINE 1: ...ELECT * from cypher('graph_name', $$CREATE (n:Person:Employee)...
                                                               ^
```

[前: Objects](04_objects_ja.md) | [次: Indexing](06_indexing_ja.md)
