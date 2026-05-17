# 第1回：MariaDB — リレーショナルデータベースの目的と基本操作

## 授業情報

| 項目 | 内容 |
|------|------|
| 前提知識 | Linuxの基本コマンド、エディタ操作（nano/vim） |
| 動作環境 | WSL2 (Ubuntu) |
| 到達目標 | MariaDBをインストールし、SQL でテーブルの作成・データ操作ができる。その後Dockerコンテナとして起動し外部から接続できる |

---

## 学習目標

この回が終わると、以下ができるようになります。

1. **RDBMSとは何か、どんな場面で使うかを説明できる**
2. **MariaDBをインストールして起動状態を確認できる**
3. **mysql_secure_installation でセキュリティ初期設定ができる**
4. **SQL（CREATE / INSERT / SELECT / UPDATE / DELETE）を使った基本操作ができる**
5. **DockerでMariaDBコンテナを起動し、ホストから接続できる**

---

## データベースの種類と MariaDB の目的

```
データベースの分類
├── RDBMS（リレーショナル）  ← 今回
│   ├── MariaDB / MySQL
│   ├── PostgreSQL
│   └── SQLite
└── NoSQL（非リレーショナル）
    ├── ドキュメント型  → MongoDB（第2回）
    └── キーバリュー型 → Redis（第3回）
```

### RDBMSとは

**RDBMS（Relational Database Management System）** は、データを **表（テーブル）** の形で管理するデータベース。

| 特徴 | 説明 |
|------|------|
| テーブル構造 | 行（レコード）と列（カラム）で構成された表形式 |
| SQL | データの操作に標準化された言語（SQL）を使う |
| トランザクション | 一連の処理を「全部成功か全部失敗」で管理できる |
| 整合性 | 外部キーなどで複数テーブル間のデータ整合性を保てる |

### MariaDB とは

**MariaDB** は MySQL から派生したオープンソースのRDBMS。MySQLと高い互換性を持ち、コマンド・設定ファイルの形式もほぼ同じ。

> 用途例：会員情報・注文履歴・商品カタログなど「構造が決まった永続データ」の管理

---

## 全体の流れ

```
【前半】WSL2 に直接インストール
  apt install → 起動確認 → セキュリティ設定 → SQL 操作

【後半】Docker コンテナとして起動
  docker run → ホストから mysql クライアントで接続
```

---

## 前半：WSL2 への直接インストール

### 1-1. インストール

```bash
sudo apt update
sudo apt install -y mariadb-server
```

起動と自動起動設定：

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl status mariadb
```

> **確認ポイント**
> `active (running)` が表示されればOK。

バージョン確認：

```bash
mariadb --version
```

---

### 1-2. セキュリティ初期設定（mysql_secure_installation）

```bash
sudo mysql_secure_installation
```

対話形式で以下を設定する：

| 質問 | 回答 | 理由 |
|------|------|------|
| Enter current password for root | そのままEnter | 初期パスワードは未設定 |
| Switch to unix_socket authentication [Y/n] | `n` | パスワード認証を維持する |
| Change the root password? [Y/n] | `n` | 今回は変更しない（本番は `y` で設定する） |
| Remove anonymous users? [Y/n] | `y` | 匿名ユーザーは削除する |
| Disallow root login remotely? [Y/n] | `y` | リモートからのroot接続を禁止する |
| Remove test database and access to it? [Y/n] | `y` | testDBは不要なので削除する |
| Reload privilege tables now? [Y/n] | `y` | 設定を即時反映する |

完了すると `Thanks for using MariaDB!` と表示される。

---

### 1-3. rootでログインする

```bash
sudo mysql -u root -p
```

> パスワードを設定していない場合はそのままEnterを押す。
> `MariaDB [(none)]>` というプロンプトが表示されればログイン成功。

---

### 1-4. データベースとテーブルを作成する

MariaDBプロンプト内で以下を入力する。

```sql
-- データベース一覧を確認
SHOW DATABASES;

-- データベースを作成
CREATE DATABASE schooldb;

-- 使用するデータベースを選択
USE schooldb;

-- テーブルを作成
CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    score INT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- テーブル構造を確認
DESC students;
```

---

### 1-5. データを操作する（CRUD）

```sql
-- データを挿入（Create）
INSERT INTO students (name, score) VALUES ('田中太郎', 85);
INSERT INTO students (name, score) VALUES ('鈴木花子', 92);
INSERT INTO students (name, score) VALUES ('佐藤次郎', 78);

-- データを参照（Read）
SELECT * FROM students;

-- 条件で絞り込む
SELECT * FROM students WHERE score >= 80;

-- データを更新（Update）
UPDATE students SET score = 88 WHERE name = '田中太郎';

-- 更新後を確認
SELECT * FROM students;

-- データを削除（Delete）
DELETE FROM students WHERE name = '佐藤次郎';

-- 最終確認
SELECT * FROM students;
```

---

### 1-6. アプリ用ユーザーを作成する

本番環境では root ではなく専用ユーザーを使うのが基本。

```sql
-- ユーザーを作成してパスワードを設定
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'apppass123';

-- schooldb への全権限を付与
GRANT ALL PRIVILEGES ON schooldb.* TO 'appuser'@'localhost';

-- 権限を反映
FLUSH PRIVILEGES;

-- ユーザー一覧を確認
SELECT user, host FROM mysql.user;

