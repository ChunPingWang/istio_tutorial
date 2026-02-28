# Istio Service Mesh 實戰學習課程

> **目標讀者**：具備 Kubernetes 基礎的 Application Architect / DevOps Engineer  
> **課程時數**：約 40 小時（含 Lab 操作）  
> **驗證方式**：每個 Lab 附帶 Shell Script，可透過 Claude Code 自動執行驗證  
> **環境需求**：Minikube / Kind / K3d 擇一，Istio 1.22+，kubectl，istioctl

---

## 課程總覽

| Module | 主題 | 時數 | 難度 |
|--------|------|------|------|
| 0 | 環境建置與驗證 | 2h | ★☆☆☆☆ |
| 1 | Istio 核心架構 | 3h | ★★☆☆☆ |
| 2 | Traffic Management | 5h | ★★★☆☆ |
| 3 | Security — mTLS & AuthZ | 5h | ★★★☆☆ |
| 4 | Observability 三本柱 | 4h | ★★★☆☆ |
| 5 | Gateway API & Ingress | 4h | ★★★☆☆ |
| 6 | Resilience Patterns | 4h | ★★★★☆ |
| 7 | Multi-Cluster & Multi-Tenant | 5h | ★★★★☆ |
| 8 | 效能調校與 Troubleshooting | 4h | ★★★★☆ |
| 9 | 企業落地案例：金融微服務場景 | 4h | ★★★★★ |

---

## Module 0：環境建置與驗證

### 學習目標

- 建立本地 Kubernetes + Istio 開發環境
- 理解 Istio 安裝 Profile 差異
- 驗證 Istio Control Plane 健康狀態

### 0.1 概念說明

Istio 提供多種安裝 Profile（`default`, `demo`, `minimal`, `ambient`），每種 Profile 啟用不同的元件組合。學習環境建議使用 `demo` Profile，它包含完整的 Istiod、Ingress Gateway 和 Egress Gateway。

Istio 的核心部署模型：

```
                    ┌─────────────────────────────────┐
                    │         Control Plane            │
                    │  ┌───────────────────────────┐   │
                    │  │         istiod             │   │
                    │  │  ┌─────┐ ┌─────┐ ┌─────┐  │   │
                    │  │  │Pilot│ │Citad│ │Galle│  │   │
                    │  │  │     │ │ el  │ │  y  │  │   │
                    │  │  └─────┘ └─────┘ └─────┘  │   │
                    │  └───────────────────────────┘   │
                    └────────────────┬────────────────┘
                                     │ xDS API
                    ┌────────────────▼────────────────┐
                    │          Data Plane              │
                    │  ┌─────────┐    ┌─────────┐     │
                    │  │ Pod A   │    │ Pod B   │     │
                    │  │ ┌─────┐ │    │ ┌─────┐ │     │
                    │  │ │App  │ │    │ │App  │ │     │
                    │  │ └──┬──┘ │    │ └──┬──┘ │     │
                    │  │ ┌──▼──┐ │    │ ┌──▼──┐ │     │
                    │  │ │Envoy│◄├────┤►│Envoy│ │     │
                    │  │ │Proxy│ │    │ │Proxy│ │     │
                    │  │ └─────┘ │    │ └─────┘ │     │
                    │  └─────────┘    └─────────┘     │
                    └─────────────────────────────────┘
```

### 0.2 Lab：安裝與驗證

```bash
#!/bin/bash
# lab-00-setup.sh — 環境建置與驗證

set -euo pipefail

echo "=== Step 1: 建立 Kind Cluster ==="
cat <<EOF | kind create cluster --name istio-lab --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
      - containerPort: 30443
        hostPort: 443
  - role: worker
  - role: worker
EOF

echo "=== Step 2: 安裝 Istio (demo profile) ==="
istioctl install --set profile=demo -y

echo "=== Step 3: 啟用 Sidecar 自動注入 ==="
kubectl label namespace default istio-injection=enabled --overwrite

echo "=== Step 4: 部署範例應用 (Bookinfo) ==="
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl wait --for=condition=ready pod -l app=productpage --timeout=120s

echo "=== Step 5: 驗證 ==="
kubectl get pods -n istio-system
istioctl analyze
```

### 0.3 驗證腳本（Claude Code 執行）

```bash
#!/bin/bash
# verify-00.sh — Claude Code 驗證 Module 0

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 0 驗證 =========="

# 1. Cluster 存在
check "Kind cluster 'istio-lab' 存在" \
  "kind get clusters | grep -q istio-lab"

# 2. istiod Running
check "istiod Pod 狀態為 Running" \
  "kubectl get pod -n istio-system -l app=istiod -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 3. Ingress Gateway Running
check "Ingress Gateway 狀態為 Running" \
  "kubectl get pod -n istio-system -l app=istio-ingressgateway -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 4. Sidecar 注入已啟用
check "default namespace 已啟用 sidecar 注入" \
  "kubectl get ns default -o jsonpath='{.metadata.labels.istio-injection}' | grep -q enabled"

# 5. Bookinfo Pod 含 2 個 container (app + sidecar)
check "Bookinfo productpage 含 Envoy sidecar (2 containers)" \
  "[ \$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].spec.containers[*].name}' | wc -w) -eq 2 ]"

# 6. istioctl analyze 無嚴重問題
check "istioctl analyze 無 Error 等級問題" \
  "! istioctl analyze 2>&1 | grep -q '\[Error\]'"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
[ $FAIL -eq 0 ] && echo "🎉 Module 0 全數通過！" || echo "⚠️ 有 $FAIL 項未通過，請檢查環境。"
```

---

## Module 1：Istio 核心架構

### 學習目標

- 深入理解 Istiod 三大功能（Pilot / Citadel / Galley）
- 理解 Envoy Sidecar 的注入機制與生命週期
- 掌握 xDS API（LDS, RDS, CDS, EDS）運作原理
- 了解 Ambient Mesh（Sidecar-less）模式的架構差異

