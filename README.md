# Hello Kubernetes on OrbStack

OrbStack の Kubernetes 機能を使い、**1つの Docker イメージから2種類の Web サービスを起動する** 構成です。
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
│   ├── kustomization.yaml     # Kustomize 設定（namespace 一括注入・適用順制御）
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
| `blue` | `blue.hello-k8s.svc.cluster.local` |
| `green` | `green.hello-k8s.svc.cluster.local` |

Namespace がリソースのスコープを提供するため、Service 名にプロジェクト接頭辞（`hello-k8s-`）を付ける必要がありません。
`blue` という短い名前でも `hello-k8s` Namespace 内で一意であれば十分です。

Namespace を指定しない場合は `default` Namespace にデプロイされ、DNS 名は `blue.default.svc.cluster.local` となります。

Namespace はリソースの論理的なグループであり、以下のメリットがあります。

- Namespace がスコープとなるため、リソース名を短くシンプルに保てる
- 関連リソースをまとめて管理できる
- `kubectl delete ns hello-k8s` で Namespace ごと一括削除できる
- チームやアプリケーション単位でリソースを分離できる

## 学習ポイント: Kustomize

本プロジェクトでは [Kustomize](https://kustomize.io/) を使ってマニフェストを管理しています。
Kustomize は kubectl に組み込まれており、追加インストールなしで使えます。

### kustomization.yaml の役割

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-k8s          # 全リソースに namespace を一括注入
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
kubectl delete -k k8s/
```
