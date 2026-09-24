# Docker & Docker Compose 完全攻略ハンドブック

Dockerの基本概念から各種コマンド、Dockerfile命令、Docker Composeの記法、および実践的な設定例までを網羅した包括的リファレンスガイド

## 1. Dockerの基本概念

* **Image**: アプリケーションと実行に必要な環境（OSレイヤー、ライブラリ、依存関係等）をパッケージ化した「読み取り専用のテンプレート」。
* **Container**: イメージをもとに作成される「実行インスタンス」。分離されたプロセスとしてホストOS上で動作します。
* **Registry**: Dockerイメージを保存・配信する保管庫（例: Docker Hub, Amazon ECR, GitHub Packages）。
* **Volume**: コンテナの破棄後もデータを保持（永続化）するためのホスト上の記憶領域。
* **Network**: コンテナ間、またはコンテナと外部世界との通信を確立するための仮想ネットワーク

---

## 2. Docker CLI コマンドリファレンス

### イメージ操作
* **`docker build -t <image-name>:<tag> <path>`**
  指定したディレクトリのDockerfileからイメージをビルドします。
  ```bash
  docker build -t my-app:v1.0 .
  ```
* **`docker images` / `docker image ls`**
  ローカルに存在するDockerイメージの一覧を表示します。
* **`docker pull <image-name>:<tag>`**
  レジストリからイメージをダウンロードします。
  ```bash
  docker pull postgres:16-alpine
  ```
* **`docker push <image-name>:<tag>`**
  ローカルのイメージをレジストリに送信します。
* **`docker rmi <image-id>` / `docker image rm <image-id>`**
  不要なイメージを削除します。
* **`docker history <image-name>`**
  イメージのレイヤー構造と生成履歴を確認します。

### コンテナ操作
* **`docker run [options] <image-name> [command]`**
  イメージから新しいコンテナを作成し、起動します。
  * 主なオプション:
    * `-d`: バックグラウンドで実行（デタッチドモード）
    * `-p <host-port>:<container-port>`: ポートマッピング
    * `-v <host-path>:<container-path>`: ボリュームマウント
    * `--name <container-name>`: コンテナ名を指定
    * `-e <key>=<value>`: 環境変数の設定
    * `--rm`: コンテナ停止時に自動削除
    * `-it`: 端末入力を有効化（シェル操作用）
  ```bash
  docker run -d -p 8080:80 --name my-web-server nginx:alpine
  ```
* **`docker ps` / `docker container ls`**
  稼働中のコンテナ一覧を表示します（`-a` オプションで停止中のコンテナも表示）。
* **`docker stop <container-id/name>`**
  実行中のコンテナを安全に停止（SIGTERM）
* **`docker start <container-id/name>`**
  停止しているコンテナを再起動
* **`docker restart <container-id/name>`**
  コンテナを再起動
* **`docker rm <container-id/name>`**
  停止中のコンテナを削除（`-f` で実行中コンテナの強制削除）
* **`docker exec -it <container-id/name> <command>`**
  実行中のコンテナ内部でコマンドを実行
  ```bash
  docker exec -it my-web-server sh
  ```
* **`docker logs [options] <container-id/name>`**
  コンテナのログを出力（`-f` でリアルタイム追跡、`--tail 100` で直近ログ指定）。

### ネットワーク管理
* **`docker network ls`**: ネットワーク一覧の表示。
* **`docker network create <network-name>`**: 新しい仮想ネットワークを作成。
* **`docker network connect <network-name> <container-name>`**: コンテナをネットワークに接続。

### ボリューム管理
* **`docker volume ls`**: 作成されているボリュームの一覧表示。
* **`docker volume create <volume-name>`**: 明示的なNamed Volumeの作成。
* **`docker volume inspect <volume-name>`**: ボリュームの詳細情報（ホスト上の実体パス等）を確認。

### システム・クリーンアップ
* **`docker system df`**: Dockerが使用しているディスク容量の確認。
* **`docker system prune`**: 未使用のデータ（停止中のコンテナ、孤立したイメージ、未使用のネットワーク）を一括削除。
  ```bash
  docker system prune -a --volumes # 完全クリーンアップ（注意して実行）
  ```

---

## 3. Dockerfile 記述ガイド＆主要命令

### 命令一覧と解説

| 命令 | 用途 | 記述例 |
|---|---|---|
| **`FROM`** | ベースイメージを指定（必須）。最初に記述。 | `FROM node:20-alpine` |
| **`WORKDIR`** | 以降の命令を実行する作業ディレクトリを指定。 | `WORKDIR /usr/src/app` |
| **`COPY`** | ホスト側のファイル/フォルダをコンテナ内にコピー。 | `COPY package*.json ./` |
| **`ADD`** | `COPY`と同等＋URLダウンロードやtar圧縮自動解凍。 | `ADD archive.tar.gz /data/` |
| **`RUN`** | イメージ**ビルド時**に実行されるコマンド（パッケージ導入等）。 | `RUN npm install --production` |
| **`ENV`** | コンテナ内の環境変数を設定。 | `ENV NODE_ENV=production` |
| **`ARG`** | ビルド時に受け取る一時的な引数を定義。 | `ARG BUILD_VERSION=1.0.0` |
| **`EXPOSE`** | コンテナが開示するポート番号を提示（ドキュメント的意味合い）。 | `EXPOSE 3000` |
| **`VOLUME`** | データ永続化領域（マウントポイント）の作成。 | `VOLUME ["/data"]` |
| **`USER`** | 以降の処理を実行するセキュリティユーザーの指定。 | `USER node` |
| **`CMD`** | **コンテナ起動時**に実行するデフォルト命令（上書き可能）。 | `CMD ["node", "server.js"]` |
| **`ENTRYPOINT`** | コンテナ起動時に**必ず実行**される固定コマンド。 | `ENTRYPOINT ["docker-entrypoint.sh"]` |

