---
name: kubernetes-patterns
description: プロダクション環境向け Kubernetes のワークロードパターン、リソース管理、RBAC、プローブ設定、オートスケーリング、ConfigMap/Secret の取り扱い、および kubectl によるトラブルシューティングとデバッグ手法。Kubernetes マニフェストの作成・レビュー、プローブ、RBAC、HPA、リソース制限のチューニング時に使用します。
metadata:
  origin: ECC
---

# Kubernetes パターン (Kubernetes Patterns)

ワークロードを安全・高可用にデプロイ、運用管理、およびデバッグするための本番環境向け Kubernetes パターン集です。

## 適用タイミング

- Kubernetes マニフェスト（Deployment、Service、Ingress、Job など）の作成
- リソースの requests / limits、liveness / readiness プローブの設計・設定
- RBAC、Namespace、ServiceAccount のセットアップ
- K8s 内での設定値（ConfigMap）や機密情報（Secret）の管理
- CrashLoopBackOff、OOMKilled、Pending 状態の Pod、ImagePull エラー等のトラブルシューティング
- HPA（Horizontal Pod Autoscaler）や PodDisruptionBudget（PDB）の構成
- セキュリティや構成妥当性の観点からの K8s YAML マニフェストのレビュー

## 使用方針

> 上記の「適用タイミング」と同様です。本リポジトリのスキル形式規約に準拠したエイリアスです。Kubernetes の YAML やワークロードの作成、レビュー、デバッグを行う際はいつでも本スキルを活用してください。

## 仕組みと構成

本スキルは、タスク別に体系化された**本番仕様のコピー＆ペースト可能な YAML パターン**と **kubectl デバッグコマンド集**を提供します：

1. **Deployment テンプレート** — セキュリティコンテキスト、ローリングアップデート戦略、3種類の全プローブ、リソース制限、ConfigMap/Secret からの環境変数注入を完備した本番用 `Deployment`。
2. **プローブ設計** — Startup / Liveness / Readiness の選定判断基準と、適切な `failureThreshold × periodSeconds` の計算根拠。
3. **Service & Ingress** — ClusterIP、LoadBalancer、および cert-manager アノテーションを含む TLS Ingress パターン。
4. **ConfigMap & Secret** — `envFrom`、ボリュームマウント、および External Secrets の運用指針。
5. **リソース管理** — ワークロード種別（Web API、JVM、ワーカー、サイドカー）ごとの requests / limits の実践的な設計目安。
6. **RBAC** — 最小特権の原則に基づく ServiceAccount → Role → RoleBinding チェーン。
7. **HPA & PDB** — オートスケーリングとノード退避（drain）時の安全性を担保する設定。
8. **Job & CronJob** — 適切な `restartPolicy` を備えた単発・定期実行ワークロードのパターン。
9. **kubectl チートシート** — ログ確認、exec、ロールバック、ポート転送、dry-run、代表的エラーの診断コマンド。
10. **アンチパターン & チェックリスト** — 回避すべきアンチパターンと、セキュリティ・信頼性・オブザーバビリティの検証項目。

## クイックリファレンス

