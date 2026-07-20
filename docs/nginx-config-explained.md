# nginx 設定解説

対象ファイル: `docker/nginx/nginx.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri.html $uri/ /index.html;
    }

    location /_next/static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## nginx とは

nginx は **Web サーバー**。ブラウザからの HTTP リクエストを受け取り、ファイルやサービスを返す役割を担う。

このプロジェクトでの流れ：

```
ブラウザ
  │  http://localhost/about  (リクエスト)
  ▼
nginx (ポート80で待機)
  │  /usr/share/nginx/html/about.html を探して返す
  ▼
ブラウザ (HTML を受け取って表示)
```

---

## 各設定の解説

### `listen 80;`

ポート `80` で HTTP 接続を待ち受ける宣言。HTTP の標準ポートは `80` なので、ブラウザは `http://localhost/` へのアクセス時に自動的に `:80` に接続する。

---

### `server_name localhost;`

このサーバーブロックが担当するドメイン名を指定する。ブラウザが送ってくる `Host` ヘッダーと照合し、どのサーバーブロックで処理するかを決める。

nginx は1台で複数のサーバーブロック（バーチャルホスト）を持てるため、「ポート × server_name」の組み合わせでリクエストを振り分ける。

**本番環境（ECS など）での考慮点：**

| 環境 | `server_name` の値 |
|---|---|
| ローカル開発 | `localhost` |
| 本番（nginx が直接ドメインを受ける） | `example.com` |
| 本番（ALB の後ろにいる） | `_`（なんでも受け付ける）|

ALB がリクエストをコンテナに転送するとき `Host` ヘッダーが書き換わるケースがあるため、ALB の後ろでは `_` を使うのが実用的。

---

### `root /usr/share/nginx/html;`

ファイルを探す際のルートディレクトリを宣言する。`docker-compose.yml` で `./out` をこのパスにマウントしているため、`next build` が生成したファイルが見つかる。

```
ホスト側 (./out/)               コンテナ内 (/usr/share/nginx/html/)
  ├── index.html          =       ├── index.html
  ├── about.html          =       ├── about.html
  └── _next/static/...    =       └── _next/static/...
```

マウントはファイルのコピーではなく「同じ実体を別名で見ている」状態。nginx は `/usr/share/nginx/html` を読んでいるつもりで、実際にはホストの `./out` を読んでいる。

**nginx の設定は `./out` を直接知らない。** コンテナ内のパスだけを参照する。

---

### `index index.html;`

ディレクトリへのリクエストが来たとき、デフォルトとして返すファイルを指定する。`/` へのアクセス時に `/index.html` を返すために必要。

---

### `location / { try_files ... }`

```nginx
location / {
    try_files $uri $uri.html $uri/ /index.html;
}
```

すべてのパスへのリクエストに対し、左から順にファイルを探し、最初に見つかったものを返す。

| 変数 | 意味 | 例: `/favicon.ico` | 例: `/about` |
|---|---|---|---|
| `$uri` | そのままのパス | `/favicon.ico` → **見つかる** | `/about` → ない |
| `$uri.html` | `.html` を付けたパス | `/favicon.ico.html` → ない | `/about.html` → **見つかる** |
| `$uri/` | ディレクトリの index.html | — | `/about/index.html` → ない |
| `/index.html` | 最終フォールバック | — | `/index.html` を返す |

**`$uri.html` が必要な理由：** `next build` の `output: "export"` モードは `about.html` というファイルを生成するが、ブラウザは `/about`（拡張子なし）でアクセスする。この変換を `$uri.html` が橋渡しする。

**`/index.html` フォールバックが必要な理由：** ブラウザで直接 `/user/123` を開いたとき、そのファイルが存在しなくても `index.html` を返して React を起動させ、JS 側でルーティングを処理させる（SPA パターン）。

---

### `location /_next/static/ { ... }`

```nginx
location /_next/static/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

`/_next/static/` 以下の JS・CSS ファイルに長期キャッシュを設定する。`location /` よりも具体的なパスなので、こちらのブロックが優先して適用される（`try_files` は実行されない）。

#### `expires 1y`

ブラウザキャッシュの有効期限を1年に設定する。

#### `Cache-Control: public`

CDN・プロキシも含めてキャッシュして良いと宣言する。

#### `Cache-Control: immutable`

「このURLのコンテンツは絶対に変わらない（再検証不要）」と宣言する。

**`immutable` なしの場合（`expires 1y` のみ）：**

```
ユーザーがリロード
  ↓
ブラウザ: サーバーに If-None-Match で確認
  ↓
nginx: 304 Not Modified
  ↓
ネットワーク通信が発生する（中身は返さないが往復は発生）
```

**`immutable` ありの場合：**

```
ユーザーがリロード
  ↓
ブラウザ: 「絶対変わらないと宣言されている。問い合わせ不要」
  ↓
ネットワーク通信ゼロ
```

---

## なぜ `immutable` を安全に使えるのか

Next.js はビルド時にファイル名にコンテンツのハッシュ値を含める：

```
_next/static/chunks/main-abc123.js   ← abc123 がハッシュ
```

**ハッシュの保証：**

```
中身が同じ → ハッシュが同じ → ファイル名が同じ
中身が違う → ハッシュが違う → ファイル名が違う（= 別のURL）
```

「ファイル名が同じ = 中身が同じ」が成立するため、`immutable` を安全に宣言できる。

**Next.js 以外での注意点：** ハッシュなしで固定ファイル名を使っている場合、デプロイ後もブラウザが古いキャッシュを使い続けるデプロイ事故が起きる。`immutable` は「ファイル名が変われば必ず別物」という仕組みとセットで初めて安全。

---

## デプロイ時のキャッシュ更新の仕組み

```
① npm run build → ./out/以下にハッシュ付きファイル生成
② nginx に新しい ./out/ をデプロイ
③ ブラウザが index.html を取得（index.html 自体はキャッシュなし）
④ index.html の script タグに新しい URL が書かれている
⑤ キャッシュにないURLなので新しい JS を取得する
```

`index.html` が「最新のURLを教える案内板」として機能する。ブラウザは案内板を毎回読み、そこに書かれたURLをキャッシュと照合するだけで、HTMLの差分比較は行わない。

---

## docker-compose.yml との対応

```yaml
nginx:
  image: nginx:1.27-alpine
  ports:
    - "80:80"                                             # ホストの80番 → コンテナの80番
  volumes:
    - ./out:/usr/share/nginx/html:ro                      # ビルド成果物をマウント
    - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro  # 設定を注入
```

**`:ro` (read-only) について：**
- nginx はファイルを読むだけで書く必要がない
- 万が一コンテナが侵害されても、ホスト側の `./out` を改ざんできない
- セキュリティと意図の明示が目的

**`/etc/nginx/conf.d/default.conf` について：**
nginx のメイン設定 (`nginx.conf`) は `conf.d/*.conf` を自動で読み込む。デフォルトの `default.conf` を上書きすることで、独自の設定を注入している。

---

## ブラウザキャッシュと容量について

「古いハッシュのファイルが1年間残るのでは？」という懸念に対して：

- ブラウザキャッシュには上限サイズ（Chrome: 数GB、Firefox: 数百MB〜1GB）があり、いっぱいになると古いものから自動削除（LRU方式）される
- JS ファイルは数KB〜数百KB程度でキャッシュ容量の誤差レベル
- `expires 1y` は「1年間ディスクに保持する命令」ではなく「1年間は古くならないのでサーバーに確認不要」という意味
- `immutable` によるネットワーク通信の削減メリットの方が遥かに大きい