### マルチステージビルドパターン

```dockerfile
# --- ステージ1: ビルド環境 ---
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# --- ステージ2: 実行環境 ---
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
# ビルドステージから成果物のみをコピー
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

## 4. Docker Compose 活用ガイド

### 基本概念と用語
* **Service**: アプリケーションの各コンポーネント（例: web, db, redis）。コンテナの設定情報を指します。
* **Project**: Composeファイル内で定義されている複数サービスをまとめた全体の単位。

### docker-compose.yml 主要キー解説

* **`version`**: Composeファイルのバージョン（※V2以降は非推奨となり省略可）。
* **`services`**: コンテナ定義の親要素。
* **`build`**: Dockerfileがあるパス、または詳細設定（`context`, `dockerfile`）。
* **`image`**: 利用するDockerイメージの指定（ビルドしない場合、またはビルド後のタグ名）。
* **`ports`**: ポートフォワーディングの設定。`"ホストポート:コンテナポート"` の形式。
* **`volumes`**: ディレクトリのマウント設定。
  * バインドマウント: `./local-dir:/app/dir`
  * ボリュームマウント: `db_data:/var/lib/postgresql/data`
* **`environment`**: コンテナに与える環境変数を記述。
* **`env_file`**: `.env` などの環境変数ファイルを直接読み込む。
* **`depends_on`**: コンテナの起動順序の依存関係を制御。
* **`restart`**: コンテナの再起動ルール（`no`, `always`, `on-failure`, `unless-stopped`）。
* **`networks`**: 接続する仮想ネットワークを指定。

### よく使うComposeコマンド

```bash
# サービスのバックグラウンド起動（必要に応じて再ビルド）
docker compose up -d --build

# サービスとネットワーク、ボリュームの停止・削除
docker compose down -v

# 起動状況・ステータスの確認
docker compose ps

# 複数コンテナのログをまとめてリアルタイム確認
docker compose logs -f [service_name]

# 実行中サービス内でのコマンド実行
docker compose exec web sh

# 設定ファイルの構文チェックとレンダリング確認
docker compose config
```

---

## 5. 【実践例】Node.js + PostgreSQL + Nginx構成

以下は、Web API（Node.js）、データベース（PostgreSQL）、リバースプロキシ（Nginx）で構成される一般的なWebサービスのテンプレートです。

### 1. ディレクトリ構造

```text
my-project/
├── docker-compose.yml
├── .env
├── app/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
└── nginx/
    └── default.conf
```

### 2. Dockerfileの設定 (`app/Dockerfile`)

```dockerfile
FROM node:20-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### 3. Nginxの設定 (`nginx/default.conf`)

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://web:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 4. Node.js アプリケーションの構築 (`app/server.js`)

```javascript
const express = require('express');
const { Pool } = require('pg');

const app = express();
const port = 3000;

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB,
});

app.get('/', async (req, res) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({ message: 'Connected to DB!', time: result.rows[0].now });
  } catch (err) {
    console.error(err);
    res.status(500).send('Database error');
  }
});

app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

### 5. docker-compose.yml の記述

```yaml
services:
  # リバースプロキシ
  proxy:
    image: nginx:alpine
    container_name: app_proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web
    networks:
      - app-network

  # Node.js アプリケーション
  web:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: app_web
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
    volumes:
      - ./app:/usr/src/app
      - /usr/src/app/node_modules
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  # PostgreSQL データベース
  db:
    image: postgres:16-alpine
    container_name: app_db
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

# 永続ボリュームの宣言
volumes:
  postgres_data:

# ネットワークの宣言
networks:
  app-network:
    driver: bridge
```

---

## 6. よく使うベストプラクティス＆トラブルシューティング

1. **キャッシュの有効活用**
   * Dockerfile内では変更頻度の低いもの（例: パッケージ定義ファイルコピーと `npm install`）を上位に記述し、ソースコードの全コピーは後ろに回します。
2. **軽量化（マルチステージビルドとAlpineイメージの活用）**
   * `alpine` や `slim` タグのベースイメージを利用し、実行用ステージから不要なビルドツールを除外します。
3. **`.dockerignore` の活用**
   * `.git` や `node_modules`, ログファイルなど不要なファイルを `.dockerignore` に記述し、ビルドコンテキストを軽量に保ちます。
4. **パーミッション問題**
   * コンテナ内での `root` ユーザー実行を避け、専用のサービスユーザー（例: `USER node`）を利用します。
5. **コンテナがすぐ停止してしまう場合**
   * 前景（フォアグラウンド）で実行されるプロセスが無いとコンテナは終了します。`CMD` にバックグラウンド化しないコマンドを指定しているか確認してください。