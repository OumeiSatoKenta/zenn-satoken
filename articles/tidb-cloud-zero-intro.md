---
title: "TiDB Cloud Zero入門 — curlひとつで手に入るMySQL互換ベクトルDB"
emoji: "🚀"
type: "tech"
topics: ["tidb", "mysql", "vectordb", "rag", "python"]
published: false
---

## この記事について

- **AI記載率**: 約65% — Claude Codeで生成し、筆者が構成指示・編集・加筆
- **動作確認**: 記事内のコマンド・設定はすべて筆者が実行確認済み

## はじめに

RAGをはじめ、AIを活用したサービスではベクトルデータベースを使う機会が増えています。ただ、ちょっと試したいだけなのにDB環境の構築やアカウント登録が面倒で、手が止まることもあるのではないでしょうか。

**TiDB Cloud Zero** は、curl 1つでMySQL互換DBを作れるサービスです。認証不要・無料・30日間有効で、ベクトル検索にも対応しています。

この記事では、TiDB Cloud Zeroのインスタンスを作成し、TiDB公式Pythonクライアントのpytidbでベクトル検索を試してみます。

## 前提条件

- **アカウント作成不要**です。API Keyも不要です
- curlコマンドが使える環境（macOS / Linux / WSL）
- Python 3.10以上（ベクトル検索の実践に使用）

## TiDB Cloud Zeroとは

