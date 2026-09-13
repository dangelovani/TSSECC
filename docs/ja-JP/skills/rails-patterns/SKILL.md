---
name: rails-patterns
description: Rails 7.1+ および 8.x アプリケーション向けの Ruby on Rails フレームワークパターン。ディレクトリ規約、サービスオブジェクトによる Skinny Controller、Form オブジェクト、Query オブジェクト、慣用的な ActiveRecord、バックグラウンドジョブ、ViewComponent、Hotwire、および Rails 8 Solid スタックを網羅。Rails アプリ、コントローラ、モデル、サービス、ジョブ、ビューの構築やレビュー時に使用します。
origin: community
---

# Rails パターン (Rails Patterns)

モダンな Ruby on Rails アプリケーション（Rails 7.1+ および 8.x）向けのフレームワークパターン集です。Rails は設計上「規約重視（opinionated）」ですが、本スキルではモデル数が50を超えるようなエンタープライズ規模でも保守性を維持できるようにコミュニティで確立された実践的パターンをまとめます。本スキルは「実装方法（How）」を扱い、「どのパターンを採用すべきかの判断（What / When）」については、リポジトリ内の Ruby パターンルール（`rules/ruby/patterns.md`、インストール時は `rules/ecc/ruby/patterns.md`）を参照してください。

## 適用タイミング

- Rails アプリケーションの新規構築（フルスタック、API 専用、またはハイブリッド構成）
- `app/` や `config/` に変更を加える PR のレビュー
- モデル、コントローラ、サービス、ジョブの新規生成
- コントローラのアクションが約10行を超えて肥大化した場合
- モデルのファイルが約200行を超えて肥大化した場合
- コントローラやビューの中に ActiveRecord クエリが直接記述され始めた場合

## コアコンセプト

### ディレクトリ規約 (The Directory Contract)

Rails アプリケーションは予測可能な構造に従います。ディレクトリは安易に追加せず、意図を持って明確に配置します。

```
app/
  models/         ActiveRecord モデル。データに密接な永続化とドメインロジック。
  controllers/    HTTP リクエスト処理。薄いオーケストレーションのみを担当。
  views/          ERB テンプレート。ビジネスロジックは含めない。
  components/     ViewComponent クラス。単体テストが必要なビューロジック。
  services/       サービスオブジェクト。複数ステップからなるビジネス操作。
  forms/          Form オブジェクト。複数モデルにまたがる複雑なフォーム処理。
  queries/        Query オブジェクト。再利用可能で合成可能な ActiveRecord クエリ。
  jobs/           バックグラウンドジョブ。Solid Queue、Sidekiq、GoodJob による非同期処理。
  mailers/        ActionMailer クラス。
  helpers/        ビューヘルパー。極小のプレゼンテーションロジックのみ。
  policies/       認可ポリシー（Pundit を使用する場合）。任意。
  channels/       ActionCable チャンネル（WebSocket 通信用）。
```

`app/lib/`、`app/utils/`、`app/managers/` のような曖昧なディレクトリの作成は避けてください。上記ディレクトリに収まらない場合は、新しいディレクトリを作るのではなく、設計そのものを再検討すべきです。真に汎用的な共通コードはルートの `lib/` に配置します。

### 薄いコントローラ (Skinny Controllers)

コントローラは「リクエストを受信し、適切なオブジェクトに委譲し、レスポンスを描画する」ことだけに専念します。ビジネスロジックは別の場所に持たせます。（Ruby パターンルールに基づき、コントローラが複数の責務を持ち始めたらサービスオブジェクトへ切り出します。）

### サービスオブジェクト (Service Objects)

単一モデルの保存にとどまらないビジネス操作のデフォルト手法です。一貫性を保つための規約は以下の通りです：

