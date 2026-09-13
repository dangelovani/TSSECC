---
name: codehealth-mcp
description: CodeScene MCPを介したリアルタイムの構造的コード健全性（Code Health）の評価 — 編集前のレビュー、変更後のスコア差分の検証、コミットおよびPRのゲート制御を行います。コード品質のレビュー、リファクタリング、AIによる変更がファイルの保守性を低下させていないかの確認、またはコミット/PR前に行う場合に使用します。
metadata:
  origin: community
---

# Code Health MCP (CodeScene)

AI支援コーディングのための構造的保守性フィードバック。スタイル/リントスキル（`coding-standards`, `plankton-code-quality`）を、**設計レベル**の健全性スコアとリグレッションゲートによって補完します。

**アップストリーム:** [codescene-oss/codescene-mcp-server](https://github.com/codescene-oss/codescene-mcp-server)  
**パッケージ:** `@codescene/codehealth-mcp`（npx 経由の stdio）

## セキュリティと境界

**オプトイン方式（ECC）:** `mcp-configs/mcp-servers.json` 内の `codescene` ブロックはテンプレートに過ぎません。ECC プラグインのインストールによってバンドルされたMCPサーバーが自動有効化されることはありません。必要な場合のみ設定にコピーしてください。ECC のインストール/同期時に `ECC_DISABLED_MCPS=codescene,...` で除外することも可能です。

**資格情報:** トークンはバンドルされていません。自身で `CS_ACCESS_TOKEN` を設定してください（アップストリームリポジトリの [getting-a-personal-access-token.md](https://github.com/codescene-oss/codescene-mcp-server/blob/main/docs/getting-a-personal-access-token.md) を参照）。トークンをリポジトリにコミットしてはいけません。

**ツールが読み取る内容:** 呼び出された際、ツールは指定された**ローカルリポジトリ内のファイルおよび Git 状態**を分析します（渡されたパス、および `analyze_change_set` のブランチコンテキスト）。勝手に単独実行されることはありません。スタンドアロンモードについては、公式のプライバシードキュメントに従ってください: [codescene-mcp-server README](https://github.com/codescene-oss/codescene-mcp-server#frequently-asked-questions) および [CodeScene policies](https://codescene.com/policies)。秘密情報、認証情報、または分析を望まないパスに対してこのスキルを使用してはいけません。

**MCPが利用不可の場合（オフライン、無効なトークン、サーバークラッシュ）:** Code Health スコアを推測・捏造してはいけません。チェックがスキップされたことをユーザーに伝えてください。ユーザーの明示的な承認がある場合にのみ続行します。MCPがダウンしているときは、ゲート制御としてリント/テスト/verification-loop を優先してください。サーバーの接続が回復したらチェックを再開します。

## 使用場面

- ユーザーが**コード品質のレビュー**、ファイルの**リファクタリング**、または**AIによる変更で保守性が低下していないか**の確認を求めたとき
- **ホットスポット**、レガシーモジュール、または不慣れなファイルを編集する前
- 保守性のセーフガードが必要な**コミット前**または**プルリクエスト前**
- エージェントが大規模なdiffを作成した後 — Code Health が低下していないかの検証
- 構造的チェックとして `verification-loop`、`tdd-workflow`、または `/quality-gate` と併用（テストやリントの代替ではありません）

## 有効化のタイミング

上記の **使用場面** と同じトリガーです（この見出しはECCのスキル自動有効化に使用されます）。

## 動作の仕組み

### 1. MCP サーバーの接続

`mcp-configs/mcp-servers.json` から `codescene` エントリをハーネスの MCP 設定にコピーします。

**Claude Code**（`~/.claude.json` → `mcpServers`）:

```json
"codescene": {
  "command": "npx",
  "args": ["-y", "@codescene/codehealth-mcp"],
  "env": {
    "CS_ACCESS_TOKEN": "YOUR_CS_ACCESS_TOKEN_HERE"
  }
}
```

**プロジェクトスコープ:** リポジトリルートの `.mcp.json` に同じブロックをマージします。

トークンの設定方法はアップストリームのリポジトリに記載されています。スタンドアロンモードでは、以下に挙げる4つのツールを利用するにあたり有料の CodeScene プラットフォームアカウントは不要です。スコアに依存する前に、セッションを再起動して `codescene` サーバーが接続されていることを確認してください。

### 2. スタンドアロンツールのみの呼び出し

| ツール | 使用場面 |
|------|-------------|
| `code_health_review` | ファイルを変更する**前**の完全な構造分析 |
| `code_health_score` | 各変更後のクイックな数値スコア（差分チェック） |
| `pre_commit_code_health_safeguard` | Code Health のリグレッションを引き起こすコミットのブロック |
| `analyze_change_set` | PR を作成する**前**のブランチレベルのチェック |

プラットフォーム専用のツール（リポジトリ全体の技術的負債ホットスポット一覧など）は**呼び出さないでください**。また、スタンドアロンでは利用できない `delta_analysis` も参照しないでください。

### 3. スコアの解釈（1.0〜10.0）

| 範囲 | 意味 | エージェントの行動方針 |
|-------|---------|----------------|
| **9.0–10.0** | 緑 — 健全 | 拡張が安全。垂直スライスを推奨 |
| **4.0–8.9** | 黄 — 負債あり | 慎重に進める。ついで（行きがけ）のリファクタリングは禁止 |
| **1.0–3.9** | 赤 — 深刻な負債 | スコープを極めて狭く限定 |

### 4. フィードバックループの実行

**ファイルに触れる前**

1. 対象パスに対して `code_health_review` を実行します。
2. ベースラインスコアと検出されたコードスメルを記録します。
3. タスクを解決するための最小限の変更を計画します。

スコアによるスコープの絞り込み: **5未満** — 最小限のdiffのみ; **5〜7** — 広範囲なリファクタリングは避ける; **7超** — リファクタリングが比較的安全（ただし変更ごとに検証）。

**各変更後**

1. 同じファイルに対して `code_health_score` を実行します。
2. `code_health_review` によるベースラインと比較します。
3. スコアが**悪化（リグレッション）**している場合は、作業を続ける前に修正します。開始時よりスコアが低下した状態でタスク完了としてはいけません。

**毎回のコミット前** — リポジトリパスに対して `pre_commit_code_health_safeguard` を実行します。

**PR 前** — ベースブランチ（`main` など）に対して `analyze_change_set` を実行します。

## 具体例

### 例: Flask の保守性改善

`pallets/flask` において、スタンドアロンツールのみを使用したエージェントループ:

1. 対象モジュールに対して `code_health_review` を実行（ベースライン **4.82**）
2. 検出されたスメルに対処する的を絞ったリファクタリング
3. 各編集後に `code_health_score` を実行
4. コミット前に `pre_commit_code_health_safeguard` を実行
5. PR 作成前に `analyze_change_set` を実行

結果: Code Health **4.82 → 9.1**（無料のスタンドアドントークンのみで達成）。

### 例: AGENTS.md での強制ルールブロック

プロジェクトの `AGENTS.md` または `CLAUDE.md` に貼り付けて使用します:

```md
## Code Health (CodeScene MCP)

ファイルを変更する前: `code_health_review` を実行し、スコアと課題を記録すること。

- スコア5未満: 問題のある範囲 — 変更スコープを狭く絞ること。
- スコア5〜7: 警告範囲 — 広範なリファクタリングは行わないこと。

各変更後: `code_health_score` を実行して差分を確認すること。

- スコアが悪化した場合: 続行する前に修正すること。スコアが低下した状態で完了を宣言しないこと。

コミット前: 必ず `pre_commit_code_health_safeguard` を実行すること。

PR前: 必ず `analyze_change_set` を実行すること。
```

### 例: アンチパターン vs 正しいループ

```markdown
# BAD: 先に編集し、後からチェックする
[code_health_review なしの大規模リファクタリング]

# BAD: スコアの低下を無視する
「テストが通った」からといって、Code Health が低下しているのに完了とする

# BAD: 赤色スコアのファイル（5未満）で広範なリファクタリングを行う
モジュール全体についでで行う行きがけのクリーンアップ

# GOOD: レビュー → 小さな変更 → スコア確認 → コミットセーフガード → analyze_change_set
```

## ECC との組み合わせ

| ECC スキル / フロー | Code Health MCP の役割 |
|------------------|----------------------|
| `coding-standards` | スタイル/命名規則。Code Health は構造/複雑度を担当 |
| `plankton-code-quality` | 記述時のリント/フォーマット。Code Health は編集前後の構造ゲートを担当 |
| `verification-loop` / `/quality-gate` | 「完了」前に構造的リグレッションチェックを追加 |
| `security-review` | セキュリティ vs 保守性 — 関連する場合は両方を適用 |
| `tdd-workflow` | テスト合格 ≠ 健全な設計 — リファクタリング後にスコアを確認 |

**コンテキスト活用のヒント:** ECC はアクティブな MCP 数を少なく保つことを推奨しています。実質的なコード編集を行う際に `codescene` を有効化し、不要なときは無効化してください。

## 関連スキル

- `coding-standards` — ベースライン規約
- `plankton-code-quality` — 記述時のリント/フォーマットフック
- `verification-loop` — ビルド/テスト/リントの品質ゲート
- `tdd-workflow` — テスト駆動開発
- `security-review` — セキュリティチェックリスト
- `documentation-lookup` — Context7 経由のライブラリドキュメント参照（直交する機能）