### 1.1 Istiod 統一控制面

Istiod 整合了原先獨立的三個元件：

- **Pilot**：將高階路由規則轉譯為 Envoy xDS 配置，推送給所有 Sidecar
- **Citadel**：管理憑證的簽發與輪替，建立 Workload Identity（SPIFFE）
- **Galley**：驗證與處理 Istio 自訂資源（CRD），確保配置正確性

### 1.2 xDS API 解析

```
┌──────────────┐          ┌──────────────┐
│   istiod     │          │  Envoy Proxy │
│              │──LDS────►│  Listener    │  監聽哪些 Port
│  Service     │──RDS────►│  Route       │  如何匹配 URL Path
│  Registry    │──CDS────►│  Cluster     │  後端服務在哪
│              │──EDS────►│  Endpoint    │  具體 Pod IP:Port
│  Config      │──SDS────►│  Secret      │  mTLS 憑證
│  Store       │          │              │
└──────────────┘          └──────────────┘
```

### 1.3 Sidecar 注入機制

Istio 使用 Kubernetes Mutating Webhook 在 Pod 建立時自動注入 Envoy sidecar：

1. Pod 建立請求送至 API Server
2. Mutating Webhook 攔截，呼叫 istiod 的 injection endpoint
3. istiod 在 Pod spec 中加入 `istio-proxy` container 與 `istio-init` initContainer
4. `istio-init` 透過 iptables 規則攔截所有流量至 Envoy

### 1.4 Lab：觀察 Sidecar 配置

```bash
#!/bin/bash
# lab-01-architecture.sh — 觀察 Istio 內部運作

set -euo pipefail

echo "=== 1. 查看 Envoy Proxy 版本 ==="
PRODUCT_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-status

echo ""
echo "=== 2. 匯出 Envoy 完整配置 ==="
istioctl proxy-config all "$PRODUCT_POD" -o json > /tmp/envoy-config.json
echo "配置已匯出至 /tmp/envoy-config.json ($(wc -c < /tmp/envoy-config.json) bytes)"

echo ""
echo "=== 3. 查看 Listener 配置 ==="
istioctl proxy-config listener "$PRODUCT_POD"

echo ""
echo "=== 4. 查看 Cluster (upstream) 配置 ==="
istioctl proxy-config cluster "$PRODUCT_POD" | head -20

echo ""
echo "=== 5. 查看 Route 配置 ==="
istioctl proxy-config route "$PRODUCT_POD" | head -20

echo ""
echo "=== 6. 查看注入的 iptables 規則 ==="
kubectl exec "$PRODUCT_POD" -c istio-proxy -- \
  pilot-agent request GET /stats | grep -c "cluster\."
echo "  (上面顯示 Envoy 追蹤的 cluster 統計數量)"

echo ""
echo "=== 7. 檢查 SPIFFE Identity ==="
istioctl proxy-config secret "$PRODUCT_POD" | head -5
```

### 1.5 驗證腳本

```bash
#!/bin/bash
# verify-01.sh — Claude Code 驗證 Module 1

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 1 驗證 =========="

PRODUCT_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')

# 1. proxy-status 可正常執行
check "istioctl proxy-status 可列出所有 proxy" \
  "istioctl proxy-status | grep -q SYNCED"

# 2. Listener 包含 inbound 和 outbound
check "Envoy 含 inbound listener (0.0.0.0:15006)" \
  "istioctl proxy-config listener $PRODUCT_POD | grep -q '0.0.0.0.*15006'"

# 3. Cluster 包含 bookinfo 相關服務
check "Envoy cluster 包含 reviews 服務" \
  "istioctl proxy-config cluster $PRODUCT_POD | grep -q reviews"

# 4. Secret 包含 SPIFFE cert
check "Envoy 持有 SPIFFE 憑證" \
  "istioctl proxy-config secret $PRODUCT_POD | grep -q 'ACTIVE'"

# 5. Sidecar container 名稱正確
check "Sidecar container 名稱為 istio-proxy" \
  "kubectl get pod $PRODUCT_POD -o jsonpath='{.spec.containers[*].name}' | grep -q istio-proxy"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 2：Traffic Management

### 學習目標

- 使用 VirtualService + DestinationRule 控制流量
- 實作 Canary / Blue-Green / A/B Testing 部署策略
- Header-based routing、Weighted routing
- Traffic Mirroring（影子流量）

### 2.1 核心資源關係

```
                  Client Request
                       │
                       ▼
              ┌─────────────────┐
              │    Gateway      │  (L4/L7 入口)
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ VirtualService  │  路由規則（match → route）
              │                 │  - URI prefix match
              │                 │  - Header match
              │                 │  - Weight-based split
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │DestinationRule  │  服務的子集與策略
              │                 │  - Subsets (v1, v2)
              │                 │  - Load balancing
              │                 │  - Connection pool
              │                 │  - Outlier detection
              └────────┬────────┘
                       │
                ┌──────┴──────┐
                ▼             ▼
           ┌────────┐   ┌────────┐
           │  v1    │   │  v2    │
           │ Pods   │   │ Pods   │
           └────────┘   └────────┘
```

### 2.2 Lab：Canary Deployment

```bash
#!/bin/bash
# lab-02-traffic.sh — Traffic Management 實戰

set -euo pipefail

echo "=== Step 1: 建立 DestinationRule ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
EOF

echo "=== Step 2: 100% 流量導向 v1 ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 100
EOF

echo "=== Step 3: Canary — 90/10 split (v1:v3) ==="
sleep 5
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 90
        - destination:
            host: reviews
            subset: v3
          weight: 10
EOF

echo "=== Step 4: Header-based routing (測試人員走 v3) ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: tester
      route:
        - destination:
            host: reviews
            subset: v3
    - route:
        - destination:
            host: reviews
            subset: v1
EOF

echo "=== Step 5: Traffic Mirroring ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
      mirror:
        host: reviews
        subset: v3
      mirrorPercentage:
        value: 100.0
EOF

