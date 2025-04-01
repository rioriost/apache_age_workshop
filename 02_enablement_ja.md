# AGEを有効にする方法

## AGEを有効にする手順

Azure Database for PostgreSQL Flexible Server（Elastic Clusterを含む）では、AGEはすでにインストールされていますが、拡張機能としてロードする必要があります。

PostgreSQLをデプロイした後、Azure Portalで`サーバーパラメータ`を選択し、以下のパラメータを2箇所で設定します：

1. azure.extensions

パラメータフィルタに**azure.extensions**を入力すると、Azure Database for PostgreSQLで利用可能なExtensionsが表示されます。`AGE`をチェックします。
![azure.extensions](images/02_param_01.png)

2. shared_preload_libraries

パラメータフィルタに**shared_preload_libraries**を入力すると、事前にロードできるExtensionsが表示されます。`AGE`をチェックします。
![azure.extensions](images/02_param_02.png)

3. 2つの項目をチェックした後、[保存]ボタンをクリックします。サーバーは再起動してAGE Extensionを有効にします。

4. 再起動が完了したら、`psql`インタープリタでPostgreSQLに接続します。接続方法は、Azure Portalで`接続`を選択すると表示されます。
![azure.extensions](images/02_connect_01.png)

5. PostgreSQLに接続したら、以下のコマンドを実行してAGE Extensionを有効にします。

```sql
CREATE EXTENSION IF NOT EXISTS AGE CASCADE;
```

`CREATE EXTENSION`が表示されたら、AGE Extensionが有効になっています。

## スキーマの設定

後述するように、AGEは`ag_catalog`というスキーマを追加し、このスキーマ内でグラフデータを扱います。したがって、`psql`やPythonなどのプログラミング言語でAGEのグラフデータを扱う場合、スキーマ検索パスに`ag_catalog`を追加する必要があります。

```sql
SET search_path=ag_catalog,"$user",public;
```

```python
import psycopg as pg
with pg.Connection.connect(con_str + " options='-c search_path=ag_catalog,\"$user\",public'") as con:
```

[前: イントロダクション](01_introduction_ja.md) | [次: AGEの実装](03_implementation_ja.md)
