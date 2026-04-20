---
theme: apple-basic
background: https://cover.sli.dev
title: Minikube
info: Kubernetes From Theory to Localhost
class: text-center
drawings:
  persist: false
transition: view-transition
mdc: true
author: Jerry Huang
routerMode: hash
---

# Kubernetes: From Theory to Localhost

<div class="text-sm opacity-70 mt-6">2026-04-20 · Jerry</div>

---

# Agenda

<Toc text-sm minDepth="1" maxDepth="2" />

1. 前期提要：需求與問題整理
2. 設計提案：Workflow Layer
3. Entity / Table Schema
4. 實作規劃、風險與決策點

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---

# Why Kubernetes?

## 問題：我們真的需要 Kubernetes 嗎？

用：

- Docker
- Docker Compose

---

# Docker Compose 的極限

問題 1：沒有 High Availability
container 掛掉 → 直接死
沒有自動 recovery

問題 2：沒有 Scaling
想要 3 個 instance？
👉 docker-compose up --scale（很弱）

問題 3：沒有 Service Discovery
IP 會變
需要自己管理

問題 4：沒有 Deployment Strategy
Rolling update ❌
Blue/Green ❌

---

# --scale 為什麼這不算 production scaling

--scale 是：

>「我多開幾個 container」

Production scaling 是：

>「我有一整套系統來確保服務在高流量與故障下仍然穩定運作」

| 能力           | `docker compose --scale` | Production（K8s / Swarm） |
| ------------ | ------------------------   | ----------------------- |
| 自動負載平衡       | ❌（要自己加）           | ✅                       |
| 自動重啟         | ❌                       | ✅                       |
| 跨主機          | ❌                       | ✅                       |
| Auto Scaling | ❌                         | ✅                       |
| 滾動更新         | ❌                       | ✅                       |

---

# 雲服務 Production 系統

✔ Self-healing

✔ Load balancing

✔ Service discovery

✔ Rolling update

✔ Scaling

> Docker Compose ❌ 做不到
>
> Kubernetes ✅

---

# Kubernetes 是什麼？

> Kubernetes = Container Orchestrator

它做的事：
- 管理 container lifecycle
- 調度（scheduling）
- 自動修復（self-healing）
- 網路抽象（service）

---

# 核心概念

Kubernetes 不是 container

| 抽象         | 說明     |
| ---------- | ------ |
| Pod        | 最小單位   |
| Deployment | 管理 Pod |
| Service    | 網路入口   |
| Node       | 機器     |

---

## Pod（最小執行單位）

Kubernetes 裡最小可部署單位
一個 Pod 裡可以有 1 個或多個 container
這些 container：
- 共享 IP
- 共享 storage（volume）

👉 可以把 Pod 想成「一個小型應用執行環境」

例子：

> 一個 web server container + 一個 sidecar（log agent）

---

## Deployment（管理 Pod）

用來管理 Pod 的生命週期
會確保：
- 有幾個 Pod（replicas）
- 自動重建掛掉的 Pod
- 滾動更新（rolling update）

👉 Deployment = 「自動幫你顧 Pod 的老闆」

你可以做的事：

- 設定 replicas: 3 → 永遠保持 3 個 Pod
- 更新 image → 自動一個一個換（不中斷服務）

---

## Service（網路入口）

給 Pod 一個穩定的存取方式
因為 Pod：
- IP 會變
- 會被重建

👉 Service = 「固定門牌 + 負載平衡」

常見類型：

- ClusterIP（內部用）
- NodePort（對外開 port）
- LoadBalancer（雲端 LB）

---

## Node（機器）

Kubernetes 裡的「一台機器」
可以是：
- 實體機
- VM（雲端）

👉 Node 上會跑：

- Pod
- kubelet（管理 Pod）
- container runtime（例如 Docker / containerd）

---

## 整體關係

```
Deployment
   ↓ 管理
Pods (多個)
   ↓ 跑在
Node (多台)
   ↑ 被存取
Service
```

---

# 架構圖

![](/k8s-architecture.jpeg){width=400px lazy}

- Pod 在 Node 上
- Deployment 管 Pod
- Service 做 routing

---

# Minikube？

Kubernetes 很重

你不可能一開始就：

- 開 3 個 node
- 搞 cluster

Minikube 是：

> 一個本地單節點 Kubernetes

---

# Minikube ≠ Production

| 項目      | Minikube | Production |
| ------- | -------- | ---------- |
| Node    | 1        | 多個         |
| HA      | ❌        | ✅          |
| Network | 簡化       | 真實         |
| LB      | 假的       | 真實         |

---

# 實戰練習

```sh
brew install minikube
minikube start
```

檢查：

```sh
docker ps
```

👉 你會看到 minikube node container

---

# Dashboard

```sh
minikube dashboard
```

👉 GUI 觀察：

- Pod
- Deployment
- Service

---

# 第一個 Deployment

```sh
touch httpbin-deployment.yaml
vim httpbin-deployment.yaml
```

```yaml
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
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
```

---

# Apply

```sh
kubectl apply -f httpbin-deployment.yaml
kubectl get po -A
```

觀察：

- Pod 被創建
- Running

---

# Service

Pod 沒有固定 IP，需要 Service

```yaml
kind: Service
```

功能：

- Load balancing
- Stable endpoint

---

# 方法 1：minikube service

```sh
minikube service httpbin-service --url
```

它做的事：

幫你建立一個從 localhost → NodePort 的 tunnel
幫你處理 VM / container 網路問題

> localhost:xxxxx → minikube VM:30080 → Service → Pod

---

# 方法 2：kubectl port-forward

```sh
kubectl port-forward deployment/httpbin 8080:80
```

> localhost:8080 → kubectl → API Server → Pod:80

✔ 完全繞過 Service / NodePort

✔ 直接打 Pod

---

# 連線方法比較

| 方法                     | 流量路徑                 | 是否經過 Service | 適合    |
| ---------------------- | -------------------- | ------------ | ----- |
| NodePort + minikube ip | NodeIP:30080 → Pod   | ✅            | 真實模擬  |
| `minikube service`     | localhost → NodePort | ✅            | ⭐ 最簡單 |
| `port-forward`         | localhost → Pod      | ❌            | debug |

---

# Reality Gap

你現在學到的是「假的 Kubernetes」

Production 會遇到：
- 多 Node scheduling
- Network policy
- Ingress Controller
- Persistent Volume
- Service mesh

Minikube 幫你理解：

✔ API

✔ Resource model

✔ 基本 flow

---

# Demo scaling + kill pod

```sh
kubectl scale deployment httpbin --replicas=3
kubectl delete pod xxx
```

觀察：

✔ Pod 被自動補上
✔ Load balancing 生效

---

# 總結

Docker Compose 解決「單機運行」
Kubernetes 解決「分散式系統運行」