- サフィックス（`InvoiceCreator`）ではなく、ドメインによる名前空間（`Invoices::Create`）を使用する。
- クラスメソッド `.call` を定義し、インスタンスの `#call` に委譲する。
- 呼び出し元が成功・失敗・エラー内容・対象レコードを条件分岐できるよう、真偽値や素のレコードではなく Result オブジェクトを返却する。
- 複数レコードの書き込みはトランザクションで囲む。
- サービスは常に単一目的（`Invoices::Create`、`Invoices::MarkPaid`）とし、`Invoices::Manager` のような汎用クラスは避ける。

### Form オブジェクト (Form Objects)

フォームが複数モデルにまたがる場合や、データベースのカラムに直接対応しない入力項目を持つ場合は、ネストした属性（nested attributes）や無関係なモデルの仮想属性ではなく、Form オブジェクトを使用します。ビューからは通常のモデルのように振る舞い（`form_with model: @form`）、背後で各レコードをクリーンに組み立てます。

### Query オブジェクト (Query Objects)

複数のコントローラやサービスで再利用される ActiveRecord クエリや、スコープにするには複雑すぎるクエリは、スコープを引数として受け取り合成可能な Query オブジェクトとして抽出します。目安として、3つ以上の条件チェーンがある場合やパラメータを多く受け取るスコープは Query オブジェクトへの分割を検討します。

### バックグラウンドジョブ (Background Jobs)

処理負荷の高い処理や時間のかかる外部通信はすべてバックグラウンドへオフロードします。（新規の Rails 8 で中規模スループットなら Solid Queue、高度なオブザーバビリティ・高スループット・既存 Redis 資産があるなら Sidekiq を推奨）。アダプタに関わらず以下の原則を徹底します：
- レコードオブジェクトではなく ID を渡す。
- `perform` メソッドは冪等（Idempotent）にする。
- `retry_on` と `discard_on` を明示的に設定する。

### パーシャルよりも ViewComponent (ViewComponent over Partials)

条件分岐によるレンダリングがある場合、引数が2つを超える場合、または3箇所以上で再利用されるビューロジックには ViewComponent を優先します。コンポーネントは独立して単体テストが可能であり、公開インターフェースが明示的です。複雑な条件ロジックを持つパーシャル（partial）は保守性の負債になりがちです。

### Hotwire: Turbo と Stimulus

Rails の標準フロントエンドスタックです。（サーバーレンダリングが中心のアプリでは Hotwire を最優先し、React/Vue はクライアント側のインタラクションの複雑さがその導入コストに見合う場合にのみ採用します）。部分的なページ更新には Turbo Frames、サーバー主導の更新には Turbo Streams、マークアップに寄り添う小さなクライアント挙動には Stimulus を使用します。

### Rails 8 Solid スタック

Rails 8 では、これまで Redis を必要としていたコンポーネントが標準でデータベース駆動の構成（Solid Queue、Solid Cache、Solid Cable）に置き換えられました。インフラ要素を1つ減らせる代わりにデータベースへの負荷が増加するというトレードオフがあります。中規模までのスループットには最適であり、超大規模環境では依然として Redis が優位です。デプロイツールとしては Docker ベースの Kamal が標準となります。

## コード例

### サービスオブジェクトを用いた薄いコントローラ

```ruby
# 悪い例: コントローラ内にビジネスロジックが記述されている
class InvoicesController < ApplicationController
  def create
    @invoice = Invoice.new(invoice_params)
    @invoice.user = current_user
    @invoice.line_items.build(invoice_params[:line_items])
    @invoice.tax_total = TaxCalculator.new(@invoice).calculate
    @invoice.total = @invoice.line_items.sum(&:amount) + @invoice.tax_total

    if @invoice.save
      InvoiceMailer.created(@invoice).deliver_later
      AccountingExportJob.perform_later(@invoice.id)
      redirect_to @invoice, notice: "請求書を作成しました"
    else
      render :new
    end
  end
end

# 良い例: コントローラはオーケストレーションのみ、処理はサービスが行う
class InvoicesController < ApplicationController
  def create
    result = Invoices::Create.call(params: invoice_params, user: current_user)

    if result.success?
      redirect_to result.invoice, notice: "請求書を作成しました"
    else
      @invoice = result.invoice
      render :new, status: :unprocessable_entity
    end
  end
end
```

