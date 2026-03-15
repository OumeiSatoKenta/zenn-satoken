---
title: "Claude CodeのTiDBスキルでAI駆動開発 — DBプロビジョニングからRAG実装まで"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "tidb", "ai", "開発環境"]
published: false
---

## はじめに

「データベースを用意して、テーブル定義して、ベクトル検索を実装して...」。RAGアプリケーションの開発では、コードを書く前にやるべきことが山ほどあります。

本記事では、**Claude Codeのスキル機能**を活用し、TiDB Cloud Zeroのプロビジョニングからpytidbによるベクトル検索の実装まで、AI駆動で高速に開発した体験を紹介します。さらに、スペック駆動開発による設計ドキュメントの自動生成と、サブエージェントによる並列レビューについても解説します。

## この記事で扱うこと

- Claude Codeスキルの仕組み（SKILL.mdとは何か）
- TiDB Cloud Zeroのプロビジョニングスキルの設計と実装
- pytidbスキルを使ったベクトル検索の実装支援
- スペック駆動開発によるドキュメント6本の自動生成
- 6つのサブエージェントによる並列レビューで49件の問題を発見した話

## 前提条件

- Claude Code がインストール済みであること
- Node.js v18以上（devcontainer環境を推奨）
- AWSアカウント（Bedrock利用時のみ）

TiDB Cloud Zeroは認証不要・無料で利用できるため、TiDBのアカウントは不要です。

## Claude Codeスキルとは

### SKILL.mdの仕組み

Claude Codeの**スキル**は、`.claude/skills/` ディレクトリに配置する `SKILL.md` ファイルで定義します。スキルは「Claude Codeへの手順書」であり、特定のタスクを実行する方法をMarkdownで記述します。

```
.claude/skills/
└── tidb-zero-provisioner/
    └── SKILL.md        # シンプルなスキルはこれだけで完成する
```

SKILL.mdの基本構造は以下の通りです。

```markdown
---
name: tidb-zero-provisioner
description: TiDB Cloud Zeroインスタンスを作成するプロビジョニングスキル
---

# スキル名

## 処理フロー

### ステップ1: ...
（Bashコマンドや手順を記述）

### ステップ2: ...
```

ポイントは、**コードではなくMarkdown**であること。Claude Codeがこの指示書を読み、Bashコマンドを実行してくれます。つまり、プログラムを書くのではなく「手順を書く」だけでツールが完成します。

スキルの呼び出しはClaude Codeのチャット上で `/スキル名` と入力するだけです。

```
$ claude
> /tidb-zero-provisioner
```

Claude CodeがSKILL.mdを読み込み、記載された手順を順番に実行してくれます。

## tidb-zero-provisionerスキルの設計と実装

### TiDB Cloud Zeroとは

TiDB Cloud Zeroは、**API Key不要・無料・即座に**MySQLインスタンスをプロビジョニングできるサービスです。インスタンスは30日で自動失効しますが、開発・プロトタイピングには十分です。

```bash
# たった1行でMySQLインスタンスが手に入る
curl -s -X POST https://zero.tidbapi.com/v1alpha1/instances \
  -H "Content-Type: application/json" \
  -d '{"tag":"pdf-rag-app"}'
```

レスポンス例:

```json
{
  "instance": {
    "id": "k6oFvE7k_D5UzETLbBFALYH2ceCdosKB",
    "connection": {
      "host": "gateway03.us-west-2.prod.aws.tidbcloud.com",
      "port": 4000,
      "username": "2PMAqxsfidag6Fe.root",
      "password": "****"
    },
    "expiresAt": "2026-04-06T16:19:16.657Z",
    "claimInfo": {
      "claimUrl": "https://tidbcloud.com/tidbs/claim/..."
    }
  }
}
```

### スキルの処理フロー

`tidb-zero-provisioner` スキルは以下の5ステップで構成しました。

**ステップ1: インスタンス作成**

```bash
RESPONSE=$(curl -s -X POST https://zero.tidbapi.com/v1alpha1/instances \
  -H "Content-Type: application/json" \
  -d '{"tag":"pdf-rag-app"}')
echo "$RESPONSE" | jq .
```

**ステップ2: 接続情報の抽出**

```bash
TIDB_HOST=$(echo "$RESPONSE" | jq -r '.instance.connection.host')
TIDB_PORT=$(echo "$RESPONSE" | jq -r '.instance.connection.port')
TIDB_USERNAME=$(echo "$RESPONSE" | jq -r '.instance.connection.username')
TIDB_PASSWORD=$(echo "$RESPONSE" | jq -r '.instance.connection.password')
```

