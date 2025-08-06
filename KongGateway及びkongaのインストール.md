OpenShift Local 上で **Kong Gateway（OSS）+ Konga GUI** をデプロイし、API Gateway として利用可能にするための **完全な手順書**

---

# ✅ OpenShift Local 上で Kong Gateway + Konga を構築する手順書

---

## 📌 前提

| 項目        | 内容                                |
| --------- | --------------------------------- |
| OpenShift | OpenShift Local (CRC) が起動済み       |
| CLI ツール   | `oc`, `kubectl`, `helm` がインストール済み |
| Namespace | `kong` というプロジェクト名で作業              |
| ポート公開方式   | `NodePort` を使用（外部アクセス対応）          |

---

## ① Namespace 作成と SCC 設定（Kong 用）

```bash
oc new-project kong

# PostgreSQL Pod が privileged を必要とするため default SA に付与
oc adm policy add-scc-to-user privileged -z default -n kong

# 複数の SA をまとめて許可したい場合はこちらも推奨
oc adm policy add-scc-to-group privileged system:serviceaccounts:kong
```

---

## ② Helm Values ファイル作成（Kong 用）

`kong-values.yaml`：

```yaml
ingressController:
  enabled: true
  ingressClass: kong

env:
  database: "postgres"
  pg_user: "kong"
  pg_database: "kong"
  pg_host: "my-kong-postgresql"

admin:
  enabled: true
  type: NodePort
  http:
    enabled: true
    nodePort: 30001

proxy:
  enabled: true
  type: NodePort
  http:
    enabled: true
    nodePort: 30080

postgresql:
  enabled: true
  auth:
    username: kong
    database: kong

podSecurityContext:
  runAsUser: null
  fsGroup: null

containerSecurityContext:
  runAsUser: null
```

---

## ③ Kong のインストール（Helm）

```bash
helm repo add kong https://charts.konghq.com
helm repo update

helm install my-kong kong/kong -n kong -f kong-values.yaml
```

---

## ④ echo サーバのデプロイ（テスト用）

`echo-app.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo
          args:
            - "-listen=:8080"
            - "-text=hello from echo server"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: kong
spec:
  selector:
    app: echo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-ingress
  namespace: kong
spec:
  ingressClassName: kong
  rules:
    - host: echo-kong.apps-crc.testing
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo
                port:
                  number: 80
```

適用：

```bash
oc apply -f echo-app.yaml
```

---

## ⑤ `/etc/hosts` に名前解決を追加

```text
127.0.0.1 echo-kong.apps-crc.testing
127.0.0.1 my-kong-kong-admin.apps-crc.testing
```

---

## ⑥ 動作確認

```bash
curl http://echo-kong.apps-crc.testing:30080/
# → hello from echo server
```

---

## ⑦ Konga をデプロイ

### 1. マニフェスト作成：`konga.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: konga
  namespace: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: konga
  template:
    metadata:
      labels:
        app: konga
    spec:
      containers:
        - name: konga
          image: pantsel/konga:latest
          ports:
            - containerPort: 1337
          env:
            - name: NODE_ENV
              value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: konga
  namespace: kong
spec:
  selector:
    app: konga
  ports:
    - protocol: TCP
      port: 80
      targetPort: 1337
```

適用：

```bash
oc apply -f konga.yaml
```

---

### 2. Route 作成

```bash
oc expose service konga -n kong
```

確認：

```bash
oc get route konga -n kong
# → http://konga-kong.apps-crc.testing など
```

---

## ⑧ Konga GUI 初期セットアップ

1. ブラウザでアクセス → `http://konga-kong.apps-crc.testing`
2. サインアップして管理ユーザーを作成
3. 「Connections」メニューで新しい接続を追加

### ✅ 接続設定例

| 項目             | 値                                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------------------- |
| Name           | kong-local                                                                                                            |
| Kong Admin URL | `http://my-kong-kong-admin.kong.svc.cluster.local:8001`（推奨）<br>`http://my-kong-kong-admin.apps-crc.testing:30001`（外部） |
| Kong Version   | auto-detect で OK                                                                                                      |

---

## 🧪 管理 API に直接アクセスする例

```bash
curl http://my-kong-kong-admin.apps-crc.testing:30001/status
```

---

## 🧼 クリーンアップ・再インストール時

```bash
helm uninstall my-kong -n kong
oc delete pvc data-my-kong-postgresql-0 -n kong
helm install my-kong kong/kong -n kong -f kong-values.yaml
```

---

## 🔚 まとめ

| 機能             | 内容                                  |
| -------------- | ----------------------------------- |
| Kong Gateway   | Helm + NodePort でデプロイ               |
| echo アプリ       | Ingress 経由で公開                       |
| Konga          | OpenShift にデプロイし GUI で管理可能          |
| Kong Admin API | 内部（ClusterIP）または外部（NodePort）経由で利用可能 |

---


