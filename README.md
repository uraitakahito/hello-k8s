# Hello Kubernetes on OrbStack

OrbStack の Kubernetes 機能を使い、**1つの Docker イメージから2種類の Web サービスを起動する**ミニマル構成です。
`ENTRYPOINT` + `CMD` パターンにより、同一イメージでも起動引数で振る舞いを切り替えられることを体験します。

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
│   ├── Dockerfile             # Nginx イメージ（ENTRYPOINT + CMD パターン）
│   ├── docker-entrypoint.sh   # CMD 引数で配信する HTML を切り替え
│   ├── index-blue.html        # Blue 版ページ
│   └── index-green.html       # Green 版ページ
├── k8s/
│   ├── namespace.yaml         # hello-k8s Namespace
│   ├── deployment-blue.yaml   # Blue Deployment（args: ["blue"]）
│   ├── deployment-green.yaml  # Green Deployment（args: ["green"]）
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
kubectl apply -f k8s/
```

`hello-k8s` Namespace が作成され、その中にリソースがデプロイされます。

> **Note:** `kubectl apply -f k8s/` はファイルをアルファベット順に処理するため、初回実行時に Namespace の作成が反映される前に Deployment が適用されエラーになることがあります。その場合は同じコマンドをもう一度実行してください。

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
curl http://hello-k8s-blue.hello-k8s.svc.cluster.local:8080

# Green
curl http://hello-k8s-green.hello-k8s.svc.cluster.local:8080
```

またはブラウザで上記 URL を開きます。

## 学習ポイント: ENTRYPOINT と CMD

### Kubernetes レベル

```yaml
spec:
  containers:
    - name: web-server
      image: hello-k8s-web
      args: ["green"]    # ← Dockerfile の CMD を上書き
```

K8s の `args` は Docker の `CMD` に対応します。
`command` は `ENTRYPOINT` に対応しますが、今回は ENTRYPOINT はそのまま使うため指定しません。

| Docker       | Kubernetes | 本プロジェクトでの値          |
|--------------|------------|-------------------------------|
| `ENTRYPOINT` | `command`  | `/docker-entrypoint.sh`       |
| `CMD`        | `args`     | `["blue"]` or `["green"]`     |

## 学習ポイント: Namespace と DNS

Kubernetes の Service には、以下の形式で DNS 名が自動的に割り当てられます。

```
<Service名>.<Namespace名>.svc.cluster.local
```

本プロジェクトでは `hello-k8s` Namespace を使っているため、DNS 名は次のようになります。

| Service | DNS 名 |
|---------|--------|
| `hello-k8s-blue` | `hello-k8s-blue.hello-k8s.svc.cluster.local` |
| `hello-k8s-green` | `hello-k8s-green.hello-k8s.svc.cluster.local` |

Namespace を指定しない場合は `default` Namespace にデプロイされ、DNS 名は `hello-k8s-blue.default.svc.cluster.local` となります。

Namespace はリソースの論理的なグループであり、以下のメリットがあります。

- 関連リソースをまとめて管理できる
- `kubectl delete ns hello-k8s` で Namespace ごと一括削除できる
- チームやアプリケーション単位でリソースを分離できる

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
kubectl delete pods -n hello-k8s -l app=hello-kubernetes
kubectl get pods -n hello-k8s -w
```

## クリーンアップ

Namespace を削除すると、中のリソースがすべて一括削除されます。

```bash
kubectl delete ns hello-k8s
```

マニフェストファイルを指定して削除する場合は以下でも可能です。

```bash
kubectl delete -f k8s/
```
