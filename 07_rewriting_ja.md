# Neo4jのCypherの書き換え

[Cypher](05_cypher.md)で説明したように、Apache AGEには一定の制限があります。
しかし、Neo4jから移行する際の主な課題は、[openCypher specification](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf)との違いにあります。

## パターンマッチングの拡張

- **パターン内のラベルの統合/交差/否定** – Neo4j 5では、ノードパターン内で `|` (OR)、`&` (AND)、`!` (NOT)を使用してノードラベルの論理組み合わせを許可します。例えば、`MATCH (c:Character|Harkonnen)`は、`Character`または`Harkonnen`ラベルを持つノードを見つけます。一方、`MATCH (c:Character&!Harkonnen)`は、`Harkonnen`ラベルを持たない`Character`ラベルを持つノードを見つけます。OpenCypher 9はインラインラベルのブール式をサポートしていません - openCypherでは、ラベルでフィルタリングするためには別々のパターンを使用するか、`WHERE`句を使用する必要があります。

```cypher
-- Neo4j 5.x example:
MATCH (p:Person|Employee)        // `p` has label Person OR Employee
RETURN p

MATCH (c:Character&!Harkonnen)   // `c` has label Character AND NOT Harkonnen
RETURN c.name
```

openCypherでは、上記は`MATCH (p) WHERE p:Person OR p:Employee`と別の`WHERE NOT c:Harkonnen`フィルタが必要となります。なぜなら、`|`、`&`、`!`ラベル構文はNeo4j特有のものだからです。

- **リレーションタイプの否定** – 同様に、Neo4jは`!`演算子を使用して特定のリレーションタイプを除外するリレーションパターンを拡張します。例えば：

```cypher
MATCH (p:Person)-[r:!FRIEND]-() RETURN p, type(r)
```

これは、`Person`からの`FRIEND`タイプでない任意のリレーションにマッチします。この構文（`:!Type`を使用）はopenCypher 9にはありません - openCypherでは、マッチの後にフィルタリングする必要があります（例：`WHERE type(r) <> "FRIEND"`）。Neo4j 5では、複数のタイプに対して否定を`&/|`で組み合わせることも許可されています（例：`[r:!ALLIES&!FAMILY]`で両方のタイプを除外）。

- **固定長パスの省略形** – Neo4jは、カーリーブレースを使用して正確なパス長を指定する代替パターン構文をサポートしています。例えば、`MATCH (a)--{2}(b)`は、ノード`a`と`b`の間の長さがちょうど2のパスを見つけます。これはCypherの標準的な`MATCH (a)-[*2]-(b)`と同等です。`{n}`または`{m..n}`のカーリーブレース表記法はNeo4jの拡張（SPARQL/GQLに触発されたもの）で、openCypher 9では定義されていません（`[*min..max]`構文を使用します）。OpenCypherでは、2ホップパターンには`-[*2]-`が必要です。

- **`MATCH`内のインラインプロパティフィルタ** – Neo4j 5は、ノードの直後に`WHERE`フィルタを直接パターン内に置く機能を導入しました。例えば：

```cypher
MATCH (c:Character **WHERE** c.Culture STARTS WITH "Z" AND c.Died IS NOT NULL)
RETURN c.name
```

これは、指定されたプロパティ述語を満たす`Character`ノードにマッチします。openCypherでは、このようなプロパティ条件は`MATCH`パターン内に存在できず、パターンの後の別の`WHERE`句に配置する必要があります。Neo4jのパターン構文のインライン`WHERE`は、openCypher 9には存在しない便利な機能です。

## サブクエリとフィルタリングの拡張

- **有無のサブクエリ（`EXISTS {...}`）** – Neo4jは、パターンをラップする、有無を調べるサブクエリで`WHERE`句を拡張します。（現在は非推奨の）`exists((n)-->(m))`関数を使用する代わりに、Neo4j 5はカーリーブレースでサブクエリブロックを必要とします。例えば：

