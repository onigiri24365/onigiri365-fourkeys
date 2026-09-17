# 実装タスク: <機能名>

- **Spec**: NNNN-feature-slug
- **Status**: draft
- **最終更新**: YYYY-MM-DD
- **要件**: [requirements.md](./requirements.md)
- **設計**: [design.md](./design.md)

> `design.md` が **approved** になるまで、このファイルを書き始めてはいけません。
> このファイルが **approved** になるまで、実装に着手してはいけません。

---

## 進め方

各タスクは **TDD** で進めます（[CLAUDE.md](../../../CLAUDE.md) の「3. テスト駆動開発」）。

1. 🔴 **Red** — テストを書き、**実行して意図した理由で失敗することを確認する**
2. 🟢 **Green** — テストを通す最小限の実装を書く
3. ♻️ **Refactor** — 緑のまま整理する

**チェックを入れてよい条件**: 該当テストが通り、**全テストが緑**で、lint / 型チェックが通っていること。

すべてのコマンドは Docker 経由で実行します。

```bash
docker compose exec api bundle exec rspec <file>:<line>
docker compose exec web npm test -- --watch
```

---

## タスク一覧

### 1. <タスクのまとまり（例: データモデルの追加）>

- [ ] **1-1 🔴 テスト**: `api/spec/models/<x>_spec.rb`
  - ケース: <...>
  - 対応: AC-1
  - 確認: テストを実行し、意図した理由で失敗すること
- [ ] **1-2 🟢 実装**: `api/db/migrate/<...>.rb`, `api/app/models/<x>.rb`
  - 1-1 のテストが通ること
- [ ] **1-3 ♻️ Refactor**: 重複除去・命名整理（必要な場合のみ）

### 2. <タスクのまとまり（例: 指標計算ロジック）>

- [ ] **2-1 🔴 テスト**: `api/spec/metrics/<x>_spec.rb`
  - ケース: 正常系 <...>
  - 対応: AC-2
- [ ] **2-2 🟢 実装**: `api/app/metrics/<x>.rb`
- [ ] **2-3 🔴 テスト**: 境界値
  - [ ] 対象データが0件のとき `null` を返す（AC-N）
  - [ ] 集計期間をまたぐ PR の扱い（AC-N）
  - [ ] チームラベルが付いていない PR は「未分類」に集計される（AC-N）
- [ ] **2-4 🟢 実装**: 境界値への対応
- [ ] **2-5 ♻️ Refactor**

### 3. <タスクのまとまり（例: API エンドポイント）>

- [ ] **3-1 🔴 テスト**: `api/spec/requests/api/v1/<x>_spec.rb`
  - ケース: 正常系 / パラメータ不正で 400
  - 対応: AC-3
- [ ] **3-2 🟢 実装**: コントローラ・シリアライザ・ルーティング
- [ ] **3-3 ♻️ Refactor**

### 4. <タスクのまとまり（例: フロントエンド）>

- [ ] **4-1 🔴 テスト**: `web/src/lib/<x>.test.ts`（データ整形の純関数）
  - 対応: AC-4
- [ ] **4-2 🟢 実装**: `web/src/lib/<x>.ts`
- [ ] **4-3 🔴 テスト**: `web/src/features/<feature>/<Component>.test.tsx`（スモーク）
  - ケース: データあり / ローディング / エラー / データなし
- [ ] **4-4 🟢 実装**: コンポーネント・API クライアント・型定義
- [ ] **4-5 ♻️ Refactor**

---

## 完了チェック

すべてのタスクが終わったら確認します。

- [ ] `requirements.md` の受入基準がすべてテストでカバーされている
  （AC-1 〜 AC-N の各番号が、上のタスクのどれかから参照されている）
- [ ] `docker compose exec api bundle exec rspec` が全緑
- [ ] `docker compose exec web npm test` が全緑
- [ ] `docker compose exec api bundle exec rubocop` が通る
- [ ] `docker compose exec web npm run lint` が通る
- [ ] `docker compose exec web npx tsc --noEmit` が通る
- [ ] スキップ・無効化したテストが残っていない
- [ ] スコープ外の機能を実装していない

---

## 変更履歴

| 日付 | 変更内容 | 理由 |
|---|---|---|
| YYYY-MM-DD | 初版 | |
