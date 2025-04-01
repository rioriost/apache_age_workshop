# AGEの実装

## 実装の概要

[Apache AGEのトップページ](https://age.apache.org)に記載されているように、AGEは以下のステップを実行します：

1. [openCypher specification](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf)に従ったパーサーを使用した関数呼び出しによりクエリを解析します。
2. Cypherクエリをサブクエリノードとして添付されるクエリツリーに変換します。
3. グラフ操作を理解し、グラフ操作に関連するプランノードを生成します。
4. グラフ操作に関連するプランノードを実行します。
5. CypherクエリはPostgreSOLの既存の完全トランザクションシステム（ACID）と連携して動作します。

## 詳細

1. Cypherクエリの解析（openCypher文法）

Cypherクエリが（`cypher()`関数呼び出しを介して）送信されると、Apache AGEは最初にopenCypher仕様に従ってCypherクエリ文字列を解析します。拡張機能はCypher用のカスタムレキサーとパーサーを提供します：これはソースファイル[src/backend/parser/cypher_gram.y](https://github.com/apache/age/blob/master/src/backend/parser/cypher_gram.y)（openCypherのBison文法）と、[cypher_clause.c](https://github.com/apache/age/blob/master/src/backend/parser/cypher_clause.c)、[cypher_expr.c](https://github.com/apache/age/blob/master/src/backend/parser/cypher_expr.c)などのサポートファイルに実装されています。これらは一緒にCypher文法を定義し、Cypherクエリの内部パースツリー（または抽象構文ツリー）を構築します。本質的に、AGEの**クエリパーサー**コンポーネントは「Cypherクエリをその構成要素に分解し、キーワード、ノード、関係、パターンを認識します」。

2. CypherクエリをPostgreSQLクエリツリーに変換する

解析後、Apache AGEはCypher ASTをPostgreSQLクエリツリー（内部クエリ構造）に変換します。これはPostgreSQLエンジンによって計画および実行できます。これは、`cypher()`呼び出しを含むSQLステートメントの解析分析/書き換えフェーズ中に発生します。具体的には、Apache AGEは`post_parse_analyze_hook`を使用してPostgreSQLのパーサーアナライザーにフックします。拡張機能のアナライザー（[src/backend/parser/cypher_analyze.c](https://github.com/apache/age/blob/master/src/backend/parser/cypher_analyze.c)に実装）は、初期クエリツリーを走査して`cypher()`関数呼び出しを見つけ、それをCypherロジックを表す新しいサブクエリでその場で置き換えます。言い換えれば、`FROM`句の`cypher()`呼び出しは、グラフパターンのSQLクエリまたはプランと同等のものを含む派生テーブル（`RTE_SUBQUERY`）に変換されます。

内部的には、関数`convert_cypher_to_subquery`がこの置き換えを実行します。これは、解析されたCypher AST（ステップ1から）を取り、それをPostgresクエリ構造（Cypherのセマンティクスをエミュレートするために必要な`SELECT/FROM/WHERE`で構成）に変換し、それをサブクエリ`RangeTblEntry`として外部クエリに添付します。この段階では、Cypherクエリは基本的にPostgreSQLが計画できる形式に変換されます：例えば、Cypherの`MATCH ... RETURN ...`は、基礎となる頂点とエッジテーブルを結合する`SELECT`になるかもしれませんし、Cypherの`CREATE`句は、特別なセット返し関数やダミースキャンによって表現されるかもしれません。特筆すべきは、`cypher()`関数自体は「プレースホルダーであり、実際には実行されない」ことです。これはパーサーにCypherクエリ文字列を運ぶためだけに存在し、分析中に削除されます。この設計は、Cypher関数の引数（グラフ名、クエリ文字列）が定数でなければならず、準備されたステートメントのパラメーターにはなり得ない理由です（実行フェーズの前に知られ、展開される必要があります）。

3. グラフ操作の計画とプランノードの生成

CypherがPostgreSQLのクエリツリーに変換されると（カスタムノードやサブクエリがある場合もあります）、PostgreSQLのプランナー/オプティマイザーが引き継ぎます。Apache AGEは、この段階で実行計画のグラフ固有の操作を処理するために統合します。特に、AGEは、グラフパターンマッチングとグラフデータの変更（頂点/エッジの作成または削除）を効率的に行う方法を計画する必要があります。この拡張機能は、PostgreSQLのプランナーフックとカスタムプランノードAPIを使用して、Cypher節のための独自の計画ロジックを注入します。

ここでの主なメカニズムは、プランナーフック`set_rel_pathlist_hook`で、これはAGEが[src/backend/optimizer/cypher_paths.c](https://github.com/apache/age/blob/master/src/backend/optimizer/cypher_paths.c)で設定します。このフックにより、AGEは計画中に各リレーション（サブクエリまたは関数結果）を調査することができます。プランナーがステップ2でCypher節のために生成された特別なサブクエリまたは関数ノードに遭遇すると、AGEのフックはそれを認識します（例えば、`CREATE ()`節または`_cypher_delete_clause`関数のプレースホルダー）そして、そのグラフ操作に特化したカスタムパスで通常の計画パスを置き換えます。例えば、Cypherの`DELETE`節の場合、AGEは、グラフ要素の削除を表す`CustomPath`（[src/backend/optimizer/cypher_pathnode.c](https://github.com/apache/age/blob/master/src/backend/optimizer/cypher_pathnode.c)で定義）を提供します。同様に、`CREATE`節の場合、新しい頂点/エッジを挿入するための`CustomPath`を提供します。これらのカスタムパスノードは、必要なデータ（作成または削除するパターンなど）をカスタムプライベートフィールド経由で持ち運びます。関連するコードファイルには、[cypher_pathnode.c](https://github.com/apache/age/blob/master/src/backend/optimizer/cypher_pathnode.c)（各種のCypher節のためのこれらの`CustomPath`オブジェクトの構築方法を定義）と、[cypher_createplan.c](https://github.com/apache/age/blob/master/src/backend/optimizer/cypher_createplan.c)（後でこれらのパスを実行可能なプランノード、つまり`CustomScan`ノードに変換）が含まれます。Apache AGEは基本的に、プランナーを拡張して「一部のグラフ操作を理解」し、それらのためのプランノードを生成することで、すべてを一般的なSQLとして扱うのではなく、特定の操作を行います。通常のCypherパターンマッチング（例えば、`MATCH`のための`JOIN`ロジック）は、変換されたSQL（PostgreSQLの既存の結合プランナーを利用）によって大部分が処理されますが、グラフを変更する節（`CREATE`、`MERGE`、`DELETE`、`SET`）については、AGEのプランナーフックがカスタムプランノードを注入して、それらのアクションを適切に実行できるようにします。

4. 計画とグラフ操作の実行

実行計画が確定すると（グラフ操作のための一部のカスタムプランノードと通常のSQL部分のための標準プランノードを含む）、PostgreSQLのエグゼキュータがクエリを実行します。Apache AGEは、この段階でグラフ関連のプランノードのためのカスタムエグゼキュータメソッドを提供することで統合します。具体的には、ステップ3で導入された各カスタムプランノードは、[src/backend/executor/cypher_create.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_create.c)、[cypher_delete.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_delete.c)、[cypher_merge.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_merge.c)、[cypher_set.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_set.c)などのファイルに対応する実行コードを持っています。これらのファイルは、それぞれの操作に対して`CustomScan`インターフェースのコールバック（`CustomExecMethods`構造体を介して）を実装しています。これには、カスタムスキャンの初期化、実行、終了のための関数が含まれます。例えば、「Cypher Create」プランノードは、[cypher_create.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_create.c)の`begin_cypher_create`、`exec_cypher_create`、`end_cypher_create`関数を使用して、実行中に新しい頂点またはエッジの作成を実際に行います。エグゼキューターがプランを実行すると、カスタムノード上で`ExecProcNode`が呼び出され、これがAGEの`exec_cypher_*`関数を呼び出します。これらの関数は通常、子ノードから必要な入力タプルを引き出します（例えば、`MATCH`パターンに基づく`CREATE`の場合、先行するマッチ結果は`ExecProcNode(lefttree)`を介して引き出されます）そして、グラフ操作を実行します。

この段階で、Apache AGEはPostgreSQLの標準的なヒープタプルの挿入と削除ルーチンを呼び出し、基礎となるストレージを変更します。AGEはグラフデータをグラフごとに2つのテーブル（ノード用とエッジ用、通常はグラフのスキーマ内で`_ag_label_vertex`と`_ag_label_edge`と名付けられます）に保存します。実行関数はこれらのテーブルにレコードを挿入または削除して、グラフへの変更を反映します。カスタムプランノードは通常のエグゼキューターに統合されているため、クエリ内で適切な順序でスケジュールされ、実行されます。したがって、AGEのExecutorコンポーネントは「実行プランに基づいてデータを取得または変更するために、PostgreSQLデータベースエンジンと直接対話します」。

5. PostgreSQLのACIDトランザクションとの統合

Apache AGEの設計の大きな利点は、CypherクエリがPostgreSQLのACID準拠のトランザクションシステムに完全に参加していることです。AGEは拡張機能（フォークではない）であるため、グラフデータのための別のストレージやトランザクションマネージャを実装していません - 代わりに、すべてのグラフデータはPostgreSQLのテーブルに保存され、すべての読み書き操作はPostgreSQLの標準的なアクセス方法を経由します。これは、グラフの更新（挿入、削除、プロパティの更新）がPostgreSQLのWAL（Write-Ahead Log）に記録され、ロックプロトコルを尊重し、他のSQL作業と一緒にコミットまたはロールバックできることを意味します。ソースコードでは、このステップのための単一の「トランザクションマネージャ」ファイルは存在せず、トランザクションの統合は暗黙的です。例えば、エグゼキューターの`exec_cypher_create`関数が新しいノードを挿入するとき、それは最終的にPostgreSQLのヒープ挿入ルーチンを呼び出します（[cypher_utils.c](https://github.com/apache/age/blob/master/src/backend/executor/cypher_utils.c)内の`insert_entity_tuple`のようなヘルパーを介して）、これは新しいタプルを割り当て、OIDを割り当て、変更をメモリに記録します - すべて現在のトランザクションの下で。周囲のSQLトランザクションがコミットすると、それらの変更が耐久性を持つようになります（WALフラッシュを介して）；トランザクションが中止されると、すべての挿入されたノード/エッジはPostgreSQLによって自動的にロールバックされます。簡単に言えば、Apache AGEは「Postgresの既存の完全なトランザクションシステム（ACID）」を活用して、グラフクエリが通常のSQLと同じ信頼性と原子性を持つようにします。

[前: Enablement](02_enablement_ja.md) | [次: Objects](04_objects_ja.md)