| タスク | 参照先 |
|------|---------|
| 本番仕様の完全な Deployment YAML | [コアワークロードパターン](#コアワークロードパターン) |
| プローブ（ヘルスチェック）設定 | [プローブ（Liveness、Readiness、Startup）](#プローブliveness-readiness-startup) |
| RBAC 最小特権セットアップ | [RBAC（Role と ServiceAccount）](#rbacrole-と-serviceaccount) |
| CrashLoopBackOff のデバッグ | [kubectl デバッグチートシート](#kubectl-デバッグチートシート) |
| オートスケーリング | [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa) |

---

## コアワークロードパターン

### Deployment — 本番環境用テンプレート

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
  labels:
    app: my-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 更新中に許可される一時的な追加 Pod 数
      maxUnavailable: 0    # 更新中も希望レプリカ数を下回らないよう保証
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0.0"
    spec:
      # Pod レベルのセキュリティコンテキスト
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001

      # グレースフルシャットダウン猶予期間
      terminationGracePeriodSeconds: 30

      containers:
        - name: my-app
          image: ghcr.io/org/my-app:1.0.0   # :latest タグは絶対に使用しない
          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 8080
              protocol: TCP

          # リソースの requests と limits は両方の指定が必須
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          # コンテナレベルのセキュリティコンテキスト
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          # プローブ設定（詳細は後述のプローブセクション参照）
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 2

          # ConfigMap および Secret からの環境変数注入
          envFrom:
            - configMapRef:
                name: my-app-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-app-secrets
                  key: db-password

          # readOnlyRootFilesystem: true 時に書き込みを許可する一時ディレクトリ
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}
```

---

## プローブ（Liveness、Readiness、Startup）

各プローブの役割と使い分けの理解はシステムの安定稼働に不可欠です：

| プローブ | 失敗時の動作 | 主な用途 |
|-------|---------------|---------|
| `startupProbe` | 起動が完了しないコンテナを強制終了・再起動 | 起動に時間がかかるアプリケーション（JVM、Python 等） |
| `livenessProbe` | コンテナを再起動 | デッドロックやプロセスのハングアップ検知 |
| `readinessProbe` | Service のエンドポイントから Pod を除外（トラフィック停止） | 一時的な過負荷や外部接続（DB 再接続）待ちのハンドリング |

```yaml
# 推奨パターン: startupProbe で起動完了まで猶予を与え、
# その後 liveness / readiness へ監視を引き継ぐ
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30  # 30回 × 5秒 = 最大 150秒の起動猶予
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 30
  failureThreshold: 3   # 3回 × 30秒 = 90秒間応答なしで再起動

readinessProbe:
  httpGet:
    path: /ready         # 独立したエンドポイント: DB、キャッシュ接続等を検証
    port: 8080
  periodSeconds: 10
  failureThreshold: 2
```

```yaml
# 誤った設定: startupProbe を使わず initialDelaySeconds だけで待つ
# アプリの起動に 60秒かかる場合は startupProbe を使用すべき
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60   # 悪手: 固定待機の推測値であり競合状態を招く
```

---

## Service と Ingress

### Service 種別

```yaml
# ClusterIP（デフォルト） — クラスタ内通信専用
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-namespace
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

```yaml
# LoadBalancer — クラウドプロバイダ経由の外部トラフィック受信用
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8080
```

### Ingress と TLS 設定

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-namespace
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: my-app-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## ConfigMap と Secret

### ConfigMap — 非機密の設定情報

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: my-namespace
data:
  LOG_LEVEL: "info"
  APP_ENV: "production"
  MAX_CONNECTIONS: "100"
  # 複雑な設定ファイルはファイルとしてマウント可能
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
```

```yaml
# ConfigMap をファイルとして Pod にマウントする例
volumes:
  - name: config
    configMap:
      name: my-app-config
      items:
        - key: app.yaml
          path: app.yaml
volumeMounts:
  - name: config
    mountPath: /etc/app
    readOnly: true
```

### Secret — 機密情報

```bash
# リテラル値から Secret を作成（CLI で作成後、Vault や SOPS で管理）
kubectl create secret generic my-app-secrets \
  --from-literal=db-password='s3cr3t' \
  --namespace=my-namespace \
  --dry-run=client -o yaml | kubectl apply -f -
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: my-namespace
type: Opaque
# 値は Base64 エンコード（暗号化ではない点に注意 — 本番では Sealed Secrets や ESO を推奨）
data:
  db-password: czNjcjN0  # 's3cr3t' の Base64 値
```

> **重要:** 生の Kubernetes Secret は Base64 エンコードされているだけであり、クラスタ側で保存時暗号化（encryption at rest）が設定されていない限り平文同然です。商用環境では [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) や [External Secrets Operator](https://external-secrets.io) の利用を推奨します。

---

## リソースの requests と limits

```yaml
resources:
  requests:       # スケジューラが Pod をノードに配置する際の基準
    cpu: "100m"   # 100 ミリコア = 0.1 CPU
    memory: "128Mi"
  limits:         # これを超えるとスロットリング（CPU）または強制終了（メモリOOM）
    cpu: "500m"
    memory: "256Mi"
```

**ワークロード別の設計目安:**

| ワークロード種別 | CPU Request | Memory Request | 備考 |
|---------------|-------------|----------------|-------|
| Web API | 100–250m | 128–256Mi | limits は requests の 2〜4倍に設定 |
| Worker / Consumer | 250–500m | 256–512Mi | 挙動予測可能性のためメモリ limits = requests を推奨 |
| JVM アプリケーション | 500m–1 | 512Mi–2Gi | JVM オーバーヘッドを考慮し `-Xmx` 以上の余裕を持たせる |
| サイドカー | 10–50m | 32–64Mi | 極力最小限に抑える |

```yaml
# 誤った設定: requests や limits が未定義 — スケジューリングが不安定化しノード枯渇を招く
containers:
  - name: app
    image: myapp:latest
    # resources: {} の欠落 — 本番では極めて危険

# 誤った設定: requests を指定せず limits のみ定義 — requests が limits と同値になり過剰予約される
resources:
  limits:
    cpu: "2"
    memory: "1Gi"
  # requests が省略されているため limits と同一値が割り当てられてしまう
```

---

## RBAC（Role と ServiceAccount）

### 最小特権の原則 (Principle of Least Privilege)

**アプリが Kubernetes API を呼び出すかどうかに応じた2つのパターン:**

#### パターン A — アプリが Kubernetes API を必要としない場合（一般的な Web アプリの大半）

ServiceAccount のトークン自動マウントを無効化します。Role や RoleBinding の作成も不要です。

```yaml
# トークンマウントを無効化した ServiceAccount — 最も安全なデフォルト
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
automountServiceAccountToken: false   # Pod 内に K8s API トークンを注入しない
```

```yaml
# Deployment 側の参照設定
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false   # 念のため Pod レベルでも明示的に無効化
```

#### パターン B — アプリが Kubernetes API を必要とする場合（Operator、コントローラ、設定ウォッチャー等）

トークンを有効化し、実際に必要最小限の権限のみをスコープを絞って付与します。

```yaml
# 1. ServiceAccount — トークン発行を有効化
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
automountServiceAccountToken: true    # アプリが K8s API を叩くためトークンが必要
```

```yaml
# 2. Role — 必要な操作のみを特定リソースに限定して許可（Namespace スコープ）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: my-namespace
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]    # 特定リソースの読み取りのみ
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["my-app-secrets"]  # 特定の名前の Secret のみに限定
    verbs: ["get"]
```

```yaml
# 3. Role と ServiceAccount のバインド
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-rolebinding
  namespace: my-namespace
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-namespace
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: my-app-role
```

```yaml
# 4. Deployment での参照
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      # automountServiceAccountToken は SA の設定に基づき自動注入される
```

---

## Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-namespace
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2      # 高可用性（HA）のため本番では常に 2 以上
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # 平均 CPU 使用率が 70% を超えたらスケールアウト
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

> HPA は使用率を「現在の使用量 / requests 値」として算出するため、対象コンテナすべてに `resources.requests` が設定されている必要があります。

---

## PodDisruptionBudget (PDB)

ノードのメンテナンス（drain）やローリングアップデート時に、過剰な Pod が同時に停止することを防止します：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: my-namespace
spec:
  minAvailable: 2           # または maxUnavailable: 1 を指定
  selector:
    matchLabels:
      app: my-app
```

---

## Namespace とマルチテナンシー

```bash
# Namespace の作成
kubectl create namespace my-namespace

# ResourceQuota を適用して Namespace のリソース消費上限を設定
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-namespace-quota
  namespace: my-namespace
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
EOF
```

---

## Job と CronJob

```yaml
# 単発実行 Job（DB マイグレーション、バッチ処理等）
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: my-namespace
spec:
  backoffLimit: 3          # 失敗時のリトライ回数上限（最大3回）
  ttlSecondsAfterFinished: 3600   # 完了後 1時間で自動削除
  template:
    spec:
      restartPolicy: OnFailure    # Job では Always は不可（OnFailure または Never）
      containers:
        - name: migrate
          image: ghcr.io/org/my-app:1.0.0
          command: ["python", "manage.py", "migrate"]
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
```

```yaml
# 定期実行 CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
  namespace: my-namespace
spec:
  schedule: "0 2 * * *"         # 毎日午前 2時実行
  concurrencyPolicy: Forbid      # 前回の処理が実行中なら新規起動を抑止
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: ghcr.io/org/cleanup:1.0.0
              resources:
                requests:
                  cpu: "50m"
                  memory: "64Mi"
```

---

## kubectl デバッグチートシート

```bash
# --- Pod のステータスとログ確認 ---
kubectl get pods -n my-namespace
kubectl get pods -n my-namespace -o wide          # 割り当てノードの表示
kubectl describe pod <pod-name> -n my-namespace   # イベントと状態詳細の確認
kubectl logs <pod-name> -n my-namespace           # 現在のログ出力
kubectl logs <pod-name> -n my-namespace --previous  # クラッシュ直前のログ出力
kubectl logs <pod-name> -n my-namespace -c <container>  # 複数コンテナ時の指定ログ

# --- 実行中コンテナへの接続 ---
kubectl exec -it <pod-name> -n my-namespace -- sh
kubectl exec -it <pod-name> -n my-namespace -- bash

# --- リソース消費状況の確認 ---
kubectl top pods -n my-namespace
kubectl top nodes

# --- Deployment の運用とロールバック ---
kubectl rollout status deployment/my-app -n my-namespace
kubectl rollout history deployment/my-app -n my-namespace
kubectl rollout undo deployment/my-app -n my-namespace      # 直前の版へロールバック
kubectl rollout undo deployment/my-app --to-revision=2 -n my-namespace

# --- 手動スケーリング ---
kubectl scale deployment my-app --replicas=5 -n my-namespace

# --- イベントログの調査（クラスタ全体の事象） ---
kubectl get events -n my-namespace --sort-by='.lastTimestamp'

# --- ローカルデバッグ用ポート転送 ---
kubectl port-forward pod/<pod-name> 8080:8080 -n my-namespace
kubectl port-forward svc/my-app 8080:80 -n my-namespace

# --- YAML の妥当性検証（Dry-Run） ---
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server   # 実クラスタのスキーマに対して検証
```

### 代表的なエラー事象の診断

```bash
# CrashLoopBackOff: コンテナが起動直後にクラッシュを繰り返している状態
kubectl logs <pod-name> --previous -n my-namespace  # クラッシュ時のログを確認
kubectl describe pod <pod-name> -n my-namespace     # 終了コードや OOMKilled の有無を確認

# ImagePullBackOff: コンテナイメージの取得に失敗
kubectl describe pod <pod-name> -n my-namespace     # Events セクションのエラー詳細を確認
# 主な原因: イメージタグの誤り、imagePullSecrets の欠落、プライベートレジストリ認証失敗

# Pending: Pod がどのノードにもスケジュールされない状態
kubectl describe pod <pod-name> -n my-namespace
# 主な原因: クラスタ内の空きリソース不足、nodeSelector / affinity の不一致、taint/toleration の不一致

# OOMKilled: メモリ制限を超過してカーネルによりプロセスが強制終了された状態
# メモリ limits の引き上げ、またはアプリケーションのメモリリークを調査
kubectl describe pod <pod-name> -n my-namespace | grep -A5 "Last State"
```

---

## アンチパターン

```yaml
# 悪手: :latest タグの使用 — デプロイの非決定性・再現不能を招く
image: myapp:latest

# 推奨: イミュータブルな特定タグ（Git コミット SHA またはセマンティックバージョニング）を指定
image: ghcr.io/org/myapp:1.4.2
# または
image: ghcr.io/org/myapp@sha256:abc123...

# ---

# 悪手: root ユーザーでのコンテナ実行
securityContext: {}    # デフォルトで root 実行になる

# 推奨: 非 root かつ UID を明示
securityContext:
  runAsNonRoot: true
  runAsUser: 1001

# ---

# 悪手: リソース制限の未設定 — 1つの Pod がノード全体のリソースを枯渇させる
containers:
  - name: app
    image: myapp:1.0.0
    # resources が未定義

# 推奨: 必ず requests と limits を定義する
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# ---

# 悪手: ConfigMap に平文のパスワードや機密情報を保存する
apiVersion: v1
kind: ConfigMap
data:
  DB_PASSWORD: "mysecretpassword"   # 絶対に禁止 — Secret または外部シークレットマネージャを使用

# ---

# 悪手: アプリケーションの ServiceAccount に ClusterAdmin 権限を付与する
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
roleRef:
  kind: ClusterRole
  name: cluster-admin    # アプリケーションにクラスタ全体の神権限を与えてしまう

# ---

# 悪手: PDB で minAvailable: 0 を設定する — 予算の意味をなさなくなる
spec:
  minAvailable: 0

# ---

# 悪手: Job で restartPolicy: Always を設定する（無限再起動ループの原因となる）
spec:
  restartPolicy: Always   # Job では OnFailure または Never を使用すること
```

---

## 本番レディ検証チェックリスト

### セキュリティ
- [ ] コンテナが非 root ユーザーで実行されている（`runAsNonRoot: true`、`runAsUser` の明示）
- [ ] `readOnlyRootFilesystem: true` が設定され、書き込み領域に `emptyDir` がマウントされている
- [ ] `allowPrivilegeEscalation: false` が設定されている
- [ ] 不要な全 Linux ケーパビリティがドロップされている（`capabilities.drop: [ALL]`）
- [ ] `default` ではなくアプリ専用の ServiceAccount が割り当てられている
- [ ] API 呼び出しが不要な場合、`automountServiceAccountToken: false` が設定されている
- [ ] RBAC が最小特権に従っている（不可欠な場合を除き `ClusterRole` ではなく `Role` を使用）
- [ ] 機密情報が Sealed Secrets または External Secrets Operator 経由で管理されている

### 信頼性・可用性
- [ ] 3つのプローブすべて（Startup + Liveness + Readiness）が適切に構成されている
- [ ] 全コンテナにリソースの requests および limits が漏れなく定義されている
- [ ] 本番ワークロードのレプリカ数が 2 以上（`minReplicas: 2+`）である
- [ ] ステートフルまたは重要サービスに対して PodDisruptionBudget（PDB）が定義されている
- [ ] `RollingUpdate` 戦略で `maxUnavailable: 0` が設定されている
- [ ] 変動トラフィックを扱うサービスに HPA が設定されている

### オブザーバビリティ
- [ ] アプリケーションが `/health`（Liveness）および `/ready`（Readiness）エンドポイントを公開している
- [ ] 構造化 JSON ログが出力されている（ログ内に個人情報・機密情報が含まれていない）
- [ ] リソースに一貫したラベル（`app`, `version`, `environment` など）が付与されている

---

## 関連スキル

- `docker-patterns` — マルチステージビルドとコンテナイメージセキュリティ
- `deployment-patterns` — CI/CD パイプライン、ロールバック戦略、ヘルスチェックエンドポイント設計
- `security-review` — 包括的なセキュリティハードニングと脆弱性検査
- `git-workflow` — K8s との GitOps 連携（ArgoCD / Flux パターン）
