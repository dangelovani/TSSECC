---
name: ito-compute
description: リアルタイムのGPU在庫の照会、認証済みItô固定レートRFQの送信、RFQまたは調達ステータスの確認、デバイス資格情報の取り消し、および個別にインストールされた公式CLIを通じた明示的ゲート付きノード適格性評価の実行を行います。ユーザーがH100/H200の計算能力の検索、固定計算レートのリクエスト、Itô計算ステータスの確認、GPUノードの検証、Itôアクセスの取り消し、またはGPU計算リソースのレンタル/購入を求め、サポート範囲の説明が必要な場合に使用します。
---

# Itô Compute（Itô コンピュート）

公式の Itô compute CLI または MCP サーバーを使用します。ECC は並列クライアント、ローカルシミュレーション、予約、ワークロードランナー、または推論サーバーを独自に実装しません。また、ECC 自体はブラウザの自動操作を行いません。

## 公式ローカルパッケージのインストール

`ito-compute-cli` は現在公開レジストリには未登録です。`npx`、`npm exec`、または未検証のパッケージを使用するのではなく、公式リポジトリからビルドしてください:

```sh
git clone https://github.com/Ito-Markets/ito-cloud-runtime.git
cd ito-cloud-runtime/cli/ito-compute-cli
npm ci
npm run check
```

`ECC_ITO_CLI_EXECUTABLE` にビルドされた実行ファイルのエントリの絶対パスを設定します:

```text
/absolute/path/to/ito-cloud-runtime/cli/ito-compute-cli/dist/bin/ito.js
```

ECC は `PATH` を介してこの資格情報を含むクライアントを自動検出することはありません。
`ecc ito login` はデバイス認証を実行し、`ITO_API_KEY` を継承しません。
検証専用の `auth`、および `find` と `status` は、設定されている場合 `ITO_API_KEY` を直接転送します（`ITO_AUTH_MODE=legacy` は不要です）。引数、追跡対象ファイル、MCP結果、ログ、またはチャット内にAPIキーやトークンを決して含めないでください。

## CLI ワークフロー

1. 初回操作の前に `ecc ito login` を実行します。ECC はこれを公式CLIのデバイス認証に委譲します。デフォルトでは Itô 検証ページがブラウザで開き、macOS Keychain にデバイストークンを永続化します。ページの引き渡しを抑制するには `ecc ito login --no-browser` を使用します。ECC 自体はブラウザ自動操作を行いません。呼び出し元のエージェントがサインインブラウザのステップを完了できない場合は、ユーザーに正確なコマンドを提示してください。承認完了後、元のタスクに戻り `ecc ito auth` を続行します。デバイストークンはデフォルトで macOS Keychain を使用します。ファイルトークンへのフォールバックは明示的であり、そのディレクトリとトークンファイルは所有者のみアクセス可能（0700 および 0600）でなければなりません。
2. 既存の資格情報を検証するために `ecc ito auth` を実行します。これはログインを開始せず、`--no-browser` フラグは拒否されます。
3. `ecc ito find` を実行する前に、RFQ を送信するための明示的な購入権限を買い手から取得します。
   - `gpu`、`count`、整数の `days`、`max-rate`、`nodes`、`gpus-per-node`、`storage-tb`、`start-window`、`form-factor`、`contract-type`、`fabric`、`region`、および分割充当（split-fill）の決定が必須です。
   - `count == nodes * gpus-per-node` である必要があります。トポロジを推測・自動補完してはいけません。
   - 買い手がファブリックやリージョンを問わないことを明示的に認めた場合にのみ `any` を使用します。
   - `--allow-split` を省略した場合は false を意味します。
4. リアルタイム RFQ コマンドを実行します:

   ```sh
   ecc ito find \
     --gpu h200 \
     --count 8 \
     --nodes 1 \
     --gpus-per-node 8 \
     --days 30 \
     --storage-tb 1 \
     --start-window 2099-08-15 \
     --max-rate 3.00 \
     --form-factor bare_metal \
     --contract-type reservation \
     --fabric infiniband \
     --region us-east-1
   ```

5. `ecc ito status` を実行して、RFQ および調達注文の状況を確認します。
   トランスポートエラーが発生して結果が曖昧な場合は、`find` を再実行する前にまずステータスを確認してください。
