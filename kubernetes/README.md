# Kubernetes ローカル学習環境

本番に近い構成をローカルで学ぶためのドキュメント集。

## 環境

- **ツール**: kind (Kubernetes in Docker)
- **目的**: nginx + gateway + API の構成を手を動かして学ぶ

## ドキュメント構成

| ファイル | 内容 |
|---------|------|
| [01-setup.md](./docs/01-setup.md) | kind のインストールとクラスタ作成 |
| [02-basic-operations.md](./docs/02-basic-operations.md) | kubectl, Pod, Service, Deployment の基本操作 |
| [03-ingress-nginx.md](./docs/03-ingress-nginx.md) | Ingress + nginx でパスベースルーティング |
| [04-deploy-api.md](./docs/04-deploy-api.md) | 自作 API を Docker イメージ化してデプロイ |

## 学習ロードマップ

1. 環境構築 (kind インストール、クラスタ作成)
2. 基本操作 (kubectl, Pod, Service, Deployment)
3. Ingress + nginx でルーティング
4. API をデプロイ
5. ConfigMap / Secret
6. Helm でパッケージ管理
