# docker-compose.yml 完全解説

## 全体構造

```yaml
services:   # 起動するコンテナの定義
networks:   # コンテナ間の仮想ネットワーク
volumes:    # データの永続化領域
```

docker-compose.yml は「複数のコンテナをまとめて定義・起動するための設計図」。  
この構成では **3つのコンテナ**（postgres・keycloak・nginx）が連携している。

---

## services.postgres

```yaml
postgres:
  image: postgres:16-alpine
  env_file: .env.docker
  volumes:
    - postgres_data:/var/lib/postgresql/data
  networks:
    - backend
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-keycloak}"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### 各キーの意味

| キー | 意味 |
|---|---|
| `image: postgres:16-alpine` | DockerHubから引っ張るイメージ。`16`がバージョン、`alpine`は軽量Linux |
| `env_file: .env.docker` | `.env.docker`から環境変数を読み込む。`POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` を注入 |
| `volumes` | コンテナの `/var/lib/postgresql/data`（DBファイルの保存場所）を `postgres_data` ボリュームにマウント |
| `networks: - backend` | このコンテナを `backend` ネットワークのみに接続。ブラウザからの直接アクセス不可 |
| `healthcheck` | コンテナが「起動した」だけでなく「DBとして使える状態か」を確認する仕組み |

### healthcheck の詳細

| キー | 意味 |
|---|---|
| `test` | `pg_isready` コマンドでPostgreSQLの接続受付を確認 |
| `interval: 10s` | チェック間隔 |
| `timeout: 5s` | タイムアウト |
| `retries: 5` | 何回失敗したら `unhealthy` と判定するか |

これが重要な理由：KeycloakはDBが使えない状態で起動するとクラッシュする。後述の `depends_on` と組み合わせて「DB準備完了まで待機」を実現している。

### volumes の理解

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
#   ↑ ボリューム名        ↑ コンテナ内のパス
```

「`/var/lib/postgresql/data` のデータをボリュームに退避する」ではなく、  
**「`postgres_data` という場所を `/var/lib/postgresql/data` としてコンテナにマウントする」** が正確な方向。

| 操作 | データの扱い |
|---|---|
| `docker compose restart` | ボリュームはそのまま → **残る** |
| `docker compose down` → `docker compose up` | ボリュームはそのまま → **残る** |
| `docker compose down -v` | ボリュームごと削除 → **消える** |
| `docker volume rm postgres_data` で手動削除 | → **消える** |

---

## services.keycloak

```yaml
keycloak:
  image: quay.io/keycloak/keycloak:26.0
  env_file: .env.docker
  environment:
    KC_DB: postgres
    KC_DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-keycloak}
    KC_DB_USERNAME: ${POSTGRES_USER:-keycloak}
    KC_DB_PASSWORD: ${POSTGRES_PASSWORD:-keycloak_local_password}
    KC_HTTP_ENABLED: "true"
    KC_HOSTNAME_STRICT: "false"
  command: start-dev
  ports:
    - "8080:8080"
  depends_on:
    postgres:
      condition: service_healthy
  networks:
    - backend
    - frontend
```

### image

`quay.io` はRedHat系のコンテナレジストリ。DockerHubではなくこちらが公式。

### env_file と environment を分けている理由

- **`env_file`** に書くのは「生のシークレット（パスワード等）」。postgres と keycloak の両方が同じ `.env.docker` を参照することでパスワードを1箇所だけで管理できる。
- **`environment`** に書くのは「それらを組み合わせて構築した値」。`KC_DB_URL` のようにホスト名・ポート・DB名を連結した文字列はKeycloak固有の知識で構築する必要があり、ymlに明示したほうが意図が伝わりやすい。

### environment の各キー（DB接続設定）

```yaml
KC_DB: postgres
KC_DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-keycloak}
KC_DB_USERNAME: ${POSTGRES_USER:-keycloak}
KC_DB_PASSWORD: ${POSTGRES_PASSWORD:-keycloak_local_password}
```