### サービスオブジェクトの実装例

```ruby
# app/services/invoices/create.rb
module Invoices
  class Create
    # Struct を使用することで Rails 7.1 がサポートする全 Ruby バージョンで動作します。
    # Ruby 3.2+ であれば、`Data.define(:success?, :invoice, :errors)` が
    # より簡潔でイミュータブルな代替手段となります。
    Result = Struct.new(:success, :invoice, :errors, keyword_init: true) do
      def success?
        success
      end
    end

    def self.call(params:, user:)
      new(params: params, user: user).call
    end

    def initialize(params:, user:)
      @params = params
      @user = user
    end

    def call
      invoice = build_invoice
      ApplicationRecord.transaction do
        invoice.save!
      end
      begin
        send_notifications(invoice)
      rescue StandardError => e
        Rails.logger.error("請求書 #{invoice.id} の通知送信に失敗しました: #{e.message}")
      end
      Result.new(success: true, invoice: invoice, errors: nil)
    rescue ActiveRecord::RecordInvalid => e
      Result.new(success: false, invoice: e.record, errors: e.record.errors)
    end

    private

    attr_reader :params, :user

    def build_invoice
      invoice = user.invoices.new(params.except(:line_items))
      invoice.line_items.build(params[:line_items])
      invoice.tax_total = TaxCalculator.call(invoice)
      invoice.total = invoice.line_items.sum(&:amount) + invoice.tax_total
      invoice
    end

    def send_notifications(invoice)
      InvoiceMailer.created(invoice).deliver_later
      AccountingExportJob.perform_later(invoice.id)
    end
  end
end
```

### Form オブジェクトの実装例

```ruby
# app/forms/signup_form.rb
class SignupForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :email, :string
  attribute :password, :string
  attribute :company_name, :string
  attribute :terms_accepted, :boolean

  validates :email, presence: true, format: URI::MailTo::EMAIL_REGEXP
  validates :password, presence: true, length: { minimum: 12 }
  validates :company_name, presence: true
  validates :terms_accepted, acceptance: true

  attr_reader :user, :company

  def save
    return false unless valid?

    ApplicationRecord.transaction do
      @company = Company.create!(name: company_name)
      @user = @company.users.create!(email: email, password: password, role: :owner)
    end
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.merge!(e.record.errors)
    false
  end
end
```

### Query オブジェクトの実装例

```ruby
# app/queries/invoices/overdue.rb
module Invoices
  class Overdue
    def self.call(scope: Invoice.all, as_of: Time.current)
      new(scope: scope, as_of: as_of).call
    end

    def initialize(scope:, as_of:)
      @scope = scope
      @as_of = as_of
    end

    def call
      scope
        .where(status: :sent)
        .where(due_date: ..as_of)
        .where.not(id: paid_invoice_ids)
        .includes(:customer, :line_items)
    end

    private

    attr_reader :scope, :as_of

    def paid_invoice_ids
      Payment.where(created_at: ..as_of).pluck(:invoice_id)
    end
  end
end
```

Query オブジェクトはスコープを受け取るため合成可能です：`Invoices::Overdue.call(scope: current_user.invoices)`。

### N+1 クエリの防止

```ruby
# 悪い例: ビューで post.author.name を呼び出す際に N+1 が発生
@posts = Post.published

# 良い例: Eager Loading を実施
@posts = Post.published.includes(:author)
```

`includes` は Rails に `preload` か `eager_load` の選択を委ねます。クエリを分ける場合は明示的に `preload` を、関連テーブルで絞り込みを行い JOIN する場合は `eager_load` を指定します。Rails 6.1 以降では、`strict_loading` を有効にすることで意図しない遅延ロード発生時に例外を発生させることができます。