**ステップ3: 接続情報をユーザーに提示**（後述のguard-secrets.shの制約により手動コピー方式）

**ステップ4: 接続検証**（mysql2パッケージで実行）

**ステップ5: データベース作成と結果表示**

### guard-secrets.shとの格闘

ここで最大のハマりポイントがありました。

:::message alert
このプロジェクトでは、`.env` ファイルの誤コミットを防ぐカスタムフック `guard-secrets.sh` を実装しています（Claude Codeの標準機能ではなく、プロジェクト固有の設定です）。このフックは `.env` ファイルへの書き込みを**すべてブロック**します。
:::

当初のスキル設計では、接続情報を自動で `.env` に書き込む想定でした。しかし実際に動かしてみると:

- `cat >> .env` — ブロック
- `echo >> .env` — ブロック
- Claude CodeのWriteツール — ブロック
- `.env.example` の作成 — これもブロック

全滅です。

**対処法**: SKILL.mdを修正し、接続情報を画面に表示してユーザーに手動コピーしてもらう方式に変更しました。

```markdown
### ステップ3: 接続情報の保存

接続情報をユーザーに表示し、`.env` ファイルへの書き込みを案内する。

**注意**: セキュリティフック（guard-secrets.sh）により、
Claude Codeから `.env` ファイルへの直接書き込みはブロックされる。
以下の内容をユーザーに提示し、手動でコピーしてもらう:

```
以下の内容を .env ファイルに追記してください:

TIDB_HOST=${TIDB_HOST}
TIDB_PORT=${TIDB_PORT}
TIDB_USERNAME=${TIDB_USERNAME}
TIDB_PASSWORD=${TIDB_PASSWORD}
TIDB_DATABASE=pdf_rag
TIDB_SSL_MODE=REQUIRED
```

セキュリティと利便性のトレードオフですが、機密情報を扱う以上、手動コピーのひと手間は妥当な設計だと考えています。

### devcontainerにmysqlクライアントがない問題

接続検証で `mysql` コマンドを使おうとしたところ、devcontainer環境にインストールされていませんでした。

:::message
devcontainer環境では必要なCLIツールが入っていないことがあります。npm/pip経由で代替手段を確保する設計が重要です。
:::

**対処法**: Node.jsの `mysql2` パッケージで代替しました。

```bash
npm install --no-save mysql2
node -e "
const mysql = require('mysql2/promise');
(async () => {
  const conn = await mysql.createConnection({
    host: '${TIDB_HOST}', port: ${TIDB_PORT},
    user: '${TIDB_USERNAME}', password: '${TIDB_PASSWORD}',
    ssl: { rejectUnauthorized: true }
  });
  const [rows] = await conn.execute(
    \"SELECT 'connected' AS status, VERSION() AS tidb_version\"
  );
  console.log(rows);
  await conn.execute('CREATE DATABASE IF NOT EXISTS pdf_rag');
  console.log('Database pdf_rag created');
  await conn.end();
})();
"
```

実行結果:

```
[ { status: 'connected', tidb_version: '8.0.11-TiDB-v8.5.3-serverless' } ]
Database pdf_rag created
```

## pytidbスキルの活用

### テーブル定義の実装支援

pytidbはTiDB専用のPython ORMで、auto-embedding機能が特徴です。Claude Codeのpytidbスキルを使うことで、正しいAPIを使った実装が支援されます。

以下はベクトル検索用テーブルの定義例です。

```python
from pytidb.schema import TableModel, Field
from pytidb import EmbeddingFunction, FullTextField

embed_fn = EmbeddingFunction("tidbcloud_free/amazon/titan-embed-text-v2")

class Chunk(TableModel):  # table=True は不要
    __tablename__ = "chunks"
    id: int = Field(primary_key=True)
    document_id: int = Field()
    text: str = FullTextField()
    text_vec: list[float] = embed_fn.VectorField(source_field="text")
    page_number: int = Field()
```

:::message alert
pytidbでは `TableModel` に `table=True` を付けてはいけません。付けるとSQLAlchemy ORMマッピングモードに入り、主キーエラーが発生します。また、`Field` は `pydantic.Field` ではなく `pytidb.schema.Field` を使う必要があります。
:::

### ベクトル検索の実装

テーブル作成からベクトル検索までの一連の流れです。

```python
from pytidb import TiDBClient