-- MariaDB から抜ける
EXIT;
```

新しいユーザーでログインして動作確認：

```bash
mysql -u appuser -p schooldb
```

---

## 後半：Docker を使った MariaDB 接続

### 2-1. Docker で MariaDB コンテナを起動する

```bash
docker run -d \
  --name mariadb-test \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=testdb \
  -e MYSQL_USER=devuser \
  -e MYSQL_PASSWORD=devpass \
  -p 3307:3306 \
  mariadb:10.11
```

| オプション | 意味 |
|------------|------|
| `-d` | バックグラウンドで起動 |
| `--name` | コンテナ名を指定 |
| `-e MYSQL_ROOT_PASSWORD` | rootパスワードを設定 |
| `-e MYSQL_DATABASE` | 起動時に作成するDB名 |
| `-e MYSQL_USER / PASSWORD` | 追加ユーザーを自動作成 |
| `-p 3307:3306` | ホストの3307番をコンテナの3306番へ転送 |

起動確認：

```bash
docker ps
docker logs mariadb-test
```

> `ready for connections` が出ればOK（初回は30秒ほどかかる場合がある）。

---

### 2-2. ホストから接続する

**方法A：ホスト側の mysql クライアントを使う**

```bash
# mysql クライアントがない場合はインストール
sudo apt install -y mysql-client

# コンテナ内MariaDB（3307番ポート）に接続
mysql -h 127.0.0.1 -P 3307 -u devuser -p testdb
```

接続後の動作確認：

```sql
SHOW DATABASES;
CREATE TABLE items (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO items (name) VALUES ('apple'), ('banana');
SELECT * FROM items;
EXIT;
```

**方法B：コンテナの中に入って接続する**

```bash
docker exec -it mariadb-test mysql -u root -p testdb
```

---

### 2-3. コンテナを停止・削除する

```bash
docker stop mariadb-test
docker rm mariadb-test
```

> コンテナを削除するとデータも消える。永続化には `-v` でボリュームをマウントする（発展課題参照）。

---

## ゴール確認チェックリスト

- [ ] `sudo systemctl status mariadb` が `active (running)` になっている
- [ ] `sudo mysql -u root -p` でログインできる
- [ ] `CREATE DATABASE` → `USE` → `CREATE TABLE` → `INSERT` → `SELECT` の一連操作ができる
- [ ] `WHERE` 句で条件を指定したSELECTができる
- [ ] アプリ用ユーザーを作成し、そのユーザーでログインできる
- [ ] Docker で MariaDB コンテナを起動し、3307番ポートで接続できる
- [ ] RDBMSとNoSQLの違いを自分の言葉で説明できる

---

## まとめ

| 概念 | 内容 |
|------|------|
| RDBMS | 表形式でデータを管理する。整合性とトランザクションが強み |
| MariaDB | MySQLと互換性のあるオープンソースRDBMS |
| SQL | データ定義（DDL）とデータ操作（DML）に分かれる |
| `CREATE TABLE` | 列名・型・制約を定義してテーブルを作る |
| `CRUD` | Create/Read/Update/Delete — データ操作の基本4操作 |
| `GRANT` | ユーザーに権限を付与するコマンド |
| Docker 接続 | `-p` でポートを公開し、ホストの mysql クライアントから接続 |

---

## 発展課題（任意）

1. `students` テーブルに `class VARCHAR(10)` 列を `ALTER TABLE` で追加し、データを更新する。
2. `ORDER BY score DESC` で成績上位から並べて表示する。
3. Docker 起動時に `-v ~/mariadb-data:/var/lib/mysql` を付けてボリュームをマウントし、コンテナを削除・再作成してもデータが残ることを確認する。
4. `docker-compose.yml` を書いて `docker compose up -d` で起動する構成に書き換える。

### 発展課題 参考サイト

| 課題 | 参考URL | ポイント |
|------|---------|---------|
| **課題1** ALTER TABLE | https://mariadb.com/kb/en/alter-table/ | `ADD COLUMN` / `MODIFY COLUMN` / `DROP COLUMN` の構文。`AFTER` 句で列の挿入位置を指定できる |
| **課題2** SELECT / ORDER BY | https://mariadb.com/kb/en/select/ | `ORDER BY` の昇順（ASC）・降順（DESC）と `LIMIT` との組み合わせ方 |
| **課題2** UPDATE 文 | https://mariadb.com/kb/en/update/ | `WHERE` 条件と `SET` 句の書き方。複数カラムを同時に更新する方法も掲載 |
| **課題3** Docker ボリューム | https://docs.docker.com/engine/storage/volumes/ | `-v` の書き方と名前付きボリュームの違い。`docker volume ls` で確認する方法 |
| **課題3** MariaDB Docker Hub | https://hub.docker.com/_/mariadb | 公式イメージの環境変数一覧（`MYSQL_ROOT_PASSWORD` など）と永続化の設定例 |
| **課題4** Docker Compose 仕様 | https://docs.docker.com/compose/compose-file/services/ | `services` / `volumes` / `environment` の書き方。ヘルスチェック設定まで掲載 |
| **課題4** MariaDB + Compose 例 | https://mariadb.com/kb/en/installing-and-using-mariadb-via-docker/ | MariaDB公式が提供するDocker・Composeの導入例 |
| 全般 MariaDB SQLリファレンス | https://mariadb.com/kb/en/sql-statements-structure/ | DDL・DML・DCLを網羅した公式SQLリファレンス |

---

## 次回予告

第2回では **MongoDB** を扱います。
RDBMSとは異なる「テーブルを持たないドキュメント型NoSQL」の考え方を理解し、JSONライクなドキュメントを使ったデータ管理を体験します。
