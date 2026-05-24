# 第2回：MongoDB — ドキュメント型NoSQLの目的と基本操作

## 授業情報

| 項目 | 内容 |
|------|------|
| 前提 | 第1回が完了していること（RDBMSとSQLの基本を理解している状態） |
| 動作環境 | WSL2 (Ubuntu) |
| 到達目標 | MongoDBをインストールし、ドキュメントのCRUD操作ができる。その後Dockerコンテナとして起動し外部から接続できる |

---

## 学習目標

この回が終わると、以下ができるようになります。

1. **MongoDBが「なぜ存在するのか」RDBMSとの違いを説明できる**
2. **MongoDBをインストールして起動状態を確認できる**
3. **mongosh でデータベース・コレクション・ドキュメントを操作できる**
4. **条件指定・更新・削除など基本的なCRUD操作ができる**
5. **DockerでMongoDBコンテナを起動し、ホストから接続できる**

---

## MongoDB の目的と特徴

### なぜ NoSQL が生まれたのか

RDBMSは「決まった構造のデータ」の管理に優れているが、次のような場面では扱いにくい：

- SNSの投稿（ユーザーごとにフィールドが違う）
- IoTセンサーデータ（スキーマが頻繁に変わる）
- ログデータ（大量の書き込みが必要）
- JSONをそのまま保存したい

こうした課題を解決するために生まれたのが **NoSQL（Not only SQL）** データベース。

### MongoDB とは

MongoDBはオープンソースの **ドキュメント型NoSQL** データベース。
データをJSONライクな **BSON（Binary JSON）** 形式のドキュメントで保存する。

```
RDBMS のデータ構造          MongoDB のデータ構造
┌──────────────────┐         ┌──────────────────────────┐
│ データベース       │         │ データベース               │
│  └─ テーブル      │   →     │  └─ コレクション           │
│      └─ 行/列     │         │      └─ ドキュメント（JSON）│
└──────────────────┘         └──────────────────────────┘
```

| RDBMS の用語 | MongoDB の用語 | 説明 |
|-------------|--------------|------|
| データベース | データベース | 変わらず |
| テーブル | コレクション | まとめる単位 |
| 行（レコード） | ドキュメント | 1件のデータ |
| 列（カラム） | フィールド | データの要素 |

### MongoDB の特徴

| 特徴 | 説明 |
|------|------|
| スキーマレス | 同じコレクション内でも各ドキュメントが異なるフィールドを持てる |
| 柔軟な構造 | ネストや配列を自然に表現できる |
| スケールアウト | シャーディングで水平分散が得意 |
| 向いている用途 | カタログ・コンテンツ管理・リアルタイム分析・ログ収集 |

> 向いていない用途：複数テーブルをまたぐ複雑な結合処理、厳密なトランザクションが必要な金融系処理

---

## 全体の流れ

```
【前半】WSL2 に直接インストール
  リポジトリ追加 → apt install → 起動確認 → mongosh で操作

【後半】Docker コンテナとして起動
  docker run → mongosh で接続 → 基本操作
```

---

## 前半：WSL2 への直接インストール

### 1-1. 公式リポジトリを追加してインストール

MongoDBはUbuntuの標準リポジトリには含まれないため、公式リポジトリを追加する。

```bash
# 必要なツールをインストール
sudo apt-get install -y gnupg curl

# MongoDB 8.0 公式GPGキーを追加
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

# リポジトリを追加（Ubuntu 24.04 / WSL2）
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# パッケージ情報を更新してインストール
sudo apt-get update
sudo apt-get install -y mongodb-org
```

> **Ubuntu 24.04 (Noble Numbat) 注意点**
> MongoDB 8.0 は Ubuntu 24.04 (Noble Numbat) を公式サポートしている。
> Ubuntu 22.04 (Jammy) 環境では `noble` の部分を `jammy` に変更すること。
> 参考：https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/

---

### 1-2. 起動と確認

WSL2 では `systemctl` が使えない場合があるため、まず確認する。

