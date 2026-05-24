# 第4回：3つのDBを組み合わせたWebアプリ開発

## 授業情報

| 項目 | 内容 |
|------|------|
| 前提知識 | 第1〜3回の完了（MariaDB / MongoDB / Redis の基本操作） |
| 動作環境 | WSL2 (Ubuntu) + Docker / Docker Compose |
| 到達目標 | 3種類のDBを役割分担させたAPIサーバーをゼロから構築できる |

---

## 完成イメージ

この演習では、**シンプルな商品詳細ページ**を作る。

```
ブラウザ → Node.js API → MariaDB   （商品名・価格・在庫）
                       → MongoDB   （商品スペック詳細）
                       → Redis     （閲覧数カウント）
```

完成すると、ブラウザで以下のような画面が表示される。

```
┌─────────────────────────────────┐
│ ワイヤレスイヤホン Pro           │
│ ￥12,800  (在庫: 35)             │
│                                 │
│ ■ 商品仕様 (MongoDB)            │
│ ・ドライバー径: 10mm            │
│ ・接続: Bluetooth 5.3          │
│                                 │
│              👁️ 閲覧数: 7 回    │
└─────────────────────────────────┘
```

---

## プロジェクト構造（完成形）

```
my-shop/
├── docker-compose.yml       ← Step 1 で作成
├── init/
│   ├── mariadb/
│   │   └── init.sql         ← Step 2 で作成
│   └── mongodb/
│       └── init.js          ← Step 3 で作成
├── package.json             ← Step 4 で作成
├── server.js                ← Step 5〜7 で実装（メイン演習）
└── public/
    └── index.html           ← Step 8 で作成
```

---

## Step 1：docker-compose.yml を作る

作業ディレクトリを作成する。

```bash
mkdir my-shop
cd my-shop
```

`docker-compose.yml` を以下の内容で作成する。

```yaml
services:
  app:
    image: node:20-alpine
    container_name: shop_app
    working_dir: /usr/src/app
    volumes:
      - .:/usr/src/app
    ports:
      - "3000:3000"
    command: sh -c "npm install && node server.js"
    depends_on:
      mariadb:
        condition: service_healthy
      mongodb:
        condition: service_started
      redis:
        condition: service_started
    restart: on-failure

  mariadb:
    image: mariadb:11
    container_name: shop_mariadb
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: shop_db
    volumes:
      - ./init/mariadb:/docker-entrypoint-initdb.d
      - mariadb_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:8.0
    container_name: shop_mongodb
    environment:
      MONGO_INITDB_DATABASE: shop_db
    volumes:
      - ./init/mongodb:/docker-entrypoint-initdb.d
      - mongodb_data:/data/db
    ports:
      - "27017:27017"

  redis:
    image: redis:8.4-alpine
    container_name: shop_redis
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

volumes:
  mariadb_data:
  mongodb_data:
  redis_data:
```

### ポイント解説

| 設定 | 理由 |
|------|------|
| `node:20-alpine` | debian ベースの `node:20-slim` より約50MB軽い |
| `healthcheck` (MariaDB) | MariaDB の起動完了を待ってから app を起動する |
| `depends_on: condition: service_healthy` | ヘルスチェック通過後に起動する設定 |
| `volumes:` (名前付き) | コンテナを削除してもデータが消えない |
| `restart: on-failure` | DB起動前に app が起動して失敗しても自動再起動 |

---

## Step 2：MariaDB の初期データを用意する

ディレクトリを作成する。

```bash
mkdir -p init/mariadb
```

`init/mariadb/init.sql` を以下の内容で作成する。

```sql
-- 商品テーブル
CREATE TABLE IF NOT EXISTS products (
    id    INT          PRIMARY KEY,
    name  VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT          NOT NULL
);

-- サンプルデータ
INSERT INTO products (id, name, price, stock) VALUES
    (101, 'ワイヤレスイヤホン Pro', 12800, 35),
    (102, 'メカニカルキーボード',    8500,  12),
    (103, 'USBハブ 7ポート',         3200,  50);
```

### ポイント解説

`/docker-entrypoint-initdb.d/` にマウントしたファイルは、**コンテナの初回起動時に自動実行**される。
`mariadb_data` ボリュームにデータが既に存在する場合は実行されない。

---

## Step 3：MongoDB の初期データを用意する

```bash
mkdir -p init/mongodb
```

`init/mongodb/init.js` を以下の内容で作成する。