echo "=== 完成！使用以下指令測試 ==="
echo "kubectl exec deploy/productpage-v1 -c istio-proxy -- curl -s http://reviews:9080/reviews/0"
```

### 2.3 驗證腳本

```bash
#!/bin/bash
# verify-02.sh — Claude Code 驗證 Module 2

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 2 驗證 =========="

# 1. DestinationRule 存在
check "DestinationRule 'reviews' 已建立" \
  "kubectl get dr reviews -o name"

# 2. DestinationRule 含 3 個 subset
check "DestinationRule 含 v1/v2/v3 三個 subset" \
  "[ \$(kubectl get dr reviews -o jsonpath='{.spec.subsets[*].name}' | wc -w) -ge 3 ]"

# 3. VirtualService 存在
check "VirtualService 'reviews' 已建立" \
  "kubectl get vs reviews -o name"

# 4. VirtualService 含 mirror 配置
check "VirtualService 含 mirror 配置" \
  "kubectl get vs reviews -o jsonpath='{.spec.http[0].mirror.host}' | grep -q reviews"

# 5. 流量可正常抵達 reviews
check "透過 productpage 可存取 reviews 服務" \
  "kubectl exec deploy/productpage-v1 -c istio-proxy -- curl -sf http://reviews:9080/reviews/0"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 3：Security — mTLS & Authorization

### 學習目標

- 理解 Istio mTLS 模式（STRICT / PERMISSIVE / DISABLE）
- 配置 PeerAuthentication 與 RequestAuthentication
- 使用 AuthorizationPolicy 實現零信任存取控制
- JWT 驗證與 RBAC 整合

### 3.1 安全架構

```
┌─────────────────────────────────────────────────┐
│                 Security Layers                  │
├─────────────────────────────────────────────────┤
│                                                  │
│  Layer 1: Transport Security (mTLS)              │
│  ┌──────────┐  mutual TLS  ┌──────────┐        │
│  │ Envoy A  │◄────────────►│ Envoy B  │        │
│  │ (SPIFFE) │  encrypted   │ (SPIFFE) │        │
│  └──────────┘              └──────────┘        │
│                                                  │
│  Layer 2: Peer Authentication                    │
│  PeerAuthentication → 驗證對方的 mTLS 憑證       │
│                                                  │
│  Layer 3: Request Authentication                 │
│  RequestAuthentication → 驗證 JWT Token          │
│                                                  │
│  Layer 4: Authorization                          │
│  AuthorizationPolicy → ALLOW / DENY / CUSTOM     │
│  基於 source / operation / condition 判斷         │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 3.2 Lab：零信任安全配置

```bash
#!/bin/bash
# lab-03-security.sh — mTLS 與 Authorization 實戰

set -euo pipefail

echo "=== Step 1: 啟用 Mesh-wide STRICT mTLS ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

echo "=== Step 2: 建立 deny-all 預設策略 ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}
EOF

echo "=== Step 3: 允許 productpage → reviews ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-reviews
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/default/sa/bookinfo-productpage"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/reviews/*"]
EOF

echo "=== Step 4: 允許 reviews → ratings ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-ratings
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/default/sa/bookinfo-reviews"
      to:
        - operation:
            methods: ["GET"]
EOF

echo "=== Step 5: JWT 驗證 (RequestAuthentication) ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
    - issuer: "https://accounts.example.com"
      jwksUri: "https://accounts.example.com/.well-known/jwks.json"
      forwardOriginalToken: true
EOF

echo "=== Step 6: 要求外部請求必須帶 JWT ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
    - from:
        - source:
            namespaces: ["default"]
EOF

echo "=== 完成！==="
echo "測試 mTLS: istioctl x describe pod \$(kubectl get pod -l app=productpage -o name)"
```

### 3.3 驗證腳本

```bash
#!/bin/bash
# verify-03.sh — Claude Code 驗證 Module 3

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 3 驗證 =========="

# 1. STRICT mTLS 已啟用
check "Mesh-wide STRICT mTLS 已啟用" \
  "kubectl get peerauthentication default -n istio-system -o jsonpath='{.spec.mtls.mode}' | grep -q STRICT"

# 2. deny-all 策略存在
check "deny-all AuthorizationPolicy 已建立" \
  "kubectl get authorizationpolicy deny-all -o name"

# 3. allow-reviews 策略存在且正確
check "allow-reviews 策略限制 source principal" \
  "kubectl get authorizationpolicy allow-reviews -o jsonpath='{.spec.rules[0].from[0].source.principals[0]}' | grep -q productpage"

# 4. allow-ratings 策略存在且正確
check "allow-ratings 策略限制 source principal" \
  "kubectl get authorizationpolicy allow-ratings -o jsonpath='{.spec.rules[0].from[0].source.principals[0]}' | grep -q reviews"

# 5. RequestAuthentication 存在
check "RequestAuthentication 'jwt-auth' 已建立" \
  "kubectl get requestauthentication jwt-auth -o name"

# 6. mTLS 確認 — 使用 istioctl 檢查
check "productpage 與 reviews 間使用 mTLS" \
  "istioctl x describe pod \$(kubectl get pod -l app=reviews -o jsonpath='{.items[0].metadata.name}') 2>&1 | grep -qi 'mTLS'"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 4：Observability 三本柱

### 學習目標

- 使用 Kiali 視覺化 Service Mesh 拓撲
- 整合 Prometheus + Grafana 監控 Istio Metrics
- 整合 Jaeger / Zipkin 實現 Distributed Tracing
- 理解 Envoy Access Log 與 Telemetry API

### 4.1 觀測架構

