# Kubernetes: Deployment & ローリングアップデート

## Pod の命名規則

```
# Deployment経由で作った場合
mynginx-deploy - 7bbd77b569 - px89b
   ①                ②            ③

① Deployment名（自分でつけた名前）
② ReplicaSetのハッシュ（PodTemplateの内容から生成）
③ Pod固有のランダムID
```

```
# ReplicaSet を直接作った場合
my-rs - px89b
  ①       ②

① ReplicaSet名
② Pod固有のランダムID
```

### 親子関係

```
Deployment:   mynginx-deploy
ReplicaSet:   mynginx-deploy-7bbd77b569
Pod:          mynginx-deploy-7bbd77b569-px89b
```

---

## イメージ更新とローリングアップデート

### コマンド

```bash
kubectl set image deployment/mynginx-deploy nginx=nginx:alpine
#                                            ↑コンテナ名  ↑新しいイメージ
```

### 内部で起きていること

イメージ変更 → PodTemplateが変わる → **新しいReplicaSetが作られる**

```
旧RS: mynginx-deploy-7bbd77b569  (nginx:latest)
新RS: mynginx-deploy-9ac34df821  (nginx:alpine)  ← 新しく作られた
```

ローリングアップデートの流れ：
1. 新RSのPodを少しずつ増やす
2. 旧RSのPodを少しずつ消す
3. 全部切り替わったら完了

→ **ダウンタイムなしで更新できる**のがDeploymentの強み

---

## 本番環境でのリリースフロー

```
コード変更
↓
CI（GitHub Actions等）でDockerイメージをビルド
↓
コンテナレジストリにpush
  例: ghcr.io/yourname/myapp:abc1234
↓
kubectl set image ... または Argo CD等が自動でDeploymentを更新
↓
ローリングアップデートで無停止リリース
```

### イメージタグの管理

本番では `latest` のような可変タグは使わず、**コミットハッシュ**で固定するのが一般的。

```bash
# NG: タグは上書きされる可能性がある
nginx:latest
myapp:v1.0

# OK: コミットハッシュで一意に確定
ghcr.io/yourname/myapp:abc1234

# より厳密: SHAダイジェストで固定
ghcr.io/yourname/myapp@sha256:a3f9...
```

同じタグでも中身が変わりうるため、デプロイの再現性・追跡可能性を担保するためにハッシュで管理する。