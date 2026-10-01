# BÀI THỰC HÀNH: THIẾT LẬP THƯ MỤC WEB AN TOÀN TẠI PHÂN VÙNG HỆ THỐNG (/OPT/)
## Môn học: DevOps (IT209) - Session 03 - Bài 5
**Mã sinh viên / Lớp:** PTIT_CNTT1_IT209  
**Đường dẫn nộp bài GitHub:** `homework/session_03/ex5/`

---

## 1. Mục tiêu bài thực hành
- **Cô lập môi trường Production**: Chuyển vị trí lưu trữ mã nguồn web tĩnh và nhật ký truy cập (logs) ra khỏi các thư mục mặc định của hệ thống (`/var/www/`, `/var/log/nginx/`) sang phân vùng `/opt/` (thư mục tiêu chuẩn dành cho các ứng dụng và phần mềm bổ sung độc lập).
- **Phân quyền nâng cao (Advanced Permissions)**:
  - Phân tách vai trò rõ ràng giữa tài khoản quản trị/phát triển thông thường (`devops`) và tiến trình vận hành nền của Web Server Nginx (`www-data`).
  - Cấp quyền đọc/ghi toàn diện cho user `devops` trên mã nguồn mà không cần đặc quyền `root`/`sudo`.
  - Cấp quyền đọc mã nguồn và quyền ghi nhật ký (Write log) cho nhóm `www-data` tại thư mục `/opt/my-app/logs/`.
- **Cấu hình Nginx Server Block tùy biến**: Định tuyến website và ghi nhận `access_log`, `error_log` trực tiếp vào đường dẫn chuyên biệt.

---

## 2. Bối cảnh & Yêu cầu kỹ thuật

### 2.1. Bối cảnh
Để chuẩn bị đưa ứng dụng vào môi trường Production thực tế, hệ thống yêu cầu cô lập hoàn toàn mã nguồn web và cấu hình logs của dự án ra khỏi phân vùng mặc định của hệ điều hành.

### 2.2. Ràng buộc hệ thống
- Tạo cấu trúc thư mục ứng dụng:
  - Thư mục chứa mã nguồn: `/opt/my-app/html/`
  - Thư mục lưu trữ logs: `/opt/my-app/logs/`
- Quyền sở hữu: Toàn bộ cây thư mục `/opt/my-app/` thuộc quyền sở hữu của user `devops` và group `www-data`.
- Phân quyền truy cập:
  - Thư mục cha và thư mục web tĩnh: `755` (`rwxr-xr-x`), tệp tin web tĩnh: `644` (`rw-r--r--`).
  - Thư mục logs: `775` (`rwxrwxr-x`) để tiến trình `www-data` có quyền tạo và ghi dữ liệu nhật ký.
- Cấu hình Nginx: Khai báo đầy đủ directive `root`, `access_log`, `error_log` trỏ chính xác về phân vùng `/opt/my-app/`.

---

## 3. Các bước triển khai chi tiết

### Bước 1: Khởi tạo cấu trúc thư mục và tệp tin mã nguồn
Thực hiện tạo thư mục và tệp tin trang chủ `index.html`:

```bash
mkdir -p /opt/my-app/html /opt/my-app/logs
echo "<h1>Welcome to Production Web at /opt/my-app</h1>" > /opt/my-app/html/index.html
```

---

### Bước 2: Thiết lập quyền sở hữu và phân quyền truy cập nâng cao
Tiến hành gán quyền sở hữu đệ quy và cấu hình phân quyền chi tiết cho thư mục ứng dụng và thư mục logs:

```bash
# Gán quyền sở hữu cho user devops và group www-data
chown -R devops:www-data /opt/my-app

# Phân quyền thư mục chuẩn (755) và file chuẩn (644)
find /opt/my-app -type d -exec chmod 755 {} \;
find /opt/my-app -type f -exec chmod 644 {} \;

# Cấp quyền Ghi (Write) cho group www-data tại thư mục logs
chmod 775 /opt/my-app/logs
```

