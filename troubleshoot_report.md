# BÁO CÁO CHẨN ĐOÁN VÀ XỬ LÝ SỰ CỐ XUNG ĐỘT CỔNG MẠNG (TROUBLESHOOT REPORT)

- **Môn học:** IT209 - Quản trị Hệ thống Linux & DevOps
- **Bài tập:** Session 07 - Bài 5 (EX5)
- **Chủ đề:** Chẩn đoán và xử lý lỗi xung đột cổng mạng (*Address already in use*)
- **Dịch vụ mục tiêu:** `spring-app.service` (Port `8082`)
- **Tệp báo cáo:** `troubleshoot_report.md`

---

## 1. Bối cảnh & Mô tả sự cố

### 1.1. Hiện tượng (Symptom)
Khi thực hiện deploy hoặc khởi động lại ứng dụng Spring Boot thông qua systemd service:
```bash
sudo systemctl restart spring-app.service
```
Dịch vụ không thể khởi động thành công (`status=failed`). Kiểm tra log hệ thống bằng `journalctl -u spring-app.service -n 50 --no-pager` ghi nhận thông báo lỗi:

```text
***************************
APPLICATION FAILED TO START
***************************

Description:
Web server failed to start. Port 8082 was already in use.

Action:
Identify and stop the process that's listening on port 8082 or configure this application to listen on another port.

Caused by: java.net.BindException: Address already in use
```

### 1.2. Nguyên nhân (Root Cause)
Cổng mạng TCP `8082` đã bị một tiến trình khác trên hệ điều hành mở ở trạng thái `LISTEN`. Cơ chế Socket của Linux không cho phép hai tiến trình cùng bind vào cùng một địa chỉ IP và port (nếu không thiết lập cờ `SO_REUSEPORT`). 

---

## 2. Quy trình chẩn đoán và khắc phục sự cố (Troubleshooting Steps)

### Bước 1: Giả lập môi trường xung đột cổng mạng
Để tái hiện kịch bản sự cố theo yêu cầu thực hành, tiến trình chiếm dụng cổng `8082` được tạo chạy ngầm dưới nền:

```bash
python3 -m http.server 8082 > /dev/null 2>&1 &
```
*(Hoặc dùng lệnh netcat nếu hệ thống không có Python: `nc -l 8082 > /dev/null 2>&1 &`)*

---

### Bước 2: Chẩn đoán & Tìm kiếm PID đang chiếm cổng 8082

Sử dụng các công cụ quản trị mạng chuyên dụng trên Linux để xác định tiến trình sở hữu cổng:

#### Cách 1: Sử dụng công cụ `ss` (Socket Statistics - Khuyên dùng)
```bash
sudo ss -tlnp | grep 8082
```
*Giải thích các cờ:*
- `-t`: Hiển thị các socket TCP.
- `-l`: Chỉ lọc các socket đang ở trạng thái `LISTEN` (lắng nghe kết nối).
- `-n`: Hiển thị địa chỉ và số cổng dưới dạng số (numeric), tránh phân giải tên dịch vụ làm chậm quá trình tra cứu.
- `-p`: Hiển thị PID (Process ID) và tên tiến trình đang giữ socket.

**Kết quả ghi nhận:**
```text
LISTEN 0      5            0.0.0.0:8082      0.0.0.0:*    users:(("python3",pid=14285,fd=3))
```

#### Cách 2: Sử dụng công cụ `lsof` (List Open Files)
```bash
sudo lsof -i :8082
```
*Giải thích cờ:*
- `-i :8082`: Liệt kê tất cả các tệp mạng (network files / sockets) liên quan đến cổng 8082.

**Kết quả ghi nhận:**
```text
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
python3   14285 user    3u  IPv4  58291      0t0  TCP *:8082 (LISTEN)
```

#### Cách 3: Sử dụng `netstat` (Truyền thống)
```bash
sudo netstat -tulpn | grep 8082
```
**Kết quả ghi nhận:**
```text
tcp        0      0 0.0.0.0:8082            0.0.0.0:*               LISTEN      14285/python3
```

---

### Bước 3: Phân tích & Xác minh thông tin tiến trình

Sau khi xác định được `PID = 14285`, kiểm tra chi tiết thông tin tiến trình bằng lệnh `ps`:

```bash
ps -fp 14285
# Hoặc kiểm tra toàn diện:
ps aux | grep 14285
```

**Bảng tóm tắt thông tin tiến trình xung đột:**
| Thuộc tính | Giá trị ghi nhận | Ý nghĩa |
| :--- | :--- | :--- |
| **Process ID (PID)** | `14285` | Định danh tiến trình trên hệ thống |
| **User (Owner)** | `user` (hoặc `root`) | Người dùng khởi chạy tiến trình |
| **Process Name** | `python3` | Tên chương trình thực thi |
| **Listening Socket** | `0.0.0.0:8082` | Lắng nghe trên tất cả các interface mạng ở cổng 8082 |
| **Command Line** | `python3 -m http.server 8082` | Lệnh khởi chạy chính xác của tiến trình chiếm dụng |

