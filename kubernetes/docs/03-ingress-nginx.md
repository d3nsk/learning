# 03. Ingress + nginx でルーティング

## 概要

Ingress を使って、1つのエントリポイントから複数の Service へパスベースでルーティングする。
nginx Ingress Controller を Ingress の実装として使う。

## 前提条件

- [02-basic-operations.md](./02-basic-operations.md) が完了していること
- `kind-mycluster` クラスタが起動中であること

---

## Ingress とは何か

### Service だけだと何が困るか

Service (NodePort) を使うと「ポート番号でアプリを公開」できるが、アプリごとにポートが増えていく。

```
クライアント
 ├── :30080 → Service A (frontend)
 ├── :30081 → Service B (api)
 └── :30082 → Service C (admin)
```

本番ではポート番号ではなく **パスやホスト名** でルーティングしたい。

```
クライアント
 └── :80
      ├── /         → Service A (frontend)
      ├── /api      → Service B (api)
      └── /admin    → Service C (admin)
```

これを実現するのが **Ingress**。

### 全体像

```
┌───────────────────────────────────────────────┐
│                  クラスタ                       │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │         Ingress Controller              │  │
│  │         (nginx が動いている Pod)         │  │
│  └──────────────────┬──────────────────────┘  │
│                     │ Ingress ルールを読んで振り分け
│          ┌──────────┴──────────┐              │
│          ↓                    ↓              │
│   ┌─────────────┐    ┌─────────────┐         │
│   │  Service A  │    │  Service B  │         │
│   │  (frontend) │    │    (api)    │         │
│   └──────┬──────┘    └──────┬──────┘         │
│          ↓                  ↓               │
│       Pod × 2            Pod × 2            │
└───────────────────────────────────────────────┘
        ↑
   Ingress リソース
   (ルーティングルールの定義)
```

- **Ingress Controller**: ルーティングを実際に処理するコンポーネント（nginx）
- **Ingress リソース**: 「このパスはこの Service へ」というルール定義

---

## 1. kind クラスタを Ingress 対応で再作成

kind はデフォルトでポートを外部に公開しない。Ingress が使えるよう設定ファイルでクラスタを作り直す。

```bash
# 既存クラスタを削除
kind delete cluster --name mycluster
```

以下の設定ファイルを作成する：

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

```bash
# 設定ファイルでクラスタを作成
kind create cluster --name mycluster --config kind-config.yaml

# ノード確認
kubectl get nodes
```

---

## 2. nginx Ingress Controller をインストール

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Ingress Controller の Pod が起動するまで待つ：

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

確認：

```bash
kubectl get pods -n ingress-nginx
# NAME                                       READY   STATUS    ...
# ingress-nginx-controller-xxxxxxxxx-xxxxx   1/1     Running   ...
```

---

## 3. アプリをデプロイする

2つの Service を用意してルーティングを試す。

### frontend (/) と api (/api) を作成

```bash
# frontend: nginx のデフォルトページを返す
kubectl create deployment frontend --image=nginx
kubectl expose deployment frontend --port=80

# api: httpbin を使って /api で JSON を返す
kubectl create deployment api --image=kennethreitz/httpbin
kubectl expose deployment api --port=80
```

確認：

```bash
kubectl get deployments
kubectl get services
```

---

## 4. Ingress リソースを作成

ルーティングルールを定義する：

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml

# Ingress の確認
kubectl get ingress
# NAME         CLASS   HOSTS   ADDRESS     PORTS   AGE
# my-ingress   nginx   *       localhost   80      10s
```

---

## 5. 動作確認

```bash
# / → frontend (nginx のデフォルトページ)
curl localhost/

# /api → api (httpbin)
curl localhost/api/get
```

ブラウザで `http://localhost/` を開くと nginx のページ、`http://localhost/api/get` を開くと JSON が返る。

---

## ルーティングの仕組みまとめ

```
curl localhost/api/get
        ↓
  nginx Ingress Controller
        ↓  /api にマッチ
  Service: api (ClusterIP)
        ↓
  Pod (httpbin)
```

`rewrite-target: /` アノテーションにより `/api` のプレフィックスが除去されてアプリに届く。

---

## クリーンアップ

```bash
kubectl delete ingress my-ingress
kubectl delete service frontend api
kubectl delete deployment frontend api
```

---

## 次のステップ

→ [04-deploy-api.md](./04-deploy-api.md) (作成予定)
