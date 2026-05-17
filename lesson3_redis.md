# 第3回：Redis — キーバリュー型NoSQLの目的と基本操作

## 授業情報

| 項目 | 内容 |
|------|------|
| 前提 | 第1回（MariaDB）・第2回（MongoDB）が完了していること |
| 動作環境 | WSL2 (Ubuntu) |
| 到達目標 | Redisをインストールし、各データ型の操作・TTL設定ができる。その後Dockerコンテナとして起動し外部から接続できる |

---

## 学習目標

この回が終わると、以下ができるようになります。

1. **Redisがどんな問題を解決するために使われるかを説明できる**
2. **Redisをインストールしてパスワード設定・起動確認ができる**
3. **String / List / Set / Sorted Set / Hash の5種類のデータ型を操作できる**
4. **TTL（有効期限）を設定してキャッシュ的な使い方を体験できる**
5. **DockerでRedisコンテナを起動し、redis-cli で接続できる**

---

## Redis の目的と特徴

### 3種類のデータベースを比較する

| 種類 | 代表製品 | データ構造 | 得意なこと |
|------|----------|------------|------------|
| RDBMS | MariaDB | テーブル・行・列 | 複雑な検索・整合性・トランザクション |
| ドキュメント型 | MongoDB | JSONドキュメント | 柔軟なスキーマ・ネスト構造 |
| **キーバリュー型** | **Redis** | **キー ＝ 値** | **超高速アクセス・キャッシュ・TTL** |

### Redis とは

**Redis（Remote Dictionary Server）** は、データをメモリ上に保持するオープンソースのキーバリュー型NoSQL。

```
Key         Value
─────────── ────────────
"user:1001" "Alice"
"session:x" "{token: ...}"
"counter"   42
```

| 特徴 | 説明 |
|------|------|
| **インメモリ** | データをRAMに保持するため読み書きが超高速（マイクロ秒オーダー） |
| **TTL** | キーごとに有効期限を設定できる（セッション・キャッシュに最適） |
| **多様なデータ型** | String / List / Set / Sorted Set / Hash の5種類 |
| **Pub/Sub** | メッセージキューとしても使える |

### Redis が向いている用途

- **セッション管理**：ログイン状態を一時的に保持（TTLで自動削除）
- **キャッシュ**：重いDBクエリの結果をRedisに保存して応答を高速化
- **リアルタイムランキング**：Sorted Set でスコアを管理
- **カウンター**：アクセス数・いいね数などのインクリメント
- **キュー**：タスクキューとして List を使う

> Redis は永続化もできるが、基本は「高速・一時的」な用途が中心。長期保存には MariaDB / MongoDB と組み合わせるのが定石。

---

## 全体の流れ

```
【前半】WSL2 に直接インストール
  apt install → 設定ファイルでパスワード設定 → 起動確認 → redis-cli で操作

【後半】Docker コンテナとして起動
  docker run → redis-cli で接続 → 基本操作
```

---

## 前半：WSL2 への直接インストール

### 1-1. インストール

```bash
sudo apt update
sudo apt install -y redis
```

起動と自動起動設定：

```bash
sudo systemctl start redis-server
sudo systemctl enable redis-server
sudo systemctl status redis-server
```

> **Ubuntu 24.04 注意点**
> サービス名は `redis` ではなく `redis-server` が正しい。
> `sudo systemctl status redis` と入力するとエラーになる場合がある。

バージョン確認：

```bash
redis-server --version
```

---

### 1-2. パスワードを設定する

Redis の設定ファイルを編集してパスワードを設定する。

```bash
sudo nano /etc/redis/redis.conf
```

以下の行を探して変更する（`requirepass` を検索：`Ctrl+W` → `requirepass`）：

```
# 変更前（コメントアウトされている）
# requirepass foobared

# 変更後（コメントを外してパスワードを設定）
requirepass redispass
```

> `Ctrl+O` → Enter → `Ctrl+X` で保存して終了。

設定を反映するために再起動：

```bash
sudo systemctl restart redis-server
sudo systemctl status redis-server
```

---

### 1-3. redis-cli で接続する

```bash
redis-cli
```

パスワード認証：

```
127.0.0.1:6379> auth redispass
OK
```

PING で動作確認：

```
127.0.0.1:6379> ping
PONG
```

---

## String 型（文字列）

Redisの最も基本的なデータ型。

