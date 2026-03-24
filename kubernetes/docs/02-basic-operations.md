# 02. 基本操作

## 概要

kubectl の基本コマンドを使って、KubernetesのコアリソースであるPod・Service・Deploymentを実際に動かしながら学ぶ。

## 前提条件

- [01-setup.md](./01-setup.md) が完了していること
- `kind-mycluster` クラスタが起動中であること

```bash
# 今どのクラスタに接続してるか確認
kubectl config current-context

# 手元にある全クラスタ一覧
kubectl config get-contexts

# クラスタを切り替える
kubectl config use-context <context-name>

# クラスタが動いているか確認
kubectl get nodes
```

---

## リソースの全体像

```
┌─────────────────────────────────────────────┐
│               Kubernetes クラスタ             │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │           Deployment                 │   │
│  │  (Pod の数・バージョンを管理)          │   │
│  │                                      │   │
│  │   ┌────────┐  ┌────────┐            │   │
│  │   │  Pod   │  │  Pod   │  ...       │   │
│  │   │[コンテナ]│  │[コンテナ]│            │   │
│  │   └────────┘  └────────┘            │   │
│  └──────────────────────────────────────┘   │
│                    ↑                        │
│              ┌──────────┐                   │
│              │ Service  │ ← Pod への窓口     │
│              └──────────┘                   │
└─────────────────────────────────────────────┘
```

- **Pod**: コンテナを動かす最小単位
- **Deployment**: Podの数やバージョンを管理する仕組み
- **Service**: Podへのネットワークアクセスを提供する窓口

---

## 1. Pod

### Podとは

コンテナを動かす最小単位。1つ以上のコンテナをまとめたグループ。

```
┌─────────────────────┐
│        Pod          │
│  ┌───────────────┐  │
│  │  コンテナ      │  │  ← 実際のアプリが動く場所
│  └───────────────┘  │
│  IPアドレス: 10.x.x.x │
└─────────────────────┘
```

> **なぜコンテナを直接使わないのか**: PodはIPアドレスや起動設定をコンテナにまとめて与える。複数コンテナを同じネットワーク・ストレージで動かしたい場合もPodでグループ化する。

### Podを作る

```bash
# nginx の Pod を起動
kubectl run mynginx --image=nginx

# Pod の一覧を確認
kubectl get pods

# 期待する出力:
# NAME       READY   STATUS    RESTARTS   AGE
# mynginx    1/1     Running   0          10s
```

### Podの詳細を見る

```bash
kubectl describe pod mynginx
```

### Podのログを見る

```bash
kubectl logs mynginx
```

### PodのIPに直接アクセスする (動作確認)

```bash
# Pod 内でコマンドを実行
kubectl exec -it mynginx -- bash

# 内部から curl で自分に接続できるか確認
curl localhost
exit
```

### Podを削除する

```bash
kubectl delete pod mynginx
```

> **注意**: Podを直接作ると、削除したら終わり。再起動・複数台管理をするには Deployment を使う。

---

## 2. Deployment

### Deploymentとは

Podを管理する仕組み。「このイメージを3台動かせ」「バージョンをアップデートしろ」などを指示できる。

```
Deployment
 └── ReplicaSet (指定した数のPodを維持する)
      ├── Pod 1
      ├── Pod 2
      └── Pod 3
```

### Deploymentを作る

```bash
# nginx を 3 台で動かす Deployment を作成
kubectl create deployment mynginx-deploy --image=nginx --replicas=3

# Deployment の確認
kubectl get deployments

# Pod が 3 つ作られているか確認
kubectl get pods
```

### Deploymentの詳細を見る

```bash
kubectl describe deployment mynginx-deploy
```

### Pod数を変える (スケール)

```bash
# 5 台に増やす
kubectl scale deployment mynginx-deploy --replicas=5

kubectl get pods  # 5つになっていることを確認
```

### イメージを更新する (ローリングアップデート)

[operations-deployment.md](./operations-deployment.md) を参照。

```bash
# nginx:alpine に更新
kubectl set image deployment/mynginx-deploy nginx=nginx:alpine

# 更新の状況を確認
kubectl rollout status deployment/mynginx-deploy
```

### 更新を元に戻す (ロールバック)

```bash
kubectl rollout undo deployment/mynginx-deploy
```

### Deploymentを削除する

```bash
kubectl delete deployment mynginx-deploy
```

---

## 3. Service

### Serviceとは

PodへのネットワークアクセスのためのIPアドレス・ポートを提供する窓口。Podは再起動するたびにIPが変わるので、固定の接続先として Service を使う。

```
クライアント
    ↓
 Service (固定IP・ポート)
    ↓
 Pod 1 / Pod 2 / Pod 3  ← ランダムに振り分け (ロードバランシング)
```

### Serviceの種類

詳しくは [service.md](./service.md) を参照

| タイプ | 用途 |
|--------|------|
| ClusterIP | クラスタ内部からのみアクセス可 (デフォルト) |
| NodePort | ノードのIPとポートで外部からアクセス可 |
| LoadBalancer | クラウドのロードバランサを使う (本番向け) |

### Deployment に Service を作る

```bash
# Deployment を先に作成
kubectl create deployment web --image=nginx --replicas=2

# NodePort タイプの Service を作成 (ポート 80 を 30080 で公開)
kubectl expose deployment web --type=NodePort --port=80

# Service の確認
kubectl get services

# 期待する出力:
# NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# web          NodePort   10.96.x.x      <none>        80:3xxxx/TCP   5s
```

### kind でブラウザからアクセスする

kindはDockerコンテナがノードなので、NodePortに直接アクセスできない。`port-forward` を使う。

```bash
# ローカルの 8080 → Service の 80 に転送
kubectl port-forward service/web 8080:80

# 別ターミナルで確認
curl localhost:8080
```

ブラウザで `http://localhost:8080` を開くと nginx のページが表示される。

### クリーンアップ

```bash
kubectl delete service web
kubectl delete deployment web
```

---

## kubectl よく使うコマンド一覧

```bash
# リソース一覧
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all          # Pod/Deployment/Service をまとめて確認

# 詳細確認
kubectl describe pod <名前>
kubectl describe deployment <名前>

# ログ
kubectl logs <Pod名>
kubectl logs -f <Pod名>  # リアルタイムで流す

# Pod 内でコマンド実行
kubectl exec -it <Pod名> -- bash

# リソース削除
kubectl delete pod <名前>
kubectl delete deployment <名前>
kubectl delete service <名前>
```

---

## 次のステップ

→ [03-ingress-nginx.md](./03-ingress-nginx.md) (作成予定)