6. 見積もりが準備され、買い手が明示的に承認したら、それを受諾します:

   ```sh
   ecc ito accept rfq_<ticket-id>
   ```

   これにより、チケットが担当デスク（Human review）にルーティングされます。これは資金の移動やリソースの確定予約を行うものではありません。買い手の明示的な承認なしに受諾してはいけません。
7. ユーザーがこのデバイスの認証取り消しを明示的に要求した場合は、`ecc ito logout` を実行します。公式CLIは、リモートでの取り消しが失敗した場合でも管理者が再試行できるようにローカルの資格情報を保持します。手動でトークンファイルを削除して代用してはいけません。

在庫価格は参考値（indicative）です。RFQ は予約された計算能力ではありません。正規の結果に null でない確定見積もり（firm quote）が含まれている場合にのみ、レートを固定として扱ってください。

## ノードのライブ適格性評価（Live node qualification）

`ecc ito evals` は、公式CLIの限定的なライブアダプターを、個別にインストールされた `sixtytwo-cli==0.3.33` に公開します。ECC を通じたローカルフィクスチャの実行は行いません。
呼び出す前に以下のすべてが必須です:

- 指定されたノードに接続するためのオペレーター承認
- `ITO_ENABLE_SIXTYTWO_LIVE=1`
- `--live-sixtytwo` フラグ
- 明示的なノードリスト
- `sixtytwo.yaml` を含む既存の絶対パス設定ディレクトリ

```sh
ecc ito evals \
  --cluster clu_prod_example \
  --live-sixtytwo \
  --nodes gpu-01,gpu-02 \
  --config-dir /absolute/path/to/qualification-config
```

公式アダプターは、ピン留めされたバージョンチェックと、明示的なノードに対する `sixtytwo test --full` のみ実行可能です。リソースのレンタル、起動、復旧、修復、リセット、購入、発注はできません。ECC は `ITO_API_KEY` やモデル/クラウド資格情報をノード適格性評価に転送しません。

## MCP ワークフロー

公式パッケージをビルドした後、絶対パスを使用して stdio サーバーを設定します:

```json
{
  "mcpServers": {
    "ito-compute": {
      "command": "node",
      "args": [
        "/absolute/path/to/ito-cloud-runtime/cli/ito-compute-cli/dist/bin/ito-mcp.js"
      ]
    }
  }
}
```

サーバーは以下のみを公開します:

- `ito_auth`
- `ito_find`
- `ito_status`
- `ito_accept`

`ito_auth` は既存の資格情報を検証します（デバイスログインは開始しません）。`ito_auth` を使用し、買い手の明示的な権限とすべてのハード制約を収集し、`ito_find` を呼び出し、必要に応じて `ito_status` でポーリングします。見積もりが準備され、買い手が明示的に承認したら、チケットIDを指定して `ito_accept` を呼び出します。デスクの見積もりはデスクが確認するまで通常参考値であり拘束力はありません。結果には `quote_class` が含まれます。

## レンタルおよび購入のセマンティクス

`find` は RFQ を送信し確定見積もりを返す場合がありますが、レンタル、購入、予約、プロビジョニング、資金移動を行うものではありません。`accept` は見積もりを担当デスクの人手によるレビューに回すものであり、資金を移動したり計算能力を予約したりするものではありません。`status` は読み取り専用ですが、プロバイダーエンドポイントが既存の調達注文を調整（reconcile）する場合があります。ECC ヘルプ内の受動的なダッシュボードリンクは独立したユーザー操作用 Web ルートです。不足している CLI 機能の代用としてダッシュボードを開いたり操作したりしないでください。

## サポートされていない操作

サポートされているクライアントインターフェースでは、見積もりのロック、計算能力の確定予約、ワークロードの実行、推論の提供は行えません。`accept` はデスクへの引き渡しであり、購入ではありません。MCP サーバーは適格性評価を公開していません（上記の明示的な CLI コマンドを使用してください）。独自の追加ツールや購入パスを作成してはいけません。ローカル CLI が存在しない場合やライブ操作が失敗した場合に、ブラウザ操作やフィクスチャで代用しないでください。欠落している機能を報告して停止してください。
