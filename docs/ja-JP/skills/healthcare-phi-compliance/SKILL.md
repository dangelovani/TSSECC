---
name: healthcare-phi-compliance
description: 保護医療情報（PHI）コンプライアンス、HIPAA準拠、およびデータセキュリティ。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# ヘルスケアPHI/PIIコンプライアンスパターン

ヘルスケアアプリケーションにおける患者データ、医療従事者データ、および財務データを保護するためのパターン。HIPAA（米国）、DISHA（インド）、GDPR（欧州）、および一般的なヘルスケアデータ保護規制に適用されます。

## いつ使用するか

- 患者の記録（カルテ）に触れる機能の構築
- 臨床システムへのアクセス制御や認証の実装
- ヘルスケアデータ用のデータベーススキーマ設計
- 患者や医療従事者のデータを返すAPIの構築
- 監査証跡（オーディットトレイル）やログの実装
- データ露出の脆弱性に対するコードレビューの実施
- マルチテナント・ヘルスケアシステムにおける行レベルセキュリティ（RLS）の設定

## 動作方法

ヘルスケアデータ保護は、**分類**（何が機密か）、**アクセス制御**（誰がそれを見られるか）、および**監査**（誰がそれを見たか）の3つのレイヤーで機能します。

### データ分類

**PHI (Protected Health Information: 保護医療情報)** — 患者を特定可能であり、かつ健康状態に関連するすべてのデータ: 患者の名前、生年月日、住所、電話番号、メールアドレス、国民識別番号（SSN、Aadhaar、NHS番号など）、カルテ番号（MRN）、診断、服薬情報、検査結果、画像データ、保険ポリシーと請求の詳細、予約と入院の記録、またはこれらを組み合わせたデータ。

**PII (Personally Identifiable Information: 個人特定情報)** — ヘルスケアシステムにおける非患者機密データ: 医療従事者/スタッフの個人詳細、医師の報酬体系と支払額、従業員の給与と銀行口座情報、ベンダーの支払い情報。

### アクセス制御: 行レベルセキュリティ (RLS)

```sql
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;

-- 施設別でアクセス範囲を制限
CREATE POLICY "staff_read_own_facility"
  ON patients FOR SELECT TO authenticated
  USING (facility_id IN (
    SELECT facility_id FROM staff_assignments
    WHERE user_id = auth.uid() AND role IN ('doctor','nurse','lab_tech','admin')
  ));

-- 監査ログ: 挿入のみ（改ざん防止）
CREATE POLICY "audit_insert_only" ON audit_log FOR INSERT
  TO authenticated WITH CHECK (user_id = auth.uid());
CREATE POLICY "audit_no_modify" ON audit_log FOR UPDATE USING (false);
CREATE POLICY "audit_no_delete" ON audit_log FOR DELETE USING (false);
```

### 監査証跡 (オーディットトレイル)

すべてのPHIアクセスまたは変更はログに記録されなければなりません：

```typescript
interface AuditEntry {
  timestamp: string;
  user_id: string;
  patient_id: string;
  action: 'create' | 'read' | 'update' | 'delete' | 'print' | 'export';
  resource_type: string;
  resource_id: string;
  changes?: { before: object; after: object };
  ip_address: string;
  session_id: string;
}
```

### 一般的な漏洩経路

**エラーメッセージ:** クライアント（ブラウザ等）にスローされるエラーメッセージに、患者を特定できる情報を含めてはなりません。詳細情報はサーバー側でのみログ出力します。

**コンソール出力:** 患者オブジェクト全体を出力してはなりません。カルテ番号、国民ID、または名前ではなく、不透明な内部レコードID（UUID）を使用します。

**URLパラメータ:** ログやブラウザ履歴に残る可能性のあるクエリ文字列やパスセグメントに、患者を特定できる情報を含めてはなりません。不透明なUUIDのみを使用します。

**ブラウザストレージ:** localStorage や sessionStorage にPHIを保存してはなりません。PHIはメモリ内でのみ保持し、オンデマンドで取得します。

**サービスロールキー (service_role keys):** クライアント側のコードで service_role キー（管理者権限キー）を使用してはなりません。常に anon/publishable キーを使用し、RLSでアクセス制御を強制します。

**ログと監視:** 患者の記録全体を出力してはなりません。カルテ番号ではなく、不透明なレコードIDのみを使用します。エラー追跡サービスにスタックトレースを送信する前にサニタイズ（クリーンアップ）します。

### データベーススキーマのタグ付け

スキーマレベルでPHI/PIIカラムにコメント等を付与して明示します：

```sql
COMMENT ON COLUMN patients.name IS 'PHI: patient_name';
COMMENT ON COLUMN patients.dob IS 'PHI: date_of_birth';
COMMENT ON COLUMN patients.aadhaar IS 'PHI: national_id';
COMMENT ON COLUMN doctor_payouts.amount IS 'PII: financial';
```

### デプロイ前チェックリスト

すべてのデプロイ前に以下を確認します：
- エラーメッセージやスタックトレースにPHIが含まれていないこと
- console.log や console.error にPHIが含まれていないこと
- URLパラメータにPHIが含まれていないこと
- ブラウザストレージにPHIが保存されていないこと
- クライアントコードに service_role キーが残っていないこと
- すべてのPHI/PIIテーブルでRLSが有効化されていること
- すべてのデータ変更に対して監査証跡が記録されていること
- セッションタイムアウトが設定されていること
- すべてのPHIエンドポイントでAPI認証が行われていること
- 施設間のデータ分離が検証されていること

## 例

### 例1：安全なエラー処理と危険なエラー処理

```typescript
// 悪い例 — エラー内でPHIが漏洩している
throw new Error(`患者 ${patient.name} が施設 ${patient.facility} で見つかりません`);

// 良い例 — 一般的なエラーを返し、サーバー側で不透明なIDのみを用いて詳細をログ出力する
logger.error('患者の検索に失敗しました', { recordId: patient.id, facilityId });
throw new Error('レコードが見つかりません');
```

### 例2：複数施設隔離のためのRLSポリシー

```sql
-- 施設Aの医師は、施設Bの患者情報を閲覧できないようにする
CREATE POLICY "facility_isolation"
  ON patients FOR SELECT TO authenticated
  USING (facility_id IN (
    SELECT facility_id FROM staff_assignments WHERE user_id = auth.uid()
  ));

-- 検証: 施設Aの医師としてログインし、施設Bの患者をクエリする
-- 期待値: 0行が返されること
```

### 例3：安全なログ出力

```typescript
// 悪い例 — 特定可能な患者データをログ出力している
console.log('患者を処理中:', patient);

// 良い例 — 不透明な内部レコードIDのみを出力している
console.log('レコードを処理中:', patient.id);
// 注意: patient.id 自体もカルテ番号ではなく、不透明なUUIDであるべきです
```