```cypher
MATCH (c:Character)
WHERE **exists { (c)-[:FAMILY]-() }**
RETURN c
```

これは、少なくとも一つの`FAMILY`リレーションを持つキャラクターをフィルタリングします。OpenCypher 9はこのサブクエリ構文を含んでいません - Neo4jでは、チェックするパターンが`exists { ... }`内に配置される構文の変更です。このブロック内に内部の`WHERE`を含めたり、新しい変数を導入したりすることもできます。openCypherでは、異なるアプローチを使用する必要がありました（例：`OPTIONAL MATCH + WHERE`または古い`exists()`述語とパターン、これはopenCypherの仕様の一部ではありませんでした）。

- **パターンカウントサブクエリ (`count{...}`)** – Neo4j 5は、クエリのカーディナリティに影響を与えることなく、パターンの発生回数をカウントするための簡潔な構文を追加しました。`count{ (n)--() }`をリターンまたは述語で使用すると、各行のそのパターンのマッチ数が得られます。例えば、各人物の次数（接続数）を取得するには：

```cypher
MATCH (p:Person)
RETURN p.name, **count{ (p)--() } AS degree**
```

そして、2つ以上のリレーションを持つノードをフィルタリングするには：`WHERE count{ (p)--() } > 2`。これは、古い`size((p)--())`構造体（それ自体がopenCypherにはない）を置き換えるNeo4j固有の拡張です。OpenCypher 9には同等のパターンカウント構文がなく、典型的な回避策は、サブクエリで`COUNT`を展開して使用するか、マッチのリストのサイズを使用することです。

- **リスト述語関数 (`ANY`, `ALL`, `NONE`, `SINGLE`)** – Neo4j Cypherは、`any(x IN list WHERE ...)`, `all(...)`, `none(...)`, `single(...)`など、リスト要素に対する条件を評価する述語関数を提供します。例えば：

```cypher
MATCH p=(a:Start)-[:HOP*1..]->(z:End)
WHERE **none(node IN nodes(p) WHERE node.class = 'D')**
RETURN p
```

これは、クラス`"D"`を持つノードがない`Start`から`End`へのパスを見つけます。これらの便利な関数はopenCypher 9の仕様の一部ではありません。OpenCypher準拠のシステム（Neptuneなど）では、リストの包括とサイズを使用してこのようなロジックを書き換える必要があります。例えば、`WHERE size([node IN nodes(p) WHERE node.class='D']) = 0`を使用して`NONE`を模倣します。Neo4jでは、`ANY`, `ALL`などは組み込みの構文構造であり、openCypher 9にはそれらがありません。

- **`reduce()`リストの削減** – Neo4jのCypherは、`reduce(accumulator = initial, elem IN list | expression)`関数をサポートしており、これを使用するとリストを単一の値（例えばリストの合計）に折りたたむことができます。例えば：

```cypher
MATCH p=(a:Airport {code:'ANC'})-[r:ROUTE*1..3]->(b:Airport {code:'AUS'})
RETURN reduce(totalDist = 0, r IN relationships(p) | totalDist + r.dist) AS totalDist
```

これは各パスの合計距離を返します。この機能的な構文はopenCypher 9には存在せず、これはopenCypherの実装が回避策を使用しなければならないことを意味します（リストを展開して合計するなど、Neptuneの例に示されています）。したがって、`reduce()`はクエリ言語の表現力に対するNeo4j固有の拡張です。

## 書き込みと読み込みの節拡張

- **`FOREACH`更新節** – Neo4jのCypherには、クエリ内のリストの各要素に対して更新を行うための`FOREACH`節が含まれています（通常は書き込みクエリの途中で）。例えば、パス上のすべてのノードに訪問フラグを設定するには：

```cypher
MATCH p=(:Airport {code:'ANC'})-[*1..2]->(:Airport {code:'AUS'})
**FOREACH** (n IN nodes(p) | SET n.visited = true)
```

