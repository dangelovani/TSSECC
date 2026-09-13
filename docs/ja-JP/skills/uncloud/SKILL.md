---
name: uncloud
description: Uncloudクラスターの管理（サービスのデプロイ、Caddyインプレスの設定、クラスター外デバイス向け静的プロキシルートの追加、ポート公開、スケーリング、ログ調査、または `uc` CLI を使用したマシンとボリュームの管理）を行う場合に使用します。
metadata:
  origin: ECC
---

# Uncloud Cluster Management（Uncloud クラスター管理）

Dockerコンテナ、WireGuardメッシュネットワーク、およびCaddyリバースプロキシを利用した分散型セルフホスティングプラットフォーム `uc` CLI のリファレンスガイド。

## 有効化のタイミング

Uncloud クラスターを操作する際、特に以下のタスクで使用します:
- `uc machine` によるマシンのブートストラップまたはクラスタへの参加
- `uc deploy` による Compose ファイルからのサービスデプロイ
- Uncloud を通じた HTTP、HTTPS、TCP、UDP ポートの公開
- `x-caddy`, `x-ports`, または `--caddyfile` による Caddy イングレスの設定
- 外部LANデバイスのクラスタプロキシ経由でのルーティング
- ログ、サービス状態、ボリューム、DNS、またはマシンの配置状態の調査

## 動作の仕組み

Uncloud は、WireGuard メッシュで相互接続されたピアマシン群にわたって Docker サービスを実行します。各マシンは対等なクラスタメンバーであり、サービスはオーバーレイネットワーク上で通信し、Caddy がグローバルに動作してパブリックな HTTP/HTTPS トラフィックを終端します。Compose ファイルでは、イングレス、配置、および自動生成される Caddy 設定のための Uncloud 拡張機能を使用でき、`uc` CLI がイメージの配布、スケジューリング、スケーリング、ログ、およびクラスタ状態を管理します。

## コマンド例

```bash
uc machine init user@host --name machine-1
uc service run --name web -p app.example.com:8080/https nginx:latest
uc deploy
```

## コアコンセプト

- **中央コントロールプレーンなし** — すべてのマシンは WireGuard で接続された対等なピア
- **Caddy** がすべてのマシン上でグローバルサービスとして稼働し、Let's Encrypt から TLS 証明書を自動取得
- **オーバーレイネットワーク** — サービスはデフォルトで `10.210.0.0/16` 経由で通信し、メッシュ内部でDNSを提供
- **Caddyfile は自動生成される** — 直接編集してはならず、代わりに `x-caddy` または `--caddyfile` を使用する

---

## CLI クイックリファレンス

### マシン管理

| コマンド | 用途 |
|---------|---------|
| `uc machine init user@host` | 最初のマシンの初期化 / 新規クラスターの作成 |
| `uc machine add user@host` | 既存クラスターへのマシンの参加 |
| `uc machine ls` | マシンの一覧表示 |
| `uc machine update NAME --public-ip IP` | イングレス用パブリックIPの更新 |
| `uc machine rm NAME` | マシンの削除 |

主要な `init` フラグ: `--name`, `--network 10.210.0.0/16`, `--no-caddy`, `--no-dns`, `--public-ip auto|IP|none`

### サービス管理

| コマンド | 用途 |
|---------|---------|
| `uc service ls` / `uc ls` | サービスの一覧表示 |
| `uc service run IMAGE` | 単一コンテナサービスの実行 |
| `uc deploy` | `compose.yaml` からのデプロイ |
| `uc deploy --no-build` | 再ビルドを行わずプッシュ済みイメージからデプロイ |
| `uc deploy --recreate` | サービスの強制再作成 |
| `uc scale SERVICE N` | レプリカ数の設定 |
| `uc service logs SERVICE` | ログの表示 |
| `uc service exec SERVICE` | コンテナ内シェルへの接続 |
| `uc service inspect SERVICE` | 詳細情報の表示 |
| `uc service rm SERVICE` | サービスの削除（名前付きボリュームは保持） |
| `uc ps` | クラスター全体の全コンテナ表示 |

### イメージ管理

```bash
uc image push myapp:latest                    # ローカルイメージを全マシンにプッシュ
uc image push myapp:latest -m machine1,machine2  # 特定のマシンにプッシュ
uc images                                     # クラスター内のイメージ一覧
```

### ボリューム管理

```bash
uc volume ls                  # 全ボリューム一覧
uc volume ls -m machine1      # 特定マシン上のボリューム
uc volume create NAME -m MACHINE
uc volume rm NAME
```

### Caddy 管理

```bash
uc caddy config    # 現在生成されている Caddyfile を表示（読み取り専用）
uc caddy deploy    # クラスター全体への Caddy のデプロイ/アップグレード
```

### DNS & コンテキスト

```bash
uc dns show        # 予約済み *.uncld.dev ドメインの表示
uc dns reserve     # 新しいドメインの予約
uc ctx ls          # クラスターコンテキストの一覧
uc ctx use prod    # コンテキストの切り替え
```

