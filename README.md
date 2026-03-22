# Hello Kubernetes on OrbStack

OrbStack の Kubernetes 機能を使って「Hello World」を動かすミニマル構成です。

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
│   ├── Dockerfile      # Nginx コンテナイメージ定義
│   └── index.html      # Hello World ページ
├── k8s/
│   ├── deployment.yaml # Deployment マニフェスト
│   └── service.yaml    # Service (NodePort) マニフェスト
└── README.md
```

## 手順

### 1. Docker イメージをビルド

OrbStack 環境ではローカルビルドしたイメージを Kubernetes から直接参照できます。

```bash
docker build -t hello-kubernetes:latest ./app
```

### 2. Kubernetes にデプロイ

```bash
kubectl apply -f k8s/
```

Pod が Running になるまで待ちます。

```bash
kubectl get pods -w
```

### 3. 動作確認

**方法 A: NodePort でアクセス**

```bash
curl http://localhost:30080
```

**方法 B: OrbStack のドメインでアクセス（推奨）**

OrbStack では Service 名でアクセスできます。

```bash
curl http://hello-k8s-service.default.svc.cluster.local
```

またはブラウザで `http://hello-k8s-service.default.svc.cluster.local` を開きます。

いずれかの方法で「Hello, Kubernetes!」が表示されれば成功です。

### 4. セルフヒーリングを体験する

Deployment は `replicas` で指定した数の Pod を常に維持しようとします。
Pod が消えても自動的に新しい Pod を起動してくれる、この仕組みを**セルフヒーリング（自己修復）**と呼びます。

まず、3つの Pod が Running であることを確認します。

```bash
kubectl get pods
```

出力例:

```
NAME                                READY   STATUS    RESTARTS   AGE
hello-kubernetes-xxxxxxxxxx-xxxxx   1/1     Running   0          60s
hello-kubernetes-xxxxxxxxxx-yyyyy   1/1     Running   0          60s
hello-kubernetes-xxxxxxxxxx-zzzzz   1/1     Running   0          60s
```

1つの Pod を手動で削除してみます。

```bash
kubectl delete pod <上の NAME から1つコピー>
```

別ターミナルで監視すると、削除された Pod の代わりに新しい Pod が即座に作成される様子を観察できます。

```bash
kubectl get pods -w
```

複数の Pod を同時に削除しても、全て削除しても、Deployment が `replicas: 3` の状態に自動復旧します。

```bash
kubectl delete pods -l app=hello-kubernetes
kubectl get pods -w
```

## クリーンアップ

```bash
kubectl delete -f k8s/
```

Docker イメージも不要であれば削除します。

```bash
docker rmi hello-kubernetes:latest
```

## トラブルシューティング

### Pod が起動しない

```bash
kubectl describe pod -l app=hello-kubernetes
kubectl logs -l app=hello-kubernetes
```

### イメージが見つからない（ErrImagePull）

`imagePullPolicy: Never` が設定されているか確認してください。
ローカルでイメージがビルド済みか `docker images | grep hello-kubernetes` で確認できます。

### NodePort に接続できない

```bash
kubectl get svc hello-k8s-service
```

PORT 列に `80:30080/TCP` と表示されていることを確認してください。