```
127.0.0.1:6379> set key1 "Hello Redis"
OK
127.0.0.1:6379> get key1
"Hello Redis"

# キーの存在確認
127.0.0.1:6379> exists key1
(integer) 1

# キーを削除
127.0.0.1:6379> del key1
(integer) 1
127.0.0.1:6379> get key1
(nil)
```

### キーの一覧表示と全削除

```
127.0.0.1:6379> set message "Hello"
OK
127.0.0.1:6379> set name "Alice"
OK
127.0.0.1:6379> keys *
1) "message"
2) "name"

# 全てのキーを削除（本番環境では注意）
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> keys *
(empty array)
```

---

## TTL（有効期限）の設定

Redisの重要機能。セッション管理やキャッシュに使う。

```
# キーをセット
127.0.0.1:6379> set session:user1 "token_abc123"
OK

# TTL を確認（-1 = 期限なし）
127.0.0.1:6379> ttl session:user1
(integer) -1

# 10秒の有効期限を設定
127.0.0.1:6379> expire session:user1 10
(integer) 1

# 残り時間を確認
127.0.0.1:6379> ttl session:user1
(integer) 7

# 期限が切れると自動削除される
127.0.0.1:6379> get session:user1
(nil)
```

セット時に最初からTTLを指定することもできる：

```
# EX オプションで秒数指定
127.0.0.1:6379> set session:user2 "token_xyz" EX 30
OK
127.0.0.1:6379> ttl session:user2
(integer) 28
```

---

## List 型（リスト・キュー）

順序付きの値の集合。キューやスタックとして使える。

```
# リストの先頭（left）に追加
127.0.0.1:6379> lpush users sasaki
(integer) 1
127.0.0.1:6379> lpush users kanno
(integer) 2

# インデックス指定で取得（0が先頭）
127.0.0.1:6379> lrange users 0 0
1) "kanno"
127.0.0.1:6379> lrange users 1 1
1) "sasaki"

# 全件取得（0 から -1 で全件）
127.0.0.1:6379> lrange users 0 -1
1) "kanno"
2) "sasaki"

# 先頭から取り出す（pop）
127.0.0.1:6379> lpop users
"kanno"

# リストの末尾（right）に追加
127.0.0.1:6379> rpush users yamamoto
(integer) 2

127.0.0.1:6379> lrange users 0 -1
1) "sasaki"
2) "yamamoto"
```

---

## Set 型（集合）

順序なし・重複なしの値の集合。

```
127.0.0.1:6379> sadd class ICT
(integer) 1
127.0.0.1:6379> sadd class GAME
(integer) 1
127.0.0.1:6379> sadd class CG
(integer) 1

# 重複は追加されない（戻り値が 0）
127.0.0.1:6379> sadd class ICT
(integer) 0

# 全メンバーを表示
127.0.0.1:6379> smembers class
1) "CG"
2) "ICT"
3) "GAME"

# メンバーが存在するか確認（1=あり / 0=なし）
127.0.0.1:6379> sismember class ICT
(integer) 1
127.0.0.1:6379> sismember class MUSIC
(integer) 0

# 要素数を確認
127.0.0.1:6379> scard class
(integer) 3

# 要素を削除
127.0.0.1:6379> srem class ICT
(integer) 1
```

---

## Sorted Set 型（スコア付き集合）

各メンバーにスコアが付く。ランキング処理に便利。

```
# スコア付きで追加（zadd キー スコア メンバー）
127.0.0.1:6379> zadd ranking 85 sasaki
(integer) 1
127.0.0.1:6379> zadd ranking 92 kanno
(integer) 1
127.0.0.1:6379> zadd ranking 78 yamamoto
(integer) 1

# スコアの低い順（昇順）で全件表示
127.0.0.1:6379> zrange ranking 0 -1
1) "yamamoto"
2) "sasaki"
3) "kanno"

# スコアも合わせて表示
127.0.0.1:6379> zrange ranking 0 -1 withscores
1) "yamamoto"
2) "78"
3) "sasaki"
4) "85"
5) "kanno"
6) "92"

# スコアの高い順（降順）で表示
127.0.0.1:6379> zrevrange ranking 0 -1 withscores
1) "kanno"
2) "92"
3) "sasaki"
4) "85"
5) "yamamoto"
6) "78"

# 順位を確認（0始まり、昇順）
127.0.0.1:6379> zrank ranking sasaki
(integer) 1

# 降順での順位
127.0.0.1:6379> zrevrank ranking sasaki
(integer) 1
```

