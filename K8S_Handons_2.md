Dưới đây là một bài tập thực hành Kubernetes (K8s) **khó hơn**, nâng cấp từ bài trước bằng cách tích hợp thêm các khái niệm phức tạp như **multi-container pod**, **custom scheduler**, **network policies**, **RBAC (Role-Based Access Control)**, và **multi-node cluster**. Bài tập này mô phỏng một hệ thống microservices thực tế với độ phức tạp cao, yêu cầu bạn phải hiểu sâu và xử lý các vấn đề kỹ thuật.

---

### **Bài tập: Triển khai hệ thống thương mại điện tử phân tán với monitoring**

**Mục tiêu**:  
Xây dựng một hệ thống thương mại điện tử gồm nhiều microservices (Frontend, API, Database, Cache, Monitoring), đảm bảo:

- **Phân quyền chặt chẽ** với RBAC.
- **Kiểm soát mạng** với Network Policy.
- **Lập lịch tùy chỉnh** với Custom Scheduler.
- **Lưu trữ phân tán** với StatefulSet và Volumes.
- **Tự động mở rộng** với HPA và Cluster Autoscaler (nếu dùng multi-node).
- **Monitoring** với Prometheus và Grafana.

---

### **Chuẩn bị**

- **Công cụ**:
  - Minikube (`minikube start --nodes 3 --memory 4096 --cpus 2`) hoặc cluster cloud (GKE, EKS).
  - Cài Ingress Controller: `minikube addons enable ingress`.
  - Cài MetalLB (nếu dùng Minikube) để hỗ trợ LoadBalancer.
- **Kiến thức cần**: Hiểu sâu về YAML, networking, và quản lý tài nguyên K8s.

---

### **Các bước thực hành**

#### **Bước 1: Tạo Namespace và RBAC**

Tạo namespace và phân quyền để giới hạn truy cập.

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
---
# rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ecommerce
  name: dev-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: ecommerce
  name: dev-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f namespace.yaml -f rbac.yaml
kubectl config set-context --current --namespace=ecommerce
```

---

#### **Bước 2: Tạo Secret và ConfigMap**

Lưu thông tin nhạy cảm và cấu hình.

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-secrets
  namespace: ecommerce
type: Opaque
data:
  db-password: ZWNvbW1lcmNl # base64 của "ecommerce"
  redis-password: cmVkaXM= # base64 của "redis"
---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-config
  namespace: ecommerce
data:
  db_host: mysql-service
  redis_host: redis-service
  env: production
```

```bash
kubectl apply -f secret.yaml -f configmap.yaml
```

---

#### **Bước 3: Triển khai MySQL và Redis với StatefulSet**

Database (MySQL) và Cache (Redis) cần danh tính cố định và lưu trữ dữ liệu.

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: ecommerce
spec:
  serviceName: mysql-service
  replicas: 2 # Multi-replica cho HA
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: db-password
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: ecommerce
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
# redis-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: ecommerce
spec:
  serviceName: redis-service
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:6.2
          args: ["--requirepass", "$(REDIS_PASSWORD)"]
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: redis-password
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: ecommerce
spec:
  ports:
    - port: 6379
  selector:
    app: redis
  clusterIP: None
```

```bash
kubectl apply -f mysql-statefulset.yaml -f redis-statefulset.yaml
```

---

#### **Bước 4: Triển khai API với Multi-Container Pod và HPA**

API dùng Flask + Redis, chạy trong pod đa container.

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: flask-api
          image: python:3.9
          command:
            [
              "sh",
              "-c",
              "pip install flask redis && python -m flask run --host=0.0.0.0",
            ]
          env:
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: ecommerce-config
                  key: redis_host
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: redis-password
          ports:
            - containerPort: 5000
          resources:
            requests:
              cpu: "200m"
            limits:
              cpu: "500m"
        - name: sidecar-logger
          image: busybox
          command:
            [
              "sh",
              "-c",
              "while true; do echo 'API running' >> /logs/api.log; sleep 5; done",
            ]
          volumeMounts:
            - name: logs
              mountPath: /logs
      volumes:
        - name: logs
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: ecommerce
spec:
  ports:
    - port: 5000
  selector:
    app: api
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 80
```

