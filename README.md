# Hello Kubernetes

1つの Docker イメージから2種類の Web サービスを起動するシンプルなKubernetesの事例です。

## 前提条件

- [OrbStack](https://orbstack.dev/) がインストール済み
- OrbStack の Kubernetes が有効（Settings → Kubernetes → Enable Kubernetes）

## プロジェクト構成

```
.
├── app/
│   ├── Dockerfile             # Nginx イメージ
│   ├── docker-entrypoint.sh   # 環境変数 VARIANT で配信する HTML を切り替え
│   ├── index-blue.html
│   └── index-green.html
├── k8s/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment-blue.yaml
│   ├── deployment-green.yaml
│   ├── service-blue.yaml      # web-blue Service（NodePort 30080）
│   └── service-green.yaml     # web-green Service（NodePort 30081）
└── README.md
```

## 手順

### 1. Docker イメージをビルド

```bash
docker build -t hello-k8s-web ./app
```

### 2. Kubernetes にデプロイ

`-k` で Kustomize で変換を行い、Kindベースのソート順で適用:

```bash
kubectl apply -k k8s/
```

`demo` Namespace が作成され、その中にリソースがデプロイされます。
Pod が Running になるまで待ちます。

```bash
kubectl get pods -n demo -w
```

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
curl http://web-blue.demo.svc.cluster.local:8080

# Green
curl http://web-green.demo.svc.cluster.local:8080
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

本プロジェクトでは `demo` Namespace を使っているため、DNS 名は次のようになります。

| Service | DNS 名 |
|---------|--------|
| `web-blue` | `web-blue.demo.svc.cluster.local` |
| `web-green` | `web-green.demo.svc.cluster.local` |

## 学習ポイント: ClusterIP

Service を作成すると、Kubernetes が **ClusterIP**（クラスタ内部でのみ有効な仮想 IP）を自動的に割り当てます。
ClusterIP はどのノードにも Pod にも紐づかず、kube-proxy が iptables / IPVS ルールで実現しています。

```bash
kubectl get svc -n demo
```

```
NAME        TYPE       CLUSTER-IP      PORT(S)          AGE
web-blue    NodePort   10.96.xxx.xxx   8080:30080/TCP   5m
web-green   NodePort   10.96.xxx.xxx   8080:30081/TCP   5m
```

`CLUSTER-IP` 列が各 Service に割り当てられた仮想 IP です。

> **注意:** `docker ps` では ClusterIP は表示されません。ClusterIP は Kubernetes のネットワーク層で管理されており、Docker のコンテナレベルからは見えません。

DNS 名 (`web-blue.demo.svc.cluster.local`) は、この ClusterIP に解決されます。

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
kubectl describe pod -n demo -l variant=blue | grep -A5 "Liveness\|Readiness"
```

Readiness Probe が失敗すると、Service の Endpoints から Pod が外れます。
Liveness Probe が失敗すると、kubelet がコンテナを再起動します。

## Kustomize

本プロジェクトでは [Kustomize](https://kustomize.io/) を使ってマニフェストを管理しています。

### kubectl apply -f と -k の違い

| コマンド | 動作 |
|----------|------|
| `kubectl apply -f k8s/` | ディレクトリ内の YAML をアルファベット順にそのまま適用 |
| `kubectl apply -k k8s/` | Kustomize で変換してから適用（namespace 注入・Kind ベースの自動ソートあり） |

## セルフヒーリングを体験する

各 Deployment は `replicas: 2` で Pod を維持します。

まず、variant ラベルで Pod を確認します。

```bash
kubectl get pods -n demo -l variant=blue
kubectl get pods -n demo -l variant=green
```

Blue の Pod を1つ削除してみます。

```bash
kubectl delete pod -n demo -l variant=blue --field-selector=status.phase=Running --grace-period=0 | head -1
```

別ターミナルで監視すると、新しい Pod が即座に作成される様子を観察できます。

```bash
kubectl get pods -n demo -l variant=blue -w
```

全ての Pod を削除しても、Deployment が `replicas: 2` の状態に自動復旧します。

```bash
kubectl delete pods -n demo -l app=web
kubectl get pods -n demo -w
```

## クリーンアップ

Namespace を削除すると、中のリソースがすべて一括削除されます。

```bash
kubectl delete ns demo
```

マニフェストファイルを指定して削除する場合は以下でも可能です。

```bash
kubectl delete -k k8s/
```

## 参考

- [hello-k8s-logging](https://github.com/uraitakahito/hello-k8s-logging) — Kubernetes でのロギング構成
