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
curl http://hello-kubernetes.default.svc.cluster.local
```

またはブラウザで `http://hello-kubernetes.default.svc.cluster.local` を開きます。

いずれかの方法で「Hello, Kubernetes!」が表示されれば成功です。

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
kubectl get svc hello-kubernetes
```

PORT 列に `80:30080/TCP` と表示されていることを確認してください。