---

## ポートの公開（Port Publishing）

### HTTP/HTTPS（Caddy リバースプロキシ経由）

```
-p [hostname:]container_port[/protocol]
```

| 例 | 意味 |
|---------|---------|
| `-p 8080/https` | 自動生成ホスト名（`service-name.cluster-domain`）での HTTPS |
| `-p app.example.com:8080/https` | カスタムホスト名での HTTPS |
| `-p 8080/http` | HTTP のみ（TLS なし） |

### TCP/UDP（ホストバインド、Caddy をバイパス）

```
-p [host_ip:]host_port:container_port[/protocol]@host
```

| 例 | 意味 |
|---------|---------|
| `-p 5432:5432@host` | 全インターフェースで TCP 5432 を公開 |
| `-p 127.0.0.1:5432:5432@host` | ループバック（127.0.0.1）のみで TCP 5432 を公開 |
| `-p 53:5353/udp@host` | UDP ポートの公開 |

---

## Compose ファイルの拡張機能

Uncloud は Docker Compose に以下の独自拡張を追加します:

### `x-ports` — ドメイン付きポート公開

```yaml
services:
  app:
    image: app:latest
    x-ports:
      - example.com:8000/https
      - www.example.com:8000/https
      - api.example.com:9000/https
```

### `x-caddy` — サービス固有のカスタム Caddy 設定

```yaml
services:
  app:
    image: app:latest
    x-caddy: |
      example.com {
        redir https://www.example.com{uri} permanent
      }
      www.example.com {
        reverse_proxy {{upstreams 8000}} {
          import common_proxy
        }
        basic_auth /admin/* {
          admin $2a$14$...
        }
      }
```

`x-caddy` 内で使用可能なテンプレート関数:
- `{{upstreams [service] [port]}}` — 正常なコンテナのIP群
- `{{.Name}}` — サービス名
- `{{.Upstreams}}` — 全サービス → IP のマップ

### `x-machines` — 配置制約（マシングループ指定）

```yaml
services:
  db:
    image: postgres:18
    x-machines: db-machine          # 単一マシン名
  app:
    image: app:latest
    x-machines:
      - machine-1
      - machine-2
```

### 複数サービスの構成例

```yaml
services:
  api:
    build: ./api
    x-ports:
      - api.example.com:3000/https
    environment:
      DATABASE_URL: postgres://db:5432/mydb

  web:
    build: ./web
    x-ports:
      - example.com:8000/https
      - www.example.com:8000/https
    environment:
      API_URL: http://api:3000

  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    x-machines: db-machine

volumes:
  db-data:
```

---

## 外部（クラスター外）デバイスへのルーティング

本物のコンテナを実行せずに、外部デバイス（BMC、NAS、ルーターUI等）を Caddy 経由で公開する方法:

**1. Caddyfile スニペットの作成**（例: `~/device.caddyfile`）:

```caddyfile
https://device.example.com {
    reverse_proxy https://192.168.1.x {
        transport http {
            tls_insecure_skip_verify   # 自己署名証明書の場合に必要
        }
    }
    log
}
```

平文（HTTP）の上流の場合: `reverse_proxy http://192.168.1.x:port`

**2. no-op コンテナをダミーとして名前付きサービスを登録:**

```bash
uc service run \
  --name device-bmc \
  --caddyfile ~/device.caddyfile \
  registry.k8s.io/pause:3.9
```

`pause` は何もしない軽量コンテナです — Uncloud が Caddyfile を紐付けるためのサービスエントリとして機能します。

**3. 検証:**

```bash
uc caddy config   # device.example.com のブロックが表示されることを確認
```

---

## 内部サービス DNS

クラスター内部のサービスは、名前で相互に名前解決できます:

| DNS名 | 解決先 |
|----------|------------|
| `service-name` | 正常な任意のコンテナ |
| `service-name.internal` | 同上 |
| `rr.service-name.internal` | ラウンドロビン |
| `nearest.service-name.internal` | 同一マシン優先 |

---

## 一般的なトラブルと対処法

| トラブル | 対処法 |
|---------|-----|
| Caddyfile を直接編集してしまう | Compose の `x-caddy` または `uc service run` の `--caddyfile` を使用する |
| 自己署名証明書を持つ上流へのプロキシが失敗する | `transport http { tls_insecure_skip_verify }` を追加する |
| `uc caddy config` にユーザー定義ブロックが表示されない | Caddy の管理ソケットに接続できていない — `uc inspect caddy` と `uc logs caddy` を確認 |
| コンテナから外部LANのIPに到達できない | Caddy コンテナのホストが対象ネットワークへのルーティング経路を持っているか確認 |
| `uc service rm` 後にボリュームが消えた | 名前付きボリューム（Named volumes）は保持されます。自動削除されるのは匿名ボリュームのみです |
