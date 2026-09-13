# **Đề tài: Xây dựng Hệ thống Giám sát và Cảnh báo Thông tin Tập trung cho Hạ tầng Công nghệ Thông tin**

> **Tác giả:** Nguyễn Đức  
> **Mô hình:** Trung tâm Điều hành Hợp nhất (Metrics - Security Logs - Network Traffic)  
> **Nền tảng chính:** Grafana, Prometheus, Wazuh SIEM, Zabbix & Telegram Alerting  

---

## 📑 MỤC LỤC

1. [CHƯƠNG 1: MỞ ĐẦU](#chương-1-mở-đầu)
   - [1.1 Lý do chọn đề tài](#11-lý-do-chọn-đề-tài)
   - [1.2 Mục tiêu nghiên cứu](#12-mục-tiêu-nghiên-cứu)
2. [CHƯƠNG 2: CƠ SỞ LÝ THUYẾT VÀ KIẾN TRÚC HỆ THỐNG](#chương-2-cơ-sở-lý-thuyết-và-kiến-trúc-hệ-thống)
   - [2.1 Tổng quan các nền tảng lõi](#21-tổng-quan-các-nền-tảng-lõi)
   - [2.2 Phân tách 3 luồng dữ liệu giám sát](#22-phân-tách-3-luồng-dữ-liệu-giám-sát)
   - [2.3 Sơ đồ Kiến trúc và Phân tích Luồng Dữ liệu (Data Flow)](#23-sơ-đồ-kiến-trúc-và-phân-tích-luồng-dữ-liệu-data-flow)
3. [CHƯƠNG 3: QUY TRÌNH HỘI TỤ VÀ GIÁM SÁT TẬP TRUNG](#chương-3-quy-trình-hội-tụ-và-giám-sát-tập-trung)
   - [3.1 Phân hệ Giám sát Tài nguyên Server (Prometheus & Node Exporter)](#31-phân-hệ-giám-sát-tài-nguyên-server-prometheus--node-exporter)
   - [3.2 Phân hệ Giám sát An toàn Thông tin (Wazuh SIEM & OpenSearch)](#32-phân-hệ-giám-sát-an-toàn-thông-tin-wazuh-siem--opensearch)
   - [3.3 Phân hệ Giám sát Hạ tầng Mạng (Zabbix & Agentless SNMP)](#33-phân-hệ-giám-sát-hạ-tầng-mạng-zabbix--agentless-snmp)
   - [3.4 Trực quan hóa và Cảnh báo Đa kênh (Grafana & Telegram)](#34-trực-quan-hóa-và-cảnh-báo-đa-kênh-grafana--telegram)
4. [CHƯƠNG 4: TÍCH HỢP ĐIỀU TRA SỰ CỐ VÀ TỰ ĐỘNG HÓA SOC](#chương-4-tích-hợp-điều-tra-sự-cố-và-tự-động-hóa-soc)
   - [4.1 Kỹ thuật nhúng mã Deep Linking RISON](#41-kỹ-thuật-nhúng-mã-deep-linking-rison)
   - [4.2 Quy trình Phản ứng Sự cố Tự động (Automated Incident Response Flow)](#42-quy-trình-phản-ứng-sự-cố-tự-động-automated-incident-response-flow)
5. [CHƯƠNG 5: KẾT LUẬN VÀ HƯỚNG PHÁT TRIỂN](#chương-5-kết-luận-và-hướng-phát-triển)
   - [5.1 Kết quả đạt được](#51-kết-quả-đạt-được)
   - [5.2 Định hình Kiến trúc AIOps tương lai](#52-định-hình-kiến-trúc-aiops-tương-lai)
6. [TÀI LIỆU THAM KHẢO VÀ PHỤ LỤC KỸ THUẬT](#tài-liệu-tham-khảo-và-phụ-lục-kỹ-thuật)

---

## CHƯƠNG 1: MỞ ĐẦU

### 1.1 Lý do chọn đề tài
Trong vận hành hệ thống CNTT Enterprise truyền thống, người quản trị thường đối mặt với rủi ro **"Bội thực công cụ" (Tool Fatigue)**. Khi có sự cố xảy ra, họ phải truy cập hàng loạt các bảng điều khiển tách biệt: xem chỉ số phần cứng trên Prometheus, kiểm tra log an ninh trên Wazuh SIEM, hay xem lưu lượng mạng trên Zabbix. Sự phân mảnh này dẫn đến thời gian trung bình để xử lý sự cố (MTTR) bị kéo dài.

Bên cạnh đó, các hệ thống cảnh báo truyền thống thường hoạt động thụ động, thiếu ngữ cảnh và dễ gây ra tình trạng "Spam cảnh báo", khiến đội ngũ điều hành dễ bỏ lỡ các mối đe dọa nghiêm trọng.

Do đó, đồ án này đề xuất và triển khai **Mô hình Trung tâm Điều hành Hợp nhất** trên nền tảng Grafana kết hợp quy trình tối ưu hóa cảnh báo phản ứng nhanh qua Telegram.

### 1.2 Mục tiêu nghiên cứu
- Phân tách và kết hợp hoàn hảo 3 luồng dữ liệu độc lập: **Metrics phần cứng**, **Logs an toàn thông tin** và **Network Traffic**.
- Triển khai **Grafana** làm trung tâm hội tụ đa nguồn (Multi-Datasource Single Interface).
- Tích hợp **Wazuh SIEM** để phát hiện các hành vi tấn công mạng theo chuẩn MITRE ATT&CK.
- Triển khai **Zabbix** để giám sát hạ tầng mạng (Firewall pfSense, Switch Cisco) theo cơ chế không cần agent (SNMPv2).
- Xây dựng cơ chế cảnh báo thông minh qua **Telegram Bot** kèm kỹ thuật **Deep Linking RISON** rút ngắn thời gian truy vết sự cố.

---

## CHƯƠNG 2: CƠ SỞ LÝ THUYẾT VÀ KIẾN TRÚC HỆ THỐNG

### 2.1 Tổng quan các nền tảng lõi
Hệ thống kết hợp sức mạnh của 6 nền tảng mã nguồn mở chuyên dụng:

1. **Prometheus:** Cơ sở dữ liệu chuỗi thời gian (TSDB) đóng vai trò thu thập chỉ số tài nguyên (Metrics) qua cơ chế Pull.
2. **Node Exporter:** Tác tử (Agent) trích xuất dữ liệu tài nguyên phần cứng hệ điều hành (CPU, RAM, Disk, Network).
3. **Wazuh SIEM:** Nền tảng phân tích log an ninh, giám sát tính toàn vẹn tệp tin (FIM) và phát hiện tấn công theo chuẩn MITRE ATT&CK.
4. **OpenSearch (Wazuh Indexer):** Cơ sở dữ liệu phân tán đánh chỉ mục log an toàn thông tin, hỗ trợ truy vấn tìm kiếm toàn văn bản (Full-text Search).
5. **Zabbix & SNMP:** Hệ thống giám sát thiết bị mạng chuyên dụng không cần agent (SNMPv2), thu thập băng thông và trạng thái kết nối các thiết bị cốt lõi.
6. **Grafana:** Nền tảng trực quan hóa tập trung, thực hiện truy vấn đồng thời tới Prometheus, OpenSearch và Zabbix để hiển thị biểu đồ và kích hoạt cảnh báo.

### 2.2 Phân tách 3 luồng dữ liệu giám sát

| Luồng dữ liệu | Công cụ thu thập | Cơ sở dữ liệu | Đặc điểm dữ liệu |
| :--- | :--- | :--- | :--- |
| **System Metrics** | Prometheus & Node Exporter | Prometheus TSDB | Dữ liệu định lượng (Numbers) biến thiên theo thời gian (`CPU = 85%`, `RAM = 90%`). |
| **Security Logs** | Wazuh Agent | OpenSearch (Wazuh Indexer) | Chuỗi văn bản chi tiết (Text JSON) lưu vết sự kiện tấn công (`Brute-force SSH from 192.168.68.181`). |
| **Network Traffic** | Zabbix Server (SNMPv2) | MySQL Database | Chỉ số băng thông cổng (Bits in/out), trạng thái cổng (`Link Down/Up`), ICMP Ping loss. |

### 2.3 Sơ đồ Kiến trúc và Phân tích Luồng Dữ liệu (Data Flow)

Khối kiến trúc được chia làm 4 phân hệ chính:
1. **Phân hệ Máy chủ Mục tiêu (Monitored Nodes):** Triển khai Node Exporter và Wazuh Agent.
2. **Phân hệ Hạ tầng Mạng (EVE-NG Lab):** Bao gồm pfSense Firewall (`.176`), pfSense Gateway 1 (`.173`), pfSense Gateway 2 (`.182`) và Core Switch Cisco (`.180`) chạy giao thức SNMPv2.
3. **Phân hệ Giám sát Trung tâm (Master Host `xyOps`):** Chạy Container Stack chứa Prometheus, Wazuh Manager, Zabbix Server, MySQL, OpenSearch và Grafana.
4. **Phân hệ Cảnh báo & Phản ứng (Alerting & Incident Response):** Kênh Telegram Bot tiếp nhận cảnh báo thời gian thực từ Grafana và Zabbix.

<img src="images/architec.png" width="750">

---

## CHƯƠNG 3: QUY TRÌNH HỘI TỤ VÀ GIÁM SÁT TẬP TRUNG

### 3.1 Phân hệ Giám sát Tài nguyên Server (Prometheus & Node Exporter)
- Prometheus thiết lập các công việc thu thập (Scrape Jobs) định kỳ trỏ về các máy chủ mục tiêu.
- Sử dụng ngôn ngữ truy vấn **PromQL** để tính toán mức độ tiêu thụ tài nguyên phần cứng theo thời gian thực.

### 3.2 Phân hệ Giám sát An toàn Thông tin (Wazuh SIEM & OpenSearch)
- Wazuh Agent đẩy nhật ký hệ thống về Wazuh Manager để phân tích qua bộ luật (Ruleset).
- Dữ liệu sự kiện an ninh được đánh chỉ mục vào OpenSearch Indexer (`wazuh-alerts-*`), mở cổng kết nối API an toàn để Grafana truy vấn.

### 3.3 Phân hệ Giám sát Hạ tầng Mạng (Zabbix & Agentless SNMP)
- Zabbix Server đóng vai trò trung tâm thu thập dữ liệu SNMPv2 từ các thiết bị mạng mà không cần cài đặt phần mềm phụ trợ.
- Tùy chỉnh tham số **Update Interval = 5s** và biểu thức Trigger `last()` giúp hệ thống nhận biết các sự cố mạng (`Link Down`, `Host Unreachable`) và phát cảnh báo trong vòng **5 đến 10 giây**.

### 3.4 Trực quan hóa và Cảnh báo Đa kênh (Grafana & Telegram)
- Grafana tích hợp đồng thời 3 Data Source (Prometheus, OpenSearch, Zabbix).
- Thiết lập quy tắc cảnh báo (Alert Rules) định tuyến thông báo sự cố mạng và bảo mật trực tiếp về kênh Telegram của đội ngũ điều hành SOC.

---

## CHƯƠNG 4: TÍCH HỢP ĐIỀU TRA SỰ CỐ VÀ TỰ ĐỘNG HÓA SOC

### 4.1 Kỹ thuật nhúng mã Deep Linking RISON
Khi một sự cố an ninh nghiêm trọng xảy ra, Grafana Alerting tự động trích xuất các thuộc tính dữ liệu JSON (như IP kẻ tấn công `data.srcip`, Rule ID) và đóng gói vào một đường link URL chuyển hướng được mã hóa theo chuẩn **RISON**.

Đường link này được nhúng trực tiếp vào nút bấm **"CRITICAL"** trên tin nhắn Telegram.

### 4.2 Quy trình Phản ứng Sự cố Tự động (Automated Incident Response Flow)
1. **Phát hiện:** Grafana / Zabbix phát hiện sự cố và gửi tin nhắn cảnh báo tới Telegram.
2. **Kích hoạt:** Chuyên viên SOC nhấn vào nút **"CRITICAL"** đính kèm dưới tin nhắn Telegram.
3. **Chuyển hướng:** Trình duyệt tự động mở giao diện OpenSearch Dashboards của Wazuh SIEM.
4. **Truy vết:** Thanh tìm kiếm Kuery (KQL) của Wazuh tự động điền sẵn bộ lọc địa chỉ IP kẻ tấn công.
5. **Khắc phục:** Chuyên viên SOC quan sát toàn bộ chuỗi hành vi tấn công trên màn hình điều tra mà không cần tra cứu log thủ công.

---

## CHƯƠNG 5: KẾT LUẬN VÀ HƯỚNG PHÁT TRIỂN

### 5.1 Kết quả đạt được
- **Giải quyết bài toán phân mảnh:** Hội tụ thành công 3 luồng dữ liệu (System Metrics, Security Logs, Network Traffic) trên một màn hình quản lý thống nhất.
- **Tối ưu hóa thời gian xử lý sự cố (MTTR):** Giảm thời gian phát hiện và truy vết sự cố xuống dưới 10 giây nhờ cơ chế Telegram Webhook và kỹ thuật Deep Linking RISON.
- **Hoàn thiện mô hình Lab mạng nâng cao:** Triển khai giám sát SNMP thành công trên 4 thiết bị mạng cốt lõi (pfSense Firewall `.176`, pfSense Gateway 1 `.173`, pfSense Gateway 2 `.182` và Core Switch Cisco `.180`) trong môi trường ảo hóa EVE-NG.

### 5.2 Định hình Kiến trúc AIOps tương lai
- **Ngữ cảnh hóa tự động (Automated Contextualization):** Đính kèm tự động biểu đồ tài nguyên hệ thống tại thời điểm phát sinh sự cố vào thông báo cảnh báo.
- **Tích hợp Tác tử AI (LLM Agents):** Định tuyến cảnh báo từ Grafana tới các Tác tử AI để tự động phân tích nguyên nhân gốc rễ (Root Cause Analysis - RCA) và đề xuất phương án khắc phục tự động.

---

## TÀI LIỆU THAM KHẢO VÀ PHỤ LỤC KỸ THUẬT

### Tài liệu Hướng dẫn Kỹ thuật Chi tiết (Technical Guides):
- [Hướng dẫn Triển khai Zabbix & Giám sát Mạng EVE-NG](file:///d:/github/xyOps/xyOps_grafana_prometheus_wazuh_alert/Huong_Dan_Trien_Khai_Zabbix_Network_Monitoring.md)
- [Hướng dẫn Triển khai Kubernetes Step-by-step](file:///d:/github/xyOps/xyOps_grafana_prometheus_wazuh_alert/Huong_Dan_Trien_Khai_K8s_Step_By_Step.md)
- [Tài liệu Triển khai Chi tiết Toàn bộ Dự án (Full README)](file:///d:/github/xyOps/xyOps_grafana_prometheus_wazuh_alert/README.md)
