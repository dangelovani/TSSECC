---
name: mailtrap-email-integration
description: サンドボックス検証、ドメイン認証、API認証を含め、MailtrapのEmail APIを使用したトランザクションメール送信機能の統合をガイドします。メール送信機能の実装、配信問題のデバッグ、安全な開発/ステージング環境でのテスト設定時に使用します。
origin: ECC
---

# Mailtrap Email統合 (Mailtrap Email Integration)

MailtrapのEmail APIおよびサンドボックスを活用して、アプリケーションにトランザクションメール送信機能を追加するためのパターン集です。API認証、環境の分離、よくある配信の落とし穴をカバーします。

## 発動タイミング

- 「メール送信」機能（新規登録確認、パスワードリセット、通知、領収書など）の実装時
- 開発/ステージング環境でメールが届かない原因のデバッグ時
- プロジェクトで初めてメール送信連携をセットアップする時
- サンドボックスによる分離を行わずにメールAPIを直接呼び出しているコードのレビュー時

## コアコンセプト

**サンドボックスと本番環境の完全分離:** Mailtrapはメールを外部配信せずにキャプチャするサンドボックスAPIを提供しており、テストメールが実在の受信トレイに誤送信されるのを防ぎます。本番送信には検証済みドメイン用の個別エンドポイントを使用します。開発環境が本番送信エンドポイントを向くような構成は絶対に避けてください。

**認証方式:** リクエストの `Authorization` ヘッダーにBearerトークンを設定します。トークンはプロジェクト単位でスコープ設定され、サンドボックスと本番では通常異なるトークンを使用します。

**ドメイン認証 (Domain Verification):** 本番送信では、Mailtrapが実際の受信者へメールを配信する前に、DNSレコード（SPF、DKIM、DMARC）による送信ドメインの認証が必須です。これを怠ると、メールが暗黙的に配信失敗するか迷惑メールフォルダに振り分けられます。

## コード例

```typescript
// Mailtrap Email API 経由での送信（本番環境用）
async function sendEmail(to: string, subject: string, html: string) {
  const response = await fetch("https://send.api.mailtrap.io/api/send", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.MAILTRAP_API_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      from: { email: "no-reply@yourverifieddomain.com", name: "Your App" },
      to: [{ email: to }],
      subject,
      html,
    }),
  });

  if (!response.ok) {
    throw new Error(`メール送信に失敗しました: ${response.status}`);
  }
  return response.json();
}
```

```typescript
// 同一の呼び出しで、非本番環境ではサンドボックスへ自動ルーティング
const MAILTRAP_ENDPOINT = process.env.NODE_ENV === "production"
  ? "https://send.api.mailtrap.io/api/send"
  : `https://sandbox.api.mailtrap.io/api/send/${process.env.MAILTRAP_INBOX_ID}`;
```

## アンチパターン

| アンチパターン | なぜ問題なのか | 代替アプローチ |
| --- | --- | --- |
| 開発/テスト環境で本番送信エンドポイントを使用する | テストメールが実在のアドレスに届き、スパム通報やテストデータの漏洩につながる | 非本番環境は必ずサンドボックスエンドポイントへルーティングする |
| ソースコード内にAPIトークンをハードコードする | バージョン管理にコミットされた際に認証情報が漏洩するリスク | 環境変数またはシークレットマネージャーからトークンを読み込む |
| ドメイン認証が完了する前に本番送信を開始する | メールが暗黙的に配信失敗するかスパム扱いになる | 本番送信を有効化する前にSPF/DKIM/DMARCの各DNSレコードを検証する |
| 送信失敗時の再試行やエラーハンドリングがない | パスワードリセットが届かないなどのサイレントな障害が発生する | レスポンスステータスを確認し、失敗をログに記録して対処可能なエラーを返す |

## ベストプラクティス

- サンドボックス用と本番用のトークンは別の環境変数で管理し、環境をまたいで同じトークンを使い回さない。
- メールが関わる本番リリースを行う前に、送信ドメインのDNSレコードを必ず検証する。
- デバッグに十分なコンテキスト（宛先、テンプレート、タイムスタンプ、レスポンスコード）を含めて配信失敗をログ出力する。
- メール送信を「失敗し得るネットワーク呼び出し」として扱い、必ず try/catch で囲み、決して成功を前提にしない。

## 関連スキル

`api-and-interface-design`, `security-and-hardening`, `ci-cd-and-automation`
