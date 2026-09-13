---
name: healthcare-eval-harness
description: ヘルスケアAIモデル評価ハーネス、臨床メトリクス、およびレギュレーション遵守の検証。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# ヘルスケア評価ハーネス — 患者安全検証

ヘルスケアアプリケーションのデプロイ用自動検証システム。「クリティカル（CRITICAL）」なテストの失敗が1件でもあれば、デプロイはブロックされます。患者の安全は妥協できません。

> **注意:** 例では参照用テストランナーとして Jest を使用しています。あなたのフレームワーク（Vitest、pytest、PHPUnit など）に合わせてコマンドを調整してください。テストカテゴリと合格基準はフレームワークに依存しません。

## いつ使用するか

- EMR/EHRアプリケーションのデプロイ前
- CDSSロジック（薬物相互作用、投与量検証、スコアリング）の変更後
- 患者データに触れるデータベーススキーマの変更後
- 認証やアクセス制御の変更後
- ヘルスケアアプリのCI/CDパイプライン設定時
- 臨床モジュールでのマージ衝突の解消後

## 動作方法

評価ハーネスは、5つのテストカテゴリを順番に実行します。最初の3つ（CDSS精度、PHI漏洩、データ整合性）は「クリティカル（CRITICAL）」ゲートであり、100%の合格率が必要です。1つでも失敗するとデプロイがブロックされます。残りの2つ（臨床ワークフロー、統合）は「高（HIGH）」ゲートであり、95%以上の合格率が必要です。

各カテゴリは、Jestのテストパスパターンにマッピングされています。CIパイプラインは、クリティカルゲートを `--bail`（最初の失敗で停止）付きで実行し、`--coverage --coverageThreshold` でカバレッジのしきい値を強制します。

### 評価カテゴリ

**1. CDSS精度 (CRITICAL — 100% 必須)**

すべての臨床意思決定支援ロジックをテストします: 薬物相互作用ペア（双方向）、投与量検証ルール、公開仕様と一致する臨床スコア、偽陰性なし、サイレントエラーなし。

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage
```

**2. PHI漏洩 (CRITICAL — 100% 必須)**

保護医療情報（PHI）の漏洩をテストします: APIのエラーレスポンス、コンソール出力、URLパラメータ、ブラウザストレージ、施設間の分離、未認証アクセス、サービスロールキーの露出。

```bash
npx jest --testPathPattern='tests/security/phi' --bail --ci
```

**3. データ整合性 (CRITICAL — 100% 必須)**

臨床データの安全性をテストします: ロックされた診察記録、監査証跡エントリ、カスケード削除の保護、同時編集の処理、孤立したレコードの禁止。

```bash
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

**4. 臨床ワークフロー (HIGH — 95%以上必須)**

エンドツーエンドのフローをテストします: 診察ライフサイクル、テンプレートのレンダリング、処方セット、医薬品/診断検索、処方箋PDF、レッドフラグアラート。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No clinical tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "臨床合格率: ${rate}% ($passed/$total)"
```

**5. 統合コンプライアンス (HIGH — 95%以上必須)**

外部システムとの統合をテストします: HL7メッセージパース（v2.x）、FHIRバリデーション、検査結果のマッピング、不正なメッセージの処理。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No integration tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "統合合格率: ${rate}% ($passed/$total)"
```

### 合否判定マトリクス

| カテゴリ | しきい値 | 失敗時のアクション |
|----------|-----------|------------|
| CDSS精度 | 100% | **デプロイをブロック** |
| PHI漏洩 | 100% | **デプロプロック** |
| データ整合性 | 100% | **デプロプロック** |
| 臨床ワークフロー | 95%以上 | 警告、要レビューで許可 |
| 統合コンプライアンス | 95%以上 | 警告、要レビューで許可 |

### CI/CD統合

```yaml
name: Healthcare Safety Gate
on: [push, pull_request]

jobs:
  safety-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci

      # クリティカルゲート — 100% 必須、最初の失敗で即停止
      - name: CDSS Accuracy
        run: npx jest --testPathPattern='tests/cdss' --bail --ci --coverage --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80}}'

      - name: PHI Exposure Check
        run: npx jest --testPathPattern='tests/security/phi' --bail --ci

      - name: Data Integrity
        run: npx jest --testPathPattern='tests/data-integrity' --bail --ci

      # 高ゲート — 95%以上必須、カスタムしきい値チェック
      - name: Clinical Workflows
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No clinical tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Clinical pass rate ${RATE}% below 95%"
          fi

      - name: Integration Compliance
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No integration tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Integration pass rate ${RATE}% below 95%"
          fi
```

### 避けるべきアンチパターン

- 「前回通ったから」という理由でCDSSテストをスキップする
- クリティカルなしきい値を100%未満に設定する
- クリティカルなテストスイートで `--no-bail` を使用する
- 統合テストでCDSSエンジンをモックする（本物のロジックをテストしなければなりません）
- セーフティゲートが赤（不合格）のときにデプロイを許可する
- CDSSテストスイートで `--coverage` なしで実行する

## 例

### 例1：すべてのクリティカルゲートをローカルで実行する

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage && \
npx jest --testPathPattern='tests/security/phi' --bail --ci && \
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

### 例2：高ゲートの合格率を確認する

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
jq '{
  passed: (.numPassedTests // 0),
  total: (.numTotalTests // 0),
  rate: (if (.numTotalTests // 0) == 0 then 0 else ((.numPassedTests // 0) / (.numTotalTests // 1) * 100) end)
}' "$tmp_json"
# 期待値: { "passed": 21, "total": 22, "rate": 95.45 }
```

### 例3：評価レポート

```
## Healthcare Eval: 2026-03-27 [commit abc1234]

### 患者の安全: PASS

| カテゴリ | テスト数 | 合格 | 失敗 | ステータス |
|----------|-------|------|------|--------|
| CDSS精度 | 39 | 39 | 0 | PASS |
| PHI漏洩 | 8 | 8 | 0 | PASS |
| データ整合性 | 12 | 12 | 0 | PASS |
| 臨床ワークフロー | 22 | 21 | 1 | 95.5% PASS |
| 統合コンプライアンス | 6 | 6 | 0 | PASS |

### カバレッジ: 84% (目標: 80%以上)
### 判定: デプロイ可能
```