### カウンターキャッシュ (Counter Cache)

```ruby
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
```

```ruby
add_column :posts, :comments_count, :integer, default: 0, null: false
```

これにより、`post.comments_count` は `COUNT(*)` クエリを発行せずカラム読み取りのみになります。上記は新規テーブルを想定しており、既存行が存在する場合は別途バックフィル（初期値投入）が必要です。

### バックグラウンドジョブの構造

レコードオブジェクトではなく ID を渡します。リトライ機構により「少なくとも1回（at-least-once）」配信されるため、外部サービスを呼び出すジョブは必ず冪等にする必要があります。さもないと、リモート呼び出し成功後に一時的エラーが発生した場合、次のリトライ試行で処理が重複実行されてしまいます。

```ruby
class AccountingExportJob < ApplicationJob
  queue_as :exports

  retry_on AccountingApi::TransientError, wait: :polynomially_longer, attempts: 5
  discard_on AccountingApi::PermanentError

  def perform(invoice_id)
    invoice = Invoice.find(invoice_id)
    export = AccountingExport.create_or_find_by!(
      invoice: invoice,
      idempotency_key: "invoice-export-#{invoice.id}-#{invoice.updated_at.to_i}"
    )
    return if export.completed_at?

    receipt = AccountingApi.export(invoice, idempotency_key: export.idempotency_key)
    export.update!(completed_at: Time.current, external_id: receipt.id)
  end
end
```

```ruby
add_index :accounting_exports, :idempotency_key, unique: true
```

一意制約インデックス（unique index）が安全性を担保します。2つの試行が競合した場合、DB が2回目の挿入を拒否し、Active Record が競合を解消して既存レコードを返します。これはジョブレベルのリトライとは無関係に機能します。API 呼び出し前のガードと、API への `idempotency_key` 伝搬によって API 呼び出し後の安全性が確保され、API 呼び出しと `update!` の間でクラッシュが発生しても重複出力を防止できます。

### ViewComponent の実装例

```ruby
# app/components/invoice_status_badge_component.rb
class InvoiceStatusBadgeComponent < ViewComponent::Base
  STATUS_CLASSES = {
    draft: "bg-gray-100 text-gray-800",
    sent: "bg-blue-100 text-blue-800",
    paid: "bg-green-100 text-green-800",
    overdue: "bg-red-100 text-red-800"
  }.freeze

  def initialize(invoice:)
    @invoice = invoice
  end

  def call
    tag.span(@invoice.status.humanize, class: "rounded-full px-2 py-1 text-sm #{status_class}")
  end

  private

  def status_class
    STATUS_CLASSES.fetch(@invoice.status.to_sym, "bg-gray-100")
  end
end
```

```erb
<%= render InvoiceStatusBadgeComponent.new(invoice: @invoice) %>
```

### Hotwire の利用例

```erb
<%# Turbo Frame: 編集リンクをクリックするとこのフレーム内のみが置換される %>
<%= turbo_frame_tag "invoice_#{@invoice.id}" do %>
  <div class="invoice">
    <%= link_to "編集", edit_invoice_path(@invoice) %>
  </div>
<% end %>
```

```erb
<%# Turbo Stream: app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", @comment %>
<%= turbo_stream.update "comment_form", partial: "form", locals: { comment: Comment.new } %>
```

```javascript
// app/javascript/controllers/copy_to_clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source"]

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value)
  }
}
```

### 許容されるコールバックと許容されないコールバック

```ruby
# 許容: 純粋なデータ正規化
class User < ApplicationRecord
  before_validation :normalize_email

  private

  def normalize_email
    self.email = email.to_s.downcase.strip
  end
end

# サービスへ移行すべき: コールバックに隠蔽された副作用
# class User < ApplicationRecord
#   after_create :send_welcome_email  # オプトアウトが難しく、テストも困難
# end
```

