# ────────────────────────────────────────
# Stage 1: Next.js ビルド
# ビルドツール（Node.js）を含む一時的なステージ
# ────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# package.json と package-lock.json を先にコピー
# ソースコードより先にコピーすることで、依存関係が変わらない限り
# npm ci のレイヤーがキャッシュされてビルドが速くなる
COPY package*.json ./
RUN npm ci

# ソースコードをコピーしてビルド実行
# next.config.ts の output: 'export' により out/ ディレクトリが生成される
COPY . .
RUN npm run build

# ────────────────────────────────────────
# Stage 2: 本番用 nginx イメージ
# Stage 1 の成果物（out/）だけを受け取る
# Node.js や node_modules は最終イメージに含まれない
# ────────────────────────────────────────
FROM nginx:1.27-alpine

# Stage 1 で生成した静的ファイルだけをコピー
COPY --from=builder /app/out /usr/share/nginx/html

# nginx の設定ファイルをコピー
# ビルドコンテキスト（プロジェクトルート）からの相対パスで指定
COPY docker/nginx/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
