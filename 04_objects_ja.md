# オブジェクト

AGEは以下のオブジェクトを提供します。ここでは、重要な項目のみを説明します。

## テーブル

```sql
table ag_catalog.ag_graph
table ag_catalog.ag_label
```

### `ag_graph`

このテーブルはグラフの情報を保持します。

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

例えば、以下のクエリの結果は、データベースに6つのグラフがあることを示しています。

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
このテーブルは各グラフのメタデータを保持します。

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

以下のクエリ結果は、グラフ`"Airroute"`が`"Airport"`とラベル付けされた頂点（ノード）と`"ROUTE"`とラベル付けされたエッジを持っていることを示しています。リレーション`_ag_label_vertex`はリレーション`"Airroute"."Airport"`の親であり、`_ag_label_edge`は`"Airroute"."ROUTE"`の親です。言い換えると、`"Airroute"."Airport"`は`_ag_label_vertex`から継承します。

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

実際のデータは各リレーションに格納されています。

ノードの場合：
```sql
ELECT * FROM "Airroute"."Airport" LIMIT 2;
       id        |                                                                                      properties
-----------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 844424930131969 | {"city": "アトランタ", "code": "ATL", "name": "ハーツフィールド・ジャクソン・アトランタ国際空港", "country": "USA", "latitude": 33.6367, "longitude": -84.4281, "passengers": 107394029}
 844424930131970 | {"city": "ロサンゼルス", "code": "LAX", "name": "ロサンゼルス国際空港", "country": "USA", "latitude": 33.9416, "longitude": -118.4085, "passengers": 88068013}
(2行)
```

エッジの場合:
```sql
SELECT * FROM "Airroute"."ROUTE" LIMIT 1;
        id        |    start_id     |     end_id      |                         properties
------------------+-----------------+-----------------+-------------------------------------------------------------
 1125899906842625 | 844424930131969 | 844424930131970 | {"distance": 3100, "flight_time": 300, "daily_flights": 35}
(1行)
```

### なぜ `id` は非常に大きいのですか？

