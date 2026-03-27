# Hello Kubernetes

**1つの Docker イメージから2種類の Web サービスを起動する** 構成です。

## 前提条件

- [OrbStack](https://orbstack.dev/) がインストール済み
- OrbStack の Kubernetes が有効（Settings → Kubernetes → Enable Kubernetes）

有効化すると `kubectl` が自動的に使えるようになります。

```bash
kubectl get nodes
```

ノードが表示されれば準備完了です。

## プロジェクト構成

```
.
├── app/
│   ├── Dockerfile             # Nginx イメージ
│   ├── docker-entrypoint.sh   # 環境変数 VARIANT で配信する HTML を切り替え
│   ├── index-blue.html
│   └── index-green.html
├── k8s/
│   ├── kustomization.yaml     # Kustomize 設定（namespace 一括注入・適用順制御）
│   ├── namespace.yaml         # hello-k8s Namespace
│   ├── deployment-blue.yaml   # Blue Deployment（env VARIANT=blue）
│   ├── deployment-green.yaml  # Green Deployment（env VARIANT=green）
│   ├── service-blue.yaml      # Blue Service（NodePort 30080）
│   └── service-green.yaml     # Green Service（NodePort 30081）
└── README.md
```

## 手順

### 1. Docker イメージをビルド

```bash
docker build -t hello-k8s-web ./app
```

### 2. Kubernetes にデプロイ

```bash
kubectl apply -k k8s/
```

`hello-k8s` Namespace が作成され、その中にリソースがデプロイされます。
Pod が Running になるまで待ちます。

```bash
kubectl get pods -n hello-k8s -w
```

Blue 2つ、Green 2つの計4 Pod が起動します。

### 3. 動作確認

**方法 A: NodePort でアクセス**

```bash
# Blue（ポート 30080）
curl http://localhost:30080

# Green（ポート 30081）
curl http://localhost:30081
```

**方法 B: OrbStack のドメインでアクセス（推奨）**

OrbStack では Service 名でアクセスできます。
この方法は Service の ClusterIP に直接ルーティングされるため、Service の `port: 8080` を指定してアクセスします。
方法 A の `:30080` / `:30081` は NodePort（ノード上の公開ポート）なので、ここでは使いません。

```bash
# Blue
curl http://blue.hello-k8s.svc.cluster.local:8080

# Green
curl http://green.hello-k8s.svc.cluster.local:8080
```

またはブラウザで上記 URL を開きます。

## 学習ポイント: ENTRYPOINT と CMD

### Kubernetes レベル

```yaml
spec:
  containers:
    - name: web-server
      image: hello-k8s-web
      env:
        - name: VARIANT
          value: "green"
```

ENTRYPOINT と CMD はそのまま使い、環境変数 `VARIANT` で配信する HTML を切り替えます。

| Docker       | Kubernetes | 本プロジェクトでの値                    |
|--------------|------------|----------------------------------------|
| `ENTRYPOINT` | `command`  | `/docker-entrypoint.sh`（変更なし）      |
| `CMD`        | `args`     | `["nginx", "-g", "daemon off;"]`（変更なし） |
| 環境変数      | `env`      | `VARIANT=blue` or `VARIANT=green`       |

## 学習ポイント: Namespace と DNS

Kubernetes の Service には、以下の形式で DNS 名が自動的に割り当てられます。

```
<Service名>.<Namespace名>.svc.cluster.local
```

本プロジェクトでは `hello-k8s` Namespace を使っているため、DNS 名は次のようになります。

| Service | DNS 名 |
|---------|--------|
| `blue` | `blue.hello-k8s.svc.cluster.local` |
| `green` | `green.hello-k8s.svc.cluster.local` |

## 学習ポイント: Liveness / Readiness Probe

Kubernetes は Pod 内のコンテナが正常かどうかを **Probe（ヘルスチェック）** で監視します。

| Probe | 役割 | 失敗時の動作 |
|-------|------|-------------|
| **Liveness Probe** | コンテナが生きているか | コンテナを再起動 |
| **Readiness Probe** | リクエストを受け付けられるか | Service のエンドポイントから除外 |

本プロジェクトでは `httpGet` 方式で `/healthz` エンドポイントを監視しています。

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 1
  periodSeconds: 5
```

### Probe の動作を確認する

Pod の Probe 設定は `describe` で確認できます。

```bash
kubectl describe pod -n hello-k8s -l variant=blue | grep -A5 "Liveness\|Readiness"
```

Readiness Probe が失敗すると、Service の Endpoints から Pod が外れます。
Liveness Probe が失敗すると、kubelet がコンテナを再起動します。

## Kustomize

本プロジェクトでは [Kustomize](https://kustomize.io/) を使ってマニフェストを管理しています。

### kustomization.yaml の役割

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-k8s
resources:                     # 適用するマニフェストと順序を定義
  - namespace.yaml
  - deployment-blue.yaml
  - ...
```

Kustomize を使うことで、以下のメリットがあります。

- **namespace の一括注入**: 個々のマニフェストに `namespace:` を書く必要がない
- **適用順の制御**: `resources:` の記載順でマニフェストが適用される（Namespace → Deployment → Service）
- **ビルド結果の確認**: `kubectl kustomize k8s/` で適用前に最終的な YAML を確認できる

### kubectl apply -f と -k の違い

| コマンド | 動作 |
|----------|------|
| `kubectl apply -f k8s/` | ディレクトリ内の YAML をアルファベット順にそのまま適用 |
| `kubectl apply -k k8s/` | Kustomize で変換してから適用（namespace 注入・順序制御あり） |

## セルフヒーリングを体験する

各 Deployment は `replicas: 2` で Pod を維持します。

まず、variant ラベルで Pod を確認します。

```bash
kubectl get pods -n hello-k8s -l variant=blue
kubectl get pods -n hello-k8s -l variant=green
```

Blue の Pod を1つ削除してみます。

```bash
kubectl delete pod -n hello-k8s -l variant=blue --field-selector=status.phase=Running --grace-period=0 | head -1
```

別ターミナルで監視すると、新しい Pod が即座に作成される様子を観察できます。

```bash
kubectl get pods -n hello-k8s -l variant=blue -w
```

全ての Pod を削除しても、Deployment が `replicas: 2` の状態に自動復旧します。

```bash
kubectl delete pods -n hello-k8s -l app=hello-k8s
kubectl get pods -n hello-k8s -w
```

## クリーンアップ

Namespace を削除すると、中のリソースがすべて一括削除されます。

```bash
kubectl delete ns hello-k8s
```

マニフェストファイルを指定して削除する場合は以下でも可能です。

```bash
kubectl delete -k k8s/
```

## 参考

- [hello-k8s-logging](https://github.com/uraitakahito/hello-k8s-logging) — Kubernetes でのロギング構成
