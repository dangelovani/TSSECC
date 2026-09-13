---
name: github-ops
description: GitHub操作、自動化、APIインテグレーション、およびCI/CDワークフロー。
origin: ECC
---

# GitHub 操作

コミュニティヘルス、CIの信頼性、コントリビューターエクスペリエンスに焦点を当てたGitHubリポジトリ管理。

## いつ使用するか

- イシューのトリアージ（分類、ラベリング、返信、重複排除）
- PRの管理（レビュー状況、CIチェック、古いPR、マージ準備）
- CI/CD失敗のデバッグ
- リリースとチェンジログの準備
- Dependabotとセキュリティアラートの監視
- オープンソースプロジェクトでのコントリビューターエクスペリエンスの管理
- ユーザーが「GitHubを確認」「イシューをトリアージ」「PRをレビュー」「マージ」「リリース」「CIが壊れた」と言った場合

## ツール要件

- すべてのGitHub API操作に **gh CLI** を使用
- `gh auth login` でリポジトリアクセスを設定済み

## イシュートリアージ

各イシューをタイプと優先度で分類する：

**タイプ:** バグ、機能リクエスト、質問、ドキュメント、改善、重複、無効、good-first-issue

**優先度:** クリティカル（破壊的/セキュリティ）、高（重大な影響）、中（あると嬉しい）、低（表面的）

### トリアージワークフロー

1. イシューのタイトル、本文、コメントを読む
2. 既存イシューとの重複がないか確認（キーワードで検索）
3. `gh issue edit --add-label` で適切なラベルを適用
4. 質問の場合：有用な回答を作成して投稿
5. 追加情報が必要なバグの場合：再現手順を求める
6. good first issueの場合：`good-first-issue` ラベルを追加
7. 重複の場合：元のイシューへのリンク付きコメント、`duplicate` ラベルを追加

```bash
# 重複の可能性を検索
gh issue list --search "keyword" --state all --limit 20

# ラベルを追加
gh issue edit <number> --add-label "bug,high-priority"

# イシューにコメント
gh issue comment <number> --body "ご報告ありがとうございます。再現手順を共有いただけますか？"
```

## PR管理

### レビューチェックリスト

1. CIステータスを確認：`gh pr checks <number>`
2. マージ可能か確認：`gh pr view <number> --json mergeable`
3. 経過時間と最終アクティビティを確認
4. レビューなしで5日以上のPRにフラグを付ける
5. コミュニティPRの場合：テストがあり規約に従っていることを確認

### 古いPRポリシー

- 14日以上アクティビティなしのイシュー：`stale` ラベルを追加し、更新を求めるコメント
- 7日以上アクティビティなしのPR：まだアクティブか確認するコメント
- 応答なしで30日後に古いイシューを自動クローズ（`closed-stale` ラベルを追加）

```bash
# 古いイシューを検索（14日以上アクティビティなし）
gh issue list --label "stale" --state open

# 最近アクティビティのないPRを検索
gh pr list --json number,title,updatedAt --jq '.[] | select(.updatedAt < "2026-03-01")'
```

## CI/CD操作

CI失敗時：

1. ワークフロー実行を確認：`gh run view <run-id> --log-failed`
2. 失敗ステップを特定
3. 不安定なテストか実際の失敗かを確認
4. 実際の失敗の場合：根本原因を特定し修正を提案
5. 不安定なテストの場合：将来の調査のためにパターンを記録

```bash
# 最近の失敗した実行をリスト
gh run list --status failure --limit 10

# 失敗した実行のログを表示
gh run view <run-id> --log-failed

# 失敗したワークフローを再実行
gh run rerun <run-id> --failed
```

## リリース管理

リリース準備時：

1. main上のすべてのCIがグリーンであることを確認
2. 未リリースの変更をレビュー：`gh pr list --state merged --base main`
3. PRタイトルからチェンジログを生成
4. リリースを作成：`gh release create`

```bash
# 前回リリース以降のマージ済みPRをリスト
gh pr list --state merged --base main --search "merged:>2026-03-01"

# リリースを作成
gh release create v1.2.0 --title "v1.2.0" --generate-notes

# プレリリースを作成
gh release create v1.3.0-rc1 --prerelease --title "v1.3.0 リリース候補1"
```

## セキュリティ監視

```bash
# Dependabotアラートを確認
gh api repos/{owner}/{repo}/dependabot/alerts --jq '.[].security_advisory.summary'

# シークレットスキャンアラートを確認
gh api repos/{owner}/{repo}/secret-scanning/alerts --jq '.[].state'

# 依存関係のバンプをレビュー — マージはユーザー承認が必要（提案のみ、自動マージ禁止）
gh pr list --label "dependencies" --json number,title
```

- 安全な依存関係のバンプをレビューし、ユーザー承認のためにマージを提案する — 自動マージは禁止
- クリティカル/高深刻度のアラートは即座にフラグを付ける
- 新しいDependabotアラートを最低週次で確認する

## 品質ゲート

GitHub操作タスクを完了する前に：
- トリアージされたすべてのイシューに適切なラベルが付いている
- 7日以上レビューまたはコメントなしのPRがない
- CI失敗が調査されている（単に再実行されただけでない）
- リリースに正確なチェンジログが含まれている
- セキュリティアラートが確認され追跡されている