```javascript
db = db.getSiblingDB('shop_db');

db.product_details.insertMany([
    {
        productId: "101",
        specs: {
            "ドライバー径": "10mm",
            "再生周波数": "20Hz - 20kHz",
            "バッテリー": "最大8時間",
            "接続": "Bluetooth 5.3",
            "重量": "5.6g (片耳)"
        }
    },
    {
        productId: "102",
        specs: {
            "スイッチ": "Cherry MX 赤軸",
            "キー数": "104キー",
            "接続": "USB-C / Bluetooth",
            "バックライト": "RGB",
            "重量": "980g"
        }
    },
    {
        productId: "103",
        specs: {
            "ポート数": "7",
            "USB規格": "USB 3.0",
            "最大転送速度": "5Gbps",
            "対応OS": "Windows / Mac / Linux",
            "ケーブル長": "30cm"
        }
    }
]);
```

### ポイント解説

RDBMSと比べて何が違うか意識しながら読む。

| 比較 | MariaDB (init.sql) | MongoDB (init.js) |
|------|-------------------|-------------------|
| 事前定義 | `CREATE TABLE` でスキーマを定義する | スキーマ定義不要、いきなり insert できる |
| データ構造 | 全行が同じカラム構成 | ドキュメントごとにフィールドが異なってもよい |
| 入れ子 | できない（別テーブルに分ける） | オブジェクト・配列をそのまま保存できる |

---

## Step 4：package.json を作成する

`my-shop/` ディレクトリ（`docker-compose.yml` と同じ場所）に `package.json` を以下の内容で作成する。

```json
{
  "name": "my-shop",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4",
    "mongodb": "^6",
    "mysql2": "^3",
    "redis": "^4"
  }
}
```

> `npm install` はコンテナ起動時に `command: sh -c "npm install && node server.js"` で自動実行されるため、ここでは不要。

---

## Step 5：server.js の骨格を作る

`my-shop/` ディレクトリ（`docker-compose.yml` と同じ場所）に `server.js` を以下の内容で作成する。

```javascript
const express = require('express');
const mysql   = require('mysql2/promise');
const { MongoClient } = require('mongodb');
const redis   = require('redis');
const path    = require('path');

const app = express();
app.use(express.static(path.join(__dirname, 'public')));

// ── DB接続セットアップ ──────────────────────────────────────

const mariaPool = mysql.createPool({
    host: 'mariadb',
    user: 'root',
    password: 'rootpassword',
    database: 'shop_db',
    waitForConnections: true
});

const mongoClient = new MongoClient('mongodb://mongodb:27017');

const redisClient = redis.createClient({ url: 'redis://redis:6379' });
redisClient.on('error', err => console.error('Redis error:', err));
redisClient.connect();

// ── APIエンドポイント ────────────────────────────────────────

app.get('/api/product/:id', async (req, res) => {
    const productId = req.params.id;

    try {
        // ============================================================
        // [演習A] MariaDB から商品の基本情報を取得する
        //   - テーブル名: products
        //   - 取得するカラム: name, price, stock
        //   - 条件: id が productId と一致する1件
        // ============================================================
        const baseProduct = null; // ← ここを実装する


        if (!baseProduct) {
            return res.status(404).json({ error: '商品が見つかりません' });
        }

        // ============================================================
        // [演習B] MongoDB から商品スペックを取得する
        //   - データベース: shop_db
        //   - コレクション: product_details
        //   - 条件: { productId: productId } に一致する1件
        // ============================================================
        const detailProduct = null; // ← ここを実装する


        // ============================================================
        // [演習C] Redis で閲覧数をカウントする
        //   - キー: `product:views:${productId}`
        //   - アクセスのたびに +1 する
        //   - 更新後の値を viewCount に代入する
        // ============================================================
        const viewCount = 0; // ← ここを実装する


        res.json({
            id:             productId,
            name:           baseProduct.name,
            price:          baseProduct.price,
            stock:          baseProduct.stock,
            specifications: detailProduct ? detailProduct.specs : {},
            views:          viewCount
        });

    } catch (error) {
        console.error(error);
        res.status(500).json({ error: 'サーバー内部エラーが発生しました' });
    }
});

app.listen(3000, () => console.log('Server running on http://localhost:3000'));
```

---

## Step 6：演習A — MariaDB からデータを取得する

`server.js` の `[演習A]` のブロックを実装する。

**ヒント：mysql2 でのクエリ実行**

```javascript
const [rows] = await mariaPool.query('SELECT ... FROM ... WHERE id = ?', [値]);
const row = rows[0]; // 1件目を取り出す
```

**確認方法：**

```bash
docker compose up -d
curl http://localhost:3000/api/product/101
```

`name`, `price`, `stock` が返ってくれば成功。

```json
{"id":"101","name":"ワイヤレスイヤホン Pro","price":"12800.00","stock":35,...}
```