この節は、リスト（しばしばパスや集約から派生したもの）を反復処理し、サブコマンドを実行しますが、openCypherの仕様の一部ではありません。OpenCypher準拠の言語は、代わりにリストを展開し、通常の`SET`で更新を行う必要があります（Neptuneの書き換えで示されています）。Neo4jの`FOREACH`は、クエリ内のバッチ更新において、純粋に構文レベルで便利です。

- **`MERGE` with `ON CREATE/ON MATCH`** – Neo4jは、パターンが作成されたかマッチしたかに応じて更新を条件付きで実行するためのオプションのサブ節を持つ`MERGE`節を拡張します。例えば：

```cypher
MERGE (u:User {id: $userId})
  ON CREATE SET u.createdAt = timestamp()
  ON MATCH  SET u.lastSeen = timestamp();
```

この構文では、ノードの作成時と既存の場合でプロパティを異なる方法で設定できます。OpenCypher 9の仕様には、`MERGE`に対するこれらの`ON CREATE/ON MATCH`修飾子は含まれていません（それらはNeo4j 3.5+で導入されました）。標準的なopenCypherでは、同じ効果を達成するためにはより冗長なパターン（例えば、別々の`MERGE`や`CASE`ロジックを使用する）が必要です。条件付きマージ構文はNeo4j固有の強化です。

- **`LOAD CSV`節** – Neo4jは、`LOAD CSV`節（`WITH HEADERS`やフィールド終端子などのオプション付き）を介してCSVファイルからのデータのインポートをサポートしています。この節は、外部のCSVデータをクエリに読み込むため、または`FOREACH`や`UNWIND`と組み合わせて一括挿入を行うために使用できます。例：

```cypher
LOAD CSV WITH HEADERS FROM "file:///people.csv" AS line
CREATE (:Person { name: line.Name, age: toInteger(line.Age) });
```

`LOAD CSV`はopenCypher 9の標準の一部ではありません。これは、一括データのロードのためのNeo4j固有の節です。他のopenCypherの実装（AWS Neptuneなど）はこれをサポートしておらず、他の手段でデータをロードする必要があります。

- **手続きの呼び出し（`CALL ... YIELD`）** – Neo4jのCypherでは、クエリ内からストアドプロシージャやユーザー定義関数を呼び出すことができます。例えば、`CALL db.index.fulltext.search("Movies", "matrix") YIELD node, score RETURN node.title`とすることで全文検索の手続きを呼び出すことができます。手続きを呼び出すためのCypherの構文は`CALL procedureName(args) YIELD resultVars`です。このメカニズム（および`CALL`キーワード）はopenCypher 9では定義されていません。openCypherでは、手続きを呼び出す標準的な方法はありません - Neo4jはこれを拡張として追加しました。（注：APOC手続きはこれを使用するライブラリの例ですが、APOC自体はこのページでは説明しません）。Neo4jの`CALL ... YIELD`節は、本来は宣言的な言語である中で、命令的なサブコールを可能にする拡張です。

## その他のクエリ拡張と修飾子

- **`CALL { ... }`によるネストされたサブクエリ** – Neo4j（4.0以降）は、`CALL { ... }`構文を使用して、大きなクエリの中に任意のサブクエリを埋め込むことをサポートしています。このサブクエリは独立して実行され、結果を包含するクエリに返すことができます。例えば：

```cypher
MATCH (q:Question)
CALL {
  MATCH (q)<-[:ANSWERED]-(:Answer)
  RETURN count(*) AS answerCount
}
RETURN q.text, answerCount
```

これは、ユニオンの後処理、複雑なフィルタリングなどに使用されます。openCypher 9は内部サブクエリの規定を持っていません。Neo4jの`CALL {}`の追加（およびそれに伴うスコープルール）は、openCypherを超えた重要な構文拡張です。外部スコープからの任意の変数は明示的に渡されなければなりません - 元々は`WITH`を介してでしたが、Neo4j 5は次に説明するように、変数を直接インポートするより直接的な方法を導入しました。

