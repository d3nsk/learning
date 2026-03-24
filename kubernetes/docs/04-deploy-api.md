# 04. API をデプロイ

## 概要

自作の API アプリを Docker イメージにして、Kubernetes にデプロイする。
kind はローカルなので、イメージのロード方法が本番と少し異なる点も押さえる。

## 前提条件

- [03-ingress-nginx.md](./03-ingress-nginx.md) が完了していること
- Docker が起動中であること

---

## 全体の流れ

```
① アプリを書く
      ↓
② Dockerfile でイメージをビルド
      ↓
③ kind クラスタにイメージをロード
      ↓
④ Deployment + Service を作成
      ↓
⑤ Ingress でパスを公開
```

---

## 1. サンプル API を作る

Go で書いたシンプルな HTTP API を使う。

```
api/
├── main.go
└── Dockerfile
```

### main.go

```go
package main

import (
    "encoding/json"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("ok"))
    })

    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        hostname, _ := os.Hostname()
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{
            "message":  "Hello from Kubernetes!",
            "hostname": hostname, // Pod 名が入る
        })
    })

    http.ListenAndServe(":8080", nil)
}
```

> `/hello` のレスポンスに `hostname` を入れている。Pod が複数あるとき、どの Pod が応答したか確認できる。

### Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY main.go .
RUN go mod init myapi && go build -o myapi .

FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/myapi .
EXPOSE 8080
CMD ["./myapi"]
```

---

## 2. Docker イメージをビルド

```bash
cd api
docker build -t myapi:v1 .

# ビルド確認
docker images | grep myapi
```

---

## 3. kind クラスタにイメージをロード

kind のノードは Docker コンテナなので、ビルドしたイメージを明示的に読み込む必要がある。

```
通常の本番環境:
  docker push → レジストリ → クラスタが pull

kind (ローカル):
  kind load → クラスタのノードに直接コピー
```

```bash
kind load docker-image myapi:v1 --name mycluster

# ロード確認 (ノード内を確認)
docker exec -it mycluster-control-plane crictl images | grep myapi
```

---

## 4. Deployment を作成

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
        - name: myapi
          image: myapi:v1
          imagePullPolicy: Never   # kind ローカルイメージを使う
          ports:
            - containerPort: 8080
          readinessProbe:          # Pod が Ready になる条件
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
```

> `imagePullPolicy: Never` — kind にロード済みのローカルイメージを使う。`IfNotPresent` だとレジストリを探してしまう場合がある。

```bash
kubectl apply -f api-deployment.yaml

# Pod が 3 つ Running になるまで待つ
kubectl get pods -w
```

---

## 5. Service を作成

```yaml
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapi
spec:
  selector:
    app: myapi
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f api-service.yaml
kubectl get services
```

---

## 6. Ingress にパスを追加

03 で作成した `ingress.yaml` に `/api` のルールを追加する（既存の Ingress がある場合は更新）。

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: myapi
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

---

## 7. 動作確認

```bash
# ヘルスチェック
curl localhost/api/healthz
# ok

# メインエンドポイント
curl localhost/api/hello
# {"hostname":"myapi-xxxxxxxxxx-xxxxx","message":"Hello from Kubernetes!"}

# 複数回叩くと hostname が変わる (ロードバランシング)
for i in $(seq 5); do curl -s localhost/api/hello | grep hostname; done
```

---

## 8. スケールして負荷分散を確認

```bash
# 5 台に増やす
kubectl scale deployment myapi --replicas=5
kubectl get pods

# 各 Pod に分散しているか確認
for i in $(seq 10); do curl -s localhost/api/hello | python3 -m json.tool | grep hostname; done
```

---

## イメージを更新するとき

```bash
# コードを修正したら再ビルド
docker build -t myapi:v2 .

# kind にロード
kind load docker-image myapi:v2 --name mycluster

# Deployment のイメージを更新 (ローリングアップデート)
kubectl set image deployment/myapi myapi=myapi:v2

# 進捗確認
kubectl rollout status deployment/myapi
```

---

## クリーンアップ

```bash
kubectl delete ingress my-ingress
kubectl delete service myapi
kubectl delete deployment myapi
```

---

## 次のステップ

→ [05-configmap-secret.md](./05-configmap-secret.md) (作成予定)
