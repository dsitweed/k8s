Nếu không sử dụng **Ingress**, bạn vẫn có thể **truy cập vào ứng dụng** đang chạy trong **Deployment** của Kubernetes bằng cách sử dụng một trong các phương thức sau:

---

### 1. **Sử dụng `kubectl port-forward`:**

Lệnh `kubectl port-forward` cho phép bạn **mở một cổng** từ máy tính của bạn (local machine) và chuyển tiếp đến một cổng trên container hoặc pod trong cluster Kubernetes.

#### Cách sử dụng:

Giả sử bạn có một Deployment tên là `my-app`, và container của bạn chạy trên cổng `8080`:

```bash
kubectl port-forward deployment/my-app 8080:8080 -n your-namespace
```

- `8080:8080`: Đây là cấu hình để chuyển tiếp từ cổng `8080` trên máy tính của bạn đến cổng `8080` của container.
- `your-namespace`: Namespace của Deployment. Nếu bạn không có namespace riêng, có thể bỏ qua `-n your-namespace`.

Sau khi chạy lệnh trên, bạn có thể truy cập ứng dụng bằng cách mở trình duyệt và vào **`http://localhost:8080`**.

---

### 2. **Sử dụng `kubectl expose` để tạo Service (ClusterIP, NodePort, hoặc LoadBalancer):**

Kubernetes cung cấp một cách để truy cập ứng dụng qua **Service**. Nếu bạn không dùng Ingress, bạn có thể tạo một **Service** để expose ứng dụng của mình.

#### Cách tạo Service (ClusterIP, NodePort, LoadBalancer):

- **ClusterIP**: Đây là loại mặc định của Service, chỉ có thể truy cập từ bên trong cluster.
  
  ```bash
  kubectl expose deployment my-app --port=8080 --target-port=8080 --name=my-app-service -n your-namespace
  ```

  Sau khi tạo Service, bạn có thể truy cập ứng dụng từ các Pod khác trong cùng cluster, nhưng không thể truy cập từ ngoài cluster.

- **NodePort**: Nếu bạn muốn truy cập ứng dụng từ bên ngoài cluster, bạn có thể sử dụng loại **NodePort**.
  
  ```bash
  kubectl expose deployment my-app --type=NodePort --port=8080 --target-port=8080 --name=my-app-service -n your-namespace
  ```

  Sau khi tạo, Kubernetes sẽ mở một cổng trên tất cả các Node của bạn và bạn có thể truy cập vào ứng dụng thông qua **`<NodeIP>:<NodePort>`**.

  Bạn có thể kiểm tra NodePort bằng cách:

  ```bash
  kubectl get svc my-app-service -n your-namespace
  ```

  Ví dụ: **`<NodeIP>:<NodePort>`**.

- **LoadBalancer**: Nếu bạn đang sử dụng một cloud provider (AWS, GCP, Azure), bạn có thể tạo một **LoadBalancer** để expose ứng dụng.

  ```bash
  kubectl expose deployment my-app --type=LoadBalancer --port=8080 --target-port=8080 --name=my-app-service -n your-namespace
  ```

  Khi sử dụng LoadBalancer, Kubernetes sẽ tự động tạo một IP công cộng cho bạn, và bạn có thể truy cập vào ứng dụng qua địa chỉ IP đó.

---

### 3. **Truy cập từ bên trong Cluster (Pod khác):**

Nếu bạn chỉ cần truy cập từ các Pod khác trong cùng cluster, bạn có thể sử dụng **Service** loại `ClusterIP` để expose Deployment của bạn và truy cập vào ứng dụng từ các Pod khác.

Giả sử bạn đã tạo Service là `my-app-service`, các Pod khác trong cùng namespace có thể truy cập qua tên `my-app-service` và cổng `8080`.

Ví dụ:

```bash
curl http://my-app-service:8080
```

---

## **Tóm lại:**

- **Không dùng Ingress**: Bạn có thể dùng `kubectl port-forward`, hoặc tạo **Service** với loại `ClusterIP`, `NodePort`, hoặc `LoadBalancer` để truy cập ứng dụng của mình.
- Nếu chỉ cần truy cập từ bên trong cluster, **ClusterIP** là đủ.
- Nếu muốn truy cập từ ngoài cluster, **NodePort** hoặc **LoadBalancer** sẽ là lựa chọn thích hợp.