```bash
# systemctl が使える場合
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod

# systemctl が使えない場合（WSL2 の一部環境）
sudo mongod --dbpath /var/lib/mongodb --logpath /var/log/mongodb/mongod.log --fork
```

バージョン確認：

```bash
mongod --version
```

---

### 1-3. mongosh（シェル）で接続する

```bash
mongosh
```

> `test>` というプロンプトが表示されればOK。

```javascript
// バージョンを確認
db.version()

// データベース一覧を表示
show dbs
```

---

### 1-4. データベースとコレクションを作成する

```javascript
// 新しいデータベースを使う（存在しない場合は自動作成）
use newdb

// コレクションの一覧を確認（空）
show collections

// コレクションを作成
db.createCollection('pref')

// コレクションが追加されたことを確認
show collections
```

---

### 1-5. ドキュメントを登録する（Create）

```javascript
// 1件挿入
db.pref.insertOne({ capital: 'sapporo', name: 'hokkaido' })

// 複数件挿入
db.pref.insertMany([
  { capital: 'aomori', name: 'aomori' },
  { capital: 'morioka', name: 'iwate' },
  { capital: 'sendai',  name: 'miyagi', population: 230 }
])
```

---

### 1-6. ドキュメントを参照する（Read）

```javascript
// 全件取得
db.pref.find()

// 件数を確認
db.pref.countDocuments()

// 条件指定で絞り込む
db.pref.find({ name: 'aomori' })

// 数値の比較（population が 200 より大きい）
db.pref.find({ population: { "$gt": 200 } })
// $gt = greater than（より大きい）
// $lt = less than（より小さい）
// $gte = greater than or equal（以上）
// $lte = less than or equal（以下）
```

---

### 1-7. ドキュメントを更新する（Update）

```javascript
// hokkaido の population フィールドを追加・更新
db.pref.updateOne(
  { name: 'hokkaido' },
  { $set: { population: 520 } }
)

// 更新後に確認
db.pref.find({ name: 'hokkaido' })
```

---

### 1-8. ドキュメントを削除する（Delete）

```javascript
// 1件削除
db.pref.deleteOne({ name: 'aomori' })

// 全件確認
db.pref.find()
```

---

### 1-9. コレクションを削除する

```javascript
// コレクションごと削除
db.pref.drop()

// 削除後の確認（空になっている）
show collections
```

---

### 1-10. mongosh を終了する

```javascript
exit
```

---

## 後半：Docker を使った MongoDB 接続

### 2-1. Docker で MongoDB コンテナを起動する

```bash
docker run -d \
  --name mongo-test \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=adminpass \
  -p 27018:27017 \
  mongo:8.0
```

| オプション | 意味 |
|------------|------|
| `-e MONGO_INITDB_ROOT_USERNAME` | adminユーザー名を設定 |
| `-e MONGO_INITDB_ROOT_PASSWORD` | adminパスワードを設定 |
| `-p 27018:27017` | ホストの27018番をコンテナの27017番へ転送 |

起動確認：

```bash
docker ps
docker logs mongo-test
```

---

### 2-2. ホストから接続する

**方法A：mongosh コマンドで接続（mongosh がインストール済みの場合）**

```bash
mongosh --host 127.0.0.1 --port 27018 \
  -u admin -p adminpass --authenticationDatabase admin
```

**方法B：コンテナの中に入って接続する**

```bash
docker exec -it mongo-test mongosh \
  -u admin -p adminpass --authenticationDatabase admin
```

---

### 2-3. 接続後の動作確認

```javascript
// データベース一覧
show dbs

// 新しいDBに切り替え
use testdb

// コレクションを作成してデータを挿入
db.users.insertMany([
  { name: 'Alice', age: 22, role: 'student' },
  { name: 'Bob',   age: 25, role: 'teacher' }
])

// 全件表示
db.users.find()

// role が student のものだけ抽出
db.users.find({ role: 'student' })

exit
```

---

### 2-4. コンテナを停止・削除する

```bash
docker stop mongo-test
docker rm mongo-test
```

---

## ゴール確認チェックリスト