```
┌───────────────────────────────────────────────────┐
│                 Envoy Sidecar                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Metrics  │  │  Traces  │  │ Access Logs  │    │
│  │(Promethe│  │(OpenTele │  │ (stdout/     │    │
│  │ us fmt) │  │ metry)   │  │  file/gRPC)  │    │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘    │
└───────┼──────────────┼───────────────┼────────────┘
        │              │               │
        ▼              ▼               ▼
  ┌──────────┐  ┌──────────┐  ┌──────────────┐
  │Prometheus│  │  Jaeger  │  │   Loki /     │
  │          │  │          │  │   EFK Stack  │
  └────┬─────┘  └────┬─────┘  └──────────────┘
       │              │
       ▼              ▼
  ┌──────────┐  ┌──────────┐
  │ Grafana  │  │  Kiali   │
  └──────────┘  └──────────┘
```

### 4.2 Lab：部署觀測工具棧

```bash
#!/bin/bash
# lab-04-observability.sh — Observability 實戰

set -euo pipefail

echo "=== Step 1: 安裝 Prometheus, Grafana, Jaeger, Kiali ==="
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/jaeger.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/addons/kiali.yaml

echo "等待所有 addon Pod Ready..."
kubectl wait --for=condition=ready pod -l app=prometheus -n istio-system --timeout=120s
kubectl wait --for=condition=ready pod -l app=grafana -n istio-system --timeout=120s
kubectl wait --for=condition=ready pod -l app=kiali -n istio-system --timeout=120s

echo "=== Step 2: 配置 Telemetry API — 自訂 Metrics ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: default
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            response_code:
              operation: UPSERT
              value: "response.code"
            request_host:
              operation: UPSERT
              value: "request.host"
EOF

echo "=== Step 3: 配置 Access Logging ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: access-logging
  namespace: default
spec:
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: "response.code >= 400"
EOF

echo "=== Step 4: 產生流量以觀察 ==="
echo "產生 50 次請求..."
PRODUCT_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
for i in $(seq 1 50); do
  kubectl exec "$PRODUCT_POD" -c istio-proxy -- \
    curl -sf http://productpage:9080/productpage > /dev/null &
done
wait
echo "流量產生完成！"

echo ""
echo "=== 存取觀測工具 ==="
echo "Kiali:      istioctl dashboard kiali"
echo "Grafana:    istioctl dashboard grafana"
echo "Jaeger:     istioctl dashboard jaeger"
echo "Prometheus: istioctl dashboard prometheus"
```

### 4.3 驗證腳本

```bash
#!/bin/bash
# verify-04.sh — Claude Code 驗證 Module 4

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 4 驗證 =========="

# 1. Prometheus Running
check "Prometheus Pod Running" \
  "kubectl get pod -n istio-system -l app=prometheus -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 2. Grafana Running
check "Grafana Pod Running" \
  "kubectl get pod -n istio-system -l app=grafana -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 3. Kiali Running
check "Kiali Pod Running" \
  "kubectl get pod -n istio-system -l app=kiali -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 4. Jaeger Running
check "Jaeger Pod Running" \
  "kubectl get pod -n istio-system -l app.kubernetes.io/name=jaeger -o jsonpath='{.items[0].status.phase}' | grep -q Running"

# 5. Telemetry 資源存在
check "Telemetry 'custom-metrics' 已建立" \
  "kubectl get telemetry custom-metrics -o name"

# 6. Access Logging Telemetry 存在
check "Telemetry 'access-logging' 已建立" \
  "kubectl get telemetry access-logging -o name"

# 7. Prometheus 可查詢 istio metrics
check "Prometheus 包含 istio_requests_total metric" \
  "kubectl exec -n istio-system deploy/prometheus -c prometheus-server -- \
    wget -qO- 'http://localhost:9090/api/v1/query?query=istio_requests_total' | grep -q result"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 5：Gateway API & Ingress

### 學習目標

- 理解 Istio Gateway vs Kubernetes Gateway API
- 配置 HTTPS/TLS termination
- 設定多域名路由與跨 namespace 路由
- Egress Gateway 控制外部流量

### 5.1 Gateway API 演進

```
  Legacy (Istio CRD)              Modern (K8s Gateway API)
  ──────────────────              ────────────────────────
  Gateway + VirtualService   →    Gateway + HTTPRoute
  (istio-specific)                (platform-agnostic)

  ┌─────────────┐                 ┌────────────────┐
  │ GatewayClass│  ← 由 Istio 提供 (istio controller)
  └──────┬──────┘
         │
  ┌──────▼──────┐
  │   Gateway   │  ← 定義 Listener (port, protocol, TLS)
  └──────┬──────┘
         │
  ┌──────▼──────┐
  │  HTTPRoute  │  ← 定義路由規則 (path, header → service)
  └─────────────┘
```

### 5.2 Lab：Gateway API 配置

```bash
#!/bin/bash
# lab-05-gateway.sh — Gateway API 與 TLS 配置

set -euo pipefail

echo "=== Step 0: 安裝 Kubernetes Gateway API CRDs ==="
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
kubectl wait --for=condition=established crd gateways.gateway.networking.k8s.io --timeout=30s

echo "=== Step 1: 產生自簽 TLS 憑證 ==="
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
  -subj '/O=Istio Lab/CN=bookinfo.example.com' \
  -keyout /tmp/bookinfo.key -out /tmp/bookinfo.crt

kubectl create secret tls bookinfo-tls \
  --cert=/tmp/bookinfo.crt --key=/tmp/bookinfo.key \
  --dry-run=client -o yaml | kubectl apply -f -

echo "=== Step 2: 使用 Kubernetes Gateway API ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: default
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: bookinfo-tls
      allowedRoutes:
        namespaces:
          from: Same
EOF

echo "=== Step 3: 建立 HTTPRoute ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-route
  namespace: default
spec:
  parentRefs:
    - name: bookinfo-gateway
  hostnames:
    - "bookinfo.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /productpage
      backendRefs:
        - name: productpage
          port: 9080
    - matches:
        - path:
            type: PathPrefix
            value: /api/v1/products
      backendRefs:
        - name: productpage
          port: 9080
EOF

echo "=== Step 4: 配置 Egress Gateway (控制外部存取) ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
    - api.example.com
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
EOF

