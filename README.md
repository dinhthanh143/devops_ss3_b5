# Báo cáo: Thiết lập thư mục Web an toàn tại phân vùng hệ thống (/opt/)

## 1. Mục tiêu
- Chuyển toàn bộ mã nguồn web tĩnh và logs sang thư mục tùy chọn `/opt/my-app/` để cô lập dữ liệu.
- Phân quyền tối ưu đảm bảo user `devops` chỉnh sửa mã nguồn không cần `sudo`.
- Tiến trình Nginx (`www-data`) có quyền đọc mã nguồn tại `/opt/my-app/html/` và quyền ghi logs tại `/opt/my-app/logs/`.
- Cấu hình Nginx Server Block ghi nhận `access_log` và `error_log` vào vị trí mới.

---

## 2. Các bước triển khai

### Bước 1: Khởi tạo cấu trúc thư mục
```bash
sudo mkdir -p /opt/my-app/html /opt/my-app/logs
```

Tạo file HTML mẫu ban đầu:
```bash
echo "<h1>Welcome to Secure My App on /opt</h1>" | sudo tee /opt/my-app/html/index.html
```

### Bước 2: Thiết lập quyền sở hữu (Ownership)
Gán quyền sở hữu toàn bộ thư mục `/opt/my-app/` cho user `devops` và group `www-data`:
```bash
sudo chown -R devops:www-data /opt/my-app/
```

### Bước 3: Phân quyền truy cập (Permissions)
- **Thư mục chung (`/opt/my-app` & `/opt/my-app/html`)**: Cấp quyền `750` hoặc `755` (Owner: `rwx`, Group: `r-x`).
- **Tệp tin mã nguồn (`/opt/my-app/html/`)**: Cấp quyền `640` hoặc `644` (Owner: `rw-`, Group: `r--`).
- **Thư mục logs (`/opt/my-app/logs`)**: Cấp quyền `775` (Owner: `rwx`, Group: `rwx`) để Nginx (`www-data`) có quyền tạo và ghi file logs.

```bash
sudo find /opt/my-app -type d -exec chmod 750 {} \;
sudo find /opt/my-app/html -type f -exec chmod 640 {} \;
sudo chmod 775 /opt/my-app/logs
```

### Bước 4: Cấu hình Nginx Server Block
Tạo file cấu hình `/etc/nginx/sites-available/my-app.conf`:
```nginx
server {
    listen 80;
    server_name localhost;

    root /opt/my-app/html;
    index index.html index.htm;

    access_log /opt/my-app/logs/access.log;
    error_log /opt/my-app/logs/error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Kích hoạt cấu hình và reload Nginx:
```bash
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 3. Minh chứng & Kiểm tra

### 3.1. Kiểm tra trạng thái phân quyền
Chạy lệnh kiểm tra phân quyền thư mục `/opt/my-app/`:

```bash
ls -la /opt/my-app/
```

**Output:**
```plaintext
total 16
drwxr-x--- 4 devops www-data 4096 Oct  1 19:40 .
drwxr-xr-x 4 root   root     4096 Oct  1 19:38 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 19:40 html
drwxrwxr-x 2 devops www-data 4096 Oct  1 19:40 logs
```

Kiểm tra chi tiết bên trong `/opt/my-app/html/`:
```bash
ls -la /opt/my-app/html/
```

**Output:**
```plaintext
total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 19:40 .
drwxr-x--- 4 devops www-data 4096 Oct  1 19:40 ..
-rw-r----- 1 devops www-data   43 Oct  1 19:40 index.html
```

---

### 3.2. Kiểm tra thao tác ghi từ user `devops` (Không dùng sudo)
Thực hiện chuyển sang user `devops` và ghi file:

```bash
su - devops
echo "Update Test" >> /opt/my-app/html/index.html
```

**Kiểm tra nội dung tệp tin:**
```bash
cat /opt/my-app/html/index.html
```

**Output:**
```plaintext
<h1>Welcome to Secure My App on /opt</h1>
Update Test
```
*(Thao tác ghi file thành công hoàn toàn mà không cần quyền root).*

---

### 3.3. Kiểm tra hoạt động Web & Ghi nhận Logs tự động
Gửi request truy cập trang web:
```bash
curl -i http://localhost/
```

**Output:**
```plaintext
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 01 Oct 2026 12:42:10 GMT
Content-Type: text/html
Content-Length: 55
Last-Modified: Thu, 01 Oct 2026 12:41:45 GMT
Connection: keep-alive
ETag: "66fbc8a9-37"
Accept-Ranges: bytes

<h1>Welcome to Secure My App on /opt</h1>
Update Test
```

Kiểm tra logs phát sinh tại `/opt/my-app/logs/access.log`:
```bash
tail -n 5 /opt/my-app/logs/access.log
```

**Output:**
```plaintext
127.0.0.1 - - [01/Oct/2026:19:42:10 +0700] "GET / HTTP/1.1" 200 55 "-" "curl/7.88.1"
```

Nginx phản hồi mã trạng thái `200 OK` và tự động ghi log truy cập vào đúng thư mục `/opt/my-app/logs/`.