| 確認項目 | 期待する結果 |
|----------|-------------|
| `sudo systemctl status mongod` | `active (running)` |
| `mongosh` でプロンプトが表示される | `test>` |
| `db.createCollection()` でコレクション作成 | `{ ok: 1 }` |
| `insertOne` / `insertMany` でデータ登録 | `acknowledged: true` |
| `find()` で全件取得 | 登録したドキュメントが表示される |
| `find({ フィールド: 値 })` で絞り込み | 条件に合うドキュメントのみ表示 |
| `updateOne` でフィールドを追加・更新 | `modifiedCount: 1` |
| `deleteOne` でドキュメントを削除 | `deletedCount: 1` |
| Docker コンテナを起動して 27018 番ポートで接続 | mongosh でデータ操作できる |

---

## まとめ

| 概念 | 内容 |
|------|------|
| コレクション | RDBMSのテーブルに相当する、ドキュメントのまとまり |
| ドキュメント | JSON形式（BSON）のデータ1件。スキーマは自由 |
| `insertOne` / `insertMany` | ドキュメントの挿入 |
| `find` | ドキュメントの検索。条件なしで全件取得 |
| `$gt` / `$lt` | 数値比較の演算子（greater than / less than） |
| `updateOne` + `$set` | 指定フィールドを更新（他のフィールドは維持） |
| `deleteOne` | 条件に合う1件のドキュメントを削除 |
| `drop()` | コレクションごと削除 |

---

## 発展課題（任意）

1. `pref` コレクションに都道府県データを5件以上登録し、`population` をソートして上位3件だけ表示する（`.sort()` と `.limit()` を使う）。
2. ドキュメントに配列フィールド（例：`tags: ['north', 'snow']`）を追加し、`$in` 演算子で配列の中の特定の値を持つドキュメントを検索する。
3. Docker 起動時に `-v ~/mongo-data:/data/db` を付け、コンテナを削除・再作成してもデータが残ることを確認する。
4. `docker-compose.yml` を書いて MongoDB コンテナを管理する構成に書き換える。

### 発展課題 参考サイト

| 課題 | 参考URL | ポイント |
|------|---------|---------|
| **課題1** `.sort()` と `.limit()` | https://www.mongodb.com/docs/manual/reference/method/cursor.sort/ | `cursor.sort({ フィールド: -1 })` で降順。`.limit(3)` で件数制限 |
| **課題1** 続き `.limit()` | https://www.mongodb.com/docs/manual/reference/method/cursor.limit/ | `sort()` と `limit()` はチェーンで繋げる |
| **課題2** `$in` 演算子 | https://www.mongodb.com/docs/manual/reference/operator/query/in/ | `{ tags: { $in: ['north'] } }` のように配列の中の値を検索できる |
| **課題3** Docker ボリューム | https://docs.docker.com/engine/storage/volumes/ | `-v` の使い方と名前付きボリューム（`-v mongo-data:/data/db`）の違い |
| **課題4** Docker Compose 公式 | https://docs.docker.com/compose/compose-file/ | `services` / `volumes` の書き方。MongoDBの`environment`設定例も掲載 |
| **課題4** MongoDB + Compose 例 | https://www.mongodb.com/docs/manual/tutorial/install-mongodb-enterprise-with-docker/ | 公式が提供する Compose ファイルの参考例 |
| 全般 MongoDB公式チュートリアル | https://www.mongodb.com/docs/manual/tutorial/ | CRUD・インデックス・集計など体系的に学べる |

---

## トラブルシューティング

| 症状 | 確認ポイント |
|------|-------------|
| `mongod` が起動しない | `sudo journalctl -u mongod` でログを確認 |
| `mongosh: command not found` | `sudo apt install -y mongodb-mongosh` でインストール |
| Docker接続で認証エラー | `--authenticationDatabase admin` が抜けていないか確認 |
| `27017` に接続できない | `-p` オプションのポート番号を確認。ホスト側は `27018` |

---

## 次回予告

第3回では **Redis** を扱います。
「テーブルもドキュメントも持たない」キーバリュー型NoSQLの考え方を学び、インメモリDBならではの高速性と TTL（有効期限）機能を体験します。
