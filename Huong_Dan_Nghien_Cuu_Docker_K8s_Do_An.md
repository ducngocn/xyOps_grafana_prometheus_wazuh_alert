# **HƯỚNG DẪN NGHIÊN CỨU & NÂNG CẤP ĐỒ ÁN VỚI DOCKER & KUBERNETES (K8S)**

> **Đề tài mở rộng:** Triển khai Hệ thống Giám sát & Cảnh báo An toàn Thông tin Tập trung cho Hạ tầng Cloud-Native / Kubernetes sử dụng Prometheus, Grafana và Wazuh SIEM.

---

## 📑 MỤC LỤC
1. [Lý do & Lợi ích khi đưa Docker & K8s vào Đồ án](#1-lý-do--lợi-ích-khi-đưa-docker--k8s-vào-đồ-án)
2. [Lộ trình Kiến thức Cần Tìm hiểu (Roadmap)](#2-lộ-trình-kiến-thức-cần-tìm-hiểu-roadmap)
   - [Phần 1: Nền tảng Docker & Containerization](#phần-1-nền-tảng-docker--containerization)
   - [Phần 2: Nền tảng Kubernetes (K8s)](#phần-2-nền-tảng-kubernetes-k8s)
   - [Phần 3: Chuyên sâu về Helm Chart (Package Manager cho K8s)](#phần-3-chuyên-sâu-về-helm-chart-package-manager-cho-k8s)
   - [Phần 4: Monitoring & Security trên K8s (Lõi Đồ án)](#phần-4-monitoring--security-trên-k8s-lõi-đồ-án)
3. [Kịch bản Triển khai Lab Thực tế (Step-by-Step)](#3-kịch-bản-triển-khai-lab-thực-tế-step-by-step)
4. [Gợi ý Cập nhật Báo cáo Đồ án](#4-gợi-ý-cập-nhật-báo-cáo-đồ-án)

---

## 1. LÝ DO & LỢI ÍCH KHI ĐƯA DOCKER & K8S VÀO ĐỒ ÁN

- **Chuẩn hóa DevSecOps / Cloud-Native:** Hiện nay các doanh nghiệp hầu hết đã dịch chuyển hạ tầng lên Container và K8s. Việc đưa K8s vào đồ án giúp bạn làm quen đúng với môi trường thực tế tại các công ty lớn.
- **Nâng tầm quy mô Đồ án:** Chuyển từ *"giám sát 1-2 máy ảo đơn lẻ"* sang *"giám sát toàn bộ cụm Container/Microservices linh hoạt"*.
- **Tính ứng dụng thực tiễn cao:** Giúp báo cáo đồ án đạt điểm cao hơn nhờ tính hiện đại, bao quát cả góc độ Vận hành (DevOps - Prometheus/Grafana) lẫn An ninh (SecOps - Wazuh SIEM).

---

## 2. LỘ TRÌNH KIẾN THỨC CẦN TÌM HIỂU (ROADMAP)

### 🔹 Phần 1: Nền tảng Docker & Containerization
1. **Khái niệm cơ bản:**
   - Container khác gì với Máy ảo (Virtual Machine - VM)? (Chia sẻ OS Kernel vs Giả lập phần cứng).
   - Docker Architecture: Docker Daemon, Docker CLI, Docker Image, Container, Docker Registry (Docker Hub).
2. **Kỹ năng thực hành Docker:**
   - Viết `Dockerfile` để đóng gói một ứng dụng mẫu (Node.js/Python/Go).
   - Quản lý Container Lifecycle: `docker run`, `docker stop`, `docker logs`, `docker exec`.
   - Lưu trữ & Mạng: Docker Volumes (Persistent Data) và Docker Networks (Bridge, Host, Overlay).
   - Sử dụng **Docker Compose**: Định nghĩa cụm nhiều container bằng tệp `docker-compose.yml` (ví dụ: chạy thử Prometheus + Grafana).

---

### 🔹 Phần 2: Nền tảng Kubernetes (K8s)
1. **Kiến trúc Cụm K8s (Cluster Architecture):**
   - **Control Plane (Master Node):**
     - `kube-apiserver`: Trung tâm tiếp nhận lệnh (REST API).
     - `etcd`: CSDL Key-Value lưu toàn bộ trạng thái cụm K8s.
     - `kube-scheduler`: Quyết định Pod nào chạy trên Worker Node nào.
     - `kube-controller-manager`: Quản lý các controller (Node, ReplicaSet...).
   - **Worker Node:**
     - `kubelet`: Agent giao tiếp với Control Plane và điều khiển Container Runtime.
     - `kube-proxy`: Quản lý Mạng và Load Balancing nội bộ.
     - `Container Runtime`: containerd / CRI-O (thay thế Docker Engine trên K8s hiện đại).
2. **Các Đối tượng Cốt lõi trong K8s (K8s Objects):**
   - **Pod:** Đơn vị tính toán nhỏ nhất (chứa 1 hoặc nhiều container).
   - **Deployment:** Quản lý khai báo, khôi phục tự động (Self-healing) và nâng cấp không gián đoạn (Rolling Update) cho các Pod.
   - **StatefulSet:** Dành cho các ứng dụng có trạng thái / CSDL (như Prometheus TSDB, OpenSearch/Wazuh Indexer).
   - **DaemonSet:** Đảm bảo chạy đúng **1 bản sao Pod trên MỌI Worker Node** (Rất quan trọng cho Node Exporter & Wazuh Agent!).
   - **Service & Ingress:** Mở cổng kết nối mạng cho Pod (ClusterIP, NodePort, LoadBalancer, Ingress Controller).
   - **ConfigMap & Secret:** Quản lý cấu hình (`prometheus.yml`, `ruleset`) và mật khẩu độc lập với mã nguồn.
   - **PersistentVolume (PV) & PersistentVolumeClaim (PVC):** Lưu trữ dữ liệu bền vững khi Pod bị khởi động lại.

---

### 🔹 Phần 3: Chuyên sâu về Helm Chart (Package Manager cho K8s)

#### 1. Helm Chart là gì?
- **Helm** đóng vai trò tương tự như `apt` trên Ubuntu hay `npm` trong Node.js, nhưng dành riêng cho Kubernetes.
- Thay vì phải viết và nạp thủ công hàng chục tệp YAML (`deployment.yaml`, `service.yaml`, `pvc.yaml`, `ingress.yaml`...), **Helm Chart** đóng gói tất cả cấu hình đó thành một bộ cài đặt hoàn chỉnh có thể truyền biến tùy biến.

#### 2. Cấu trúc cơ bản của 1 Helm Chart:
```text
my-chart/
├── Chart.yaml          # Chứa thông tin về Chart (tên, phiên bản, mô tả)
├── values.yaml         # Chứa các biến cấu hình mặc định (có thể override khi install)
├── templates/          # Chứa các file template YAML K8s (Deployment, Service...)
└── charts/             # Chứa các chart phụ thuộc (dependencies)
```

#### 3. Các câu lệnh Helm cốt lõi cần nhớ:
- **Thêm Repository (Kho chứa Chart):**
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  ```
- **Triển khai ứng dụng (Install):**
  ```bash
  helm install <tên_release> <tên_repo>/<tên_chart> -f custom-values.yaml -n <namespace> --create-namespace
  ```
- **Kiểm tra trạng thái & Cập nhật (List / Upgrade / Rollback):**
  ```bash
  helm list -n <namespace>                              # Xem các ứng dụng đã cài
  helm upgrade <tên_release> <tên_chart> -f values.yaml # Cập nhật cấu hình
  helm rollback <tên_release> <revision>                # Quay lại phiên bản cũ nếu lỗi
  helm uninstall <tên_release> -n <namespace>           # Gỡ bỏ ứng dụng
  ```

#### 4. Ứng dụng Helm Chart trực tiếp vào Đồ án này:
- **Triển khai `kube-prometheus-stack`:**
  - Chỉ cần 1 lệnh `helm install` để tự động dựng toàn bộ: Prometheus Operator, Grafana, Alertmanager, Node-Exporter DaemonSet, kube-state-metrics.
- **Triển khai `wazuh-kubernetes`:**
  - Helm chart chính thức từ Wazuh giúp dựng cụm Wazuh Indexer, Wazuh Manager & Dashboard trên K8s cực kỳ nhanh chóng.
- **Tự động hóa với `custom-values.yaml`:**
  - Thay vì chạy lệnh `grafana-cli plugins install ...` bằng tay như bài trước, bạn chỉ cần khai báo vào file `values.yaml` của Grafana:
    ```yaml
    grafana:
      plugins:
        - grafana-opensearch-datasource
    ```
  - Helm sẽ tự động cài đặt plugin này khi Grafana khởi chạy!

---

### 🔹 Phần 4: Monitoring & Security trên K8s (Lõi Đồ án)

#### A. Giám sát Metrics (Prometheus & Grafana)
- **Prometheus Operator & Helm Chart `kube-prometheus-stack`:**
  - Tự động triển khai Prometheus, Alertmanager, Grafana, `kube-state-metrics` và `node-exporter`.
- **Thành phần thu thập Metrics trong K8s:**
  - **cAdvisor:** Trích xuất thông số tài nguyên (CPU/RAM/Network) của từng Container/Pod.
  - **kube-state-metrics:** Trích xuất trạng thái cụm K8s (số lượng Pod đang Running/CrashLoopBackOff, số Node Ready...).
  - **Node Exporter (DaemonSet):** Thu thập tài nguyên phần cứng của bản thân các Worker Node.

#### B. Giám sát An toàn Thông tin (Wazuh SIEM trên K8s)
- **Wazuh Agent dưới dạng DaemonSet:**
  - Triển khai Wazuh Agent trên từng Node để phát hiện mã độc, lỗ hổng OS, file bị thay đổi trái phép (Syscheck) và brute-force tấn công vào Node/Container.
- **Tích hợp K8s Audit Logging (Cực kỳ đắt giá cho Đồ án):**
  - Cấu hình K8s API Server đẩy **Audit Logs** về Wazuh Manager.
  - Wazuh đối soát Rule để phát hiện:
    - AI vừa thực thi lệnh chui vào Pod: `kubectl exec -it ...`
    - AI vừa tạo Pod có quyền Root nguy hiểm (`privileged: true`).
    - AI vừa sửa đổi ConfigMap / Secret trái phép.

#### C. Tự động hóa Cảnh báo & Điều tra (Grafana Alert + Telegram + Deep Link)
- Khi Pod bị quá tải hoặc bị tấn công ➔ Grafana phát hiện qua PromQL/OpenSearch query.
- Gửi tin nhắn Telegram kèm nút bấm **Deep Link RISON**.
- Người quản trị bấm nút ➔ Chuyển hướng trực tiếp đến OpenSearch/Wazuh Dashboards lọc đúng Pod/IP đang bị sự cố.

---

## 3. KỊCH BẢN TRIỂN KHAI LAB THỰC TẾ (STEP-BY-STEP)

Để làm Lab trên máy cá nhân mà không tốn quá nhiều RAM, bạn nên đi theo thứ tự sau:

1. **Bước 1: Dựng cụm K8s siêu nhẹ với K3s**
   - Tạo 1 hoặc 2 máy ảo Ubuntu (RAM ~ 4GB/máy).
   - Cài K3s (chỉ mất 1 câu lệnh):
     ```bash
     curl -sfL https://get.k3s.io | sh -
     ```
2. **Bước 2: Cài đặt Stack Giám sát với Helm**
   - Cài Helm và deploy `kube-prometheus-stack`:
     ```bash
     helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
     helm repo update
     helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
     ```
3. **Bước 3: Triển khai Wazuh Agent DaemonSet**
   - Triển khai Wazuh Manager (có thể chạy trên Docker/VM riêng hoặc K8s).
   - Triển khai Wazuh Agent dưới dạng DaemonSet trên K8s để tự động giám sát tất cả các Node.
4. **Bước 4: Deploy ứng dụng thử nghiệm (Sample App)**
   - Triển khai 1 ứng dụng web Microservice đơn giản (ví dụ: Nginx/Web App).
   - Giả lập quá tải CPU hoặc gửi request tấn công vào Web App ➔ Đánh giá luồng cảnh báo Telegram.

---

## 4. GỢI Ý CẬP NHẬT BÁO CÁO ĐỒ ÁN

| Chương | Nội dung cần bổ sung / điều chỉnh |
| :--- | :--- |
| **Chương 1: Mở đầu** | Bổ sung lý do dịch chuyển sang hạ tầng Cloud-Native / K8s và bài toán giám sát container. |
| **Chương 2: Cơ sở lý thuyết** | Thêm mục **2.5: Tổng quan về Containerization (Docker) và Kubernetes (K8s)**<br>Thêm mục **2.6: Tổng quan về Helm Chart - Package Manager cho K8s**<br>Thêm mục **2.7: Mô hình K8s Audit Logging và K8s Metrics Architecture**. |
| **Chương 3: Triển khai thực tế** | Thay thế các lệnh cài đặt `docker-compose` đơn lẻ bằng quy trình triển khai K8s Manifests / Helm Charts (`kube-prometheus-stack`, DaemonSet Wazuh Agent). |
| **Chương 4: Tích hợp & Điều tra** | Xây dựng kịch bản thử nghiệm: Giám sát sự cố trên Pod K8s, bóc tách `pod_name`, `namespace`, `src_ip` đưa vào Telegram Alert & RISON Deep Link. |
| **Chương 5: Kết luận** | Nhấn mạnh tính mở rộng cao của K8s và khả năng sẵn sàng tích hợp AIOps/Auto-scaling. |

---
*Chúc bạn hoàn thành xuất sắc đồ án!*
