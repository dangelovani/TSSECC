---
name: orch-build-mvp
description: 設計文書やスペックドキュメントから動作するMVPのブートストラップをオーケストレーション — ドキュメントの取り込み、薄い垂直スライスの計画、最初のエンドツーエンドスライスのスキャフォールド、TDD実装、レビュー、ゲート付きコミット。SDD/PRDを動作する出発点に変換する場合に使用。
metadata:
  origin: ECC
---

# orch-build-mvp

アクター・アクション・ターゲット: **orch · build · mvp**。[`orch-pipeline`](../orch-pipeline/SKILL.md) の共有エンジン上の薄いラッパー。

## いつ使用するか

- ユーザーが**設計/スペックドキュメント**（SDD、PRD、system_design）を持ち、そこからワーキングな垂直スライスをブートストラップしたい場合。
- ドキュメントパスを引数として受け取る。例: `civicpulse/docs/SDD-v0.6.md`

## 操作設定

- **デフォルトサイズ下限:** large — Scaffoldを含む完全パイプライン。
- **フェーズマスク:** 0（スペック読み込み）→ 1 → 2（重い）→ 3（scaffold）→ 4 → 5 → 6。
- **最初のアクション（フェーズ0→2）:** ドキュメントを読み、スコープ、確定済み決定、機能リストを抽出。**薄い垂直スライス**（全モデル→全ビューではなく、最初に1つのエンドツーエンドパス）に順序付け。フェーズ3でその最初のスライスを構築。

## 仕組み

1. 上記の設定で `orch-pipeline` エンジンを実行。
2. **既存のGANハーネスを再利用**（手動ループの代わりに）：
   - SDDを `gan-harness/spec.md` + `gan-harness/eval-rubric.md` に変換（`gan-planner` が生成するものの代替 — スペックは既にある）。
   - `/gan-build "<一行概要>" --skip-planner` でビルドを駆動（デフォルト: `--max-iterations 15`, `--pass-threshold 7.0`, `--eval-mode playwright`。非UIスライスには `--eval-mode code-only`）。
   - そのコマンドが `gan-generator` → `gan-evaluator` ループを実行し、スコアがパスまたはプラトーになるまで `gan-harness/feedback/feedback-NNN.md` を書き込む。
3. **ゲート1**（スライス計画）と**ゲート2**（コミット前）で停止。スキャフォールドと各スライスを個別の `feat:` コミットとしてコミット。
4. セキュリティトリガーに触れるスライスには `security-reviewer` を追加。

## 例

```
orch-build-mvp: civicpulse/docs/SDD-v0.6.md
→ SDD読み込み → スライスリスト（垂直）→ スライス1をスキャフォールド  [ゲート1: 承認]
→ /gan-build --skip-planner (generator → evaluatorループ) スペックに対するスコア → レビュー
→ feat: コミット  [ゲート2: 確認] → 次のスライス
```