### 適切な Concern と不適切な Concern

```ruby
# 適切: 真に横断的であり、無関係な複数モデル間で再利用される
# app/models/concerns/soft_deletable.rb
module SoftDeletable
  extend ActiveSupport::Concern

  included do
    scope :active, -> { where(deleted_at: nil) }
    scope :deleted, -> { where.not(deleted_at: nil) }
  end

  def soft_delete! = update!(deleted_at: Time.current)
  def restore! = update!(deleted_at: nil)
end

# 不適切: 単一のモデルからしか使われておらず、本来そのモデルに属するロジック
# app/models/concerns/invoice_calculations.rb
module InvoiceCalculations
  extend ActiveSupport::Concern

  def calculate_total
    line_items.sum(&:amount) + tax_total
  end
end
# このモジュールは Invoice のみから include されています。横断的関心事ではなく、
# 単にモデルを「薄く」見せるためにモジュールに隠しただけです。Invoice クラス本体に戻すべきです。
```

単一クラスからしか使われない Concern は単なるコード移動に過ぎず、元のクラスに配置すべきです。Concern は少なくとも2つ以上の無関係なモデル間で再利用される場合にのみ作成します。

## アンチパターン

### 神コントローラ (God Controllers)
約80行を超えるコントローラは責務を持ちすぎています。コントローラを分割するか、サービスオブジェクトへ処理を抽出してください。

### 30以上のメソッドを持つ肥大化したモデル (Fat Models)
モデルは自身のデータについての知識のみを持つべきです。他のモデルの調整、通知送信、ワークフローの連携を行うメソッドはサービスオブジェクトに移譲します。

### コールバックの連鎖 (Callback Chains)
`after_save :update_cache, :send_notifications, :enqueue_export` のような構成はデバッグの悪夢の始まりです。これらを明示的に実行するサービスオブジェクトに移行します。

### 複雑なフォームに対する nested attributes
`accepts_nested_attributes_for` は単純なユースケースには適していますが、条件付きバリデーションや複数モデル間の複雑なロジックには Form オブジェクトを使用します。

### 重要モデルへの default_scope の乱用
`default_scope { where(deleted: false) }` はアプリケーション内の全クエリからレコードを暗黙的に除外してしまい、サポートやデバッグ時の調査を阻害します。明示的な名前付きスコープを使用してください。

### データベースの概念にちなんだモデル名
`UserRole`、`OrderStatus`、`InvoiceState` などは通常、モデルではなく enum の適用候補です。

### Hotwire を検討する前の安易な JS フレームワーク採用
時折インタラクションが発生する程度のサーバーレンダリングページであれば、Hotwire の方が迅速にリリースできます。React や Vue は真に SPA 構成が必要な領域に限定します。

## ベストプラクティス

- コントローラは薄く保ち、ビジネスロジックはサービスオブジェクトへ委譲する。
- サービスからは Result オブジェクトを返し、例外ではなく実行結果で呼び出し元を分岐させる。
- 複数レコードの書き込みはトランザクションで囲む。通知や副作用の失敗はログに残し、主処理の書き込みをロールバックさせない。
- ジョブにはレコード ID を渡し、`perform` は冪等にし、リトライや破棄の条件を明示する。
- デフォルトで Eager Loading を行い、偶発的な N+1 クエリは不具合として扱う。
- Concern は少なくとも2つ以上の無関係なモデルで共有される振る舞いに限定する。
- サーバーレンダリング中心のアプリでは、クライアントフレームワークの前にまず Hotwire の適用を検討する。

## 関連スキル

- `backend-patterns` — サービス境界とアダプタパターン（Ruby パターンルールで参照）
- Ruby パターンルール（`rules/ruby/patterns.md`、インストール時は `rules/ecc/ruby/patterns.md`） — 本スキルが実装する意思決定と採用指針
