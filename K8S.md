Dưới đây là giải thích ngắn gọn và dễ hiểu bằng tiếng Việt về sự khác biệt giữa **NodePort**, **LoadBalancer** và **Ingress** trong Kubernetes, cùng với cách chúng hoạt động và ứng dụng thực tế.
---
### **1. NodePort**
- **Định nghĩa**: Là một loại Service trong Kubernetes, mở một cổng (port) cụ thể trên **tất cả các node** trong cluster để truy cập từ bên ngoài.
- **Cách hoạt động**:
  - Kubernetes ánh xạ một cổng trong khoảng 30000-32767 trên mỗi node đến cổng của pod (thông qua Service).
  - Bạn truy cập bằng cách dùng `IP của node:cổng NodePort`.
- **Đặc điểm**:
  - Phạm vi: Chỉ dùng được trong nội bộ hoặc môi trường thử nghiệm.
  - Không có cân bằng tải thực sự từ bên ngoài, bạn phải tự chọn node để truy cập.
  - Chỉ hỗ trợ giao thức TCP/UDP, không quản lý được HTTP/HTTPS phức tạp.
- **Ví dụ**: Nếu Service có `nodePort: 30007`, bạn truy cập qua `http://<node-ip>:30007`.
- **Ứng dụng**: Dùng cho thử nghiệm hoặc khi không cần truy cập phức tạp từ bên ngoài.
---
### **2. LoadBalancer**
- **Định nghĩa**: Là một loại Service tích hợp với **cloud provider** (như AWS, GCP, Azure) để cung cấp một địa chỉ IP công cộng và cân bằng tải đến các pod.
- **Cách hoạt động**:
  - Tạo một load balancer bên ngoài (do cloud provider quản lý).
  - Traffic từ internet đi qua load balancer, sau đó được phân phối đến các pod trong Service.
- **Đặc điểm**:
  - Cung cấp một IP công cộng duy nhất để truy cập (không cần biết IP của node).
  - Tự động cân bằng tải giữa các pod.
  - Phụ thuộc vào cloud provider, không hoạt động tốt trên môi trường local (như Minikube) trừ khi có cấu hình đặc biệt (ví dụ: MetalLB).
  - Chỉ hỗ trợ TCP/UDP cơ bản, không xử lý được định tuyến HTTP/HTTPS theo path hay domain.
- **Ví dụ**: Triển khai Service `type: LoadBalancer`, bạn nhận được URL như `http://<load-balancer-ip>`.
- **Ứng dụng**: Dùng cho ứng dụng sản phẩm cần truy cập công cộng trên cloud.
---
### **3. Ingress**
- **Định nghĩa**: Không phải là một loại Service mà là một tài nguyên riêng trong Kubernetes, dùng để quản lý truy cập HTTP/HTTPS từ bên ngoài đến các Service theo quy tắc thông minh (dựa trên domain, path...).
- **Cách hoạt động**:
  - Cần một **Ingress Controller** (như Nginx, Traefik) để xử lý logic.
  - Định tuyến traffic dựa trên URL (ví dụ: `example.com/api` đến Service A, `example.com/web` đến Service B).
  - Thường kết hợp với Service loại `ClusterIP` bên trong cluster.
- **Đặc điểm**:
  - Hỗ trợ HTTP/HTTPS, định tuyến theo domain, path, hoặc header.
  - Không cung cấp IP công cộng trực tiếp, mà cần kết hợp với LoadBalancer hoặc NodePort để nhận traffic từ ngoài.
  - Linh hoạt hơn, phù hợp với ứng dụng web phức tạp.
- **Ví dụ**:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example-ingress
  spec:
    rules:
    - host: example.com
      http:
        paths:
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: api-service
              port:
                number: 80
  ```
  - Truy cập: `http://example.com/api` sẽ đến `api-service`.
- **Ứng dụng**: Dùng cho hệ thống web lớn, cần định tuyến thông minh và quản lý nhiều domain/path.
---
### **So sánh chính**
| **Tiêu chí**           | **NodePort**                  | **LoadBalancer**             | **Ingress**                  |
|-------------------------|-------------------------------|------------------------------|------------------------------|
| **Mục đích**           | Truy cập cơ bản qua node     | Truy cập công cộng qua cloud | Quản lý truy cập HTTP/HTTPS |
| **IP công cộng**       | Không (dùng IP node)         | Có (từ cloud provider)      | Không (phụ thuộc Service)   |
| **Cân bằng tải**       | Không tự động                | Có (do cloud provider)       | Có (do Ingress Controller)  |
| **Giao thức**          | TCP/UDP                      | TCP/UDP                      | HTTP/HTTPS                  |
| **Định tuyến thông minh** | Không                     | Không                        | Có (domain, path)           |
| **Môi trường phù hợp** | Local, thử nghiệm            | Cloud sản phẩm               | Web phức tạp                |
---
### **Khác biệt chính**
1. **NodePort vs LoadBalancer**:
   - NodePort mở cổng trên từng node, không tự động cân bằng tải, phù hợp môi trường nhỏ.
   - LoadBalancer cung cấp IP công cộng và cân bằng tải, dùng tốt trên cloud nhưng tốn tài nguyên hơn.
2. **NodePort/LoadBalancer vs Ingress**:
   - NodePort và LoadBalancer chỉ định tuyến TCP/UDP cơ bản, không hiểu logic HTTP (domain, path).
   - Ingress chuyên xử lý HTTP/HTTPS, linh hoạt hơn nhưng cần cấu hình thêm Ingress Controller.
---
### **Khi nào dùng cái nào?**
- **NodePort**: Thử nghiệm nhanh hoặc môi trường không cần truy cập công cộng phức tạp.
- **LoadBalancer**: Ứng dụng cần IP công cộng đơn giản trên cloud (như API hoặc game server).
- **Ingress**: Hệ thống web cần định tuyến theo domain/path (như microservices hoặc website lớn).
Nếu bạn cần ví dụ cụ thể hoặc cách triển khai từng loại, hãy cho tôi biết nhé!
