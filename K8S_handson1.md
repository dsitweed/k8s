Dưới đây là một bài tập thực hành Kubernetes (K8s) nâng cao, tích hợp đầy đủ các nội dung từ các bài tập trước (Pod, Deployment, Service, ConfigMap, Secret, HPA, Volumes, StatefulSet, Ingress, v.v.) và thêm độ khó để bạn thử thách kỹ năng. Tôi sẽ mô tả chi tiết bằng tiếng Việt, bạn có thể thực hành trên Minikube hoặc cluster cloud.

---

### **Bài tập: Triển khai hệ thống Blog đa tầng với WordPress và MySQL**

**Mục tiêu**:
Xây dựng một hệ thống blog hoàn chỉnh gồm WordPress (ứng dụng web) và MySQL (database), sử dụng tất cả các thành phần K8s đã học để đảm bảo:
- Tự động mở rộng (HPA).
- Lưu trữ dữ liệu lâu dài (Volumes/StatefulSet).
- Định tuyến truy cập thông minh (Ingress).
- Tách cấu hình và thông tin nhạy cảm (ConfigMap/Secret).
- Tổ chức tài nguyên (Namespace).

---

### **Chuẩn bị**
- **Công cụ**: Minikube (`minikube start --memory 4096 --cpus 2`), `kubectl`, và Ingress Controller (cài Nginx Ingress: `minikube addons enable ingress`).
- **Thời gian ước tính**: 1-2 giờ.

---

### **Các bước thực hành**

#### **Bước 1: Tạo Namespace**
Tổ chức tài nguyên trong một namespace riêng.
```bash
kubectl create namespace blog-system
kubectl config set-context --current --namespace=blog-system
```

---

#### **Bước 2: Tạo Secret cho MySQL và WordPress**
Lưu thông tin nhạy cảm như mật khẩu.
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: blog-secrets
  namespace: blog-system
type: Opaque
data:
  mysql-root-password: cm9vdHBhc3M=  # base64 của "rootpass"
  wordpress-db-password: d29yZHByZXNz  # base64 của "wordpress"
```
```bash
kubectl apply -f secret.yaml
```

---

#### **Bước 3: Tạo ConfigMap cho WordPress**
Tách cấu hình môi trường.
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: blog-system
data:
  db_host: mysql-service
  db_name: wordpress_db
```
```bash
kubectl apply -f configmap.yaml
```

---

#### **Bước 4: Triển khai MySQL với StatefulSet và Persistent Volume**
Sử dụng StatefulSet để đảm bảo danh tính và thứ tự cho database, kết hợp Persistent Volume để lưu dữ liệu.

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: blog-system
spec:
  serviceName: mysql-service
  replicas: 1
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
              name: blog-secrets
              key: mysql-root-password
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: db_name
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blog-secrets
              key: wordpress-db-password
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
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: blog-system
spec:
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
  clusterIP: None  # Headless service cho StatefulSet
```
```bash
kubectl apply -f mysql-statefulset.yaml
```

---

#### **Bước 5: Triển khai WordPress với Deployment và HPA**
Sử dụng Deployment để triển khai WordPress, kết hợp HPA để tự động mở rộng khi tải tăng.

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: blog-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: db_host
        - name: WORDPRESS_DB_NAME
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: db_name
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blog-secrets
              key: wordpress-db-password
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  namespace: blog-system
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: wordpress
  type: ClusterIP
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: blog-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```
```bash
kubectl apply -f wordpress-deployment.yaml
```

---

#### **Bước 6: Thiết lập Ingress để truy cập WordPress**
Dùng Ingress để định tuyến HTTP đến WordPress.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: blog-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: blog.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress-service
            port:
              number: 80
```
1. Áp dụng Ingress:
   ```bash
   kubectl apply -f ingress.yaml
   ```
2. Thêm `blog.local` vào `/etc/hosts` (trên máy local):
   ```bash
   echo "$(minikube ip) blog.local" | sudo tee -a /etc/hosts
   ```

---

#### **Bước 7: Kiểm tra và thử tải**
1. **Kiểm tra trạng thái**:
   ```bash
   kubectl get all -n blog-system
   kubectl get ingress -n blog-system
   ```
2. Truy cập: Mở trình duyệt và vào `http://blog.local`. Bạn sẽ thấy giao diện cài đặt WordPress.
3. **Thử tải với HPA**:
   ```bash
   kubectl run -i --tty load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://blog.local; done"
   ```
   Theo dõi HPA:
   ```bash
   kubectl get hpa -n blog-system -w
   ```

---

#### **Bước 8: Dọn dẹp**
Xóa toàn bộ tài nguyên:
```bash
kubectl delete namespace blog-system
```

---

### **Điểm khó và tích hợp**
- **StatefulSet + Volume**: MySQL cần lưu dữ liệu lâu dài, yêu cầu quản lý danh tính và PVC.
- **HPA**: WordPress tự động mở rộng khi tải tăng, đòi hỏi cấu hình tài nguyên chính xác.
- **Ingress**: Định tuyến HTTP phức tạp hơn NodePort/LoadBalancer.
- **Namespace**: Tổ chức tài nguyên trong không gian riêng biệt.
- **ConfigMap/Secret**: Tách cấu hình và bảo mật thông tin nhạy cảm.
- **Service**: Liên kết MySQL và WordPress trong cluster.

---

### **Mở rộng (tùy chọn nếu muốn thử thách thêm)**
1. Thêm LoadBalancer cho WordPress thay vì Ingress (dùng trên cloud).
2. Sử dụng Helm để triển khai WordPress thay vì YAML thủ công (`helm install wordpress bitnami/wordpress`).
3. Thêm replica cho MySQL (MySQL Cluster) với cấu hình phức tạp hơn.

---

### **Kết quả mong đợi**
- Hệ thống blog chạy ổn định, truy cập được qua `blog.local`.
- WordPress tự động mở rộng khi tải tăng.
- Dữ liệu MySQL được lưu trữ an toàn dù pod restart.

Nếu bạn cần hỗ trợ chi tiết hơn khi thực hành (debug lỗi, giải thích sâu), cứ hỏi nhé! Chúc bạn thành công!