echo "=== 完成！==="
echo "測試: curl -k https://bookinfo.example.com/productpage --resolve bookinfo.example.com:443:\$(kubectl get svc bookinfo-gateway-istio -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

### 5.3 驗證腳本

```bash
#!/bin/bash
# verify-05.sh — Claude Code 驗證 Module 5

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 5 驗證 =========="

# 1. TLS Secret 存在
check "TLS Secret 'bookinfo-tls' 已建立" \
  "kubectl get secret bookinfo-tls -o name"

# 2. Gateway API Gateway 存在
check "Gateway 'bookinfo-gateway' 已建立" \
  "kubectl get gateway.gateway.networking.k8s.io bookinfo-gateway -o name"

# 3. Gateway 含 HTTPS listener
check "Gateway 包含 HTTPS listener" \
  "kubectl get gateway.gateway.networking.k8s.io bookinfo-gateway -o jsonpath='{.spec.listeners[*].protocol}' | grep -q HTTPS"

# 4. HTTPRoute 存在
check "HTTPRoute 'bookinfo-route' 已建立" \
  "kubectl get httproute bookinfo-route -o name"

# 5. HTTPRoute 引用正確的 Gateway
check "HTTPRoute parentRef 指向 bookinfo-gateway" \
  "kubectl get httproute bookinfo-route -o jsonpath='{.spec.parentRefs[0].name}' | grep -q bookinfo-gateway"

# 6. ServiceEntry 存在
check "ServiceEntry 'external-api' 已建立" \
  "kubectl get serviceentry external-api -o name"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 6：Resilience Patterns

### 學習目標

- 配置 Circuit Breaker（熔斷器）
- 實作 Retry / Timeout 策略
- 使用 Fault Injection 進行 Chaos Testing
- Rate Limiting 與 Connection Pooling

### 6.1 韌性模式總覽

```
  Request Flow with Resilience
  ─────────────────────────────
  
  Client → [Timeout: 3s]
           → [Retry: 2 attempts, 5xx only]
              → [Circuit Breaker]
                 │
                 ├─ CLOSED (正常) → Backend
                 │
                 ├─ OPEN (熔斷中) → 503 fast-fail
                 │
                 └─ HALF-OPEN (探測中) → 少量請求測試

  Fault Injection (測試用)
  ─────────────────────────
  Client → [Inject delay: 5s, 10%]
         → [Inject abort: 503, 5%]
            → Backend
```

### 6.2 Lab：韌性配置

```bash
#!/bin/bash
# lab-06-resilience.sh — Resilience Patterns 實戰

set -euo pipefail

echo "=== Step 1: Timeout + Retry ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - timeout: 3s
      retries:
        attempts: 2
        perTryTimeout: 2s
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
      route:
        - destination:
            host: reviews
            subset: v1
EOF

echo "=== Step 2: Circuit Breaker ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 10
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
EOF

echo "=== Step 3: Fault Injection — Delay ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - fault:
        delay:
          percentage:
            value: 20.0
          fixedDelay: 5s
        abort:
          percentage:
            value: 5.0
          httpStatus: 503
      route:
        - destination:
            host: ratings
            subset: v1
EOF

# 確保 ratings DestinationRule 存在
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
    - name: v1
      labels:
        version: v1
EOF

echo "=== 完成！==="
echo "壓測工具測試: kubectl exec deploy/productpage-v1 -c istio-proxy -- sh -c 'for i in \$(seq 1 20); do curl -s -o /dev/null -w \"%{http_code}\n\" http://reviews:9080/reviews/0; done'"
```

### 6.3 驗證腳本

```bash
#!/bin/bash
# verify-06.sh — Claude Code 驗證 Module 6

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 6 驗證 =========="

# 1. VirtualService reviews 含 timeout
check "reviews VirtualService 設有 timeout" \
  "kubectl get vs reviews -o jsonpath='{.spec.http[0].timeout}' | grep -q 3s"

# 2. VirtualService reviews 含 retry
check "reviews VirtualService 設有 retries" \
  "kubectl get vs reviews -o jsonpath='{.spec.http[0].retries.attempts}' | grep -q 2"

# 3. DestinationRule 含 outlierDetection (Circuit Breaker)
check "reviews DestinationRule 含 outlierDetection" \
  "kubectl get dr reviews -o jsonpath='{.spec.trafficPolicy.outlierDetection.consecutive5xxErrors}' | grep -q 5"

# 4. DestinationRule 含 connectionPool
check "reviews DestinationRule 含 connectionPool 限制" \
  "kubectl get dr reviews -o jsonpath='{.spec.trafficPolicy.connectionPool.tcp.maxConnections}' | grep -q 100"

# 5. Fault Injection — delay
check "ratings VirtualService 含 fault delay" \
  "kubectl get vs ratings -o jsonpath='{.spec.http[0].fault.delay.fixedDelay}' | grep -q 5s"

# 6. Fault Injection — abort
check "ratings VirtualService 含 fault abort (503)" \
  "kubectl get vs ratings -o jsonpath='{.spec.http[0].fault.abort.httpStatus}' | grep -q 503"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 7：Multi-Cluster & Multi-Tenant

### 學習目標

- 理解 Istio 多叢集部署模型（Primary-Remote, Multi-Primary）
- 配置 Namespace 隔離與多租戶策略
- 跨叢集服務發現與流量路由
- Sidecar 資源限縮（減少不必要的 xDS 推送）

### 7.1 多叢集拓撲