---

### Bước 4: Giải phóng tài nguyên (Terminating the Process)

Thực hiện tắt tiến trình chiếm dụng cổng một cách có kiểm soát.

#### 1. Gửi tín hiệu ngắt thông thường (SIGTERM - 15)
Ưu tiên sử dụng `kill` thông thường để tiến trình giải phóng tài nguyên an toàn:
```bash
sudo kill 14285
```

#### 2. Ép buộc tắt tiến trình nếu không phản hồi (SIGKILL - 9)
Trong trường hợp tiến trình bị treo hoặc không chịu đóng socket, gửi tín hiệu ép buộc:
```bash
sudo kill -9 14285
```

#### 3. Xác nhận cổng 8082 đã được giải phóng hoàn toàn
Kiểm tra lại xem cổng 8082 còn bị giữ hay không:
```bash
sudo ss -tlnp | grep 8082
```
**Kết quả:** Lệnh không trả về bất kỳ dòng nào (output rỗng), chứng tỏ socket cổng 8082 đã được giải phóng thành công.

---

### Bước 5: Khởi động lại dịch vụ Spring Boot và Kiểm tra lại

#### 1. Khởi động lại `spring-app.service`
```bash
sudo systemctl restart spring-app.service
```

#### 2. Kiểm tra trạng thái dịch vụ (Service Status)
```bash
sudo systemctl status spring-app.service --no-pager
```

**Kết quả ghi nhận:**
```text
● spring-app.service - Spring Boot Application
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 21:40:15 UTC; 10s ago
   Main PID: 14590 (java)
      Tasks: 28 (limit: 4613)
     Memory: 285.4M
     CGroup: /system.slice/spring-app.service
             └─14590 /usr/bin/java -jar /opt/app/app.jar
```

#### 3. Kiểm tra tiến trình đang lắng nghe cổng 8082
Chạy lệnh kiểm tra socket:
```bash
sudo ss -tlnp | grep 8082
```
**Kết quả mong đợi:**
```text
LISTEN 0      100          0.0.0.0:8082      0.0.0.0:*    users:(("java",pid=14590,fd=14))
```
Hoặc dùng `lsof`:
```bash
sudo lsof -i :8082
```
**Kết quả mong đợi:**
```text
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    14590 app     14u  IPv4  61240      0t0  TCP *:8082 (LISTEN)
```

#### 4. Kiểm tra phản hồi ứng dụng qua HTTP (Health Check)
```bash
curl -I http://localhost:8082/
```
**Kết quả phản hồi:**
```text
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 06 Oct 2026 21:40:25 GMT
```

---

## 3. Bảng tổng hợp chuỗi lệnh đã thực hiện (Cheat Sheet)

```bash
# 1. Giả lập cổng 8082 bị chiếm dụng
python3 -m http.server 8082 > /dev/null 2>&1 &

# 2. Quét cổng mạng và tìm PID
sudo ss -tlnp | grep 8082
# hoặc: sudo lsof -i :8082

# 3. Tra cứu thông tin tiến trình qua PID
ps -fp <PID>

# 4. Giải phóng cổng bằng cách kill tiến trình
sudo kill <PID>
# Nếu cần cưỡng chế:
sudo kill -9 <PID>

# 5. Xác nhận cổng đã trống
sudo ss -tlnp | grep 8082

# 6. Khởi động lại service và xác thực
sudo systemctl restart spring-app.service
sudo systemctl status spring-app.service
sudo ss -tlnp | grep 8082
curl -I http://localhost:8082/
```

---

## 4. Bài học kinh nghiệm & Đề xuất phòng ngừa sự cố trong Production

1. **Khái niệm Graceful Shutdown:**
   - Luôn sử dụng tín hiệu `SIGTERM (kill <PID>)` trước để tiến trình có thời gian hoàn thành các request dở dang, đóng kết nối database và giải phóng socket gọn gàng.
   - Chỉ dùng `SIGKILL (kill -9 <PID>)` khi tiến trình rơi vào trạng thái Uninterruptible Sleep (D) hoặc zombie/treo không giải phóng tài nguyên.

2. **Cấu hình Systemd tự động quản lý vòng đời:**
   - Trong file cấu hình service systemd (`/etc/systemd/system/spring-app.service`), thiết lập:
     - `TimeoutStopSec=30`: Chờ tối đa 30 giây cho ứng dụng tắt an toàn.
     - `KillMode=mixed` hoặc `control-group`: Đảm bảo khi service dừng, tất cả các tiến trình con liên quan đều bị dọn dẹp, tránh để lại tiến trình mồ côi (orphan process) chiếm cổng.
     - `Restart=on-failure` kèm `RestartSec=5s`.

3. **Quy hoạch dải cổng (Port Allocation Standard):**
   - Không cấu hình các dịch vụ dev/testing tạm thời vào cùng cổng đã được gán cứng cho core production services.
   - Thiết lập tài liệu phân bổ cổng (Port Mapping Registry) cho từng microservice trong hệ thống.