**Bảng ma trận phân quyền chi tiết:**

| Đường dẫn | Sở hữu (Owner:Group) | Quyền Octal | Quyền Ký hiệu | Ý nghĩa thực tế |
|---|---|---|---|---|
| `/opt/my-app/` | `devops:www-data` | `755` | `rwxr-xr-x` | Thư mục gốc ứng dụng, mọi tiến trình có thể truy cập qua |
| `/opt/my-app/html/` | `devops:www-data` | `755` | `rwxr-xr-x` | Thư mục web, `devops` có quyền ghi/sửa, `www-data` có quyền đọc |
| `/opt/my-app/html/index.html` | `devops:www-data` | `644` | `rw-r--r--` | Tệp web tĩnh, `devops` sửa không cần sudo, Nginx đọc phục vụ web |
| `/opt/my-app/logs/` | `devops:www-data` | `775` | `rwxrwxr-x` | Thư mục log, cho phép nhóm `www-data` tạo và ghi file log |

---

### Bước 3: Cấu hình Server Block của Nginx
Tạo tệp cấu hình `/etc/nginx/sites-available/my-app` với nội dung trỏ về thư mục `/opt/`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /opt/my-app/html;
    index index.html;

    access_log /opt/my-app/logs/access.log;
    error_log /opt/my-app/logs/error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Kích hoạt cấu hình mới và tắt các cấu hình mặc định cũ:

```bash
# Tạo symlink kích hoạt site
ln -sf /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/

# Vô hiệu hóa cấu hình mặc định cũ
rm -f /etc/nginx/sites-enabled/default
rm -f /etc/nginx/sites-enabled/ptit-web

# Kiểm tra cú pháp cấu hình Nginx
nginx -t

# Tải lại cấu hình Nginx
systemctl reload nginx
```

---

## 4. Minh chứng thực tế & Kết quả kiểm thử (Verification & Logs)

Dưới đây là toàn bộ nhật ký thực thi thực tế trên máy chủ (`root@ptit-web-devops`):

### 4.1. Nhật ký tạo thư mục và phân quyền
```bash
root@ptit-web-devops:~# id devops || adduser --gecos "" devops
uid=1000(devops) gid=1000(devops) groups=1000(devops),27(sudo),100(users)
root@ptit-web-devops:~# mkdir -p /opt/my-app/html /opt/my-app/logs
root@ptit-web-devops:~# echo "<h1>Welcome to Production Web at /opt/my-app</h1>" > /opt/my-app/html/index.html
root@ptit-web-devops:~# chown -R devops:www-data /opt/my-app
root@ptit-web-devops:~# find /opt/my-app -type d -exec chmod 755 {} \;
root@ptit-web-devops:~# find /opt/my-app -type f -exec chmod 644 {} \;
root@ptit-web-devops:~# chmod 775 /opt/my-app/logs
```

### 4.2. Kiểm tra trạng thái phân quyền (`ls -la /opt/my-app/`)
```bash
root@ptit-web-devops:~# ls -la /opt/my-app/
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  1 10:23 .
drwxr-xr-x 3 root   root     4096 Oct  1 10:23 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 10:23 html
drwxrwxr-x 2 devops www-data 4096 Oct  1 10:23 logs
```

**Nhận xét kết quả:**
- Thư mục `html/` có quyền `drwxr-xr-x` (`755`), thuộc sở hữu của `devops:www-data`.
- Thư mục `logs/` có quyền `drwxrwxr-x` (`775`), thuộc sở hữu của `devops:www-data`.

---

### 4.3. Kiểm tra cú pháp và kích hoạt Nginx
```bash
root@ptit-web-devops:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
root@ptit-web-devops:~# systemctl reload nginx
```

---

### 4.4. Kiểm tra ghi file bằng tài khoản thường `devops` (Không dùng `sudo`)
Chuyển phiên làm việc sang user `devops` và thực hiện ghi nối thêm dữ liệu vào file `index.html`:

```bash
root@ptit-web-devops:~# su - devops -c 'echo "Update Test" >> /opt/my-app/html/index.html'
root@ptit-web-devops:~# cat /opt/my-app/html/index.html
<h1>Welcome to Production Web at /opt/my-app</h1>
Update Test
```

**Nhận xét kết quả:**
- User `devops` ghi file thành công mà không nhận bất kỳ thông báo lỗi quyền (`Permission denied`).
- Nội dung tệp tin được cập nhật chính xác dòng `Update Test`.

---

### 4.5. Kiểm tra truy cập Web Server và kiểm tra Log phát sinh
Gửi yêu cầu HTTP tới máy chủ Nginx qua lệnh `curl`:

```bash
root@ptit-web-devops:~# curl http://localhost/
<h1>Welcome to Production Web at /opt/my-app</h1>
Update Test
```

Kiểm tra nội dung tệp log truy cập `/opt/my-app/logs/access.log`:

```bash
root@ptit-web-devops:~# cat /opt/my-app/logs/access.log
::1 - - [01/Oct/2026:10:29:54 +0700] "GET / HTTP/1.1" 200 62 "-" "curl/8.5.0"

root@ptit-web-devops:~# tail -f /opt/my-app/logs/access.log
::1 - - [01/Oct/2026:10:29:54 +0700] "GET / HTTP/1.1" 200 62 "-" "curl/8.5.0"
```

**Nhận xét kết quả:**
- Mã phản hồi `HTTP 200` hiển thị đầy đủ nội dung HTML mới.
- Tiến trình Nginx tự động sinh file `access.log` và ghi log thành công vào `/opt/my-app/logs/access.log` nhờ quyền `775` của group `www-data`.
- Không có lỗi `403 Forbidden` hay lỗi phân quyền ghi log.

---

## 5. Tổng kết & Đánh giá kết quả

| Hạng mục kiểm tra | Yêu cầu đề bài | Kết quả thực tế | Trạng thái |
|---|---|---|---|
| **Vị trí lưu trữ** | Chuyển sang `/opt/my-app/` (`html/` và `logs/`) | Đã tạo và cấu hình đúng tại `/opt/my-app/` | Đạt yêu cầu |
| **Quyền sở hữu (chown)** | Thuộc về `devops:www-data` | `drwxr-xr-x` devops www-data | Đạt yêu cầu |
| **Phân quyền truy cập (chmod)** | `755`/`644` cho web, `775` cho logs | Thư mục: `755`, File: `644`, Logs: `775` | Đạt yêu cầu |
| **Thao tác ghi của user devops** | Sửa file không cần sudo | Ghi `Update Test` thành công | Đạt yêu cầu |
| **Hoạt động của Nginx** | Không lỗi 403, phục vụ đúng mã nguồn | Trả về HTTP 200 OK với đầy đủ nội dung | Đạt yêu cầu |
| **Ghi nhận nhật ký (Logging)** | Tự động ghi vào `/opt/my-app/logs/` | Log truy cập ghi nhận chuẩn xác tại `access.log` | Đạt yêu cầu |

---

## 6. Bài học kinh nghiệm & Tiêu chuẩn DevOps
1. **Cô lập ứng dụng**: Việc tách mã nguồn sang `/opt/` giúp hệ thống quản lý các ứng dụng độc lập tốt hơn, tránh xung đột với các gói hệ thống mặc định của Ubuntu.
2. **Quyền ghi Log tối thiểu**: Cung cấp quyền ghi cho nhóm `www-data` tại duy nhất thư mục `logs/` (`775`), trong khi thư mục mã nguồn `html/` chỉ cho phép `www-data` quyền đọc (`755`/`644`), giúp ngăn chặn triệt để nguy cơ tin tặc chèn mã độc (webshell) trực tiếp vào thư mục web thông qua lỗi của web server.