```
  Model 1: Primary-Remote          Model 2: Multi-Primary
  ──────────────────────           ────────────────────────
  
  ┌───────────────┐                ┌───────────────┐
  │  Cluster A    │                │  Cluster A    │
  │  (Primary)    │                │  ┌──────────┐ │
  │  ┌──────────┐ │                │  │  istiod  │ │
  │  │  istiod  │ │                │  └──────────┘ │
  │  └────┬─────┘ │                │  ┌──────────┐ │
  │  ┌────▼─────┐ │                │  │  Envoy   │ │
  │  │  Envoy   │ │                │  └──────────┘ │
  │  └──────────┘ │                └───────┬───────┘
  └───────┬───────┘                        │
          │ xDS                    Cross-cluster
          │                        service discovery
  ┌───────▼───────┐                        │
  │  Cluster B    │                ┌───────▼───────┐
  │  (Remote)     │                │  Cluster B    │
  │  (no istiod)  │                │  ┌──────────┐ │
  │  ┌──────────┐ │                │  │  istiod  │ │
  │  │  Envoy   │ │                │  └──────────┘ │
  │  └──────────┘ │                │  ┌──────────┐ │
  └───────────────┘                │  │  Envoy   │ │
                                   │  └──────────┘ │
                                   └───────────────┘
```

### 7.2 Lab：Namespace 隔離與 Sidecar Scope

```bash
#!/bin/bash
# lab-07-multitenant.sh — Multi-Tenant 隔離

set -euo pipefail

echo "=== Step 1: 建立多租戶 Namespace ==="
for ns in tenant-a tenant-b; do
  kubectl create ns $ns --dry-run=client -o yaml | kubectl apply -f -
  kubectl label ns $ns istio-injection=enabled --overwrite
done

echo "=== Step 2: 限縮 Sidecar Scope (減少 xDS 推送) ==="
for ns in tenant-a tenant-b; do
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: $ns
spec:
  egress:
    - hosts:
        - "./*"
        - "istio-system/*"
EOF
done

echo "=== Step 3: 租戶間流量隔離 ==="
for ns in tenant-a tenant-b; do
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ns-isolation
  namespace: $ns
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces:
              - "$ns"
    - from:
        - source:
            namespaces:
              - "istio-system"
EOF
done

echo "=== Step 4: 部署測試工作負載 ==="
for ns in tenant-a tenant-b; do
  kubectl apply -n $ns -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: docker.io/kong/httpbin:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  ports:
    - port: 8000
      targetPort: 80
  selector:
    app: httpbin
EOF
done

echo "=== 完成！==="
echo "跨租戶測試 (應被拒絕):"
echo "  kubectl exec -n tenant-a deploy/httpbin -- curl -sf http://httpbin.tenant-b:8000/get"
```

### 7.3 驗證腳本

```bash
#!/bin/bash
# verify-07.sh — Claude Code 驗證 Module 7

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 7 驗證 =========="

# 1. Namespace 存在且啟用 sidecar 注入
for ns in tenant-a tenant-b; do
  check "Namespace '$ns' 存在且啟用 sidecar 注入" \
    "kubectl get ns $ns -o jsonpath='{.metadata.labels.istio-injection}' | grep -q enabled"
done

# 2. Sidecar scope 已配置
for ns in tenant-a tenant-b; do
  check "Sidecar scope 已限縮於 '$ns'" \
    "kubectl get sidecar default -n $ns -o name"
done

# 3. AuthorizationPolicy 隔離
for ns in tenant-a tenant-b; do
  check "AuthorizationPolicy 'ns-isolation' 存在於 '$ns'" \
    "kubectl get authorizationpolicy ns-isolation -n $ns -o name"
done

# 4. 工作負載已部署
for ns in tenant-a tenant-b; do
  check "httpbin 已部署於 '$ns'" \
    "kubectl get deploy httpbin -n $ns -o name"
done

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 8：效能調校與 Troubleshooting

### 學習目標

- 調校 Envoy resource limits 與 concurrency
- istiod 效能調校（xDS 推送頻率、debounce）
- 常見問題排查方法論
- 使用 istioctl 診斷工具

### 8.1 調校要點

```
Performance Tuning Areas
────────────────────────

1. Sidecar Resources
   └─ CPU/Memory requests & limits
   └─ Concurrency (worker threads)
   └─ Stats 範圍控制

2. Control Plane
   └─ PILOT_DEBOUNCE_AFTER / PILOT_DEBOUNCE_MAX
   └─ PILOT_PUSH_THROTTLE
   └─ Resource limits for istiod

3. Data Plane
   └─ Sidecar CRD 限制 egress scope
   └─ 減少 Service/Endpoint 數量可見性
   └─ Access log sampling

4. Protocol Detection
   └─ 明確定義 port naming (http-*, grpc-*, tcp-*)
   └─ 避免 Protocol Sniffing overhead
```

### 8.2 Lab：效能調校與診斷

```bash
#!/bin/bash
# lab-08-tuning.sh — 效能調校與 Troubleshooting

set -euo pipefail

echo "=== Step 1: 調整 Sidecar Resource Limits ==="
echo "(使用 istioctl install 更新 mesh 配置，而非直接 apply IstioOperator CR)"
istioctl install --set profile=demo \
  --set meshConfig.defaultConfig.concurrency=2 \
  --set 'meshConfig.defaultConfig.proxyStatsMatcher.inclusionRegexps[0]=.*circuit_breakers.*' \
  --set 'meshConfig.defaultConfig.proxyStatsMatcher.inclusionRegexps[1]=.*upstream_rq.*' \
  --set values.global.proxy.resources.requests.cpu=50m \
  --set values.global.proxy.resources.requests.memory=64Mi \
  --set values.global.proxy.resources.limits.cpu=500m \
  --set values.global.proxy.resources.limits.memory=256Mi \
  -y

echo "=== Step 2: Troubleshooting Toolkit ==="
echo ""
echo "--- 2a. Proxy Status ---"
istioctl proxy-status

echo ""
echo "--- 2b. Config Validation ---"
istioctl analyze --all-namespaces

echo ""
echo "--- 2c. 檢查特定 Pod 的配置同步狀態 ---"
PRODUCT_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "")
if [ -n "$PRODUCT_POD" ]; then
  echo "Pod: $PRODUCT_POD"
  istioctl x describe pod "$PRODUCT_POD" 2>/dev/null || echo "(describe 不可用)"
fi

