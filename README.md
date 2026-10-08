# Báo Cáo Thực Hành: Thiết Lập Cơ Sở Dữ Liệu & Cấu Hình Dịch Vụ Systemd Cho Spring Boot

- **Bài tập**: Bài 3 - Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot (Session 07 / Systemd Service & MySQL Database Integration)
- **Đường dẫn nộp bài**: `SS07/Bai03/README.md` (hoặc `homework/session_07/ex3/`)
- **Tệp đính kèm**: `spring-app.service`, `README.md`

---

## 1. Bối cảnh & Mục tiêu

### Bối cảnh
Trong môi trường Production, ứng dụng Spring Boot không được chạy thủ công qua lệnh `java -jar` trên Terminal cá nhân của quản trị viên. Việc chạy thủ công dễ khiến ứng dụng bị ngừng hoạt động khi đóng cửa sổ SSH, thiếu cơ chế tự khởi động lại khi crash và vi phạm nguyên tắc an toàn thông tin nếu chạy dưới tài khoản `root`.

### Mục tiêu
- Khởi tạo cơ sở dữ liệu `springboot_db` và người dùng `spring-admin` trên hệ quản trị MySQL với mật khẩu mạnh.
- Khởi tạo người dùng hệ thống Linux `spring-runner` (không có shell đăng nhập `/bin/false`) để thực thi tiến trình ứng dụng theo nguyên tắc đặc quyền tối thiểu.
- Thiết kế tệp tin cấu hình dịch vụ Systemd `/etc/systemd/system/spring-app.service` tự động hóa quản lý vòng đời tiến trình.
- Thiết lập cơ chế tự động khôi phục dịch vụ khi gặp sự cố crash bất ngờ (`Restart=on-failure`, `RestartSec=10`).
- Kiểm tra trạng thái dịch vụ và cổng lắng nghe `8082` bằng công cụ `systemctl` và `ss`.

---

## 2. Thông Số Cấu Hình Hệ Thống

| Thông số | Giá trị | Ghi chú |
| :--- | :--- | :--- |
| **MySQL Database** | `springboot_db` | Cơ sở dữ liệu cho ứng dụng Spring Boot |
| **MySQL User** | `spring-admin` | Mật khẩu: `SpringSecure@123` (Toàn quyền trên DB) |
| **Linux System User** | `spring-runner` | User hệ thống không có Shell Login (`/bin/false`) |
| **App Binary Path** | `/opt/spring-app/app.jar` | Tệp tin thực thi Spring Boot Executable JAR |
| **Systemd Service File** | `/etc/systemd/system/spring-app.service` | Đơn vị quản lý dịch vụ hệ thống |
| **Application Port** | `8082/tcp` | Cổng dịch vụ lắng nghe REST API |

---

## 3. Các bước thực hiện chi tiết

### Bước 1: Khởi tạo Cơ sở dữ liệu và Phân quyền người dùng trong MySQL

Đăng nhập vào MySQL Console qua quyền `root`:

```bash
sudo mysql -u root -p
```

Thực thi các câu lệnh SQL khởi tạo Database và User:

```sql
-- 1. Tạo cơ sở dữ liệu springboot_db
CREATE DATABASE IF NOT EXISTS springboot_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 2. Tạo người dùng spring-admin với mật khẩu bảo mật
CREATE USER IF NOT EXISTS 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';

-- 3. Cấp toàn quyền trên cơ sở dữ liệu springboot_db cho spring-admin
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';

-- 4. Áp dụng thay đổi phân quyền
FLUSH PRIVILEGES;

-- 5. Thoát khỏi MySQL Console
EXIT;
```

---

### Bước 2: Khởi tạo người dùng hệ thống `spring-runner` và chuẩn bị thư mục ứng dụng

Tạo tài khoản Linux giới hạn `spring-runner` không cho phép login shell:

```bash
# 1. Tạo người dùng hệ thống spring-runner không có home folder và không có shell login
sudo useradd -r -s /bin/false spring-runner

# 2. Tạo thư mục chứa ứng dụng
sudo mkdir -p /opt/spring-app

# 3. Phân quyền sở hữu thư mục cho spring-runner
sudo chown -R spring-runner:spring-runner /opt/spring-app
sudo chmod 755 /opt/spring-app
```

---

### Bước 3: Viết tệp tin cấu hình dịch vụ Systemd (`/etc/systemd/system/spring-app.service`)

Tạo tệp cấu hình `/etc/systemd/system/spring-app.service`:

```bash
sudo nano /etc/systemd/system/spring-app.service
```

Nội dung tệp tin cấu hình:

```ini
[Unit]
Description=Spring Boot Application Service
After=network.target mysql.service
Wants=mysql.service

[Service]
User=spring-runner
Group=spring-runner
Type=simple
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=spring-app

[Install]
WantedBy=multi-user.target
```

---

### Bước 4: Nạp lại Systemd Daemon, Kích hoạt và Khởi động Dịch vụ

```bash
# 1. Nạp lại thông tin cấu hình systemd
sudo systemctl daemon-reload

# 2. Cho phép dịch vụ tự động khởi động cùng hệ thống
sudo systemctl enable spring-app.service

# 3. Khởi động dịch vụ Spring Boot
sudo systemctl start spring-app.service
```

---

## 4. Trích Xuất Log Terminal Chứng Minh (Verification Logs)

### 1. Kiểm tra trạng thái dịch vụ (`sudo systemctl status spring-app.service`)

```console
$ sudo systemctl status spring-app.service
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-10-08 15:05:00 UTC; 2min 30s ago
   Main PID: 14520 (java)
      Tasks: 28 (limit: 1115)
     Memory: 185.4M
        CPU: 4.821s
     CGroup: /system.slice/spring-app.service
             └─14520 /usr/bin/java -jar /opt/spring-app/app.jar

Oct 08 15:05:00 ubuntu-server systemd[1]: Started spring-app.service - Spring Boot Application Service.
Oct 08 15:05:03 ubuntu-server spring-app[14520]:  :: Spring Boot ::                (v3.2.0)
Oct 08 15:05:05 ubuntu-server spring-app[14520]: Started Application in 4.123 seconds (process running for 4.8)
```
> **Đánh giá**: Dịch vụ `spring-app.service` hiển thị trạng thái `active (running)`, hoạt động dưới quyền PID `14520` của lệnh `/usr/bin/java -jar /opt/spring-app/app.jar`.

---

### 2. Kiểm tra cổng lắng nghe của ứng dụng (`ss -tlnp | grep 8082`)

```console
$ ss -tlnp | grep 8082
LISTEN 0      100          *:8082            *:*    users:(("java",pid=14520,fd=14))
```
> **Đánh giá**: Cổng `8082` đang ở trạng thái `LISTEN`, được quản lý bởi tiến trình Java của user `spring-runner`.

---

## 5. Kết luận

- Khởi tạo thành công Cơ sở dữ liệu `springboot_db` và phân quyền cho `spring-admin` trên MySQL.
- Tự thiết kế và triển khai tệp tin dịch vụ Systemd `spring-app.service` chuẩn công nghiệp.
- Ứng dụng vận hành an toàn dưới quyền user giới hạn `spring-runner`, tự động lắng nghe cổng `8082` và sẵn sàng tự khôi phục sau 10 giây khi gặp sự cố crash.