# 接続
db = TiDBClient.connect(
    host=TIDB_HOST,
    port=TIDB_PORT,
    username=TIDB_USERNAME,
    password=TIDB_PASSWORD,
    database="pdf_rag",
    enable_ssl=True,
    ensure_db=True
)

# テーブル作成（既存なら何もしない）
db.create_table(schema=Chunk, if_exists="skip")

# テーブル取得
table = db.open_table("chunks")

# データ挿入（auto-embeddingでベクトルが自動計算される）
table.insert([
    Chunk(document_id=1, text="TiDBはHTAPデータベースです", page_number=1),
    Chunk(document_id=1, text="ベクトル検索をサポートしています", page_number=2),
])

# ベクトル検索
results = table.search("TiDBの特徴は？").limit(5).to_list()
for r in results:
    print(r["text"], r["_distance"])
```

### pytidb APIのハマりポイント集

開発中に踏んだ地雷をまとめます。APIの確認は `inspect.signature()` が最も確実です。

```python
import inspect
from pytidb.client import TiDBClient
print(inspect.signature(TiDBClient.connect))
```

| 誤り | 正しい書き方 |
|------|-------------|
| `db.create_table(schema=Model, mode="exist_ok")` | `db.create_table(schema=Model, if_exists="skip")` |
| `db.get_table("name")` | `db.open_table("name")` |
| `table.query(filter={...})` | `table.query(filters={...})` （複数形） |
| `table.query(filters={...})` のfiltersと混同して `table.search(...).filters({...})` | `table.search(...).filter({...})` （searchチェーンではfilterは単数形） |
| `TiDBClient.connect(ssl_verify=True)` | `TiDBClient.connect(enable_ssl=True)` |

## スペック駆動開発の流れ

### /setup-projectでドキュメント6本を自動生成

`/setup-project` と後述の `/review-docs` は、このリポジトリに同梱されたカスタムスキルです。`.claude/skills/` にSKILL.mdを置くことで、同様のコマンドを自分のプロジェクトにも追加できます。

`/setup-project` を実行すると、対話的に6つの設計ドキュメントが生成されます。

```
$ claude
> /setup-project
```

生成されるドキュメント:

| ドキュメント | 役割 |
|------------|------|
| `docs/product-requirements.md` | プロダクト要求定義書（PRD） |
| `docs/functional-design.md` | 機能設計書 |
| `docs/architecture.md` | アーキテクチャ設計書 |
| `docs/repository-structure.md` | リポジトリ構造定義書 |
| `docs/development-guidelines.md` | 開発ガイドライン |
| `docs/glossary.md` | 用語集（ユビキタス言語定義） |

流れとしては、まず `docs/ideas/` にアイデアを書き出し、Claude Codeに読み込ませます。そこから PRD → 機能設計 → アーキテクチャ と段階的にドキュメントが生成されていきます。1ファイルずつ作成され、ユーザーの承認を得てから次に進む設計です。

### 技術選定の判断理由

ドキュメント作成の過程で固まった主な技術選定です。

| 技術 | 選定理由 |
|------|---------|
| TiDB Cloud Zero | 認証不要・無料・即座にプロビジョニング。読者が再現しやすい |
| pytidb | TiDB専用ORM。auto-embeddingで外部APIキー不要 |
| Amazon Bedrock | AWSエコシステムとの統合。ECSタスクロールで認証情報管理が容易 |
| FastAPI | 非同期対応、自動OpenAPIドキュメント生成 |
| React + TanStack Query | サーバー状態管理が宣言的 |

## サブエージェント並列レビュー

### 6エージェントで49件の問題を発見

ドキュメント6本の作成後、`/review-docs docs` コマンドで並列レビューを実行しました。

```
$ claude
> /review-docs docs
```

Claude Codeの `doc-reviewer` サブエージェントが4〜6並列で起動し、以下の観点でレビューします。

1. **完全性**: 必要な項目がすべて含まれているか
2. **具体性**: 曖昧な表現がないか
3. **一貫性**: 他のドキュメントと整合しているか
4. **技術的正確性**: APIやコマンドが正しいか

### レビュー結果

| ドキュメント | スコア | 主な指摘 |
|------------|--------|---------|
| product-requirements.md | 3.6-4.2/5 | F5が機能要件として誤分類、KPI測定方法未記載 |
| functional-design.md | 3.2-3.4/5 | TableModel構文がpytidb公式と不一致 |
| architecture.md | 3.5-3.6/5 | VPCネットワーク構成未定義 |
| repository-structure.md | 4.25-4.6/5 | auth_middleware.pyの配置先未記載 |
| development-guidelines.md | 3.75-3.8/5 | devcontainerと手動ステップの区別が曖昧 |
| glossary.md | 4.0-4.75/5 | Frontend技術用語が欠落 |

### 発見された重大な問題

レビューで発見された問題の中で、特にインパクトが大きかったものを5つ挙げます。

:::message alert
**1. pytidb API不一致**
テーブル作成の引数が `mode="exist_ok"` と記載されていたが、正しくは `if_exists="skip"`。さらに `table=True` も不要だった。ドキュメント全体に波及する修正が必要になった。
:::

**2. チャンク分割のバグ**
`current_page = page.number` がループ末尾にあり、ページ番号が1ページずれる。設計段階で見つかったため、実装でのバグを未然に防げた。

**3. トークン vs 文字数の混在**
ドキュメントでは「500トークン」と記載しているが、実装は `len(buffer)` で文字数ベース。トークンと文字数は異なるため、「500文字」に統一した。

**4. VPC設計の欠落**
ECS Fargateのネットワーク構成（パブリック/プライベートサブネット）が未定義だった。本番環境で致命的になるため、architecture.mdに追記した。

**5. Terraform検証順序の誤り**
`validate → init` と記載していたが、`init` で依存モジュールをダウンロードしないと `validate` は動作しない。正しくは `init → validate`。

### 一括修正

6つのサブエージェントを並列起動し、全ドキュメントを一括修正しました。合計49件の変更を適用。主な修正内容:

- **PRD**: F5を「開発者向けツール」に分離、KPI測定にLangFuseを導入
- **機能設計書**: TableModel構文修正、`FullTextField()` 採用
- **アーキテクチャ**: VPC/サブネット設計追加、IAMポリシー定義
- **開発ガイドライン**: docker-compose起動手順追加、Terraform順序修正
- **用語集**: Frontend技術用語4つ追加

## 開発者体験の変化

### ドキュメントからコードへの高速な移行

従来の開発フローとの比較です。

**従来のフロー:**
1. 設計ドキュメントを手書き（数日）
2. レビュー会議（半日〜1日）
3. 修正（数日）
4. DB環境構築（手動で数時間）
5. コード実装開始

**スキル活用後のフロー:**
1. アイデアを壁打ち → `/setup-project` でドキュメント6本生成（数時間）
2. `/review-docs` で並列レビュー + 一括修正（数十分）
3. `tidb-zero-provisioner` でDB即座にプロビジョニング（数分）
4. pytidbスキルの知識ベースを参照しながらコード実装開始

特に効果が大きかったのは以下の3点です。

- **DB環境構築の時間短縮**: TiDB Cloud ZeroのAPI一発で完了。アカウント作成もAPI Key取得も不要
- **API調査の時間短縮**: pytidbの正しいAPIをスキルが把握しているため、ドキュメントを探し回る必要がない
- **設計品質の向上**: 並列レビューにより、実装前にバグや設計漏れを49件発見できた

## まとめ

Claude Codeのスキル機能は、「手順書をMarkdownで書くだけ」でツールが作れる軽量な仕組みです。TiDB Cloud Zeroと組み合わせることで、認証不要・無料のデータベースを数分でプロビジョニングできます。

本記事で紹介した開発フローのポイント:

1. **SKILL.mdは手順書** — コードではなくMarkdownで書く
2. **セキュリティフックとの共存** — `.env` への自動書き込みはブロックされるが、手動コピー方式で対応可能
3. **pytidbのAPI確認は `inspect.signature()` が最も確実** — 公式ドキュメントと実際のAPIにズレがある場合がある
4. **並列レビューは強力** — 6エージェントで49件の問題を発見し、設計段階でバグを潰せた
5. **スペック駆動 + スキル活用** — ドキュメントベースの設計からコード実装への移行が高速になる

## シリーズ記事

この記事は「TiDB Cloud Zero × Claude Code で作る PDF RAG アプリ」シリーズの一部です。

1. [TiDB Cloud Zero入門 — curlひとつで手に入るMySQL互換ベクトルDB](tidb-cloud-zero-intro)
2. **Claude CodeのTiDBスキルでAI駆動開発** ← 本記事
3. [TiDB + Amazon BedrockでPDF RAGアプリを作る](tidb-bedrock-pdf-rag)

## 参考リンク

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [TiDB Cloud Zero API](https://zero.tidbapi.com)
- [pytidb GitHub](https://github.com/pingcap/pytidb)
- [TiDB Cloud](https://tidbcloud.com)