---

## Hash 型（ハッシュ）

1つのキーに複数のフィールドと値を持てる。ユーザー情報などの管理に使う。

```
# ハッシュにフィールドをセット
127.0.0.1:6379> hset user:1001 lname sasaki
(integer) 1
127.0.0.1:6379> hset user:1001 fname hiroyuki
(integer) 1
127.0.0.1:6379> hset user:1001 age 22
(integer) 1

# 特定フィールドの取得
127.0.0.1:6379> hget user:1001 lname
"sasaki"

# 全フィールドと値を取得
127.0.0.1:6379> hgetall user:1001
1) "lname"
2) "sasaki"
3) "fname"
4) "hiroyuki"
5) "age"
6) "22"

# フィールドの存在確認（1=あり / 0=なし）
127.0.0.1:6379> hexists user:1001 fname
(integer) 1

# フィールドを削除
127.0.0.1:6379> hdel user:1001 age
(integer) 1

127.0.0.1:6379> hgetall user:1001
1) "lname"
2) "sasaki"
3) "fname"
4) "hiroyuki"
```

---

### データベースの切り替え

Redisは0〜15番の16個のデータベースを持つ。

```
# DB 1 に切り替え
127.0.0.1:6379> select 1
OK

# DB 1 は空
127.0.0.1:6379[1]> keys *
(empty array)

# DB 0 に戻る
127.0.0.1:6379[1]> select 0
OK

# 存在しない DB 番号はエラー
127.0.0.1:6379> select 16
(error) ERR DB index is out of range

# redis-cli を終了
127.0.0.1:6379> quit
```

---

## 後半：Docker を使った Redis 接続

### 2-1. Docker で Redis コンテナを起動する

```bash
docker run -d \
  --name redis-test \
  -p 6380:6379 \
  redis:7.2 \
  redis-server --requirepass redispass
```

| オプション | 意味 |
|------------|------|
| `-p 6380:6379` | ホストの6380番をコンテナの6379番へ転送 |
| `redis-server --requirepass redispass` | コンテナ起動時にパスワードを設定 |

起動確認：

```bash
docker ps
docker logs redis-test
```

---

### 2-2. ホストから接続する

**方法A：redis-cli コマンドで直接接続**

```bash
redis-cli -h 127.0.0.1 -p 6380 -a redispass
```

接続後の動作確認：

```
127.0.0.1:6380> ping
PONG
127.0.0.1:6380> set greeting "Hello Docker Redis"
OK
127.0.0.1:6380> get greeting
"Hello Docker Redis"
127.0.0.1:6380> set session:demo "abc123" EX 60
OK
127.0.0.1:6380> ttl session:demo
(integer) 58
127.0.0.1:6380> quit
```

**方法B：コンテナの中に入って接続する**

```bash
docker exec -it redis-test redis-cli -a redispass
```

---

### 2-3. コンテナを停止・削除する

```bash
docker stop redis-test
docker rm redis-test
```

---

## ゴール確認チェックリスト

| 確認項目 | 期待する結果 |
|----------|-------------|
| `sudo systemctl status redis-server` | `active (running)` |
| `redis-cli` + `auth` でログインできる | `OK` |
| `set` / `get` で String の読み書きができる | 設定した値が取得できる |
| `expire` で TTL を設定し、`ttl` で確認できる | 残り秒数が表示される |
| TTL が切れたキーが自動削除されている | `get` で `(nil)` が返る |
| `lpush` / `lrange` で List の操作ができる | 追加した値が正しい順序で取得できる |
| `sadd` / `smembers` で Set の操作ができる | 重複なしで値が格納されている |
| `zadd` / `zrange` で Sorted Set の操作ができる | スコア順に並んでいる |
| `hset` / `hgetall` で Hash の操作ができる | フィールドと値が取得できる |
| Docker コンテナを起動して 6380 番で接続できる | redis-cli で `PONG` が返る |

---

## まとめ

| データ型 | コマンド | 主な用途 |
|----------|----------|----------|
| String | `set` / `get` / `del` / `expire` | セッション、カウンター、キャッシュ |
| List | `lpush` / `rpush` / `lrange` / `lpop` | キュー、タスクリスト |
| Set | `sadd` / `smembers` / `sismember` / `srem` | タグ、ユニークなIDの集合 |
| Sorted Set | `zadd` / `zrange` / `zrank` | ランキング、スコアボード |
| Hash | `hset` / `hget` / `hgetall` / `hdel` | ユーザー情報、設定データ |