`graphid` タイプは [int64として定義されています](https://github.com/apache/age/blob/c75d9e477e2f83016a7a144291d2c15038cf229f/src/include/utils/graphid.h#L29)。

`graphid` は [graphid.c](https://github.com/apache/age/blob/c75d9e477e2f83016a7a144291d2c15038cf229f/src/backend/utils/adt/graphid.c#L200) の `make_graphid()` 関数によって作成されます。

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

これは、上位16ビットを `label_id` の保存に、下位48ビットを `entry_id` の保存に使用します。

例えば、ノードテーブル（例："Airroute"."Airport"）の `id` が3（名前空間内のOID）の場合、"Airport"というラベルのノードの最初のIDは常に844424930131969（計算式：3 << 48 + 1）となります。

## タイプ

```sql
type ag_catalog.agtype
type ag_catalog.graphid
type ag_catalog.label_id
type ag_catalog.label_kind
```

### agtypeの概要

Apache AGEは、グラフクエリ結果を表現するためのカスタムデータタイプとして**agtype**を導入します。これは基本的にJSONのスーパーセットで、PostgreSQLの`jsonb`形式と非常に似た構造で実装されています。実際には、`agtype`は基本的なJSON値（null、boolean、numeric、string、arrays、objects）だけでなく、頂点（ノード）、エッジ、パスなどの**グラフ特有のタイプ**もサポートするために`jsonb`を拡張します。AGEのすべてのクエリ結果は`agtype`として返されるため、この拡張機能の中心的なデータタイプとなります。

内部的には、`agtype`は**varlena（可変長）**データとして保存され、最初の4バイトが全体のサイズを示します。その後に続く内容は、バイナリJSONのようなツリー構造を使用します。`jsonb`と同様に、`agtype`はその内容を高速なアクセスと圧縮を可能にする単一の連続した構造に組織します。各値は、小さな固定サイズのヘッダー（**agtentry**と呼ばれる）とその実際のデータペイロードからなるノードとして保存されます。オブジェクトや配列のような複雑な値の場合、最上部のコンテナヘッダーが要素の数とタイプ（オブジェクト対配列）を記述します。この設計により、`jsonb`と同様に効率的な走査とルックアップが可能です。主な違いは、`agtype`がこの構造を拡張してCypher/グラフ値タイプを追加的に処理し、実証済みの`jsonb`レイアウトを再利用することです。

### `agtype.h`の中のコア構造
[agtype.h](https://github.com/apache/age/blob/master/src/include/utils/agtype.h)には、`agtype`のディスク上およびメモリ内表現を定義するいくつかのC構造体があります。以下は、主要な構造体とその役割の概要です：

#### agtype_container – 配列/オブジェクトのディスク上のコンテナ

`agtype_container`は、JSON配列またはオブジェクトをバイナリ形式で表現します（これはPostgreSQLの`JsonbContainer`に密接に対応しています）。これにはヘッダと子要素のエントリの配列が含まれています：
  - **ヘッダ（uint32 header）** – コンテナ内の要素（またはキー-値ペア）の数と、コンテナタイプを示すフラグビットを格納します。下位28ビットがカウントを保持し、上位4ビットがフラグです。例えば、このコンテナがオブジェクトか配列かを示すフラグがあります。具体的には、`AGT_FOBJECT`と`AGT_FARRAY`は、それぞれオブジェクトと配列のヘッダに設定されます。また、`AGT_FSCALAR`フラグは、スカラコンテナを示すために使用されます。これは、単独のスカラがコンテナとして格納される特殊なケースです（これは`jsonb`がトップレベルのスカラを表現する方法を反映しています）。
	-	**子供たち（agtentry children[]）** – コンテナ内の各子要素の**エントリヘッダ**（`agtentry`型）の柔軟な配列です。各`agtentry`は、1つの子要素のタイプとコンテナ内の長さ/オフセットを記述する32ビットのメタデータ値です。すべての子供たちの実際のデータ（例えば、文字列の内容、数値のバイト、サブコンテナなど）は、この`agtentry`ヘッダの配列の後に順次格納されます。このメタデータとデータの分離は、空間効率を向上させ、キーまたは要素のスキャン時のキャッシュの局所性を向上させるために行われます。

メモリ内の`children`配列の後には、各子ノードの可変長コンテンツを含む**データ領域**が続きます。コンテナがオブジェクトである場合、子エントリはすべてのキーが最初に来るように（辞書順にソートされて）並べられ、その後にすべての値が続きます。これは`jsonb`から継承されており、キーの検索を高速化します（キーが連続していてソートされているため）が、ストレージ内での元の挿入順序を失うコストがかかります。ただし、`agtype`はオブジェクトフィールドの元の挿入順序を別の構造体で追跡します（これについては`agtype_pair`で見ていきます）。コンテナが配列である場合、エントリは順序に従った要素に対応します。コンテナのヘッダフラグはオブジェクトと配列を区別します。

**フラグとマクロ**：`agtype_container`のフラグはマクロでアクセスできます。例えば、`AGTYPE_CONTAINER_IS_OBJECT(container)`はオブジェクトフラグをチェックします。同様に、`AGTYPE_CONTAINER_SIZE(container)`は子供の数（オブジェクトのキーはこの合計で1つとしてカウントされます）を提供します。この設計により、`agtype`値のルートは常にコンテナであるか、スカラコンテナとして扱われます。ルート値のための別のエントリヘッダは格納されません。なぜなら、コンテナヘッダ自体がそれがオブジェクトか配列かを教えてくれるからです。

#### agtentry – 値のメタデータエントリ

コンテナ内の各要素またはフィールドは、`agtentry`（`jsonb`の`JEntry`に相当）によって記述されます。`agtentry`は、値のタイプとその長さまたはオフセットの両方をエンコードする4バイトの値です。`agtentry`のレイアウトと定数は次のとおりです：
  - **タイプビット（上位3ビット）** – エントリが表現するデータの種類を示します。`agtype`はいくつかのタイプを定義します：文字列、数値、ブール値（真/偽は別々のコードを持つ）、null、コンテナ（ネストされた配列/オブジェクト用）、またはグラフ固有または拡張タイプの特別な**「拡張agtype」**タイプです。例えば、`AGTENTRY_IS_STRING (0x00000000)`、`AGTENTRY_IS_NUMERIC (0x10000000)`、`AGTENTRY_IS_BOOL_FALSE (0x20000000)`、`AGTENTRY_IS_BOOL_TRUE (0x30000000)`、`AGTENTRY_IS_NULL (0x40000000)`、そして`AGTENTRY_IS_CONTAINER (0x50000000)`は基本的なJSONタイプをカバーしています。さらに、`AGTENTRY_IS_AGTYPE (0x70000000)`は、AGEによって導入された拡張タイプのタイプ指定子です。
  -	**HAS_OFFフラグ（1ビット）** – 次のビット（4番目に高いビット）は、残りの28ビットがどのように解釈されるべきかを示すフラグです。`AGTENTRY_HAS_OFF (0x80000000)`が設定されている場合、下位28ビットはデータ領域の開始からこの値の内容の開始までの**オフセット**を格納します。フラグが設定されていない場合、下位28ビットは値の内容の**長さ**を格納します。このスキームは、直接アドレッシングと圧縮のバランスを取るために使用されます：通常、ほとんどのエントリはより良い圧縮のために長さを格納しますが、定期的にエントリは絶対オフセットを格納して位置を計算することを可能にします（これはPostgreSQLのJSONB戦略に従ってTOAST圧縮効率を追求します）。
	-	**長さ/オフセット（下位28ビット）** – 上記のフラグにより、このフィールドは値のデータのバイト長さ、またはコンテナのデータ開始位置に対する値のデータの終了位置（オフセット+1）のいずれかです。いずれの場合でも、値のバイトを取得することができます。大きな値や間隔位置で発生する値はオフセットを持ち、他の値は長さを持つことで、アクセスパターンを最適化します。

拡張型（`AGTENTRY_IS_AGTYPE`）の場合、エントリのデータは特定の値のサブタイプを識別するための**AGタイプヘッダー**でプレフィックスされます。AGEはこの目的のためにいくつかの`AGT_HEADER_*`定数を定義します。拡張エントリのデータの最初の4バイトはこれらのヘッダーの1つで、その後のバイトの解釈方法を示します。定義されたヘッダーには、`AGT_HEADER_INTEGER (0x00000000)`、`AGT_HEADER_FLOAT (0x00000001)`、`AGT_HEADER_VERTEX (0x00000002)`、`AGT_HEADER_EDGE (0x00000003)`、`AGT_HEADER_PATH (0x00000004)`が含まれます。つまり、`agtentry`のタイプが「AGTYPE」の場合、内容は「これは64ビット整数です」（または浮動小数点数、頂点など）という4バイトのタグで始まり、その後に実際の値データ（例えば、8バイトの整数データ、または頂点のためのネストされたコンテナ）が続きます。このメカニズムにより、`agtype`はグラフエンティティと新しいスカラータイプをJSONと同じコンテナフレームワークでエンコードすることができます。

#### agtype – トップレベルのデータ構造

`agtype`構造体自体は、PostgreSQLのタプルに格納されるトップレベルのvarlenaオブジェクトを表します。メモリ内では以下のように定義されています：

```c
typedef struct {
    int32 vl_len_;         /* varlena header (total size) */
    agtype_container root;
} agtype;
```

これは単に4バイトの長さヘッダーの後に`agtype_container`（ルートコンテナ）を埋め込むだけです。ルートはオブジェクトまたは配列のコンテナであり、実際には単一のスカラーを保持している場合にはスカラーコンテナとしてマークされる可能性があります。`AGT_ROOT_IS_OBJECT(ag)`や`AGT_ROOT_COUNT(ag)`のようなユーティリティマクロが提供されており、ルートのヘッダーフラグとカウントを調査します。本質的に、ディスク上またはメモリ内の`agtype`データのバイトは、長さヘッダーの後に直接`agtype_container`構造体（または拡張エントリ、スカラーが格納されている場合）に対応します。これはPostgreSQLの`jsonb`（`Jsonb`構造体が`JsonbContainer`ルートを含む）のレイアウトを反映しています。

#### agtype_value – メモリ内でデシリアライズされた値

`agtype_container`と`agtentry`がコンパクトなバイナリ形式を管理する一方で、`agtype_value`は`agtype`コンテンツの便利なメモリ内操作（PostgreSQLの`JsonbValue`に類似）に使用されます。`agtype_value`は、任意の`agtype`値を分解形式で表現できるC構造体であり、手動のバイトオフセットなしで値を構築または調査するのが容易になります。以下のように定義されています：

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

この構造体では、typeフィールドは、ユニオンのどのメンバーが有効であるかをタグ付けする**enum agtype_value_type**です。可能なタイプの値には、すべてのJSONタイプと拡張されたAGEタイプが含まれます：
  -	**スカラー**：`AGTV_NULL`、`AGTV_BOOL`、`AGTV_INTEGER`、`AGTV_FLOAT`、`AGTV_NUMERIC`、`AGTV_STRING`は、プリミティブタイプをカバーします。（ここで`AGTV_BOOL`は、真偽値フィールドが実際の値を格納するとともに、true/falseの両方に使用されます。）特筆すべきは、`AGTV_INTEGER`と`AGTV_FLOAT`は、AGEの新しいスカラータイプであり（バニラのJSONBでは、任意の数値に`AGTV_NUMERIC`を使用します）。これらは、Cypherの64ビット整数と倍精度浮動小数点数タイプに対応します。列挙型の値は、すべてのスカラータイプ（これらの新しいものを含む）が連続した範囲になるように順序付けられており、これはソートや型チェックマクロにとって重要です。
  - **グラフエンティティタイプ**：`AGTV_VERTEX`、`AGTV_EDGE`、`AGTV_PATH`は、ノード、エッジ、パスの値を表す追加のタイプです。内部的には、これらのタイプの値は通常、複合構造体として格納されます（例えば、頂点はid、ラベル、プロパティマップを含む）。メモリ内の`agtype_value`では、頂点やエッジは、オブジェクトのプロパティとして見ることができるノードやエッジを表すために、オブジェクトのユニオンメンバーを使用するか、ディスク上のフォーマットを指すバイナリメンバーを使用する可能性があります。タイプタグ`AGTV_VERTEX`（またはEDGE/PATH）は、それを一般的なオブジェクトと区別します。パスは、頂点とエッジの順序付きリストとして見ることができるため、配列メンバーを介して表現されるか、専用のパス構造を介して表現されるかもしれません。いずれにせよ、これらの列挙型の値の存在は、システムがこれらを比較や処理の中で第一級のタイプとして扱うことができることを意味します。
  - **複合タイプ**：`AGTV_ARRAY`と`AGTV_OBJECT`は、`agtype_value`が配列またはオブジェクトを値として保持していることを示します（デシリアライズされた）。これらの場合、配列またはオブジェクトのユニオンメンバーが使用され、サブ値へのポインタを含みます。例えば、`type == AGTV_ARRAY`の場合、`val.array.num_elems`と`val.array.elems`が要素を記述します。配列構造体の`raw_scalar`フラグは、この配列が「生のスカラー」（孤立したスカラーを配列として表現する特別なケース）を表している場合に真となります。`type == AGTV_OBJECT`の場合、`val.object.num_pairs`と`val.object.pairs`が設定され、`agtype_pair`（以下で説明）が各キーと値を保持します。
  - **バイナリコンテナ**：`AGTV_BINARY`は、値がすでにバイナリ形式（`agtype_container`）であり、完全にデシリアライズせずに値として扱いたい場合に使用される特別なタイプです。この場合、`val.binary`メンバーは`agtype_container`とその長さを指します。これは、パースを遅延させるために便利です - 例えば、関数がサブオブジェクトを取得し、それを渡すか出力するだけであれば、バイナリ表現を直接使用できます。`AGTV_BINARY`タイプは、基本的にディスクフォーマットの配列やオブジェクトを`agtype_value`内にラップします。（これは、JSONBの`JsonbValue`の`jbvBinary`に類似しています。）

`agtype_value`構造体は、プログラム的に複雑な値を構築するのを容易にします。例えば、新しい頂点を構築するときには、そのプロパティマップのために`AGTV_OBJECT`タイプの`agtype_value`を割り当てるか、プロパティがすでにコンテナ形式である場合は`AGTV_BINARY`を使用し、それを上位レベルの`AGTV_VERTEX` `agtype_value`でラップするかもしれません。最終的に、`agtype_value_to_agtype()`のような関数は、メモリ内の表現をコンパクトな`agtype`コンテナ形式に戻すことができます。

#### agtype_pair – オブジェクトの構築用キー・バリューペア

オブジェクト（マップ）値をメモリ内で構築するとき、AGEは`agtype_pair`構造体を使用して、シリアライズ前の各キーと値のペアを保持します。それは以下のように定義されています：

```c
struct agtype_pair {
    agtype_value key;    /* must be AGTV_STRING type */
    agtype_value value;  /* can be any type */
    uint32      order;   /* original insertion order index */
};
```

これはディスク上の構造ではなく、`AGTV_OBJECT`値を形成する際に一時的に使用されます。キーは文字列であることが期待される`agtype_value`で、値は任意のタイプの`agtype_value`です。orderフィールドは、元の入力シーケンスでのペアのインデックスを記録します。これは、同じキーが複数回出現する場合、システムがどの出現を保持するかを選択できるため重要です。AGE（およびJSONB）では、重複するキーに対するルールは「最後に観察された値が勝つ」です。シリアライゼーション中に、キーはストレージのためにソートされますが、この順序は入力がどのように順序付けられたかを覚えていることで重複を決定するのに役立ちます。処理後、必要に応じて重複するキーが削除され、最後の出現だけが保存されます。

#### agtype_parse_state – パース/ビルド状態（内部）

`agtype_parse_state`は、JSONをパースしたり、`agtype`値を段階的に構築する際に使用される内部スタックフレームです（`JsonbParseState`に似ています）。これは部分的な`cont_val`（構築中のコンテナ値）、現在の子の数、および親状態へのポインタを保持します。また、最後に更新された値へのポインタを追跡し、これはネストした構造を構築する際の特定のキャストや変更シナリオで使用されます。この構造は実装にとって重要ですが、ユーザーに直接公開されるものではなく、`push_agtype_value()`のような関数を介して`agtype`値の段階的な組み立てを可能にします。

#### agtype_iterator – agtypeを順次読み取る

`agtype`値を走査するために（例えば、テキストとして出力するか、包含クエリを評価するために）、AGEはイテレータインターフェースを提供します。`agtype_iterator`構造体は、クライアントがその生のレイアウトを公開することなく、複雑な`agtype`構造を通過することを可能にします。内部的に、`agtype_iterator`には以下が含まれます：
  - 現在反復処理されている`agtype_container`へのポインタ（コンテナ），
  - その中の要素の数（`num_elems`）と現在のインデックス，
  - コンテナの`agtentry`配列（`children`）へのポインタとデータ領域（`data_proper`）へのポインタ，
  - スカラーコンテナケースを処理するための`is_scalar`のようなフラグ，
  - 現在の要素のデータオフセットの追跡（オブジェクト値の`curr_data_offset`と`curr_value_offset`），
  - 配列要素、オブジェクトキー、オブジェクト値など、どこにいるかを示す`agt_iterator_state`という状態列挙型，
  - ネストした構造体の親イテレータノードへのリンク（イテレータがサブコンテナを終了した後に巻き戻すことができるように）。

イテレータは`agtype_iterator_next()`を呼び出すことで進められ、これは何のタイプのアイテムが遭遇したかを示す`agtype_iterator_token`を返します（オブジェクトの開始、キー、値、配列の終了など）。これらのトークン（例えば、`WAGT_BEGIN_OBJECT`、`WAGT_KEY`、`WAGT_VALUE`、`WAGT_END_OBJECT`など）は、呼び出し元が階層構造を再構築するのを可能にします。この設計はJSONBイテレータに直接モデル化されており、`agtype`をJSONテキストとして出力したり、クエリ操作のためにそれをナビゲートするのを容易にします。イテレータは拡張型も理解しています - 例えば、頂点を持つ`AGTENTRY_IS_AGTYPE`エントリに遭遇した場合、それはおそらく値トークンとして返され、`agtype_value`が生成するタイプ`AGTV_VERTEX`を調べることができます。

### PostgreSQL jsonbとの比較

`agtype`は`jsonb`の拡張として構築されているため、PostgreSQLのJSONBタイプと多くの内部特性を共有していますが、グラフデータをサポートするために導入された重要な違いがあります。以下に、その構造と機能性の比較を示します：
  - **全体構造（Varlenaとコンテナ）**：`jsonb`と`agtype`の両方で、データは単一のvarlena blobにコンテナヘッダと一連のエントリに続くデータとともに格納されます。ヘッダにはカウントとタイプフラグ（オブジェクト/配列/スカラー）があり、固定サイズのメタデータエントリの配列が共通しています。実際、`agtype_container`と`JsonbContainer`はレイアウトと目的が同じです。これは、`agtype`が`jsonb`が使用する効率的なパースとインデックス付きアクセス技術（キーのバイナリ検索、配列要素の高速スキップなど）の恩恵を受けることを意味します。また、エントリのオフセット/長さ戦略が似ているため、ストレージはtoast圧縮可能です。異なるのは、エンコードできる値のタイプとそのエンコーディングの詳細です。
  - **タイプカバレッジ – JSONプリミティブ vs グラフエンティティ**：PostgreSQLの`jsonb`はJSONプリミティブタイプをサポートしています：null、ブーリアン、テキスト文字列、数値（JSONBではすべてが一貫性のためにPostgreSQL Numericタイプを使用して格納されます）、および複合タイプ：配列とオブジェクト。Apache AGEの`agtype`はこれらすべてを含みますが、グラフ特有のデータをカバーするためにタイプシステムを拡張します：
    - **頂点（ノード）**：`agtype`では`AGTV_VERTEX`として表現されます。頂点値にはグラフ識別子（通常は64ビットのid）、ラベル、プロパティセット（キー/値マップ）が含まれます。JSONBでは、頂点を一般的なJSONオブジェクト（例：`{"id": ..., "label": ..., "properties": {...}}`）としてしか表現できません。`agtype`では、頂点は第一級：それらは内部的に明確なタイプタグを持ちます。これにより、AGEは値を頂点として認識し、例えば特定の形式で印刷したり、グラフのアイデンティティに関して等価性を処理したりすることができます。内部的には、頂点はおそらくそのフィールドを格納するためにオブジェクトコンテナを使用していますが、バイナリ形式では`AGT_HEADER_VERTEX`タグでプレフィックスされているため、システムはそのオブジェクトがグラフ頂点を表していることを知っています。
    - **エッジ**：`agtype`では`AGTV_EDGE`として表現されます。エッジにはid、ラベル（関係のタイプ）、開始頂点idと終了頂点id、および可能ならプロパティが含まれます。頂点と同様に、`agtype`のエッジはタイプで区別され、`AGT_HEADER_EDGE`マーカーで格納されます。生のJSONBでは、エッジは再びそれらのフィールドを持つ単なるオブジェクトに過ぎず、組み込みの認識はありません。
    - **パス**：`AGTV_PATH`として表現されます。パスは基本的に接続された頂点とエッジの順序付けられたシーケンスです（例えば、Cypherクエリの可変長パスマッチの結果）。AGEはパスを単一の`agtype`値としてエンコードし、独自のタイプタグを持つことができます。おそらく、パスは頂点とエッジのコンポーネントのリスト/配列、または内部的に特殊な構造として格納されますが、いずれにせよ`jsonb`には直接的な同等物はありません（JSONBではおそらくオブジェクトの配列になるでしょう）。
    - **数値タイプ**：`jsonb`は整数、浮動小数点数、または小数の間で区別しません - 任意のJSON数値はNumericとして格納されます。`agtype`はここでCypherのタイプシステムに合わせて分岐します。それは提供します：
      - `AGTV_INTEGER`は64ビット整数値用です。
      - `AGTV_FLOAT`は倍精度浮動小数点値用です。
      - `AGTV_NUMERIC`は任意精度の数値（PostgreSQLのNumericタイプを使用）用です。
これは、`agtype`が小さな整数や浮動小数点数をより直接的かつ効率的に保存できることを意味します。例えば、整数値42は、`AGT_HEADER_INTEGER`タグと8バイトの2の補数表現で保存され、Numeric varlenaに変換されることはありません。これにより、パフォーマンスが向上し、型の忠実性が保たれます（intとfloatの間で意図しない変換が行われることはありません）。対照的に、JSONBは42や4.2をNumeric形式に変換します（これは大きく、処理が遅くなる可能性があります）。トレードオフとして、`agtype`は3つの数値型を処理するロジックを維持しなければならないのに対し、JSONBは1つだけです。しかし、ユーザーの視点から見れば、これはCypherの期待に合致しています：整数は整数のまま、浮動小数点数は浮動小数点数のまま、そして非常に大きな数値や高精度の数値はNumericにフォールバックできます。これら3つはすべてテキストJSONにシリアライズ可能です（出力は同じに見えます） - これは純粋に内部的な区別です。
  - **内部表現とエンコーディング**：標準的なJSONデータについては、`agtype`は`jsonb`と同じバイナリエンコーディングを使用します。`agtype`内の文字列は、`jsonb`内のものと同じ形式で保存されます（型コードと長さ、その後に文字が続き、null終端されません）。真偽値とnullは、`jsonb`と同様に特別な`agtentry`コードとして保存されます（ヘッダー以外に追加のペイロードバイトはありません）。`agtype`内の配列とオブジェクトは、`jsonb`内のものと同じコンテナ構造と順序付けルールを持っています。この共通性により、多くの`jsonb`操作（包含チェック、ハッシュ化、反復など）は、最小限の変更で`agtype`に適応できます。実際、AGEはPostgreSQLの`jsonb`内部の多くを再利用しています：例えば、配列やオブジェクトを反復処理するロジック、またはGINインデックスエントリを作成するロジックは、新しい型を考慮に入れて大幅に同じです。`agtype_iterator`と`agtype_parse_state`は、それぞれのJSONBの対応物と密接に並行しており、`agtype.c`内のシリアライズ/デシリアライズコードの多くは、PostgreSQLの実装に基づいています。
  - **グラフ固有の追加と拡張性**：`agtype`の主な拡張はグラフモデルをサービスしますが、それらが実装された方法は、必要に応じてさらなる型を追加する余地を残しています。`AGTENTRY_IS_AGTYPE`エントリタイプと小さなヘッダーを使用して新しいサブタイプを区別することで、設計者は`agtype`を拡張可能にしました。現在のセット（整数、浮動小数点数、頂点、エッジ、パス）は、openCypherが必要とするものをカバーしています。将来のグラフ機能が新しい種類の値を必要とする場合（例えば、空間型やカスタムグラフ値など）、理論的には別のヘッダーコードを使用して全体の形式を再設計することなく追加できます。JSONBだけでは、このような追加の型の概念はありません - JSONのドメインに限定されています。拡張性の観点から、`agtype`は、JSONBの基盤を新しいドメイン（グラフデータ）に拡張しながら、データ形式の一般的なモデルを保持する方法を示しています。
  - **JSONBとの互換性**: 機能的には、`agtype`は`jsonb`のスーパーセットと見ることができます - 任意の有効なJSONBデータは`agtype`として表現することができます（実際には共通部分については同じバイナリ形式を使用します）。しかし、グラフ型の存在により、すべての`agtype`データが有効な`jsonb`であるわけではありません。例えば、純粋なJSONオブジェクトや配列が`agtype`に格納されている場合、同じフラグと構造を使用するため、AGEの値である['hello', 123]（文字列と整数の配列）は、`jsonb`が["hello", 123]を格納する方法とバイナリ互換性があります（ただし、123はNumericの代わりに`AGTV_INTEGER`として格納されるかもしれません）。しかし、頂点やエッジを含む値は`jsonb`に相当するものがなく、それを`jsonb`として解釈しようとすると、`AGTENTRY_IS_AGTYPE`型コードは認識されません。したがって、`jsonb`と`agtype`の間の直接的なキャストは、テキストを介してシリアライズ/デシリアライズするか、それらの拡張型を明示的に処理する必要があります。内部的には、AGEは`JsonbContainer`のロジックを再利用していますが、未使用の型コードスペースを新たな目的に割り当てることでストレージフォーマットを分岐させています。これは、`agtype`が一般的に`jsonb`とバイナリ互換性がないことを意味します。それは並行するフォーマットです。それは言われて、インデックスのサポートと演算子のロジック（例えば、キーの存在、構造の包含）の多くは`jsonb`から借りています。AGEの`agtype`のGINインデックスは、JSONBと同様にキーと値をインデックス化します（存在演算子をサポートするために、文字列配列の要素がキーであるかのようにインデックス化されるトリックを含む）。`agtype`の演算子、例えば`->`（フィールドアクセス）、`@>`（含む）、などは、新しい型を適切に処理するように拡張された形で、それらのJSONBの対応物と同様に実装されています。

## キャスト

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

## 関数

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

`create_graph`はApache AGEでグラフを作成するための初期関数です。引数は`name`のみです。対照的に、`drop_graph`は2つ目の引数を取ります。これは削除をカスケードするかどうかを示すフラグです。上記のように、グラフは`ag_graph`のエントリ、`SERIAL`シーケンス、それに関連するノード、エッジ、およびそのシーケンスで構成されます。グラフの削除には、これらのメタデータ要素の**カスケード**削除が必要です。

### create_vlabel(name,name) / create_elabel(name,name) / drop_label(name,name,boolean)

これらの関数は厳密には必要ではありません。ノードとエッジはCypherクエリを使用して作成および削除できます。

### cypher(name,cstring,ag_catalog.agtype)

`cypher`関数はApache AGEの中核です。[AGEの実装](03_implementation.md)で述べられているように、これはCypherクエリの**入口**です。

### load_labels_from_file(name,name,text,boolean) / load_edges_from_file(name,name,text)

これらの関数は、ラベル（ノード）とエッジをファイルからロードするために使用されます。引数は、グラフ名、ラベル名、ファイルパスです。`load_labels_from_file`には、ファイルから`id`フィールドを除外するかどうかを示すブール値フラグが4つ目の追加引数としてあります。

残念なことに、この2つの関数がファイルパスとして取り得る引数は**ローカル**のファイルであり、Azure Database for PostgreSQLでは利用することができません。

## 演算子

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

[前: 実装](03_implementation_ja.md) | [次: Cypher Query with AGE](05_cypher_ja.md)