- **サブクエリに変数を渡す（スコープ付き`CALL`）** – Neo4j 5では、`CALL`の後に括弧を書くことで、外部の変数を直接サブクエリに渡すことができます。例えば：

```cypher
MATCH (p:Person)
OPTIONAL CALL **(p)** {
  MATCH (p)-[:OWNS]->(d:Dog)
  RETURN d.name AS dogName
}
RETURN p.name, dogName
```

ここでは、外部クエリの`p`が、追加の`WITH`なしでサブクエリ内で利用可能になります。このスコープ付き`CALL`構文（`CALL(var1, var2) { ... }`）や、`CALL`の前に`OPTIONAL`を付ける能力はNeo4j固有のものです。openCypherはサブクエリをパラメータ化する方法や、`OPTIONAL CALL`を定義していません。これらの強化は、Cypherにオプションの結合に似た組成能力を与えます。（例えば、NeptuneのopenCypherは`CALL(n) {}`構文を全くサポートしていません。）

- **サブクエリのバッチ実行** – Neo4jでは、サブクエリを`CALL { ... } **IN TRANSACTIONS [OF N ROWS]**`の構文を使用して、定期的なコミットを伴うバッチモードで実行することができます。これにより、大量の書き込みを複数のトランザクションに分割することができます。例えば：

```cypher
MATCH (n:OldLabel)
CALL {
  WITH n
  DELETE n
} **IN TRANSACTIONS OF 1000 ROWS**
```

は、1000件の削除ごとにコミットします。`IN TRANSACTIONS`節はサブクエリの一部ではなく、Neo4j固有のものであり（4.xで追加されました）、古い`USING PERIODIC COMMIT`（`LOAD CSV`に特化したもの）を置き換えます。OpenCypher 9には、単一のクエリ内でトランザクションをバッチ処理するための類似の構造はありません。

- **プランナーヒント（`USING INDEX/SCAN`）** – Neo4jのCypherは、`USING`キーワードを使用してクエリに指定されるクエリヒントをサポートしています（例：`USING INDEX n:Person(name)`や`USING JOIN`や`USING SCAN`）。これらは、インデックスの使用方法や他の計画決定についてのクエリプランナーへの指示です。例えば：

```cypher
MATCH (n:Person) **USING INDEX n:Person(name)**
WHERE n.name = "Alice"
RETURN n
```
そのようなヒントはopenCypher 9の仕様の一部ではなく（クエリの計画は実装に任されています）、それらはNeo4j固有の構文追加です。openCypherを実装する他のデータベースでは、これらのヒントを無視したり、サポートしない場合があります。例えば、Neptuneは特定のエンジンバージョンで`USING INDEX`の部分的なサポートを追加しました。

- **`EXPLAIN/PROFILE`** – Neo4jでは、クエリは`EXPLAIN`または`PROFILE`というキーワードでプレフィックスを付けることで実行計画を調査できます。Cypherの`MATCH…​WHERE…​RETURN`シーケンス内の節ではありませんが、これらのキーワードはNeo4jのクエリ言語インターフェースの一部です。それらはopenCypher 9（宣言的なクエリ自体に焦点を当てている）で説明されていません。例えば、`EXPLAIN MATCH (n:Person) RETURN count(n)`はクエリプランを生成しますが、実際にはクエリを実行しません。これはデバッグと最適化のためのNeo4j固有の機能で、openCypherの仕様には含まれていません。これは純粋なクエリセマンティクスの外側にあります。

上記の各構文拡張はNeo4jのCypherをより強力または便利にしますが、それらを使用するクエリは純粋なopenCypher-9プラットフォームで実行するために書き換えが必要になる可能性があります。これらの拡張（新しいパターン構文からカスタム節まで）は、Neo4jのCypherがopenCypher v9標準を超えて進化したことを強調しています。示されたすべての例はNeo4j 5.xで有効であり、openCypherの参照実装には存在しない機能を示しています。

[前: インデックス作成](06_indexing_ja.md)