KeycloakはデフォルトでH2というインメモリDBを使う。コンテナを止めたらデータが消える。  
それを防ぐために「外部のPostgreSQLを使え」と明示的に指示する必要がある。

#### JDBC URLの解剖

```
jdbc:postgresql://postgres:5432/${POSTGRES_DB:-keycloak}
```

| 部分 | 意味 |
|---|---|
| `jdbc:` | プロトコルの種類。JavaアプリがDBに接続する際の規格（Java Database Connectivity） |
| `postgresql:` | DBの種類。MySQLなら `mysql:`、Oracleなら `oracle:thin:` になる |
| `//postgres` | ホスト名。DockerネットワークのDNS名 = サービス名 `postgres` |
| `:5432` | ポート番号。PostgreSQLのデフォルトポート |
| `/${POSTGRES_DB:-keycloak}` | データベース名 |

**JDBCとは：** JavaアプリとDBの差を吸収する標準インターフェース。JDBCがあることでDBを替えてもアプリ側のコードをほぼ変えなくてよい。フロントエンドで言うなら `fetch` がブラウザ差異を吸収するイメージに近い。

```
# Prisma（Node.js）の接続文字列
postgresql://user:password@localhost:5432/dbname

# JDBC（Java）の接続文字列
jdbc:postgresql://localhost:5432/dbname
```

**なぜホスト名が `postgres` なのか：** Docker Composeでは同じネットワーク内のサービスはサービス名でDNS解決できる。サービス名を `db` に変えたらURLも `jdbc:postgresql://db:5432/...` になる。

#### `${VAR:-default}` 構文

```
${POSTGRES_DB:-keycloak}
```

「環境変数 `POSTGRES_DB` が定義されていればその値を使い、未定義なら `keycloak` をフォールバック値として使う」という変数展開。

#### .env.docker との連動

```
.env.docker
  POSTGRES_USER=keycloak
  POSTGRES_PASSWORD=secret
  POSTGRES_DB=keycloak
       ↓ env_file で両コンテナに注入
┌─────────────────┐        ┌──────────────────────────┐
│ postgres コンテナ │        │ keycloak コンテナ          │
│                 │        │                          │
│ ユーザー: keycloak│ ←接続─ │ KC_DB_URL: ...//postgres  │
│ パスワード: secret│        │ KC_DB_USERNAME: keycloak  │
│ DB名: keycloak  │        │ KC_DB_PASSWORD: secret    │
└─────────────────┘        └──────────────────────────┘
```

`.env.docker` の1箇所を変えれば、postgresの初期ユーザー作成とKeycloakの接続設定が同時に連動して変わる設計。

### environment のその他のキー

```yaml
KC_HTTP_ENABLED: "true"
KC_HOSTNAME_STRICT: "false"
```

本番のKeycloakはHTTPS必須・ホスト名の厳格検証がデフォルト。ローカル開発でHTTPを使えるようにこれらを緩和している。

### command

```yaml
command: start-dev
```

Dockerイメージのデフォルト起動コマンドを上書き。`start-dev` は開発モード（ホットリロード・詳細ログあり）。**本番では `start` を使う。**

### ports

```yaml
ports:
  - "8080:8080"
```

`ホストのポート:コンテナのポート` のマッピング。`localhost:8080` でKeycloak管理コンソールにアクセスできるようになる。`ports` がない場合、コンテナ外（ホストのブラウザ）からはアクセスできない。

### depends_on

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

`condition: service_healthy` により、postgresの `healthcheck` が **healthy になるまで** keycloakは起動しない。`service_started`（コンテナが起動したら即OK）よりも厳格。

### networks

```yaml
networks:
  - backend   # postgresと通信
  - frontend  # ブラウザ経由のOIDCリダイレクト
```

keycloakだけが両ネットワークに参加している。nginxが配信するフロントエンドからOIDCのリダイレクトを受け取る必要があるため `frontend` にも接続している。

---

## services.nginx

```yaml
nginx:
  image: nginx:1.27-alpine
  ports:
    - "80:80"
  volumes:
    - ./out:/usr/share/nginx/html:ro
    - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
  networks:
    - frontend
  depends_on:
    - keycloak
```