echo ""
echo "--- 2d. Envoy Stats (重要指標) ---"
if [ -n "$PRODUCT_POD" ]; then
  echo "Circuit Breaker Stats:"
  kubectl exec "$PRODUCT_POD" -c istio-proxy -- \
    pilot-agent request GET /stats 2>/dev/null | grep -c "circuit_breakers" || echo "0 entries"
  
  echo ""
  echo "Upstream Connection Stats:"
  kubectl exec "$PRODUCT_POD" -c istio-proxy -- \
    pilot-agent request GET /stats 2>/dev/null | grep "upstream_cx_active" | head -5 || echo "N/A"
fi

echo ""
echo "--- 2e. istiod Log Level 調整 (臨時 Debug) ---"
echo "istioctl admin log --level ads:debug,authorization:debug"
echo "(上面是範例指令，不自動執行以避免影響效能)"
```

### 8.3 Troubleshooting 決策樹

```
問題：請求 503 / Connection refused
──────────────────────────────────
│
├─ istioctl proxy-status → 有 STALE？
│   └─ Yes → istiod 推送問題 → 檢查 istiod logs & resources
│
├─ istioctl analyze → 有 Warning/Error？
│   └─ Yes → 配置問題 → 按提示修正 CRD
│
├─ kubectl logs <pod> -c istio-proxy → 403 / RBAC?
│   └─ Yes → AuthorizationPolicy 問題
│          → istioctl x authz check <pod>
│
├─ istioctl proxy-config cluster <pod> → 目標 cluster 存在？
│   └─ No → DestinationRule / ServiceEntry 缺失
│
├─ istioctl proxy-config endpoint <pod> → endpoint healthy?
│   └─ UNHEALTHY → Outlier detection ejected
│                → 檢查 outlierDetection 配置
│
└─ 確認 Port Naming → http-xxx / grpc-xxx / tcp-xxx
    └─ 未命名 → Protocol detection 錯誤
```

### 8.4 驗證腳本

```bash
#!/bin/bash
# verify-08.sh — Claude Code 驗證 Module 8

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 8 驗證 =========="

# 1. istioctl proxy-status 可執行
check "istioctl proxy-status 正常回應" \
  "istioctl proxy-status"

# 2. istioctl analyze 可執行（允許 Warning/Info，僅在 [Error] 時失敗）
check "istioctl analyze 無 Error 等級問題 (default namespace)" \
  "test -z \"\$(istioctl analyze 2>&1 | grep '\[Error\]')\""

# 3. 所有 proxy 為 SYNCED
check "所有 Proxy 配置已同步 (無 STALE)" \
  "test -z \"\$(istioctl proxy-status | grep STALE)\""

# 4. Envoy admin endpoint 可訪問
PRODUCT_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "")
if [ -n "$PRODUCT_POD" ]; then
  check "Envoy admin stats endpoint 可訪問" \
    "kubectl exec $PRODUCT_POD -c istio-proxy -- pilot-agent request GET /stats 2>/dev/null | head -1"
fi

# 5. 無 CRD 配置錯誤
check "istioctl analyze 無 Error 等級問題 (all namespaces)" \
  "test -z \"\$(istioctl analyze --all-namespaces 2>&1 | grep '\[Error\]')\""

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
```

---

## Module 9：企業落地案例 — 金融微服務場景

### 學習目標

- 設計符合金融監管要求的 Service Mesh 架構
- 實作審計日誌、加密通訊、存取控制的完整方案
- 整合 WAF、Rate Limiting、API Gateway 層
- 規劃 Production-ready 的 Istio 部署策略

### 9.1 金融場景架構

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │   WAF / CDN │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ API Gateway │  (Rate Limit, API Key)
                    │ (Kong/APISIX)│
                    └──────┬──────┘
                           │
               ════════════╧════════════
               ║    Istio Mesh Boundary  ║
               ╠═════════════════════════╣
               ║                         ║
               ║  ┌─────────────────┐    ║
               ║  │ Istio Ingress   │    ║
               ║  │ Gateway         │    ║
               ║  └────────┬────────┘    ║
               ║           │             ║
               ║  ┌────────▼────────┐    ║
               ║  │  BFF / GraphQL  │    ║  ← ns: frontend
               ║  └────────┬────────┘    ║
               ║           │ mTLS        ║
               ║  ┌────────▼────────┐    ║
               ║  │ Account Service │    ║  ← ns: core-banking
               ║  │ Payment Service │    ║
               ║  │ Loan Service    │    ║
               ║  └────────┬────────┘    ║
               ║           │ mTLS        ║
               ║  ┌────────▼────────┐    ║
               ║  │ Audit Logger    │    ║  ← ns: compliance
               ║  │ Fraud Detection │    ║
               ║  │ KYC Service     │    ║
               ║  └────────┬────────┘    ║
               ║           │             ║
               ║  ┌────────▼────────┐    ║
               ║  │ Egress Gateway  │    ║  ← 控制外部 API 存取
               ║  └────────┬────────┘    ║
               ╚═══════════╧═════════════╝
                           │
                    ┌──────▼──────┐
                    │ External:    │
                    │ SWIFT / FISC │
                    │ Credit Bureau│
                    └─────────────┘
```

### 9.2 Lab：金融合規配置