| 概念 | 内容 |
|------|------|
| インメモリ | データをRAMに置くため超高速。サーバ再起動でデータは消える（設定で永続化可） |
| TTL | キーの有効期限。セッション管理・キャッシュに不可欠 |
| DB番号 | 0〜15番の16個のDB空間。用途ごとに分けて使う |
| `flushall` | 全DBの全キーを削除。本番では絶対に注意 |

---

## 3回の総まとめ：データベースの使い分け

```
どのデータベースを選ぶか？

  構造が決まっている / 複雑な検索 / トランザクション が必要
  └─→ MariaDB（RDBMS）

  スキーマが柔軟 / JSONをそのまま保存 / 大量ドキュメント
  └─→ MongoDB（ドキュメント型NoSQL）

  高速が必要 / 一時的なデータ / TTL / ランキング / キャッシュ
  └─→ Redis（キーバリュー型NoSQL）
```

実際のWebシステムでは MariaDB + Redis の組み合わせ（永続データはMariaDB、セッション・キャッシュはRedis）が定番構成。

---

## 発展課題（任意）

1. `set session:user1 "token" EX 10` でセッションを作り、`ttl` で残り時間を観察する。10秒後に `get` で `(nil)` になることを確認する。
2. Sorted Set に学生名とテスト点数を5人分登録し、`zrange ... withscores` で点数順のランキングを表示する。
3. Docker 起動時に `-v ~/redis-data:/data` と `--save 60 1` オプションを追加し、RDB（スナップショット）による永続化を試す。
4. `docker-compose.yml` を書いて MariaDB + Redis を同時に起動する構成を作る。

### 発展課題 参考サイト

| 課題 | 参考URL | ポイント |
|------|---------|---------|
| **課題1** TTL / EXPIRE | https://redis.io/docs/latest/commands/expire/ | `EXPIRE key 秒` / `TTL key` の詳細仕様。`EX` オプションで `SET` と同時指定する方法も掲載 |
| **課題1** SET（EXオプション） | https://redis.io/docs/latest/commands/set/ | `SET key value EX seconds` の全オプション一覧 |
| **課題2** Sorted Set 概念 | https://redis.io/docs/latest/data-types/sorted-sets/ | `ZADD` / `ZRANGE` / `ZREVRANGE` / `ZRANK` の使い分けを図解で解説 |
| **課題2** ZRANGE コマンド | https://redis.io/docs/latest/commands/zrange/ | Redis 6.2以降は `ZRANGE` が昇順・降順両対応。`REV` オプションで `ZREVRANGE` を置き換えられる |
| **課題3** 永続化（RDB/AOF） | https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/ | RDB（スナップショット）とAOF（追記ログ）の違い。`--save` オプションの意味を理解できる |
| **課題4** Docker Compose + Redis | https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/ | 公式のDocker Compose例。`command` でオプションを渡す書き方が参考になる |
| **課題4** Compose ファイル仕様 | https://docs.docker.com/compose/compose-file/services/ | `depends_on` でMariaDBが先に起動するよう依存関係を設定する方法 |
| 全般 Redis公式コマンドリファレンス | https://redis.io/docs/latest/commands/ | 全コマンドをデータ型別に検索できる。引数・戻り値・複雑度まで掲載 |

---

## トラブルシューティング

| 症状 | 確認ポイント |
|------|-------------|
| `redis-cli` で `NOAUTH Authentication required` | `auth パスワード` を先に実行する |
| `Connection refused` | `sudo systemctl status redis-server` でサービスが起動しているか確認 |
| Docker接続で `WRONGPASS` | `--requirepass` で設定したパスワードと `-a` で指定したパスワードが一致しているか確認 |
| `TTL` が `-1` のまま | `expire` コマンドを実行したか確認。または `set key value EX 秒数` を使う |

---

お疲れさまでした。
3回の授業を通じて **MariaDB（RDBMS）・MongoDB（ドキュメント型NoSQL）・Redis（キーバリュー型NoSQL）** の目的と基本操作、そしてDockerを使った接続方法を習得しました。
それぞれの特性を理解して、用途に合ったデータベースを選べるようになりましょう。