### ports

`localhost:80` でアクセスできる。ブラウザで `http://localhost` とだけ打てばアクセスできるのはポート80がデフォルトだから。

### volumes の2つのマウント

| マウント | 目的 |
|---|---|
| `./out:/usr/share/nginx/html:ro` | `next build` で生成された静的ファイル（HTML/CSS/JS）をnginxの配信ルートに置く |
| `./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro` | nginxの設定ファイルを注入する |

末尾の `:ro` は **read-only**。コンテナ内からファイルを書き換えられないようにするセキュリティ設定。

### depends_on

`condition` の指定がないので `service_started`（起動したら即OK）扱い。postgresほどシビアな依存関係ではないため。

---

## networks

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

### `driver: bridge`

Dockerのデフォルトネットワーク方式。ホストOSと分離された仮想ネットワーク。

### なぜ2つに分けるのか

```
[ブラウザ] → nginx (frontend) → keycloak (frontend & backend) → postgres (backend)
```

`backend` ネットワークには `keycloak` と `postgres` だけが参加している。`nginx` は `backend` に参加していないので、nginx→postgres への直接通信は物理的に不可能。最小権限の原則。

### ネットワーク分離の仕組み

**「backend」という名前が特別なのではない。** 名前は単なるラベル。分離を生んでいるのは：

1. **`postgres` に `ports` がない** → ホストOS（localhost）からアクセスできない
2. **`nginx` と `postgres` が異なるネットワークにいる** → コンテナ間通信もできない

この2つの組み合わせによってpostgresが保護されている。

### コンテナ間通信と ports の関係

`ports` は「ホストOS ↔ コンテナ」の穴あけであり、**コンテナ間通信には無関係**。  
同じDockerネットワークに属していれば、`ports` 宣言なしに内部ポートへ直接アクセスできる。

```
# nginxからpostgresへの接続を許可したい場合に必要なのは
# ports の追加ではなく、同じネットワークへの参加だけ

nginx:
  networks:
    - frontend
    - backend   # ← これだけ追加すればOK（ports追加は不要）
```

---

## volumes（トップレベル）

```yaml
volumes:
  postgres_data:
```

**名前付きボリューム**の宣言。`docker volume ls` で確認でき、`docker compose down` してもデータが残る（`docker compose down -v` で削除される）。サービス定義内の `- postgres_data:/var/...` と対応している。

---

## 全体の通信フロー

```
[ブラウザ]
    │
    ├─ localhost:80   → nginx   (frontendネットワーク)
    │                    ↓ 静的ファイル配信
    │
    └─ localhost:8080 → keycloak (frontend & backendネットワーク)
                            ↓ JDBC接続
                        postgres (backendネットワーク)
```

nginx は静的ファイルを返すだけ。認証はブラウザが直接keycloakと通信する（OIDCのリダイレクトフロー）。postgresは外部から完全に隔離。

---

## セキュリティに関する補足

### DB名・ユーザー名にKeycloakという名前を使うことについて

このPostgreSQLはKeycloak専用インスタンスのため、名前から用途が明確に読み取れるという意味で合理的。  
ただし **PostgreSQLを複数サービスで共有する場合は問題になる**。

| 観点 | 評価 |
|---|---|
| Keycloakという名前をDB名・ユーザー名に使う | 専用インスタンスなら問題なし |
| パスワードの強度 | ローカル限定なら許容、本番流用は危険 |
| 将来Next.jsアプリもDBを使いたくなったとき | その時点で設計を見直す必要がある |

### 本当に改善すべき点

```dotenv
POSTGRES_PASSWORD=keycloak_local_password  # 平文・単純すぎる
KEYCLOAK_ADMIN_PASSWORD=admin_local_password  # "admin"は推測されやすい
```

コメントに「本番環境では Docker Secrets や環境変数管理ツールを使用すること」とあるが、CIで `.env.docker` の存在チェックを入れるなどの本番流用防止策も検討が必要。