PingCAP社が提供するTiDB Cloudの無料インスタント環境です（[公式サイト](https://zero.tidbcloud.com/)）。

| 特徴 | 内容 |
|------|------|
| 認証 | **不要**（API Keyなし） |
| 料金 | **無料** |
| 有効期限 | 作成から**30日間**（自動失効） |
| プロトコル | **MySQL 8.0互換**（TiDB v8.5.3-serverless） |
| ベクトル検索 | **対応**（VECTOR型 + コサイン類似度） |
| 永続化 | claimUrlでTiDB Cloud Starterに変換可能 |

開発・検証用の使い捨て環境として使えます。

## curlでインスタンスを作成する

curl 1行でインスタンスを作成できます。

```bash
curl -s -X POST https://zero.tidbapi.com/v1alpha1/instances \
  -H "Content-Type: application/json" \
  -d '{"tag":"my-first-instance"}' | jq .
```

`tag` は用途を識別するための任意のラベルです。

### レスポンス例

```json
{
  "instance": {
    "id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "connection": {
      "host": "gateway03.us-west-2.prod.aws.tidbcloud.com",
      "port": 4000,
      "username": "zyx123abc.root",
      "password": "xxxxxxxxxxxxxxxx"
    },
    "expiresAt": "2026-04-06T16:19:16.657Z",
    "claimInfo": {
      "claimUrl": "https://tidbcloud.com/tidbs/claim/..."
    }
  }
}
```

`username` の `.root` の前（`zyx123abc`）はクラスタごとに発行される一意のプレフィックスです。TiDB Cloud Serverlessではこのプレフィックスでクラスタを識別します。なお、Zeroで作成されるインスタンスは `us-west-2`（オレゴン）リージョンに配置されます。

## mysqlコマンドで接続確認

TLS接続が必須なので `--ssl-mode=REQUIRED` を付けてください。

```bash
mysql -h gateway03.us-west-2.prod.aws.tidbcloud.com -P 4000 \
  -u 'zyx123abc.root' -p --ssl-mode=REQUIRED
```


## Python（pytidb）で接続する

pytidbはTiDB公式のPythonクライアントです。

```bash
pip install pytidb
```

```python
from pytidb import TiDBClient

db = TiDBClient.connect(
    host="gateway03.us-west-2.prod.aws.tidbcloud.com",
    port=4000,
    username="zyx123abc.root",
    password="YOUR_PASSWORD",
    database="test_db",
    enable_ssl=True,
    ensure_db=True,
)

print("Connected:", db is not None)
```


## ベクトル検索を試す

pytidbのauto-embeddingで、テキスト挿入だけでベクトル化とセマンティック検索ができます。

### スキーマ定義

```python
from pytidb.schema import TableModel, Field, FullTextField
from pytidb.embeddings import EmbeddingFunction

# TiDB Cloud組み込みのEmbeddingモデル（外部APIキー不要）
embed_fn = EmbeddingFunction("tidbcloud_free/amazon/titan-embed-text-v2")

class Chunk(TableModel):
    __tablename__ = "chunks"
    id: int = Field(primary_key=True)
    text: str = FullTextField()                                       # 全文検索インデックスを付与
    text_vec: list[float] = embed_fn.VectorField(source_field="text") # textから自動ベクトル化
```

`tidbcloud_free/amazon/titan-embed-text-v2` はTiDB Cloud組み込みの無料Embeddingモデルで、外部APIキー不要です。`source_field="text"` を指定すると、`text` の内容から自動的にベクトルを生成します。

### テーブル作成とデータ挿入

```python
# テーブル作成（既にある場合はスキップ）
table = db.create_table(schema=Chunk, if_exists="skip")

# データ挿入（auto-embedding: textから自動的にベクトル生成）
# id は auto-increment のため省略可能
table.bulk_insert([
    {"text": "TiDBは分散SQLデータベースです"},
    {"text": "MySQLと互換性のあるプロトコルを持っています"},
    {"text": "ベクトル検索にも対応しています"},
    {"text": "水平スケーリングが可能なHTAPデータベースです"},
    {"text": "PingCAP社が開発しています"},
])
```

`bulk_insert` でテキストを渡すだけで、`text_vec` にベクトルが自動保存されます。

### セマンティック検索

```python
results = table.search("分散データベースとは？").limit(5).to_list()

for r in results:
    # _distance はコサイン距離（0に近いほど類似度が高い）
    print(f"- {r['text']} (distance: {r['_distance']:.4f})")
```

`table.search()` はコサイン距離によるベクトル検索を実行します。`_distance` が0に近いほど類似度が高いです。

## claimUrlで永続化する

インスタンスは30日で失効しますが、レスポンスの `claimUrl` をブラウザで開くとTiDB Cloud Starter（無料枠あり）に変換でき、有効期限なしで使い続けられます。具体的な手順は[クラスメソッドさんの記事](https://dev.classmethod.jp/articles/tidb-cloud-zero-mysql-curl-mcp/)が詳しいです。

## ハマりポイントまとめ

:::message
### pytidbのAPI名がドキュメントと異なる

[公式README](https://github.com/pingcap/pytidb)の例と実際のAPIに差異があります（2026年3月時点）。

| ドキュメント等で見かける表記 | 実際のAPI |
|---|---|
| `ssl_verify=True` | `enable_ssl=True` |
| `mode="exist_ok"` | `if_exists="skip"` |
| `db.get_table("name")` | `db.open_table("name")` |

`inspect.signature()` で確認できます:

```python
import inspect
from pytidb.client import TiDBClient
print(inspect.signature(TiDBClient.connect))
```
:::

## まとめ

- **curl 1行**でインスタンス作成（認証不要・無料）
- **MySQL 8.0互換**なので既存ツール・ライブラリがそのまま使える
- **pytidbのauto-embedding**でベクトル検索が数行で実現
- **claimUrl**で永続化可能

## この先で作るもの

本シリーズでは、TiDB Cloud Zeroのベクトル検索を土台に、PDFを自然言語で質問できるRAGアプリを作ります。Python + Amazon Bedrock + React（SSEストリーミング）という構成です。

![PDF RAG Searchアプリ](/images/tidb-zero-rag-app.png)

次回はClaude CodeのTiDBスキルを活用した開発フローを紹介します。

## シリーズ記事

この記事は「TiDB Cloud Zero × Claude Code で作る PDF RAG アプリ」シリーズの一部です。

1. **TiDB Cloud Zero入門** ← 本記事
2. [Claude CodeのTiDBスキルでAI駆動開発](claude-code-tidb-skills)
3. [TiDB + Amazon BedrockでPDF RAGアプリを作る](tidb-bedrock-pdf-rag)

## 参考リンク

- [TiDB Cloud Zero](https://zero.tidbcloud.com/)
- [pytidb（GitHub）](https://github.com/pingcap/pytidb)
- [TiDB Vector Search ドキュメント](https://docs.pingcap.com/tidbcloud/vector-search-overview)
