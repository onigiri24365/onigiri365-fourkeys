# CLAUDE.md

このファイルは、本リポジトリで作業する Claude Code への指示書です。**作業開始時に必ず全文を読んでください。**

---

## 1. プロジェクト概要

**onigiri365-fourkeys** は、GitHub のコミットと Pull Request を元に、開発チームの **Four Keys（DORA 指標）** と **サイクルタイム** を、組織全体・チーム別に可視化する社内向けウェブアプリケーションです。

- **利用者**: 社内の開発チームメンバー、エンジニアリングマネージャー
- **目的**: デリバリーパフォーマンスを定量的に把握し、ボトルネック（特にレビュー待ち・デプロイ待ち）を特定する
- **前提**: 社内ネットワーク限定で運用するため、**MVP ではアプリケーション認証を実装しません**

---

## 2. 最重要ルール：仕様書駆動開発（Spec-Driven Development）

> **仕様書なしにアプリケーションコードを書き始めてはいけません。**

すべての機能追加・変更は `docs/specs/<NNNN>-<feature-slug>/` 配下の仕様書から始めます。

### 2.1 3つのフェーズと承認ゲート

| # | フェーズ | 成果物 | 次へ進む条件 |
|---|---|---|---|
| 1 | 要件定義 | `requirements.md` | **ユーザーの承認** |
| 2 | 設計 | `design.md` | **ユーザーの承認** |
| 3 | タスク分解 | `tasks.md` | **ユーザーの承認** → 実装開始 |

各フェーズの終わりには、**「この内容で承認いただけますか？次のフェーズに進んでよいですか？」と明示的に確認し、ユーザーの承認を得るまで次に進まない**こと。承認を得ずに設計を書き始めたり、実装に着手したりしてはいけません。

承認を得たら、該当ファイル冒頭の `Status:` を `approved (YYYY-MM-DD)` に更新します。

### 2.2 実装中に仕様との乖離が判明したら

コードを先に変えてはいけません。次の順序で対応します。

1. 作業を止め、何がどう食い違っているかをユーザーに報告する
2. `requirements.md` または `design.md` を更新する
3. 再承認を得る
4. `tasks.md` を必要に応じて更新し、実装を再開する

### 2.3 仕様書が不要な例外

以下は仕様書なしで進めて構いません。

- タイポ・誤字の修正
- コメント・ドキュメントの修正
- 依存パッケージのバージョン更新（挙動が変わらないもの）
- 仕様書そのものの修正
- Lint / フォーマッタによる機械的な整形
- 承認済み `tasks.md` に沿った実装作業そのもの

判断に迷ったら「仕様書を書く」側に倒してください。

詳細な運用手順は [docs/specs/README.md](docs/specs/README.md) を参照。

---

## 3. テスト駆動開発（TDD）

> **それを要求する失敗するテストなしに、プロダクションコードを書いてはいけません。**

`tasks.md` の各タスクは必ず Red → Green → Refactor のサイクルで進めます。

### 3.1 サイクル

1. **Red** — 受入基準を表すテストを先に書く。
   **実際にテストを実行し、意図した理由で失敗することを確認する。**
   （書いただけで次に進まない。「実装がないので失敗するはず」と推測しない）
2. **Green** — そのテストを通す最小限の実装を書く。先回りして機能を足さない。
3. **Refactor** — テストが緑のまま、重複の除去と命名の整理を行う。

### 3.2 ルール

- 1タスク = 1〜数個のテスト。細かく回す
- **コミットは全テストが緑の状態で行う**
- **テストをスキップ・削除・無効化して緑にしない。** `skip` / `xit` / `it.skip` を緑化目的で使わない
- テストの方が間違っていると判断した場合は、**なぜ間違いなのかを説明してから**直す
- バグ修正も TDD。まず**バグを再現する失敗テスト**を書く
- テストが書きにくいと感じたら、それは設計の問題。`design.md` に戻って相談する

### 3.3 レイヤ別の方針

**api/ (Rails)**
- モデルスペック、サービススペック、リクエストスペックを書く
- GitHub API は **VCR カセット + WebMock** で固定。テスト中に実 API を叩かない
- **指標計算ロジックは必ずユニットテストを書く。** 最低限カバーする境界:
  - デプロイが0件の期間
  - 障害が0件の期間
  - 集計期間をまたぐ PR（期間開始前にコミット、期間内にマージ）
  - チームラベルが付いていない PR
  - 同一 PR に複数レビューがある場合
  - レビューなしでマージされた PR

**web/ (React)**
- 指標の変換・整形を行う純関数はユニットテスト（Vitest）
- 主要画面は Testing Library でスモークテスト（描画され、主要な数値が表示される）
- API はモックし、実際のバックエンドに依存しない

### 3.4 タスク完了の定義

以下をすべて満たしたときのみ `tasks.md` のチェックボックスを埋めます。