```bash
#!/bin/bash
# lab-09-financial.sh — 金融微服務 Istio 配置

set -euo pipefail

echo "=== Step 1: 建立金融業務 Namespace ==="
for ns in frontend core-banking compliance; do
  kubectl create ns $ns --dry-run=client -o yaml | kubectl apply -f -
  kubectl label ns $ns istio-injection=enabled --overwrite
done

echo "=== Step 2: Mesh-wide STRICT mTLS ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: mesh-strict-mtls
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

echo "=== Step 3: Namespace 隔離策略 ==="
# core-banking 只接受 frontend 和 compliance 的流量
cat <<'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: core-banking-access
  namespace: core-banking
spec:
  action: ALLOW
  rules:
    - from:
        - source:
            namespaces: ["frontend"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    - from:
        - source:
            namespaces: ["compliance"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/*/audit", "/api/*/status"]
    - from:
        - source:
            namespaces: ["istio-system"]
EOF

echo "=== Step 4: 審計日誌 — 全量 Access Log ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: audit-logging
  namespace: core-banking
spec:
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: "true"
EOF

echo "=== Step 5: Rate Limiting (EnvoyFilter) ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit
  namespace: frontend
spec:
  workloadSelector:
    labels:
      app: bff
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 100
                tokens_per_fill: 100
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
EOF

echo "=== Step 6: Egress 控制 — 只允許白名單外部 API ==="
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: core-banking
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "./core-banking/*"
        - "./compliance/*"
        - "istio-system/*"
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: swift-api
  namespace: core-banking
spec:
  hosts:
    - api.swift.com
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
EOF

echo "=== 完成！==="
echo ""
echo "金融合規檢查清單："
echo "  ✓ STRICT mTLS (加密通訊)"
echo "  ✓ Namespace 隔離 (最小權限)"
echo "  ✓ 全量審計日誌 (合規要求)"
echo "  ✓ Rate Limiting (防護)"
echo "  ✓ Egress 白名單 (資料外洩防護)"
```

### 9.3 驗證腳本

```bash
#!/bin/bash
# verify-09.sh — Claude Code 驗證 Module 9

PASS=0; FAIL=0

check() {
  local desc="$1"; shift
  if eval "$@" > /dev/null 2>&1; then
    echo "✅ PASS: $desc"; ((PASS++))
  else
    echo "❌ FAIL: $desc"; ((FAIL++))
  fi
}

echo "========== Module 9 驗證（金融合規） =========="

# 1. Namespace 建立
for ns in frontend core-banking compliance; do
  check "Namespace '$ns' 存在且啟用 sidecar 注入" \
    "kubectl get ns $ns -o jsonpath='{.metadata.labels.istio-injection}' | grep -q enabled"
done

# 2. STRICT mTLS
check "Mesh-wide STRICT mTLS 已啟用" \
  "kubectl get peerauthentication mesh-strict-mtls -n istio-system -o jsonpath='{.spec.mtls.mode}' | grep -q STRICT"

# 3. core-banking 隔離策略
check "core-banking AuthorizationPolicy 存在" \
  "kubectl get authorizationpolicy core-banking-access -n core-banking -o name"

check "core-banking 策略限制來源為 frontend" \
  "kubectl get authorizationpolicy core-banking-access -n core-banking -o jsonpath='{.spec.rules[0].from[0].source.namespaces[0]}' | grep -q frontend"

# 4. 審計日誌
check "core-banking 審計日誌 Telemetry 已配置" \
  "kubectl get telemetry audit-logging -n core-banking -o name"

# 5. Rate Limiting EnvoyFilter
check "Rate Limiting EnvoyFilter 已配置於 frontend" \
  "kubectl get envoyfilter rate-limit -n frontend -o name"

# 6. Egress 控制 — REGISTRY_ONLY
check "core-banking Sidecar outbound 為 REGISTRY_ONLY" \
  "kubectl get sidecar default -n core-banking -o jsonpath='{.spec.outboundTrafficPolicy.mode}' | grep -q REGISTRY_ONLY"

# 7. SWIFT ServiceEntry
check "SWIFT API ServiceEntry 已建立" \
  "kubectl get serviceentry swift-api -n core-banking -o name"

echo ""
echo "========== 結果：$PASS passed / $FAIL failed =========="
[ $FAIL -eq 0 ] && echo "🎉 金融合規配置全數通過！" || echo "⚠️ 有 $FAIL 項未通過。"
```

---

## 附錄 A：Claude Code 一鍵驗證腳本

以下腳本讓 Claude Code 一次跑完所有 Module 驗證：

```bash
#!/bin/bash
# verify-all.sh — Claude Code 全課程驗證

echo "╔══════════════════════════════════════════════╗"
echo "║   Istio Learning Course — Full Verification  ║"
echo "╚══════════════════════════════════════════════╝"
echo ""

TOTAL_PASS=0; TOTAL_FAIL=0

for module in 00 01 02 03 04 05 06 07 08 09; do
  script="verify-${module}.sh"
  if [ -f "$script" ]; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    bash "$script"
    echo ""
  fi
done

echo "╔══════════════════════════════════════════════╗"
echo "║              全課程驗證完成                    ║"
echo "╚══════════════════════════════════════════════╝"
```

---

## 附錄 B：推薦學習資源

- **Istio 官方文件**：https://istio.io/latest/docs/
- **Envoy Proxy 文件**：https://www.envoyproxy.io/docs
- **Kubernetes Gateway API**：https://gateway-api.sigs.k8s.io/
- **SPIFFE/SPIRE**：https://spiffe.io/
- **Kiali**：https://kiali.io/docs/
- **書籍**：*Istio in Action* (Manning), *Service Mesh Patterns* (O'Reilly)

---

## 附錄 C：常用指令速查表

| 用途 | 指令 |
|------|------|
| 安裝 Istio | `istioctl install --set profile=demo -y` |
| 檢查 Proxy 狀態 | `istioctl proxy-status` |
| 分析配置問題 | `istioctl analyze --all-namespaces` |
| 查看 Listener | `istioctl proxy-config listener <pod>` |
| 查看 Route | `istioctl proxy-config route <pod>` |
| 查看 Cluster | `istioctl proxy-config cluster <pod>` |
| 查看 Endpoint | `istioctl proxy-config endpoint <pod>` |
| 查看 Secret/Cert | `istioctl proxy-config secret <pod>` |
| 描述 Pod 網路 | `istioctl x describe pod <pod>` |
| AuthZ 檢查 | `istioctl x authz check <pod>` |
| Dashboard | `istioctl dashboard kiali/grafana/jaeger` |
| 注入狀態 | `kubectl get ns --show-labels | grep istio` |
| Envoy 日誌等級 | `istioctl admin log --level ads:debug` |
