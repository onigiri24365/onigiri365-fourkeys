# 開発環境（Docker）

**開発環境は Docker で完結します。** ホストマシンに Ruby / Node.js / PostgreSQL / Redis をインストールする必要はありません。

> ⚠️ **ホストで `bundle` / `rails` / `rspec` / `npm` / `npx` を直接実行しないでください。**
> このドキュメントに載っているコマンドはすべて Docker 経由です。

---

## 1. 必要なもの

- Docker Desktop（または Docker Engine + Docker Compose v2）
- Git

以上です。

---

## 2. 初回セットアップ

```bash
# 1. リポジトリを取得
git clone git@github.com:onigiri24365/onigiri365-fourkeys.git
cd onigiri365-fourkeys

# 2. 環境変数ファイルを用意
cp .env.example .env
#    .env を開いて GITHUB_TOKEN を設定する（下記「3. 環境変数」参照）

# 3. イメージをビルド
docker compose build

# 4. 起動
docker compose up -d

# 5. データベースを作成・マイグレーション
docker compose exec api bin/rails db:create db:migrate

# 6. 初期データ（追跡リポジトリ・チーム設定）を投入
docker compose exec api bin/rails db:seed
```

起動確認:

- API: http://localhost:3000/up
- Web: http://localhost:5173

---

## 3. 環境変数

`.env`（`.gitignore` 済み）で設定します。`.env.example` を必ず最新に保ってください。

| 変数 | 必須 | 説明 |
|---|---|---|
| `GITHUB_TOKEN` | yes | GitHub Personal Access Token。必要なスコープ: `repo`（private リポジトリを追跡する場合）または `public_repo` |
| `GITHUB_API_ENDPOINT` | no | GitHub Enterprise を使う場合のみ |
| `DATABASE_URL` | no | 既定でコンテナ内の `db` を指す |
| `REDIS_URL` | no | 既定でコンテナ内の `redis` を指す |
| `TZ` | no | 既定 `Asia/Tokyo` |

> 🔒 **トークンなどの秘匿値をコード・仕様書・コミットに書かないでください。**

---

## 4. 日常の操作

### 起動・停止

```bash
docker compose up -d              # バックグラウンド起動
docker compose ps                 # 状態確認
docker compose stop               # 停止（データは残る）
docker compose down               # 停止 + コンテナ削除（ボリュームは残る）
docker compose down -v            # ボリュームごと削除（DB が消えるので注意）
docker compose restart api        # 個別に再起動
```

### ログ

```bash
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f web
docker compose logs -f            # すべて
```

### シェル・コンソール

```bash
docker compose exec api bash
docker compose exec api bin/rails console
docker compose exec db psql -U postgres onigiri365_fourkeys_development
docker compose exec redis redis-cli
```

---

## 5. データベース

```bash
docker compose exec api bin/rails db:migrate
docker compose exec api bin/rails db:rollback
docker compose exec api bin/rails db:migrate:status
docker compose exec api bin/rails db:seed

# テスト用 DB の準備（スキーマ変更後）
docker compose exec api bin/rails db:test:prepare

# 作り直す（開発データが消える）
docker compose exec api bin/rails db:drop db:create db:migrate db:seed
```

---

## 6. テスト

TDD で開発します（[CLAUDE.md](../CLAUDE.md) の「3. テスト駆動開発」）。

### Rails

```bash
# 全テスト
docker compose exec api bundle exec rspec

# 単体で回す（TDD の内側ループ）
docker compose exec api bundle exec rspec spec/metrics/lead_time_spec.rb
docker compose exec api bundle exec rspec spec/metrics/lead_time_spec.rb:42

# 失敗したものだけ再実行
docker compose exec api bundle exec rspec --only-failures
```

### React

```bash
# 全テスト（1回実行）
docker compose exec web npm test

# watch モード（TDD の内側ループ）
docker compose exec web npm test -- --watch

# 特定ファイル
docker compose exec web npm test -- src/lib/formatDuration.test.ts
```

---

## 7. Lint / 型チェック

```bash
# Ruby
docker compose exec api bundle exec rubocop
docker compose exec api bundle exec rubocop -a        # 自動修正

# TypeScript / React
docker compose exec web npm run lint
docker compose exec web npm run lint -- --fix
docker compose exec web npx tsc --noEmit              # 型チェック
```

---

## 8. 依存パッケージの追加

**必ずコンテナ内で実行し、更新されたロックファイルをコミットします。**

### gem

```bash
docker compose exec api bundle add <gem-name>
docker compose exec api bundle add <gem-name> --group development,test
```

`Gemfile` を直接編集した場合:

```bash
docker compose exec api bundle install
```

ネイティブ拡張を含む gem を追加した場合は再ビルドが必要です。

```bash
docker compose build api && docker compose up -d api
```

### npm パッケージ

```bash
docker compose exec web npm install <package-name>
docker compose exec web npm install -D <package-name>
```

`Gemfile.lock` / `package-lock.json` の変更を**必ずコミットしてください**。

---

## 9. 再ビルドが必要なとき

| 変更したもの | 対応 |
|---|---|
| `Dockerfile` | `docker compose build <service>` |
| `compose.yaml` | `docker compose up -d`（差分が反映される） |
| `Gemfile`（ネイティブ拡張あり） | `docker compose build api` |
| `package.json` | 通常は `npm install` で十分。壊れたら再ビルド |
| アプリのコード | 不要（ボリュームマウントで即反映） |

キャッシュを無視して作り直す:

```bash
docker compose build --no-cache api
```

---

## 10. GitHub データの取り込み

```bash
# 追跡リポジトリのバックフィル（過去分の一括取り込み）
docker compose exec api bin/rails github:backfill[owner/repo]

# 差分取り込みを手動実行
docker compose exec api bin/rails github:sync

# Sidekiq の状態
docker compose logs -f worker
```

> レート制限に注意してください。バックフィルは時間がかかることがあります。
> 進捗は `docker compose logs -f worker` で確認できます。

---

## 11. よくあるトラブル

### ポートが衝突する

`3000` / `5173` / `5432` / `6379` が他のプロセスに使われている場合、`.env` でホスト側のポートを変更します。

```bash
lsof -i :3000     # 使用中のプロセスを確認
```

### `node_modules` が見つからない / 壊れた

`node_modules` は名前付きボリュームで管理しており、ホストの内容と混ざりません。壊れた場合は作り直します。

```bash
docker compose down
docker volume rm onigiri365-fourkeys_web_node_modules
docker compose build web && docker compose up -d
```

### ファイルの所有権が root になる

コンテナ内で生成したファイル（`rails generate` など）が root 所有になる場合があります。

```bash
sudo chown -R "$(id -u):$(id -g)" .
```

`Dockerfile` でホストと同じ UID/GID のユーザーを作ることで回避できます。

### DB に接続できない

`db` の起動完了前に `api` が接続しにいくと起きます。

```bash
docker compose ps                 # db が healthy か確認
docker compose restart api
```

### 変更がブラウザに反映されない

```bash
docker compose logs -f web        # Vite がエラーを出していないか確認
docker compose restart web
```

### ディスク容量が足りない

```bash
docker system df
docker system prune -a            # 未使用のイメージ・コンテナを削除（注意）
```

---

## 12. 完全にリセットする

```bash
docker compose down -v            # コンテナとボリュームを削除
docker compose build --no-cache
docker compose up -d
docker compose exec api bin/rails db:create db:migrate db:seed
```