- [ ] 該当タスクのテストが通っている
- [ ] **全テストが緑**（他を壊していない）
- [ ] Lint / 型チェックが通っている

---

## 4. 開発環境は Docker で完結する

> **ホストマシンで直接コマンドを実行してはいけません。**

ホストに Ruby / Node.js / PostgreSQL / Redis をインストールしません。すべての実行は Docker Compose 経由です。

### 4.1 禁止事項

- ホストでの `bundle`, `bundle install`, `rails`, `rake`, `rspec`, `rubocop` の直接実行
- ホストでの `npm`, `npx`, `node`, `yarn`, `vite`, `vitest` の直接実行
- ホストの PostgreSQL / Redis への接続
- **ユーザーへの手順説明でも、ホスト実行のコマンドを提示しない**

### 4.2 サービス構成（`compose.yaml`）

| サービス | 役割 |
|---|---|
| `api` | Rails API サーバ |
| `web` | Vite 開発サーバ（React） |
| `db` | PostgreSQL |
| `redis` | Sidekiq のキュー |
| `worker` | Sidekiq ワーカー（GitHub データの定期取り込み） |

### 4.3 依存パッケージの追加

必ずコンテナ内で実行し、更新されたロックファイルをコミットします。

```bash
docker compose exec api bundle add <gem>
docker compose exec web npm install <package>
```

`Gemfile` / `package.json` を直接編集した場合、および `Dockerfile` を変更した場合は**イメージの再ビルドが必要**です。

```bash
docker compose build api && docker compose up -d api
```

### 4.4 環境変数

- `.env.example` をリポジトリにコミットし、`.env` は `.gitignore` に入れる
- `GITHUB_TOKEN` などの秘匿値は**絶対にコードやコミットに含めない**
- `.env` 経由、または Rails credentials 経由で渡す

セットアップ手順の全量は [docs/development.md](docs/development.md) を参照。

---

## 5. ディレクトリ構成

```
.
├── CLAUDE.md               ← このファイル
├── compose.yaml            開発環境の全サービス定義
├── .env.example
├── api/                    Rails 8 (API mode)
│   ├── Dockerfile
│   └── app/
│       ├── models/
│       ├── services/github/    GitHub API 取り込み
│       ├── metrics/            指標計算ロジック
│       ├── serializers/        JSON レスポンス組み立て (PORO)
│       ├── jobs/               Sidekiq ジョブ
│       └── controllers/api/v1/
├── web/                    React + TypeScript + Vite
│   ├── Dockerfile
│   └── src/
│       ├── api/                API クライアントと型定義
│       ├── components/
│       │   └── charts/         グラフコンポーネント
│       ├── features/           画面単位のまとまり
│       └── lib/                純粋な変換関数
└── docs/
    ├── glossary.md         指標・用語の唯一の定義元
    ├── development.md      Docker 開発環境の手順
    └── specs/
        ├── README.md       仕様書駆動の運用ルール
        ├── _template/      仕様書テンプレート
        └── NNNN-<slug>/    各機能の仕様書
```

---

## 6. 技術スタック

