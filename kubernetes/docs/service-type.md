# Kubernetes Service type

## Serviceの本質

Serviceとは、Pod群への通信を抽象化する仕組み。以下の3要素がセットで動く。

| 役割 | 担当 |
|------|------|
| 名前解決（名前 → ClusterIP） | CoreDNS |
| 仮想IPのインターセプト | iptables（kube-proxyが設定） |
| 実Podへの転送先リスト | Endpoints / EndpointSlice |

```
Pod
 └─ CoreDNS（名前 → ClusterIP）
      └─ iptablesがインターセプト
           └─ 実PodにDNAT（ランダム分散）
```

どのtypeを選んでもこの内部の仕組みは変わらない。

---

## typeによる上位互換

typeはClusterIPの機能を基盤に、外部への入口が増えるだけ。減ることはない。

```
LoadBalancer
 └─ NodePort
      └─ ClusterIP（すべての基盤）
```

| type | 追加される入口 | 主な用途 |
|------|--------------|---------|
| ClusterIP | なし（クラスタ内部のみ） | マイクロサービス間通信、DB接続 |
| NodePort | `NodeIP:NodePort` で外部からアクセス可 | 開発・検証環境 |
| LoadBalancer | クラウドLBが自動作成・外部IPが払い出される | 本番の外部公開 |
| ExternalName | DNS CNAMEで外部サービスを参照 | 外部サービスへのエイリアス |

---

## LoadBalancer typeの2段階構造

LBはPodを直接知らない。NodeとPodで2回振り分けが起きる。

```
外部ユーザー
 └─ クラウドLB（① Nodeレベル：死活・負荷を見てNodeに振る）
      └─ Node:NodePort
           └─ iptables（② Podレベル：ランダムにPodへ分散）
                └─ 実Pod
```

### 問題：余分なホップ

クラウドLBが「Podのいないノード」に振ることがある。
そのNodeのiptablesが別Nodeの実Podに転送するため、無駄なホップが発生する。

### 解決策：externalTrafficPolicy: Local

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

- そのNodeにいるPodにしか転送しない
- クラウドLB側がHealthCheckでPodのいるNodeだけに振るようになる
- 余分なホップがなくなる

---

## 外部公開の設計パターン

### 基本方針

「外部に出したいServiceだけLoadBalancer、それ以外はClusterIP」

ClusterIPにしておけば外部から到達する手段がそもそも存在しないため、セキュリティ的な隔離にもなる。

```
外部ユーザー
 └─ Service B（type: LoadBalancer） ← 外部公開
      └─ Pod B（APIサーバー）
           └─ Service A（type: ClusterIP） ← 内部専用
                └─ Pod A（DB、内部サービス）
```

### YAMLサンプル

**外部公開するService（APIサーバーなど）**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

**内部専用Service（DBなど）**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  type: ClusterIP   # 省略も可
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## ポイントまとめ

- Serviceの本質は「CoreDNSの名前解決 ＋ iptablesのDNAT」
- typeはその外側の入口の話であり、内部の仕組みは変わらない
- LoadBalancerはクラウドLBのプロビジョニング指示であり、Pod直接通信ではない
- クラウドLBはNodeレベル、iptablesはPodレベルの2段階で振り分けが行われる
- 本番では `externalTrafficPolicy: Local` で余分なホップを防ぐのが推奨