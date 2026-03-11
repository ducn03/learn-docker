# ☸️ Kubernetes (K8s) — Từ A đến Z

> Tài liệu đầy đủ: khái niệm → cài đặt local → deploy thực tế → Helm → Cloud & VPS

---

## Mục lục

1. [Kubernetes là gì? Tại sao cần?](#1-kubernetes-là-gì)
2. [Kiến trúc K8s](#2-kiến-trúc-k8s)
3. [Cài đặt môi trường](#3-cài-đặt-môi-trường)
4. [Các Object cốt lõi](#4-các-object-cốt-lõi)
5. [kubectl — CLI chính](#5-kubectl--cli-chính)
6. [Deploy Spring Boot App lên K8s](#6-deploy-spring-boot-app)
7. [Service & Networking](#7-service--networking)
8. [ConfigMap & Secret](#8-configmap--secret)
9. [Storage — PV & PVC](#9-storage--pv--pvc)
10. [Scaling & Auto-scaling](#10-scaling--auto-scaling)
11. [Helm — Package Manager cho K8s](#11-helm)
12. [Deploy lên Cloud (GKE / EKS / AKS)](#12-deploy-lên-cloud)
13. [Deploy lên VPS tự cài (kubeadm)](#13-deploy-lên-vps-tự-cài)
14. [CI/CD tích hợp K8s](#14-cicd-tích-hợp-k8s)
15. [Monitoring & Logging](#15-monitoring--logging)
16. [Troubleshooting thường gặp](#16-troubleshooting)

---

## 1. Kubernetes là gì?

### Vấn đề Docker Compose chưa giải quyết được

Docker Compose rất tốt cho môi trường **development** hoặc **single server**. Nhưng khi production cần:

| Vấn đề | Docker Compose | Kubernetes |
|--------|---------------|------------|
| Chạy trên nhiều server | ❌ | ✅ |
| Tự restart khi crash | ⚠️ Cơ bản | ✅ Thông minh |
| Scale tự động theo traffic | ❌ | ✅ |
| Rolling update không downtime | ❌ | ✅ |
| Load balancing tự động | ❌ | ✅ |
| Self-healing (thay thế pod lỗi) | ❌ | ✅ |

### Kubernetes là gì?

Kubernetes (K8s) là hệ thống **orchestration** (điều phối) container — tự động hóa việc deploy, scale, và quản lý các containerized application trên **cụm nhiều máy chủ**.

```
Không có K8s:
Server 1: container A, B, C      ← Quản lý thủ công
Server 2: container D, E         ← SSH vào từng máy
Server 3: container F            ← Dễ lỗi, khó scale

Có K8s:
┌─────────────────── Kubernetes Cluster ──────────────────┐
│                                                          │
│  Bạn chỉ cần nói: "Tôi muốn chạy 3 instance của app A" │
│                                                          │
│  K8s tự lo:                                             │
│  ├── Đặt container đúng server có tài nguyên            │
│  ├── Restart nếu container crash                        │
│  ├── Scale lên/xuống theo CPU/memory                    │
│  └── Update không downtime                              │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Kiến trúc K8s

### Cluster = Control Plane + Worker Nodes

```
┌──────────────────────────────────────────────────────────────┐
│                     K8s CLUSTER                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 CONTROL PLANE (Master)               │    │
│  │                                                      │    │
│  │  ┌──────────────┐  ┌──────────┐  ┌───────────────┐ │    │
│  │  │  API Server  │  │   etcd   │  │   Scheduler   │ │    │
│  │  │ (cổng vào K8s│  │(database │  │(phân công Pod │ │    │
│  │  │  kubectl↕)   │  │  state)  │  │  vào Node)    │ │    │
│  │  └──────────────┘  └──────────┘  └───────────────┘ │    │
│  │                                                      │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │          Controller Manager                   │  │    │
│  │  │  (giám sát, đảm bảo trạng thái mong muốn)    │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│          ┌────────────────┼────────────────┐                 │
│          ▼                ▼                ▼                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Worker Node │  │  Worker Node │  │  Worker Node │      │
│  │      1       │  │      2       │  │      3       │      │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │      │
│  │  │  Pod   │  │  │  │  Pod   │  │  │  │  Pod   │  │      │
│  │  │(App A) │  │  │  │(App A) │  │  │  │(App B) │  │      │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │      │
│  │  kubelet     │  │  kubelet     │  │  kubelet     │      │
│  │  kube-proxy  │  │  kube-proxy  │  │  kube-proxy  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Các thành phần Control Plane

| Thành phần | Vai trò |
|-----------|---------|
| **API Server** | Cổng vào duy nhất của cluster. kubectl, CI/CD đều giao tiếp qua đây |
| **etcd** | Database phân tán lưu toàn bộ state của cluster |
| **Scheduler** | Quyết định Pod nào chạy trên Node nào |
| **Controller Manager** | Đảm bảo số lượng Pod luôn đúng như mong muốn (desired state) |

### Các thành phần Worker Node

| Thành phần | Vai trò |
|-----------|---------|
| **kubelet** | Agent chạy trên mỗi Node, nhận lệnh từ Control Plane, quản lý Pod |
| **kube-proxy** | Quản lý network rules, load balancing giữa các Pod |
| **Container Runtime** | Chạy container thực sự (containerd, CRI-O) |

---

## 3. Cài đặt môi trường

### Option A: Minikube (Local — Khuyên dùng để học)

Minikube tạo một K8s cluster **single-node** trên máy local bằng VM hoặc Docker.

```bash
# ── Cài kubectl trước ──────────────────────────────────────
# macOS
brew install kubectl

# Ubuntu/Debian
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Kiểm tra
kubectl version --client

# ── Cài Minikube ───────────────────────────────────────────
# macOS
brew install minikube

# Ubuntu/Debian
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# ── Khởi động cluster ──────────────────────────────────────
minikube start --driver=docker --cpus=2 --memory=4g

# Kiểm tra
minikube status
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.28.0

# Các lệnh Minikube hữu ích
minikube stop           # Dừng cluster
minikube delete         # Xóa cluster
minikube dashboard      # Mở Web UI
minikube tunnel         # Expose LoadBalancer services ra localhost
eval $(minikube docker-env)  # Dùng Docker daemon của minikube
```

### Option B: kind (Kubernetes in Docker)

kind chạy K8s cluster bằng Docker containers — nhẹ hơn, phù hợp CI/CD.

```bash
# Cài kind
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/

# Tạo cluster đơn giản
kind create cluster --name learn-k8s

# Tạo cluster multi-node
cat > kind-config.yml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --config kind-config.yml --name learn-k8s

# Load image local vào kind (thay vì push lên registry)
kind load docker-image my-app:latest --name learn-k8s

# Xóa cluster
kind delete cluster --name learn-k8s
```

### Option C: k3d (k3s trong Docker — nhẹ nhất)

```bash
# Cài k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Tạo cluster
k3d cluster create learn-k8s --agents 2 --port "80:80@loadbalancer"

# Xóa
k3d cluster delete learn-k8s
```

### So sánh

| | Minikube | kind | k3d |
|---|---|---|---|
| Dễ dùng | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| Nhanh | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| Multi-node | ✅ | ✅ | ✅ |
| Dashboard | ✅ built-in | ❌ | ❌ |
| Phù hợp | Học, dev | CI/CD, test | Nhẹ, production-like |

---

## 4. Các Object cốt lõi

### 4.1 Pod — Đơn vị nhỏ nhất

Pod là **nhóm 1 hoặc nhiều container** chạy cùng nhau, chia sẻ network và storage.

```yaml
# pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: ghcr.io/username/spring-app:latest
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"      # 250 millicores = 0.25 CPU
        limits:
          memory: "512Mi"
          cpu: "500m"
```

> ⚠️ Trong thực tế **không tạo Pod trực tiếp** — dùng Deployment để K8s tự quản lý.

### 4.2 Deployment — Quản lý Pod tự động

Deployment khai báo **trạng thái mong muốn** (desired state): bao nhiêu Pod, image nào, update strategy ra sao.

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  namespace: default
spec:
  replicas: 3               # Muốn 3 Pod chạy
  selector:
    matchLabels:
      app: spring-app       # Quản lý các Pod có label này

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Tối đa thêm 1 Pod khi update
      maxUnavailable: 0     # Không được giảm Pod khi update → 0 downtime

  template:                 # Template để tạo Pod
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: ghcr.io/username/spring-app:latest
          ports:
            - containerPort: 8080

          # Biến môi trường
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password

          # Giới hạn tài nguyên
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Health checks
          livenessProbe:     # Container có còn sống không?
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:    # Container đã sẵn sàng nhận traffic?
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
```

### 4.3 Service — Expose Pod ra ngoài

Pod có IP động (thay đổi khi restart). **Service** tạo một địa chỉ ổn định để truy cập Pod.

```yaml
# service.yml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-svc
spec:
  selector:
    app: spring-app         # Tìm các Pod có label này
  ports:
    - protocol: TCP
      port: 80              # Port của Service
      targetPort: 8080      # Port của Pod
  type: ClusterIP           # Chỉ accessible trong cluster
```

**4 loại Service:**

```
ClusterIP (mặc định)
└── Chỉ truy cập được trong cluster
└── Dùng cho giao tiếp nội bộ (API ↔ DB)

NodePort
└── Expose qua port của Node (30000-32767)
└── Dùng để test, không dùng production

LoadBalancer
└── Tạo cloud load balancer (AWS ELB, GCP LB...)
└── Dùng production trên cloud

ExternalName
└── Map service đến DNS bên ngoài
└── Dùng khi cần gọi service external
```

### 4.4 Ingress — HTTP Routing

Ingress quản lý routing HTTP/HTTPS từ bên ngoài vào các Service.

```yaml
# ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: spring-app-svc
                port:
                  number: 80

    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
  tls:
    - hosts:
        - api.myapp.com
        - myapp.com
      secretName: tls-secret
```

### 4.5 Namespace — Phân chia môi trường

```yaml
# Tạo namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

```bash
# Dùng namespace trong kubectl
kubectl get pods -n production
kubectl apply -f deployment.yml -n production

# Đặt namespace mặc định
kubectl config set-context --current --namespace=production
```

---

## 5. kubectl — CLI chính

### Cấu trúc lệnh

```bash
kubectl [verb] [resource] [name] [flags]

# Ví dụ:
kubectl get pods
kubectl describe pod my-pod
kubectl delete deployment my-deploy
kubectl apply -f manifest.yml
```

### Các lệnh hay dùng nhất

```bash
# ── Xem tài nguyên ─────────────────────────────────────────
kubectl get pods                        # Xem pods
kubectl get pods -o wide                # Thêm thông tin (Node, IP)
kubectl get pods --watch                # Theo dõi realtime
kubectl get all                         # Xem tất cả resource
kubectl get nodes                       # Xem nodes
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get namespaces
kubectl get pvc                         # PersistentVolumeClaim

# ── Chi tiết & Debug ───────────────────────────────────────
kubectl describe pod <pod-name>         # Chi tiết + Events
kubectl describe node <node-name>
kubectl logs <pod-name>                 # Xem logs
kubectl logs <pod-name> -f              # Follow logs
kubectl logs <pod-name> --previous      # Logs của lần chạy trước (sau crash)
kubectl logs <pod-name> -c <container>  # Logs container cụ thể (multi-container pod)

# ── Thực thi lệnh trong Pod ────────────────────────────────
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh
kubectl exec <pod-name> -- env          # Xem biến môi trường

# ── Apply / Delete ─────────────────────────────────────────
kubectl apply -f deployment.yml         # Tạo hoặc cập nhật
kubectl apply -f ./k8s/                 # Apply toàn bộ thư mục
kubectl delete -f deployment.yml
kubectl delete pod <pod-name>
kubectl delete deployment <name>

# ── Scale ──────────────────────────────────────────────────
kubectl scale deployment spring-app --replicas=5

# ── Update image ───────────────────────────────────────────
kubectl set image deployment/spring-app spring-app=ghcr.io/user/app:v2.0

# ── Rollout ────────────────────────────────────────────────
kubectl rollout status deployment/spring-app   # Theo dõi rolling update
kubectl rollout history deployment/spring-app  # Xem lịch sử deploy
kubectl rollout undo deployment/spring-app     # Rollback về version trước
kubectl rollout undo deployment/spring-app --to-revision=2

# ── Port forwarding (test local) ───────────────────────────
kubectl port-forward pod/<pod-name> 8080:8080
kubectl port-forward service/spring-app-svc 8080:80

# ── Copy file ──────────────────────────────────────────────
kubectl cp <pod-name>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod-name>:/app/config.json

# ── Context / Cluster ──────────────────────────────────────
kubectl config get-contexts              # Xem danh sách cluster
kubectl config use-context minikube      # Chuyển sang cluster khác
kubectl config current-context          # Cluster đang dùng
```

### Alias hữu ích (thêm vào ~/.bashrc hoặc ~/.zshrc)

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kdp='kubectl describe pod'
alias klf='kubectl logs -f'
```

---

## 6. Deploy Spring Boot App

### Cấu trúc thư mục

```
learn-docker/
├── src/
├── Dockerfile
├── pom.xml
└── k8s/
    ├── namespace.yml
    ├── deployment.yml
    ├── service.yml
    ├── ingress.yml
    ├── configmap.yml
    └── secret.yml
```

### k8s/namespace.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
```

### k8s/configmap.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  SPRING_PROFILES_ACTIVE: "production"
  SERVER_PORT: "8080"
  LOG_LEVEL: "INFO"
```

### k8s/secret.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: myapp
type: Opaque
data:
  # Giá trị phải encode base64: echo -n "mypassword" | base64
  DB_PASSWORD: bXlwYXNzd29yZA==
  JWT_SECRET: bXlqd3RzZWNyZXQ=
```

### k8s/deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  namespace: myapp
  labels:
    app: spring-app
    version: "1.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      # Đảm bảo các Pod trải đều trên các Node
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: spring-app

      containers:
        - name: spring-app
          image: ghcr.io/username/spring-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080

          # Lấy config từ ConfigMap
          envFrom:
            - configMapRef:
                name: app-config

          # Lấy secret
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DB_PASSWORD
            - name: DB_URL
              value: "jdbc:postgresql://postgres-svc:5432/mydb"

          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

      # Kéo image từ private registry
      imagePullSecrets:
        - name: ghcr-secret
```

### k8s/service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-svc
  namespace: myapp
spec:
  selector:
    app: spring-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### k8s/ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spring-app-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: spring-app-svc
                port:
                  number: 80
```

### Deploy lên K8s

```bash
# Tạo secret để pull image từ GHCR
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_TOKEN \
  --namespace=myapp

# Apply tất cả manifest
kubectl apply -f k8s/

# Kiểm tra
kubectl get all -n myapp

# Xem chi tiết nếu có lỗi
kubectl describe pod -n myapp
kubectl logs -n myapp -l app=spring-app
```

---

## 7. Service & Networking

### Service Discovery

Trong K8s, các service tìm nhau qua **DNS**:

```
Format: <service-name>.<namespace>.svc.cluster.local
Ví dụ: postgres-svc.myapp.svc.cluster.local

# Hoặc ngắn gọn nếu cùng namespace:
postgres-svc
```

```yaml
# Application config dùng tên service thay vì IP
env:
  - name: DB_URL
    value: "jdbc:postgresql://postgres-svc:5432/mydb"
    # K8s DNS tự resolve "postgres-svc" → IP của service
```

### Ingress Controller

Để Ingress hoạt động cần cài **Ingress Controller** (nginx là phổ biến nhất):

```bash
# Cài nginx ingress controller trên Minikube
minikube addons enable ingress

# Cài trên cluster thông thường
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Kiểm tra
kubectl get pods -n ingress-nginx
```

### NetworkPolicy — Giới hạn traffic giữa Pods

```yaml
# Chỉ cho phép spring-app gọi vào postgres
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: spring-app    # Chỉ cho phép từ spring-app
      ports:
        - protocol: TCP
          port: 5432
```

---

## 8. ConfigMap & Secret

### ConfigMap — Cấu hình không nhạy cảm

```bash
# Tạo ConfigMap từ file
kubectl create configmap app-config --from-file=application.properties

# Tạo từ literal
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=SERVER_PORT=8080

# Xem
kubectl get configmap app-config -o yaml
```

### Dùng ConfigMap trong Pod

```yaml
spec:
  containers:
    - name: app
      # Cách 1: Inject tất cả key thành env vars
      envFrom:
        - configMapRef:
            name: app-config

      # Cách 2: Inject từng key cụ thể
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL

      # Cách 3: Mount thành file
      volumeMounts:
        - name: config-volume
          mountPath: /app/config

  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

### Secret — Dữ liệu nhạy cảm

```bash
# Tạo secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret

# Tạo secret TLS
kubectl create secret tls tls-secret \
  --cert=server.crt \
  --key=server.key

# Encode giá trị base64
echo -n "mypassword" | base64   # → bXlwYXNzd29yZA==

# Decode
echo "bXlwYXNzd29yZA==" | base64 --decode
```

> ⚠️ **Secret trong K8s không thực sự "bí mật"** — chỉ được encode base64, không encrypt. Production nên dùng **Sealed Secrets**, **External Secrets**, hoặc **Vault**.

---

## 9. Storage — PV & PVC

### Vấn đề

Pod bị xóa → dữ liệu mất. Cần storage **tách biệt khỏi lifecycle của Pod**.

### Các khái niệm

```
StorageClass          ← Loại storage (fast SSD, standard HDD, ...)
    │
    ▼
PersistentVolume (PV) ← Ổ đĩa thực tế (admin tạo hoặc tự động qua StorageClass)
    │
    ▼
PersistentVolumeClaim ← Yêu cầu storage của ứng dụng (dev tạo)
(PVC)
    │
    ▼
Pod                   ← Mount PVC vào container
```

### PersistentVolumeClaim

```yaml
# pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce      # Chỉ 1 Node được mount (phù hợp DB)
    # ReadOnlyMany       # Nhiều Node đọc
    # ReadWriteMany      # Nhiều Node đọc/ghi (cần NFS/CephFS)
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard   # Dùng StorageClass mặc định
```

### Dùng PVC trong Deployment

```yaml
spec:
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data

      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc    # Tên PVC vừa tạo
```

### StatefulSet — Cho Database

Database cần Pod có **tên ổn định** và **storage riêng cho mỗi replica**. Dùng StatefulSet thay vì Deployment:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_DB
              value: mydb
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

  # Mỗi replica tự động tạo PVC riêng
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## 10. Scaling & Auto-scaling

### Manual Scaling

```bash
kubectl scale deployment spring-app --replicas=5
```

### Horizontal Pod Autoscaler (HPA)

Tự động scale dựa trên CPU/Memory:

```yaml
# hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-app-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up nếu CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Cần cài metrics-server trước
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Minikube
minikube addons enable metrics-server

# Apply HPA
kubectl apply -f hpa.yml

# Xem HPA hoạt động
kubectl get hpa -n myapp -w
```

### Resource Quota — Giới hạn tài nguyên cả namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

---

## 11. Helm

### Helm là gì?

Helm là **package manager cho Kubernetes** — giống `apt` trên Ubuntu hay `npm` cho Node.js.

```
Không có Helm:
├── deployment.yml
├── service.yml
├── ingress.yml
├── configmap.yml
├── secret.yml
└── hpa.yml
→ Phải apply từng file, quản lý version thủ công

Có Helm:
helm install my-app ./my-chart --values values-prod.yml
helm upgrade my-app ./my-chart --set image.tag=v2.0
helm rollback my-app 1
helm uninstall my-app
```

### Cài đặt Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Kiểm tra
helm version
```

### Cấu trúc Helm Chart

```
my-app/                        ← Tên chart
├── Chart.yaml                 ← Metadata (tên, version, description)
├── values.yaml                ← Giá trị mặc định
├── values-staging.yml         ← Override cho staging
├── values-production.yml      ← Override cho production
└── templates/
    ├── _helpers.tpl           ← Helper templates
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa.yaml
    └── NOTES.txt              ← Hướng dẫn sau khi install
```

### Tạo Chart mới

```bash
helm create my-app
cd my-app
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: Spring Boot application
type: application
version: 0.1.0          # Chart version
appVersion: "1.0.0"     # App version
```

### values.yaml

```yaml
# Giá trị mặc định — có thể override khi deploy
replicaCount: 2

image:
  repository: ghcr.io/username/spring-app
  tag: "latest"
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.myapp.com
  tls: true

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  springProfile: production
  logLevel: INFO

secrets:
  dbPassword: ""    # Truyền khi deploy, không commit!
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.config.springProfile }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-app.fullname" . }}-secret
                  key: dbPassword
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### values-production.yml

```yaml
# Override cho production
replicaCount: 3

image:
  tag: "1.2.3"    # Pin version cụ thể cho production

ingress:
  host: api.myapp.com

resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "2000m"

config:
  logLevel: WARN
```

### Các lệnh Helm

```bash
# ── Install / Upgrade ──────────────────────────────────────
# Install lần đầu
helm install my-app ./my-app -n myapp --create-namespace \
  --values ./my-app/values-production.yml \
  --set secrets.dbPassword=supersecret

# Upgrade (update)
helm upgrade my-app ./my-app -n myapp \
  --values ./my-app/values-production.yml \
  --set image.tag=v2.0

# Install hoặc upgrade (idempotent — dùng trong CI/CD)
helm upgrade --install my-app ./my-app -n myapp \
  --values ./my-app/values-production.yml

# ── Quản lý Release ────────────────────────────────────────
helm list -n myapp                    # Xem các release
helm history my-app -n myapp          # Lịch sử deploy
helm rollback my-app 1 -n myapp       # Rollback về revision 1
helm uninstall my-app -n myapp        # Xóa release

# ── Debug ──────────────────────────────────────────────────
helm template my-app ./my-app         # Xem YAML được render (dry-run)
helm lint ./my-app                    # Kiểm tra syntax
helm install my-app ./my-app --dry-run --debug  # Test không apply thật

# ── Dùng chart từ repository ───────────────────────────────
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/postgresql
helm install postgres bitnami/postgresql \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=10Gi \
  -n myapp
```

---

## 12. Deploy lên Cloud

### Google Kubernetes Engine (GKE)

```bash
# Cài Google Cloud SDK
# https://cloud.google.com/sdk/docs/install

# Login
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Tạo GKE cluster
gcloud container clusters create learn-k8s \
  --zone asia-southeast1-a \
  --num-nodes 2 \
  --machine-type e2-standard-2 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 5

# Kết nối kubectl vào cluster
gcloud container clusters get-credentials learn-k8s \
  --zone asia-southeast1-a

# Kiểm tra
kubectl get nodes

# Xóa cluster (tránh bị tính tiền)
gcloud container clusters delete learn-k8s --zone asia-southeast1-a
```

### Amazon EKS

```bash
# Cài eksctl
# https://eksctl.io/installation/

# Tạo cluster
eksctl create cluster \
  --name learn-k8s \
  --region ap-southeast-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4

# Cập nhật kubeconfig
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name learn-k8s

# Xóa cluster
eksctl delete cluster --name learn-k8s --region ap-southeast-1
```

### Azure AKS

```bash
# Login
az login

# Tạo resource group
az group create --name learn-k8s-rg --location southeastasia

# Tạo AKS cluster
az aks create \
  --resource-group learn-k8s-rg \
  --name learn-k8s \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5

# Kết nối kubectl
az aks get-credentials \
  --resource-group learn-k8s-rg \
  --name learn-k8s

# Xóa
az group delete --name learn-k8s-rg --yes
```

### Sau khi tạo cluster trên Cloud

```bash
# Cài nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Cài cert-manager (TLS tự động với Let's Encrypt)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Deploy app
helm upgrade --install my-app ./my-app \
  -n myapp --create-namespace \
  --values values-production.yml
```

---

## 13. Deploy lên VPS tự cài (kubeadm)

Phù hợp khi muốn tự kiểm soát hoàn toàn, không dùng cloud managed service.

### Yêu cầu

- Tối thiểu 2 VPS: 1 master + 1 worker
- Mỗi máy: 2 CPU, 2GB RAM, Ubuntu 22.04
- Các máy có thể ping nhau

### Bước 1: Chuẩn bị tất cả các node

```bash
# Chạy trên TẤT CẢ nodes (cả master lẫn worker)

# Tắt swap (K8s yêu cầu)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Cài containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Cài kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Bước 2: Khởi tạo Master node

```bash
# Chạy trên MASTER node
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>

# Sau khi init xong, copy config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Cài Flannel CNI (network plugin)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Lưu lại lệnh join (sẽ dùng cho worker)
# kubeadm join <MASTER_IP>:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx
```

### Bước 3: Join Worker nodes

```bash
# Chạy trên mỗi WORKER node
# Dùng lệnh kubeadm join từ bước 2
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>

# Kiểm tra trên master
kubectl get nodes
# NAME       STATUS   ROLES           AGE
# master     Ready    control-plane   5m
# worker-1   Ready    <none>          2m
```

### Bước 4: Deploy ứng dụng

```bash
# Cài nginx ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml

# Deploy app
helm upgrade --install my-app ./my-app \
  -n myapp --create-namespace \
  --values values-production.yml

# Kiểm tra ingress nodeport
kubectl get svc -n ingress-nginx
# Truy cập qua http://<WORKER_IP>:<NODEPORT>
```

---

## 14. CI/CD tích hợp K8s

### GitHub Actions → Deploy lên K8s

```yaml
# .github/workflows/deploy-k8s.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      # Setup kubectl với kubeconfig từ secret
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > ~/.kube/config

      # Cài Helm
      - name: Set up Helm
        uses: azure/setup-helm@v3

      # Deploy với Helm
      - name: Deploy with Helm
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace myapp \
            --create-namespace \
            --values ./helm/my-app/values-production.yml \
            --set image.tag=${{ github.sha }} \
            --set secrets.dbPassword=${{ secrets.DB_PASSWORD }} \
            --wait \
            --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/my-app -n myapp
          kubectl get pods -n myapp
```

### Lấy KUBECONFIG để thêm vào GitHub Secrets

```bash
# Cloud (GKE/EKS/AKS) — lấy kubeconfig hiện tại
cat ~/.kube/config | base64 -w 0

# Thêm vào GitHub Secrets với tên: KUBECONFIG
```

---

## 15. Monitoring & Logging

### Prometheus + Grafana (Stack phổ biến nhất)

```bash
# Cài kube-prometheus-stack qua Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# Truy cập Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# http://localhost:3000 (admin/admin123)

# Truy cập Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```

### Expose Spring Boot metrics cho Prometheus

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

```yaml
# ServiceMonitor — để Prometheus tự scrape metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spring-app-metrics
  namespace: myapp
spec:
  selector:
    matchLabels:
      app: spring-app
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

### Logging với Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

---

## 16. Troubleshooting

### Pod không start — Debug flow

```bash
# Bước 1: Xem trạng thái
kubectl get pods -n myapp

# Bước 2: Xem sự kiện
kubectl describe pod <pod-name> -n myapp
# Tìm phần "Events:" ở cuối output

# Bước 3: Xem logs
kubectl logs <pod-name> -n myapp
kubectl logs <pod-name> -n myapp --previous  # Nếu pod đã crash

# Bước 4: Vào trong pod debug
kubectl exec -it <pod-name> -n myapp -- sh
```

### Các trạng thái lỗi thường gặp

#### ❌ CrashLoopBackOff

```
Pod crash liên tục → K8s backoff trước khi restart lại
```

```bash
kubectl logs <pod> --previous   # Xem log lần crash trước
# Thường do: lỗi app, thiếu biến môi trường, DB chưa sẵn sàng
```

#### ❌ ImagePullBackOff / ErrImagePull

```
K8s không pull được image
```

```bash
kubectl describe pod <pod>
# Kiểm tra: tên image đúng không? Secret để pull đã tạo chưa?

# Tạo lại imagePullSecret
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=USERNAME \
  --docker-password=TOKEN \
  -n myapp
```

#### ❌ Pending — Pod không được schedule

```bash
kubectl describe pod <pod>
# Events thường cho thấy:
# "Insufficient memory" → Node hết RAM, tăng replicas hoặc thêm Node
# "No nodes available" → Tất cả Node đều không phù hợp
# "Unschedulable" → Kiểm tra resource requests có quá cao không
```

#### ❌ Service không truy cập được

```bash
# Kiểm tra service selector có đúng label không
kubectl get pods -n myapp --show-labels
kubectl describe service spring-app-svc -n myapp

# Test kết nối từ trong cluster
kubectl run test-pod --image=busybox -it --rm --restart=Never -- wget -qO- http://spring-app-svc
```

#### ❌ PVC Pending

```bash
kubectl describe pvc <pvc-name>
# Thường do: không có StorageClass, hoặc StorageClass không hỗ trợ accessMode
kubectl get storageclass
```

### Lệnh debug hữu ích

```bash
# Xem resource usage của nodes
kubectl top nodes

# Xem resource usage của pods
kubectl top pods -n myapp

# Xem events toàn cluster
kubectl get events -n myapp --sort-by='.lastTimestamp'

# Xem tất cả trong một namespace
kubectl get all -n myapp

# Dump toàn bộ config của một resource
kubectl get deployment spring-app -n myapp -o yaml

# Diff — xem sự khác biệt trước khi apply
kubectl diff -f deployment.yml
```

---

## Tóm tắt lộ trình

```
Tuần 1: Khái niệm + Minikube local
    └── Pod, Deployment, Service, kubectl cơ bản

Tuần 2: Config & Storage
    └── ConfigMap, Secret, PVC, StatefulSet

Tuần 3: Deploy ứng dụng thực tế
    └── Spring Boot + PostgreSQL trên K8s

Tuần 4: Helm
    └── Tạo chart, values theo môi trường

Tuần 5: Cloud / VPS
    └── GKE hoặc kubeadm trên VPS

Tuần 6: Production-ready
    └── CI/CD, HPA, Monitoring, TLS
```

---

## Cấu trúc repo hoàn chỉnh

```
learn-docker/
├── src/
├── Dockerfile
├── pom.xml
├── docker-compose.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy-k8s.yml
├── k8s/                        ← Raw manifests (dev/staging)
│   ├── namespace.yml
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
└── helm/                       ← Helm chart (production)
    └── my-app/
        ├── Chart.yaml
        ├── values.yaml
        ├── values-staging.yml
        ├── values-production.yml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml
```

---

*Tài liệu thuộc series `learn-docker` — Phần 3: Kubernetes*