**api/**
- Ruby 3.3+ / Rails 8（API mode）
- PostgreSQL
- Octokit（GitHub API クライアント）
- Sidekiq + Redis（定期ポーリング）
- RSpec / FactoryBot / VCR + WebMock
- RuboCop（`rubocop-rails-omakase`）

**web/**
- React 19 / TypeScript（strict）
- Vite
- TanStack Query（サーバ状態管理）
- Recharts（グラフ描画）
- Vitest + Testing Library
- ESLint + Prettier

> バージョンは環境構築時に確定します。確定したらこの節を実際の値に更新してください。

---

## 7. ドメインモデルと指標定義

> **指標の定義の唯一の出典は [docs/glossary.md](docs/glossary.md) です。**
> 計算式や判定ロジックをコードに書く前に必ず glossary を参照し、記述がなければ**推測せずユーザーに質問**してください。glossary とコードに齟齬があれば、コードではなく glossary を先に直します。

### 7.1 主要エンティティ

| エンティティ | 説明 |
|---|---|
| `Repository` | 追跡対象の GitHub リポジトリ。**指標の判定ルール設定を持つ** |
| `Team` | チーム。PR ラベルとの対応ルールを持つ |
| `TeamLabelRule` | チーム ↔ PR ラベルの紐付け（1チームに複数ラベル可） |
| `PullRequest` | 取り込んだ PR。作成・初回レビュー・approve・マージの各時刻を持つ |
| `Commit` | PR に紐づくコミット。最初のコミット時刻がリードタイムの起点 |
| `Deployment` | デプロイとみなされたイベント |
| `Incident` | 障害とみなされたイベント |
| `MetricSnapshot` | 日次などの集計結果キャッシュ |

### 7.2 設定で切り替わる判定ルール（Repository 単位）

指標の定義を**コードにハードコードしない**こと。リポジトリごとに設定で切り替えられる必要があります。

**デプロイ判定**
- `merge_to_default` — デフォルトブランチへの PR マージをデプロイとみなす（既定）
- `github_deployment` — GitHub Deployments API のデプロイイベント
- `release_tag` — Release の作成

**障害判定**
- `issue_label` — 指定ラベル（例: `incident`）が付いた Issue
- `pr_label` — 指定ラベル（例: `hotfix`）が付いた PR

**チーム判定**
- PR に付いたラベルをチームに対応付ける
- 複数のチームラベルが一致した場合の優先順位を決める（既定: `TeamLabelRule` の priority 昇順で先勝ち）
- どのチームにも一致しない PR は「未分類」チームに集約する

---

## 8. コーディング規約

### 8.1 Rails（api/）

- **Fat Model を避ける。** 責務ごとにディレクトリを分ける
  - GitHub API の呼び出しと取り込み → `app/services/github/`
  - 指標の計算 → `app/metrics/`
  - JSON レスポンスの組み立て → `app/serializers/`（Jbuilder は使わず、プレーンな PORO）
- **コントローラやモデルから直接 Octokit を呼ばない。** 必ずサービスクラス経由
- 集計処理は Ruby 側でのループではなく、可能な限り **SQL 側に寄せる**（`group`, `SELECT` 集約関数）
- N+1 を作らない。`includes` / `preload` を明示する
- 外部 API 呼び出しにはタイムアウトとリトライ、レート制限への対応を必ず入れる

### 8.2 React（web/）

- 関数コンポーネント + hooks のみ（クラスコンポーネント禁止）
- サーバ状態は **TanStack Query**、UI ローカル状態は `useState`。両者を混同しない
- **`any` 禁止。** `tsconfig` は strict
- API のレスポンス型は `web/src/api/types.ts` に集約
- グラフは `web/src/components/charts/` に切り出す。
  **データの整形・集計はコンポーネントの外の純関数（`web/src/lib/`）に置き、単体でテストできるようにする**
- コンポーネントに計算ロジックを埋め込まない

### 8.3 共通

- 識別子・ファイル名・型名は**英語**
- UI の表示文言、コミットメッセージ、仕様書は**日本語**
- コミットメッセージは変更の理由がわかる粒度で書く

---

## 9. よく使うコマンド

すべて Docker 経由です。全量は [docs/development.md](docs/development.md) を参照。

```bash
# 起動・停止
docker compose up -d
docker compose down
docker compose logs -f api

# Rails
docker compose exec api bin/rails db:migrate
docker compose exec api bin/rails console
docker compose exec api bundle exec rspec                        # 全テスト
docker compose exec api bundle exec rspec spec/metrics/foo_spec.rb:12   # TDD の内側ループ
docker compose exec api bundle exec rubocop
docker compose exec api bundle exec rubocop -a

# React
docker compose exec web npm test
docker compose exec web npm test -- --watch                      # TDD の内側ループ
docker compose exec web npm run lint
docker compose exec web npx tsc --noEmit
```

> コマンドが確定・変更されたら、この節と `docs/development.md` の両方を更新してください。

---

## 10. Claude への行動指示

**やること**

- 作業開始前に該当する仕様書（`docs/specs/`）を読む
- **テストを先に書く。** 実装から始めない
- **すべてのコマンドを `docker compose` 経由で実行する**
- 指標の定義は `docs/glossary.md` を参照する。記述がなければユーザーに質問する
- 変更が完了したら、該当テストと全テストを実行して結果を報告する
- テストが失敗したら、失敗したことを出力とともに正直に報告する

**やらないこと**

- 仕様書のフェーズを飛ばす、承認を得ずに次へ進む
- 承認済みの仕様書をユーザーの承認なしに書き換える
- 失敗テストなしにプロダクションコードを書く
- テストをスキップ・削除して緑にする
- ホストで直接コマンドを実行する、ホスト実行の手順を提示する
- 指標の定義を推測で決める
- **認証・認可のコードを勝手に追加する**（MVP スコープ外）
- **MVP スコープ外の機能を先回りして実装する**
- `GITHUB_TOKEN` などの秘匿値をコード・仕様書・コミットに書く

---

## 11. MVP のスコープ

**含む**
- GitHub API（Octokit）の定期ポーリングによるコミット・PR・Issue の取り込み
- Four Keys 4指標（デプロイ頻度 / 変更リードタイム / 変更失敗率 / 平均復旧時間）の全体表示
- 同 4指標のチーム別表示（チームは PR ラベルで判定）
- **サイクルタイム内訳**（コーディング / ピックアップ / レビュー / デプロイ待ち）の可視化
- 期間指定によるフィルタ

**含まない（先回りして実装しない）**
- アプリケーション認証・認可
- 複数チームを並べる比較ビュー、DORA パフォーマンスレベル判定の UI
- PR 個別の一覧・詳細へのドリルダウン
- GitHub Webhook の受信
- CI / 本番デプロイの構成
