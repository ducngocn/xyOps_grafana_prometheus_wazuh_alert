# **HƯỚNG DẪN TRIỂN KHAI ZABBIX VÀ GIÁM SÁT THIẾT BỊ MẠNG (EVE-NG)**

> **Mục tiêu:** Tích hợp hệ thống giám sát mạng chuyên dụng (Zabbix) lên máy chủ `xyOps` thông qua Docker Compose. Kết nối Zabbix với môi trường lab ảo EVE-NG để thu thập lưu lượng mạng (SNMP) và trực quan hóa lên Grafana.

---

## 📑 MỤC LỤC
1. [Kiến trúc Triển khai](#1-kiến-trúc-triển-khai)
2. [BƯỚC 1: Triển khai Zabbix trên máy chủ xyOps](#bước-1-triển-khai-zabbix-trên-máy-chủ-xyops)
3. [BƯỚC 2: Cấu hình Thiết bị mạng trong EVE-NG](#bước-2-cấu-hình-thiết-bị-mạng-trong-eve-ng)
4. [BƯỚC 3: Đưa Thiết bị mạng vào lưới giám sát của Zabbix](#bước-3-đưa-thiết-bị-mạng-vào-lưới-giám-sát-của-zabbix)
5. [BƯỚC 4: Tích hợp Zabbix vào Grafana (Trực quan hóa)](#bước-4-tích-hợp-zabbix-vào-grafana-trực-quan-hóa)
6. [BƯỚC 5: Cấu hình Cảnh báo Telegram (Alerting)](#bước-5-cấu-hình-cảnh-báo-telegram-alerting)

---

## 1. KIẾN TRÚC TRIỂN KHAI

Theo nhu cầu của bạn, chúng ta sẽ **GỘP CHUNG** Zabbix vào file `docker-compose.yml` hiện có của Prometheus và Grafana tại `/opt/monitoring/`.
- Ưu điểm của cách này là bạn chỉ cần quản lý duy nhất 1 file cấu hình cho toàn bộ hệ thống giám sát. Chỉ cần 1 lệnh `docker compose up -d` là khởi động tất cả.
- Zabbix cần 3 thành phần chính: **Zabbix Server** (Core xử lý), **Zabbix Web** (Giao diện UI), và **MySQL** (Lưu trữ dữ liệu cấu hình và metrics mạng).

---

## BƯỚC 1: CẬP NHẬT FILE DOCKER-COMPOSE.YML HIỆN TẠI

**1. Di chuyển vào thư mục Monitoring cũ:**
Mở Terminal trên máy `xyOps` và chạy lệnh sau:
```bash
cd /opt/monitoring
sudo nano docker-compose.yml
```

**2. Khai báo thêm cấu hình Zabbix:**
Kéo xuống tận cùng của file `docker-compose.yml`, và thêm 3 dịch vụ của Zabbix cùng với volume dữ liệu của MySQL vào **bên dưới các dịch vụ hiện có**.

Hãy cẩn thận thụt lề cho thẳng hàng với các `services` cũ:

```yaml
  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --default-authentication-plugin=mysql_native_password
    volumes:
      - zabbix_db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=zabbix_root_pass
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pass
      - MYSQL_DATABASE=zabbix
    restart: unless-stopped

  zabbix-server:
    image: zabbix/zabbix-server-mysql:latest
    container_name: zabbix-server
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pass
    ports:
      - "10051:10051"
    depends_on:
      - mysql-server
    restart: unless-stopped

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:latest
    container_name: zabbix-web
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbix_pass
      - ZBX_SERVER_HOST=zabbix-server
      - PHP_TZ=Asia/Ho_Chi_Minh
    ports:
      - "8080:8080"
    depends_on:
      - mysql-server
      - zabbix-server
    restart: unless-stopped

volumes:
  zabbix_db_data:
```
*(Nếu trong file cũ của bạn đã có thẻ `volumes:` ở cuối cùng rồi, thì bạn chỉ cần khai báo thêm `zabbix_db_data:` bên dưới nó thôi).*

**3. Khởi chạy lại toàn bộ hệ thống:**
```bash
sudo docker compose down
sudo docker compose up -d
```
Đợi khoảng 2-3 phút để MySQL khởi tạo xong. Sau đó, mở trình duyệt truy cập: `http://192.168.68.181:8080`
- **Tài khoản mặc định:** `Admin` (chữ A viết hoa)
- **Mật khẩu:** `zabbix`

---

## BƯỚC 2: CẤU HÌNH THIẾT BỊ MẠNG TRONG EVE-NG

### 2.1. Cấu hình các thiết bị pfSense (Firewall .176, Gateway 1 .173, Gateway 2 .182)

Dựa trên sơ đồ kiến trúc hệ thống, mạng lab bao gồm **3 thiết bị pfSense**:
- **pfSense Firewall chính:** IP `192.168.68.176`
- **pfSense Gateway 1:** IP `192.168.68.173`
- **pfSense Gateway 2:** IP `192.168.68.182`

Cả 3 thiết bị pfSense đều được cắm nối ra mạng ngoài (Cloud `Net`) cùng dải mạng với máy chủ `xyOps` (`192.168.68.181`). Thao tác cấu hình SNMP cho từng con như sau:

**1. Bật giao thức SNMP trên từng thiết bị pfSense:**
Thao tác trên giao diện Web GUI của từng con (`https://192.168.68.176`, `https://192.168.68.173`, và `https://192.168.68.182`):
- Trên thanh menu ngang, chọn **Services** ➔ **SNMP**.
- Cấu hình các thông số sau:
  - Tích chọn ô **Enable SNMP daemon** (Bật dịch vụ SNMP).
  - **SNMP Community:** Đổi thành `public`.
- Kéo xuống dưới cùng và nhấn **Save**.

**2. Tạo Rule mở cổng WAN SNMP:**
- Vào **Firewall** ➔ **Rules** ➔ **WAN** ➔ Nhấp **Add** (thêm luật Pass):
  - **Action:** Pass
  - **Protocol:** UDP
  - **Destination Port Range:** `161` (SNMP)
- Nhấn **Save** ➔ **Apply Changes**.

**3. Kiểm tra kết nối từ máy xyOps:**
```bash
ping 192.168.68.176
ping 192.168.68.173
ping 192.168.68.182
```

### 2.2. Cấu hình Core Switch Cisco (IP: 192.168.68.180)

Để giám sát Switch dễ dàng nhất mà không cần NAT/Routing phức tạp, bạn hãy kéo một đám mây **Net (Cloud0)** trong EVE-NG và cắm vào cổng trống của Switch (ví dụ `Gi0/2`).

Vào CLI của Switch cấu hình như sau:
```text
Switch> enable
Switch# configure terminal
Switch(config)# interface GigabitEthernet 0/2
Switch(config-if)# no switchport
Switch(config-if)# ip address 192.168.68.180 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# exit
Switch(config)# snmp-server community public RO
Switch(config)# end
Switch# write memory
```

---

## BƯỚC 3: ĐƯA CÁC THIẾT BỊ MẠNG VÀO LƯỚI GIÁM SÁT CỦA ZABBIX

Đăng nhập vào giao diện Web của Zabbix (`http://192.168.68.181:8080`).

### 3.1. Add các Firewall & Gateway pfSense vào Zabbix
Vào **Data collection** ➔ **Hosts** ➔ Click **Create Host**:
- **pfSense Firewall:** Host name: `pfSense-Core-Firewall` | Templates: `pfSense SNMP` | Group: `Firewalls` | Interface SNMP: `192.168.68.176` | Community: `public`
- **pfSense Gateway 1:** Host name: `pfSense-GW1` | Templates: `pfSense SNMP` | Group: `Firewalls` | Interface SNMP: `192.168.68.173` | Community: `public`
- **pfSense Gateway 2:** Host name: `pfSense-GW2` | Templates: `pfSense SNMP` | Group: `Firewalls` | Interface SNMP: `192.168.68.182` | Community: `public`

### 3.2. Add Core Switch Cisco vào Zabbix
Vào **Data collection** ➔ **Hosts** ➔ Click **Create Host**:
- **Core Switch:** Host name: `Core-Switch-Cisco` | Templates: `Cisco IOS SNMP` | Group: `Switches` | Interface SNMP: `192.168.68.180` | Community: `public`

> **❗ Lưu ý đặc thù của EVE-NG:**
> Khi xem Latest data của Switch Cisco trong Lab ảo, bạn có thể thấy một số thông số bị báo chấm than đỏ (Not supported) như Nhiệt độ, Nguồn điện (PSUs), Số serial phần cứng. 
> Điều này là **hoàn toàn bình thường**, vì Switch trong EVE-NG là phần mềm mô phỏng, không có phần cứng vật lý nên không thể trả lời Zabbix các chỉ số này. Tuy nhiên, toàn bộ dữ liệu băng thông của 24/48 cổng mạng sẽ vẫn được thu thập đầy đủ.


---

## BƯỚC 4: TÍCH HỢP ZABBIX VÀO GRAFANA (TRỰC QUAN HÓA)

Mặc dù Zabbix vẽ biểu đồ mạng rất tốt, nhưng để Đồ án của bạn chuyên nghiệp và hướng tới **mô hình giám sát tập trung**, chúng ta sẽ đẩy dữ liệu mạng từ Zabbix sang Grafana để hiển thị chung với các thông số của Prometheus.

**1. Cài đặt Plugin Zabbix cho Grafana:**
Truy cập vào máy chủ `xyOps`, chạy lệnh sau để cài Plugin và khởi động lại Grafana:
```bash
sudo docker exec -it grafana grafana cli plugins install alexanderzobnin-zabbix-app
sudo docker restart grafana
```

**2. Bật Plugin trên Grafana:**
- Đăng nhập vào Grafana (`http://192.168.68.181:3000`).
- Vào **Administration** ➔ **Plugins** ➔ Tìm chữ `Zabbix` ➔ Click vào plugin Alexanderzobnin Zabbix và chọn **Enable**.

**3. Cấu hình Zabbix làm Data Source:**
- Vào **Connections** ➔ **Data sources** ➔ **Add data source**.
- Chọn **Zabbix**.
- Điền các thông số:
  - **URL:** `http://192.168.68.181:8080/api_jsonrpc.php`
  - Ở mục **Zabbix API details**:
    - **Username:** `Admin`
    - **Password:** `zabbix`
- Kéo xuống cuối click **Save & test**. Nếu báo xanh "Zabbix API version: x.x.x" là thành công!

**4. Vẽ biểu đồ lưu lượng mạng:**
Giờ đây khi tạo Dashboard mới trên Grafana, bạn có thể chọn Data Source là Zabbix. Chọn Group: `Routers`, Host: `Cisco-Router-Core`, Item: `Interface e0/0: Bits received` để vẽ ra biểu đồ Băng thông Mạng rực rỡ và chuyên nghiệp.

---

## BƯỚC 5: CẤU HÌNH CẢNH BÁO TELEGRAM (ALERTING)

Để bắn cảnh báo sự cố mạng (Switch rớt mạng, port Down, CPU cao, pfSense mất kết nối) về Telegram tương tự như Prometheus Alertmanager và Wazuh, Zabbix hỗ trợ sẵn **Telegram Webhook** cực kỳ dễ dùng và chuyên nghiệp.

### 5.1. Chuẩn bị Bot Token & Chat ID Telegram
1. **Lấy Bot Token:** Dùng Bot Telegram hiện tại của bạn hoặc tạo mới qua `@BotFather` để lấy **Bot Token** (ví dụ: `123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ`).
2. **Lấy Group Chat ID:**
   - Tạo một Group Telegram (ví dụ: `Net_notification`).
   - Thêm con **Bot Telegram của bạn** VÀ bot **`@IDBot`** vào nhóm.
   - Trong ô chat nhóm, gõ lệnh `/id` ➔ Bot `@IDBot` sẽ trả về **Chat ID của Nhóm** (chuỗi số bắt đầu bằng dấu trừ, ví dụ: `-1001987654321`).

### 5.2. Cấu hình Telegram Media Type trên Zabbix Web
1. Đăng nhập Zabbix Web (`http://192.168.68.181:8080`).
2. Vào **Alerts** ➔ **Media types**.
3. **Bật trạng thái Telegram:** Kiểm tra cột *Status* của dòng **Telegram**, nếu đang là `Disabled` (màu đỏ) thì click trực tiếp vào chữ đó để chuyển sang `Enabled` (màu xanh lá).
4. Nhấp vào tên **Telegram** để sửa thông số trong bảng **Parameters**:
   - `api_token`: Thay thế `<PLACE YOUR TOKEN>` bằng **Bot Token** của bạn.
   - `api_parse_mode`: Thay thế `<PLACE PARSE MODE>` bằng **`html`**.
   - Giữ nguyên `api_chat_id` là `{ALERT.SENDTO}`.
5. Nhấn **Update** để lưu lại.

### 5.3. Gán Telegram Media cho Tài khoản Admin
1. Vào **Users** ➔ **Users** ➔ Nhấp vào tài khoản **Admin**.
2. Chuyển sang tab **Media** ➔ Nhấp nút **Add**:
   - **Type:** Chọn `Telegram`.
   - **Send to:** Nhập `Telegram Group Chat ID` bạn vừa lấy ở bước 5.1 (ví dụ: `-1001987654321`).
   - **When active:** `1-7,00:00-24:00` (nhận cảnh báo 24/7).
   - **Severity:** Tích chọn tất cả các mức độ mong muốn (Warning, Average, High, Disaster).
3. Nhấn **Add** ở cửa sổ nhỏ ➔ Nhấn **Update** ở trang chính để lưu tài khoản Admin.

### 5.4. Kích hoạt Action tự động gửi Telegram
1. Vào **Alerts** ➔ **Actions** ➔ **Trigger actions**.
2. Tìm quy tắc mặc định: **Report problems to Zabbix administrators**.
3. Đảm bảo cột Status của nó đang là **Enabled** (Màu xanh). Nếu là Disabled, hãy nhấp chọn để Bật nó lên.

🎉 **KẾT QUẢ:** Mỗi khi Switch Cisco rút dây, port rớt link hoặc pfSense bị ngắt kết nối, Zabbix sẽ ngay lập tức phát hiện qua SNMP và tự động gửi tin nhắn Telegram đẹp mắt kèm trạng thái `[PROBLEM]` và `[RESOLVED]` khi mạng bình thường trở lại!

### 5.5. Tối ưu tốc độ phát hiện Cảnh báo (Fast Alerting - 5 đến 10 giây)

Mặc định Zabbix cần 3 phút (3 lần ping hỏng liên tiếp) mới phát tin báo động để tránh cảnh báo ảo. Để Zabbix gửi tin về Telegram **siêu nhanh (chỉ sau 5-10 giây)**, bạn cần tùy chỉnh tham số **Update Interval**.

> **💡 Khái niệm Update Interval (Tần suất cập nhật):**
> Là khoảng thời gian nghỉ giữa 2 lần Zabbix Server chủ động gửi gói tin ping / SNMP đi truy vấn thiết bị.
> - `Update interval = 1m`: Cứ 60 giây Zabbix mới ping 1 lần ➔ Phát hiện sự cố trễ tối đa 60 giây.
> - `Update interval = 5s`: Cứ 5 giây Zabbix ping 1 lần ➔ Phát hiện sự cố ngay lập tức!

#### Các bước thực hiện:
1. **Chỉnh tần suất quét ping (Item Update Interval):**
   - Vào **Data collection** ➔ **Hosts** ➔ Click vào **Items** của thiết bị (ví dụ `Core-Switch-Cisco`).
   - Tìm item **ICMP ping** ➔ Sửa ô **Update interval** từ `1m` thành `5s` (5 giây) hoặc `10s`.
   - Nhấn **Update** để lưu.

2. **Chỉnh số lần chờ thất bại (Trigger Expression):**
   - Vào **Data collection** ➔ **Hosts** ➔ Click vào **Triggers** của thiết bị.
   - Click chọn trigger **Unavailable by ICMP ping**.
   - Sửa biểu thức logic từ `max(/.../icmpping,#3)=0` (chờ 3 lần hỏng) thành `last(/.../icmpping)=0` (báo ngay lần đầu hỏng).
   - Nhấn **Update**.

📌 **Lưu ý kinh nghiệm (Best Practice):**
- **Khi bảo vệ Đồ án (Demo):** Nên cấu hình `Update interval = 5s` để khi bạn thao tác ngắt cáp trong EVE-NG, tin nhắn Telegram nổ tức thì trước mặt Hội đồng chấm đồ án.
- **Khi triển khai Thực tế (Production):** Nên để `30s` đến `1m` để tránh làm Zabbix Server quá tải và tiêu tốn băng thông mạng do gửi gói tin liên tục.

### 5.6. Hướng dẫn Tùy biến (Custom) Templates trong Zabbix

Zabbix cho phép bạn tùy biến ở 2 cấp độ: **Mẫu tin nhắn Telegram** và **Template Giám sát Thiết bị**.

#### A. Tùy biến Mẫu tin nhắn Telegram (Message Templates)
Giúp tin nhắn bắn về Telegram đẹp mắt hơn (thêm icon 🚨, 💥, ✅, tiếng Việt, IP thiết bị):
1. Vào **Alerts** ➔ **Media types** ➔ Nhấp vào **Telegram**.
2. Chuyển sang tab **Message templates**.
3. Danh sách các mẫu tin nhắn hiện ra:
   - **Problem:** Mẫu tin nhắn khi phát sinh sự cố mới.
   - **Problem recovery:** Mẫu tin nhắn khi sự cố được khôi phục.
   - **Problem update:** Mẫu tin nhắn khi cập nhật thông tin sự cố.
4. Click nút **Edit** bên phải từng loại để sửa nội dung `Subject` và `Message` (hỗ trợ HTML và các biến Zabbix Macro như `{HOST.NAME}`, `{EVENT.NAME}`, `{EVENT.SEVERITY}`).

#### B. Tùy biến Template Giám sát Thiết bị (Device Templates)
Giúp thêm/bớt chỉ số thu thập (Items) hoặc luật cảnh báo (Triggers) áp dụng cho nhiều thiết bị:
1. Vào **Data collection** ➔ **Templates**.
2. Tìm tên Template (ví dụ: `Cisco IOS SNMP` hoặc `pfSense SNMP`).
3. Tại đây bạn có thể:
   - Click **Items** ➔ Thêm chỉ số OID SNMP mới muốn giám sát.
   - Click **Triggers** ➔ Tạo thêm luật cảnh báo tùy chỉnh.
   - Click **Full clone** ở cuối trang ➔ Nhân bản thành Template mới riêng của bạn (ví dụ: `My Custom Cisco Template`) để tha hồ chỉnh sửa mà không sợ hỏng template mặc định.

---

## 6. CÁC KỊCH BẢN THỬ NGHIỆM CẢNH BÁO MẠNG (DEMO TEST SCENARIOS)

Dưới đây là **6 kịch bản sự cố mạng thực tế** phổ biến nhất mà bạn có thể dùng để dựng kịch bản kiểm thử (Test Cases) hoặc làm bài diễn hoạt khi bảo vệ Đồ án tốt nghiệp:

| STT | Kịch bản Sự cố | Thao tác Giả lập (Lab EVE-NG / pfSense) | Cảnh báo Telegram kỳ vọng |
| :--- | :--- | :--- | :--- |
| **1** | **Mất kết nối Toàn bộ Thiết bị** *(Host Down)* | Tắt nguồn VM pfSense hoặc ngắt kết nối card mạng VMware. | 🔴 `[DISASTER] Core-Switch-Cisco: Unavailable by ICMP ping` |
| **2** | **Rớt Cổng / Đứt Dây Mạng** *(Link Down)* | Vào CLI Switch Cisco: `interface Gi0/1` ➔ `shutdown`. | ⚠️ `[HIGH] Core-Switch-Cisco: Interface GigabitEthernet0/1: Link down` |
| **3** | **Khởi động lại Đột ngột** *(Device Restart)* | Vào CLI Switch Cisco gõ lệnh khởi động lại: `reload`. | 🟡 `[WARNING] Core-Switch-Cisco: System has been restarted (uptime < 10m)` |
| **4** | **Nghẽn Băng thông Mạng** *(High Bandwidth)* | Chạy `iperf3` hoặc tải file lớn liên tục từ Client qua pfSense/Switch. | 🟡 `[AVERAGE] pfSense: High bandwidth usage on interface wan (> 80%)` |
| **5** | **Suy hao Mạng / Mất gói** *(Packet Loss)* | Chuột phải vào dây cáp EVE-NG ➔ **Edit Quality** ➔ Chọn Loss 30%. | 🟡 `[WARNING] Core-Switch-Cisco: High ICMP ping loss (> 20%)` |
| **6** | **Tắt Dịch vụ Core trên Firewall** *(Service Down)* | Trên Web GUI pfSense ➔ **Status** ➔ **Services** ➔ Stop dịch vụ `dhcpd`. | 🔴 `[HIGH] pfSense: DHCP server is not running` |






