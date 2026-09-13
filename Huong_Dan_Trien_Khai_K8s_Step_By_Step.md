# **HƯỚNG DẪN CHI TIẾT THỰC HÀNH TRIỂN KHAI KUBERNETES (K3S) & K8S MONITORING**

> **Đồ án mở rộng:** *Triển khai Hệ thống Giám sát & Cảnh báo An toàn Thông tin Tập trung cho Hạ tầng Cloud-Native sử dụng Prometheus, Grafana và Wazuh SIEM trên Kubernetes.*

---

## 📑 MỤC LỤC
1. [Chuẩn bị Môi trường & An toàn (Snapshot)](#1-chuẩn-bị-môi-trường--an-toàn-snapshot)
2. [BƯỚC 1: Dựng cụm Kubernetes siêu nhẹ (K3s)](#bước-1-dựng-cụm-kubernetes-siêu-nhẹ-k3s)
3. [BƯỚC 2: Cài đặt Helm (Package Manager cho K8s)](#bước-2-cài-đặt-helm-package-manager-cho-k8s)
4. [BƯỚC 3: Triển khai Stack Giám sát (`kube-prometheus-stack`)](#bước-3-triển-khai-stack-giám-sát-kube-prometheus-stack)
5. [BƯỚC 4: Triển khai Wazuh Agent dạng DaemonSet trên K8s](#bước-4-triển-khai-wazuh-agent-dạng-daemonset-trên-k8s)
6. [BƯỚC 5: Deploy Web App Mẫu & Giả lập Sự cố An ninh](#bước-5-deploy-web-app-mẫu--giả-lập-sự-cố-an-ninh)
7. [BƯỚC 6: Hướng dẫn Cập nhật Báo cáo Đồ án (LaTeX)](#bước-6-hướng-dẫn-cập-nhật-báo-cáo-đồ-án-latex)

---

## 1. CHUẨN BỊ MÔI TRƯỜNG & AN TOÀN (SNAPSHOT)

> ⚠️ **Yêu cầu bắt buộc trước khi thực hiện:**
> Bạn phải tạo **Snapshot** cho 2 máy ảo Ubuntu của bạn (`xyOps` - `192.168.68.181` và `wazuh` - `192.168.68.170`) trên VMware Workstation để sẵn sàng rollback nếu gặp lỗi.

---

## BƯỚC 1: DỰNG CỤM KUBERNETES SIÊU NHẸ (K3S)

### 1.1 Cài đặt K3s trên máy ảo Ubuntu (`xyOps`)
Đăng nhập vào máy ảo **`xyOps`** và chạy câu lệnh cài đặt K3s tự động:

```bash
curl -sfL https://get.k3s.io | sh -
```

### 1.2 Cấu hình phân quyền `kubectl` cho User thường (Không cần gõ `sudo`)
Mặc định file cấu hình K8s chỉ đọc được bởi `root`. Chạy khối lệnh sau để User của bạn (`ducnn`) gõ lệnh `kubectl` thoải mái:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc
```

### 1.3 Kiểm tra trạng thái Cluster K8s
Chạy lệnh kiểm tra Node:

```bash
kubectl get nodes
```

**KẾT QUẢ KỲ VỌNG:**
```text
NAME    STATUS   ROLES                  AGE   VERSION
xyops   Ready    control-plane,master   30s   v1.30.x+k3s1
```
*(Nếu cột `STATUS` hiển thị là **`Ready`** nghĩa là cụm K8s đã sẵn sàng!)*

---

## BƯỚC 2: CÀI ĐẶT HELM (PACKAGE MANAGER CHO K8S)

### 2.1 Tải và cài đặt Helm CLI
Helm giúp bạn cài cả bộ Prometheus + Grafana chỉ bằng 1 câu lệnh mà không cần viết hàng trăm file YAML thủ công.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Kiểm tra phiên bản Helm:
```bash
helm version
```

### 2.2 Thêm Repository chứa các Chart chuẩn của cộng đồng Prometheus
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## BƯỚC 3: TRIỂN KHAI STACK GIÁM SÁT (`kube-prometheus-stack`)

### 3.1 Tạo Namespace riêng cho Giám sát (`monitoring`)
```bash
kubectl create namespace monitoring
```

### 3.2 Tạo tệp tùy biến cấu hình `custom-values.yaml`
Tạo file cấu hình để bật sẵn plugin OpenSearch/Wazuh cho Grafana và mở cổng truy cập NodePort:

```bash
cat <<EOF > custom-values.yaml
grafana:
  enabled: true
  service:
    type: NodePort
    nodePort: 30000
  plugins:
    - grafana-opensearch-datasource
  adminPassword: "admin"

prometheus:
  enabled: true
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
EOF
```

### 3.3 Chạy lệnh Helm Install cài đặt toàn bộ Stack Giám sát
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f custom-values.yaml
```

### 3.4 Kiểm tra các Pods đang khởi tạo trong Namespace `monitoring`
```bash
kubectl get pods -n monitoring
```

**KẾT QUẢ KỲ VỌNG:** Bạn sẽ thấy danh sách các Pod:
* `prometheus-monitoring-kube-prometheus-prometheus-...`
* `monitoring-grafana-...`
* `monitoring-kube-prometheus-node-exporter-...` (DaemonSet)
* `monitoring-kube-state-metrics-...`

*(Đợi 1-2 phút cho tất cả các Pod chuyển trạng thái sang **`Running`**).*

---

## BƯỚC 4: TRIỂN KHAI WAZUH AGENT DẠNG DAEMONSET TRÊN K8S

### 4.1 Tạo tệp Manifest `wazuh-agent-daemonset.yaml`
Tạo file khai báo DaemonSet để K8s tự động đẩy Wazuh Agent lên tất cả các Worker Node:

```bash
cat <<EOF > wazuh-agent-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: wazuh-agent
  namespace: monitoring
  labels:
    app: wazuh-agent
spec:
  selector:
    matchLabels:
      app: wazuh-agent
  template:
    metadata:
      labels:
        app: wazuh-agent
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: wazuh-agent
        image: wazuh/wazuh-agent:4.9.0
        securityContext:
          privileged: true
        env:
        - name: WAZUH_MANAGER
          value: "192.168.68.170"
        - name: WAZUH_AGENT_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
EOF
```

### 4.2 Lệnh nạp DaemonSet vào K8s
```bash
kubectl apply -f wazuh-agent-daemonset.yaml
```

### 4.3 Kiểm tra trạng thái DaemonSet
```bash
kubectl get daemonset -n monitoring
```

---

## BƯỚC 5: DEPLOY WEB APP MẪU & GIẢ LẬP SỰ CỐ AN NINH

### 5.1 Deploy một ứng dụng Nginx thử nghiệm lên K8s
```bash
kubectl create deployment sample-web --image=nginx:alpine
kubectl expose deployment sample-web --port=80 --type=NodePort
```

### 5.2 Thử nghiệm Kịch bản Tấn công / Can thiệp chui vào Pod (`kubectl exec`)
Chạy lệnh chui vào bên trong Pod Nginx để giả lập hacker đột nhập:

```bash
# Lấy tên Pod Nginx vừa tạo
POD_NAME=$(kubectl get pods -l app=sample-web -o jsonpath="{.items[0].metadata.name}")

# Thực hiện lệnh chui vào Pod (Tương đương hành vi tấn công)
kubectl exec -it $POD_NAME -- /bin/sh
```

👉 **Đánh giá:** Wazuh SIEM thông qua K8s Audit Log sẽ bắt được hành vi `kubectl exec` này và Grafana Alerting sẽ gửi tin nhắn **Telegram** cảnh báo kèm nút bấm **RISON Deep Link**!

---

## BƯỚC 6: HƯỚNG DẪN CẬP NHẬT BÁO CÁO ĐỒ ÁN (LATEX)

Sau khi làm xong Lab, bạn mở tệp [bao_cao_do_an.tex](file:///d:/github/xyOps/xyOps_grafana_prometheus_wazuh_alert/bao_cao_do_an.tex) và bổ sung nội dung vào các chương:

1. **Chương 2 (Cơ sở lý thuyết):** Thêm mục `\subsection{Tổng quan về Kubernetes (K8s) và Helm Chart}`.
2. **Chương 3 (Triển khai):** Thêm đoạn mã Helm Install và Manifest `wazuh-agent-daemonset.yaml` ở trên vào báo cáo.
3. **Chương 4 (Điều tra sự cố):** Thêm kịch bản thử nghiệm sự cố bằng lệnh `kubectl exec` vào Pod.
