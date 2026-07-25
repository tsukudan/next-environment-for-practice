# Dockerfile 解説

## 全体構造：マルチステージビルド

`FROM` が複数あることが **マルチステージビルド** の定義。
`FROM` が来るたびに全く新しいファイルシステムがゼロから始まり、前のステージのファイルは `COPY --from=<ステージ名>` で明示的に取り出さない限り引き継がれない。

```
FROM node:20-alpine AS builder   # ← ステージ1
...

FROM nginx:1.27-alpine           # ← ステージ2（新しいFROM = 新しいステージ）
...
```

---

## Stage 1: ビルドステージ（`builder`）

```dockerfile
FROM node:20-alpine AS builder
```

Node.js 20 が入った Alpine Linux（軽量）イメージを使う。`AS builder` で名前をつけ、後のステージから参照できるようにする。

---

```dockerfile
WORKDIR /app
```

コンテナ内の作業ディレクトリを `/app` に設定。以降の `COPY` や `RUN` はここが起点になる。

> **注意**: ここでいう `/app` はコンテナ内のパスであり、プロジェクトの `app/` ディレクトリとは別物。
> - `WORKDIR /app` → コンテナのファイルシステム上の作業ディレクトリ（名前は慣習で `/app` にしているだけ）
> - プロジェクトの `app/` → Next.js の App Router のソースコード

`COPY . .` 実行後、コンテナ内の構造は以下のようになる：

```
/app/                     ← WORKDIR（コンテナ内）
  ├── app/                ← ホストの app/ がここにコピーされてくる
  │    ├── page.tsx
  │    └── layout.tsx
  ├── package.json
  ├── next.config.ts
  └── ...
```

---

```dockerfile
COPY package*.json ./
RUN npm ci
```

**意図的に2ステップに分けている**（レイヤーキャッシュの活用）。

### レイヤーキャッシュとは

Dockerfile の各命令（`FROM`, `COPY`, `RUN` など）は **レイヤー** という差分ファイル層を1つずつ生成する。

```
レイヤー1: FROM node:20-alpine
レイヤー2: WORKDIR /app
レイヤー3: COPY package*.json ./      ← package.json が変わったか？
レイヤー4: RUN npm ci                 ← 変わってなければキャッシュ使用
レイヤー5: COPY . .                   ← ソースコードが変わったか？
レイヤー6: RUN npm run build
```

ビルド時、Docker は **上から順に「前回と同じか？」を検査** する。あるレイヤーが変化していれば、それ以降は全部再実行される。

```
# ソースコードだけ変更した場合
レイヤー3: COPY package*.json ./  → 変化なし → キャッシュ流用 ✓
レイヤー4: RUN npm ci             → キャッシュ流用 ✓（何分もかかる処理をスキップ）
レイヤー5: COPY . .               → 変化あり → 再実行
レイヤー6: RUN npm run build      → 再実行
```

もし `COPY . .` を先に書いていたら、ソースコード1行変えるだけで毎回 `npm ci` が走ってしまう。**順番に意図がある**のはそのため。

---

```dockerfile
COPY . .
RUN npm run build
```

### `COPY . .` の意味

```
COPY <コピー元> <コピー先>
       ↑           ↑
       .           .
  ホスト側      コンテナ側
  (ビルド     (WORKDIR で
  コンテキスト)  指定した /app)
```

- 左の `.` = `docker build` を実行したディレクトリ（プロジェクトルート）
- 右の `.` = コンテナ内の現在地（`WORKDIR /app` なので `/app`）

「ホストのプロジェクト全体を、コンテナの `/app/` 以下にコピーせよ」という意味。

### `RUN npm run build`

`WORKDIR /app` を設定しているため、`/app` 直下で `npm run build` を実行することになる。ターミナルで言えば以下と同義：

```bash
cd /app
npm run build
```

`next.config.ts` に `output: "export"` が設定されているため、ビルド結果は `/app/out/` に純粋な静的ファイル（HTML/CSS/JS）として出力される。Node.js サーバーは不要になる。

---

## Stage 2: 本番ステージ（nginx）

```dockerfile
FROM nginx:1.27-alpine
```

軽量な nginx イメージを新たに起動。**ここからは全く別のコンテナ** で、Node.js も `node_modules`（数百MB）も含まれない。

---

```dockerfile
COPY --from=builder /app/out /usr/share/nginx/html
```

Stage 1 の成果物（`out/` の静的ファイル）だけを nginx の配信ディレクトリにコピー。`--from=builder` がステージ間のファイル受け渡しのポイント。ビルド成果物のパスが `/app/out` になっているのは、`WORKDIR` が `/app` だったため。

---

```dockerfile
COPY docker/nginx/nginx.conf /etc/nginx/conf.d/default.conf
```

`nginx.conf` を配置。`try_files $uri $uri.html $uri/ /index.html;` により、Next.js のファイルベースルーティング（`/about` → `about.html`）と、404 時の SPA フォールバックを実現している。

---

```dockerfile
EXPOSE 80
```

コンテナがポート 80 を使うことを Docker に宣言（実際の公開は `docker-compose.yml` の `ports` 設定が行う）。

---

## なぜマルチステージにするのか

| | ビルドステージ | 本番ステージ |
|---|---|---|
| 含まれるもの | Node.js, npm, node_modules, ソースコード | nginx バイナリ, 静的ファイルのみ |
| サイズ感 | 数百〜1GB 超 | 数十 MB |
| 本番イメージに含まれる | **No** | **Yes** |

本番イメージを小さく・安全に保つのがマルチステージビルドの目的。攻撃面積（ビルドツール、ソースコードの漏洩など）も減らせる。