---

## Step 7：演習B — MongoDB からデータを取得する

`[演習B]` のブロックを実装する。

**ヒント：mongodb ドライバーでのドキュメント取得**

```javascript
const db = mongoClient.db('データベース名');
const doc = await db.collection('コレクション名').findOne({ フィールド: 値 });
```

**確認方法：**

`specifications` にスペック情報が入ってくれば成功。

```json
{
  "specifications": {
    "ドライバー径": "10mm",
    "接続": "Bluetooth 5.3"
  }
}
```

---

## Step 8：演習C — Redis で閲覧数をカウントする

`[演習C]` のブロックを実装する。

**ヒント：Redis のカウンターコマンド**

```javascript
const newCount = await redisClient.incr('キー名');
```

`incr` はキーの値を +1 して、更新後の数値を返す。キーが存在しない場合は 0 から始める。

**確認方法：**

`curl` を何度か実行すると `views` の数字が増えることを確認する。

```bash
curl http://localhost:3000/api/product/101   # views: 1
curl http://localhost:3000/api/product/101   # views: 2
curl http://localhost:3000/api/product/101   # views: 3
```

---

## Step 9：フロントエンドを作る

```bash
mkdir public
```

`public/index.html` を以下の内容で作成する。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>3つのDB連携演習</title>
    <style>
        body  { font-family: sans-serif; background: #f4f7f6; padding: 40px; display: flex; justify-content: center; }
        .card { background: white; padding: 30px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,.1); max-width: 450px; width: 100%; }
        h2    { margin-top: 0; color: #333; border-bottom: 2px solid #ddd; padding-bottom: 10px; }
        .price  { font-size: 1.2em; color: #e44d26; font-weight: bold; }
        .specs  { background: #f9f9f9; padding: 15px; border-radius: 5px; margin: 20px 0; border-left: 4px solid #007bff; }
        .specs p { margin: 5px 0; font-size: 0.9em; }
        .counter { text-align: right; font-size: 0.85em; color: #666; }
    </style>
</head>
<body>
<div class="card">
    <h2 id="name">読み込み中...</h2>
    <p class="price">￥<span id="price">-</span> <small>(在庫: <span id="stock">-</span>)</small></p>
    <div id="specs-area" class="specs"></div>
    <div class="counter">👁️ このページの閲覧数: <strong id="views">-</strong> 回</div>
</div>

<script>
async function fetchProductData(id) {
    try {
        const res  = await fetch(`/api/product/${id}`);
        if (!res.ok) throw new Error('Network error');
        const data = await res.json();

        document.getElementById('name').textContent  = data.name;
        document.getElementById('price').textContent = Number(data.price).toLocaleString();
        document.getElementById('stock').textContent = data.stock;
        document.getElementById('views').textContent = data.views;

        const area = document.getElementById('specs-area');
        area.innerHTML = '<strong>■ 商品仕様 (MongoDB)</strong>';
        if (Object.keys(data.specifications).length === 0) {
            area.innerHTML += '<p>詳細仕様なし</p>';
        } else {
            for (const [key, val] of Object.entries(data.specifications)) {
                area.innerHTML += `<p>・${key}: ${val}</p>`;
            }
        }
    } catch (e) {
        document.getElementById('name').textContent = 'エラー: データの取得に失敗';
        console.error(e);
    }
}

fetchProductData('101');
</script>
</body>
</html>
```

ブラウザで `http://localhost:3000` を開き、商品情報が表示されることを確認する。

---

## Step 10：動作確認まとめ

全体が完成したらコンテナを再起動して確認する。

```bash
docker compose down
docker compose up --build
```

チェックリスト：

- [ ] `curl http://localhost:3000/api/product/101` で JSON が返る
- [ ] `name`, `price`, `stock` が表示される（MariaDB）
- [ ] `specifications` にスペック情報が入っている（MongoDB）
- [ ] `curl` を繰り返すと `views` の数字が増える（Redis）
- [ ] ブラウザで `http://localhost:3000` を開くと画面が表示される

---

## 解答例

<details>
<summary>クリックして解答を表示</summary>

```javascript
// [演習A] MariaDB
const [rows] = await mariaPool.query(
    'SELECT name, price, stock FROM products WHERE id = ?',
    [productId]
);
const baseProduct = rows[0];

// [演習B] MongoDB
const db = mongoClient.db('shop_db');
const detailProduct = await db.collection('product_details').findOne({ productId });

// [演習C] Redis
const viewCount = await redisClient.incr(`product:views:${productId}`);
```

</details>

---

## 発展課題

基本課題が終わったら以下に挑戦する。難易度順に並んでいる。

### 課題1：商品選択UIを追加する（★☆☆）

現在は商品ID `101` が固定されている。`index.html` にボタンを追加して、
101 / 102 / 103 を切り替えられるようにしてみよう。

**ヒント：**

```html
<button onclick="fetchProductData('101')">イヤホン</button>
<button onclick="fetchProductData('102')">キーボード</button>
<button onclick="fetchProductData('103')">USBハブ</button>
```

---

### 課題2：Redis でキャッシュを実装する（★★☆）

現在は毎回 MariaDB と MongoDB に問い合わせている。
Redis を **キャッシュ層**として使い、2回目以降はDBを叩かずに済むようにしてみよう。

**考え方：**

```
リクエスト
  ↓
Redis に `product:cache:${id}` があるか確認
  ├── ある → Redis から返す（DBアクセスなし）
  └── ない → MariaDB + MongoDB から取得 → Redis に保存（TTL: 60秒）
```

**使うコマンド：**

```javascript
// 保存（60秒で自動削除）
await redisClient.set(key, JSON.stringify(data), { EX: 60 });

// 取得
const cached = await redisClient.get(key);
if (cached) return res.json(JSON.parse(cached));
```

---

### 課題3：在庫数を更新するAPIを追加する（★★☆）

`PUT /api/product/:id/stock` というエンドポイントを追加し、
リクエストボディで受け取った数量で MariaDB の `stock` を更新してみよう。

```bash
# 在庫を 30 に更新する
curl -X PUT http://localhost:3000/api/product/101/stock \
     -H "Content-Type: application/json" \
     -d '{"stock": 30}'
```

**ヒント：**

```javascript
app.use(express.json());  // JSONボディを受け取るため先頭に追加

app.put('/api/product/:id/stock', async (req, res) => {
    const { stock } = req.body;
    // UPDATE 文を実装する
});
```

**ヒント②：MariaDB の UPDATE 構文**

```javascript
await mariaPool.query(
    'UPDATE products SET stock = ? WHERE id = ?',
    [stock, productId]
);
```

`?` はプレースホルダー。値は第2引数の配列で順番に渡す（SQLインジェクション対策）。

**ヒント③：更新後の確認**

更新後に `result` として受け取ると、実際に変更された行数が確認できる。

```javascript
const [result] = await mariaPool.query(...);
if (result.affectedRows === 0) {
    return res.status(404).json({ error: '商品が見つかりません' });
}
res.json({ message: '在庫を更新しました', stock });
```

`affectedRows` が 0 のときは存在しないIDが指定されたということ。

**ヒント④：キャッシュの削除を忘れずに**

課題2のキャッシュを実装済みの場合、在庫を更新しても Redis に古いデータが残ってしまう。
更新後にキャッシュも削除しよう。

```javascript
await redisClient.del(`product:cache:${productId}`);
```

---

### 課題4：MongoDB にレビューを追加する（★★★）

`POST /api/product/:id/review` で商品レビューを投稿し、
`GET /api/product/:id` のレスポンスにレビュー一覧を含めてみよう。

```javascript
// 投稿するドキュメントのイメージ
{
    productId: "101",
    user: "alice",
    rating: 5,
    comment: "音質が良くて気に入っています",
    createdAt: new Date()
}
```

RDBMSとの違いを感じるポイント：
- スキーマ変更なしに新しいフィールドを追加できる
- 配列を使って1ドキュメントにレビュー一覧を埋め込む方法もある

---

## トラブルシューティング

### アプリが起動しない

```bash
docker compose logs app
```

ログを確認する。`Cannot find module` は `npm install` が失敗している可能性がある。

```bash
docker compose down
docker compose up --build
```

### MariaDB に接続できない

```bash
docker compose logs mariadb
```

ヘルスチェックが通るまで待つ。初回は30〜60秒かかることがある。

### データが初期化されない（テーブルが空）

ボリュームにデータが残っていると init.sql は再実行されない。

```bash
docker compose down -v   # ボリュームごと削除
docker compose up
```

---

## まとめ

| DB | 役割 | 使ったコマンド/操作 |
|----|------|-------------------|
| MariaDB | 商品の基本情報（構造化データ） | `SELECT` / `UPDATE` |
| MongoDB | 商品スペック（スキーマレス） | `findOne` / `insertMany` |
| Redis | 閲覧カウンター（高速・一時データ） | `INCR` / `SET` with TTL |

3つのDBを使い分けることで、**それぞれの得意分野を活かした設計**ができることを体感できたはず。
発展課題のキャッシュ実装（課題2）は、実際の本番システムでも広く使われているパターンなので、ぜひ挑戦してみよう。