```bash
kubectl apply -f api-deployment.yaml
```

---

#### **Bước 5: Triển khai Frontend với NodePort và LoadBalancer**

Frontend dùng Nginx, thử cả hai loại Service.

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
  namespace: ecommerce
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30080
  selector:
    app: frontend
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  namespace: ecommerce
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: frontend
```

```bash
kubectl apply -f frontend-deployment.yaml
```

---

#### **Bước 6: Thiết lập Ingress**

Định tuyến traffic thông minh.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
spec:
  rules:
    - host: shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-lb
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 5000
```

```bash
kubectl apply -f ingress.yaml
echo "$(minikube ip) shop.local" | sudo tee -a /etc/hosts
```

---

#### **Bước 7: Tạo Network Policy**

Giới hạn giao tiếp giữa các service.

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-access
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 5000
```

```bash
kubectl apply -f network-policy.yaml
```

---

#### **Bước 8: Triển khai Monitoring với Prometheus và Grafana**

Dùng Helm để cài đặt nhanh.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n ecommerce
```

Truy cập Grafana:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n ecommerce
```

Mở `http://localhost:3000` để xem dashboard.

---

#### **Bước 9: Tạo Custom Scheduler (Tùy chọn nâng cao)**

Tạo scheduler tùy chỉnh để ưu tiên node có CPU cao.

1. Tạo file `custom-scheduler.yaml`:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: custom-scheduler
     namespace: kube-system
   spec:
     containers:
       - name: scheduler
         image: k8s.gcr.io/scheduler-plugins/scheduler:latest
         args:
           - --config=/etc/kubernetes/scheduler-config.yaml
         volumeMounts:
           - name: config
             mountPath: /etc/kubernetes
     volumes:
       - name: config
         configMap:
           name: scheduler-config
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: scheduler-config
     namespace: kube-system
   data:
     scheduler-config.yaml: |
       apiVersion: kubescheduler.config.k8s.io/v1beta3
       kind: KubeSchedulerConfiguration
       profiles:
       - schedulerName: custom-scheduler
         plugins:
           score:
             disabled:
             - name: NodeResourcesFit
             enabled:
             - name: NodeResourcesBalancedAllocation
   ```
2. Áp dụng và cấu hình pod dùng scheduler này trong `spec.schedulerName: custom-scheduler`.

---

#### **Bước 10: Kiểm tra và thử tải**

1. Truy cập `http://shop.local` (Frontend) và `http://shop.local/api` (API).
2. Tạo tải:
   ```bash
   kubectl run -i --tty load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://shop.local/api; done"
   ```
3. Theo dõi HPA và monitoring:
   ```bash
   kubectl get hpa -n ecommerce
   kubectl port-forward svc/monitoring-prometheus-oper-prometheus 9090:9090 -n ecommerce
   ```

---

#### **Bước 11: Dọn dẹp**

```bash
kubectl delete namespace ecommerce
helm uninstall monitoring -n ecommerce
```

---

### **Điểm khó**

- **Multi-container Pod**: API có sidecar để log, tăng độ phức tạp quản lý.
- **RBAC**: Phân quyền chặt chẽ, yêu cầu hiểu về security.
- **Network Policy**: Giới hạn traffic, đòi hỏi hiểu networking.
- **Custom Scheduler**: Can thiệp vào lập lịch K8s, mức độ nâng cao.
- **Multi-replica StatefulSet**: Quản lý MySQL cluster phức tạp hơn single instance.
- **Monitoring**: Tích hợp Prometheus/Grafana để theo dõi hiệu suất.

---

### **Kết quả mong đợi**

- Hệ thống chạy ổn định, Frontend giao tiếp với API, API dùng Redis/MySQL.
- HPA tự mở rộng khi tải tăng.
- Monitoring cung cấp thông tin CPU, memory, và traffic.

Nếu bạn cần giải thích sâu hơn hoặc hỗ trợ triển khai, hãy báo tôi nhé! Đây là bài tập rất khó, đòi hỏi bạn phải thành thạo K8s. Chúc bạn thành công!
