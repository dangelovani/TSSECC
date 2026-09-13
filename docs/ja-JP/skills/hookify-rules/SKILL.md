---
name: hookify-rules
description: 自動フック実装、イベントドリブン実行、およびルール駆動ワークフロー。
---

# Hookify ルールの記述

## 概要

Hookifyルールは、監視するパターンとパターンが一致した場合に表示するメッセージを定義するYAMLフロントマター付きのMarkdownファイルです。ルールは `.claude/hookify.{rule-name}.local.md` ファイルに保存されます。

## ルールファイル形式

### 基本構造

```markdown
---
name: rule-identifier
enabled: true
event: bash|file|stop|prompt|all
pattern: regex-pattern-here
---

このルールがトリガーされた場合にClaudeに表示するメッセージ。
Markdownフォーマット、警告、提案などを含めることができます。
```

### フロントマターフィールド

| フィールド | 必須 | 値 | 説明 |
|-------|----------|--------|-------------|
| name | はい | ケバブケース文字列 | 一意の識別子（動詞先頭: warn-*、block-*、require-*） |
| enabled | はい | true/false | 削除せずにトグル |
| event | はい | bash/file/stop/prompt/all | どのフックイベントでトリガーするか |
| action | いいえ | warn/block | warn（デフォルト）はメッセージを表示、blockは操作を防止 |
| pattern | はい* | 正規表現文字列 | マッチするパターン（*複雑なルールにはconditionsを使用） |

### 高度な形式（複数条件）

```markdown
---
name: warn-env-api-keys
enabled: true
event: file
conditions:
  - field: file_path
    operator: regex_match
    pattern: \.env$
  - field: new_text
    operator: contains
    pattern: API_KEY
---

.envファイルにAPIキーを追加しようとしています。このファイルが.gitignoreに含まれていることを確認してください！
```

**イベント別の条件フィールド：**
- bash: `command`
- file: `file_path`、`new_text`、`old_text`、`content`
- prompt: `user_prompt`

**演算子:** `regex_match`、`contains`、`equals`、`not_contains`、`starts_with`、`ends_with`

ルールがトリガーされるには、すべての条件が一致する必要があります。

## イベントタイプガイド

### bash イベント
Bashコマンドパターンにマッチ：
- 危険なコマンド: `rm\s+-rf`、`dd\s+if=`、`mkfs`
- 権限昇格: `sudo\s+`、`su\s+`
- パーミッション問題: `chmod\s+777`

### file イベント
Edit/Write/MultiEdit操作にマッチ：
- デバッグコード: `console\.log\(`、`debugger`
- セキュリティリスク: `eval\(`、`innerHTML\s*=`
- 機密ファイル: `\.env$`、`credentials`、`\.pem$`

### stop イベント
完了チェックとリマインダー。パターン `.*` は常にマッチ。

### prompt イベント
ワークフロー強制のためにユーザープロンプトコンテンツにマッチ。

## パターン記述のヒント

### 正規表現の基本
- 特殊文字のエスケープ: `.` → `\.`、`(` → `\(`
- `\s` 空白、`\d` 数字、`\w` 単語文字
- `+` 1つ以上、`*` 0個以上、`?` オプション
- `|` OR演算子

### よくある落とし穴
- **広すぎる**: `log` は "login"、"dialog" にもマッチする — `console\.log\(` を使用
- **具体的すぎる**: `rm -rf /tmp` — `rm\s+-rf` を使用
- **YAMLエスケープ**: クォートなしのパターンを使用。クォートされた文字列には `\\\\s` が必要

### テスト
```bash
python3 -c "import re; print(re.search(r'your_pattern', 'test text'))"
```

## ファイル構成

- **場所**: プロジェクトルートの `.claude/` ディレクトリ
- **命名**: `.claude/hookify.{descriptive-name}.local.md`
- **Gitignore**: `.gitignore` に `.claude/*.local.md` を追加

## コマンド

- `/hookify [description]` - 新しいルールを作成（引数なしの場合は会話を自動分析）
- `/hookify-list` - すべてのルールをテーブル形式で表示
- `/hookify-configure` - ルールのオン/オフをインタラクティブに切り替え
- `/hookify-help` - 完全なドキュメント

## クイックリファレンス

最小限のルール：
```markdown
---
name: my-rule
enabled: true
event: bash
pattern: dangerous_command
---
ここに警告メッセージ
```
