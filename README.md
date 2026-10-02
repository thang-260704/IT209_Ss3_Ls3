# Bài 3: Bảo mật tài nguyên bằng HTTP Basic Authentication

## Mục tiêu
- Dùng `htpasswd` (gói `apache2-utils`) để tạo tài khoản với mật khẩu đã được băm.
- Cấu hình Nginx bảo vệ đường dẫn `/admin` bằng **HTTP Basic Authentication**.

## File nộp bài

| File | Vị trí trên máy chủ | Ghi chú |
|---|---|---|
| [`my-web.conf`](my-web.conf) | `/etc/nginx/sites-available/my-web.conf` | Server Block có thêm `location /admin` |
| [`.htpasswd`](.htpasswd) | `/etc/nginx/.htpasswd` | **File mẫu**, xem ghi chú bên dưới |
| [`admin-index.html`](admin-index.html) | `/var/www/my-web/html/admin/index.html` | Trang nội bộ của khu vực admin |

> **Về file `.htpasswd` mẫu:** file nộp lên GitHub được băm từ mật khẩu demo `DemoPass@2026` (thuật toán APR1-MD5, tiền tố `$apr1$`), dùng để minh hoạ định dạng `user:hash`. Mật khẩu thật trên máy chủ là mật khẩu khác và **không** được đưa lên repo công khai. Đây là thực hành bảo mật cơ bản: không bao giờ commit thông tin xác thực thật, kể cả ở dạng đã băm.

## Môi trường

| Thông tin | Giá trị |
|---|---|
| Máy chủ | VPS AZVPS – Ubuntu 22.04.5 LTS |
| IPv4 | `160.187.229.88` |
| User thao tác | `devops` (dùng `sudo`), SSH qua cổng `2222` |
| Tài khoản Basic Auth | `admin_user` |

---

## Các bước thực hiện

### Bước 1 – Cài `apache2-utils`

```bash
sudo apt install apache2-utils -y
```

### Bước 2 – Tạo file mật khẩu `/etc/nginx/.htpasswd`

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin_user
sudo chown root:www-data /etc/nginx/.htpasswd
sudo chmod 640 /etc/nginx/.htpasswd
ls -l /etc/nginx/.htpasswd
```

- `-c` tạo file mới. Lệnh sẽ hỏi mật khẩu 2 lần, và mật khẩu được lưu ở dạng **băm**, không lưu ở dạng thô.
- File được đặt ở `/etc/nginx/` chứ **không** đặt trong `/var/www/`. Như vậy file không bao giờ bị phục vụ ra ngoài như một file tĩnh, nên không có nguy cơ bị tải về qua URL.
- Quyền `640` với `root:www-data`: chỉ root được ghi, còn Nginx (chạy bằng user `www-data`) được đọc. Các user khác trên máy chủ không đọc được.

### Bước 3 – Tạo nội dung cho khu vực `/admin`

```bash
sudo mkdir -p /var/www/my-web/html/admin
sudo nano /var/www/my-web/html/admin/index.html   # nội dung: admin-index.html
```

### Bước 4 – Thêm `location /admin` vào Server Block

```nginx
location /admin {
    auth_basic           "Restricted Admin Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
    try_files $uri $uri/index.html $uri/ =404;
}
```

- `auth_basic` bật xác thực. Chuỗi `"Restricted Admin Area"` là realm, được hiển thị trong header `WWW-Authenticate`.
- `auth_basic_user_file` trỏ tới file chứa danh sách `user:hash`.
- `try_files ... $uri/index.html ...` cho phép `/admin` (không có dấu `/` cuối) trả thẳng `200` với trang `admin/index.html`, thay vì redirect.

### Bước 5 – Kiểm tra và reload

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Kết quả thực tế:

```text
devops@wean2607:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
devops@wean2607:~$ sudo systemctl reload nginx
```

![Cấu hình trên máy chủ](images/01-server-setup.png)

---

## Kiểm tra (máy cá nhân)

```text
C:\Users\ADMIN>curl -I http://160.187.229.88/admin
HTTP/1.1 401 Unauthorized
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 01:01:25 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Restricted Admin Area"

C:\Users\ADMIN>curl -I -u "admin_user:**********" http://160.187.229.88/admin
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 01:03:46 GMT
Content-Type: text/html
Content-Length: 679
Last-Modified: Fri, 02 Oct 2026 00:58:09 GMT
Connection: keep-alive
ETag: "6abf01a1-2a7"
Accept-Ranges: bytes
```

> Mật khẩu thật đã được che bằng `**********` trong log và ảnh chụp.

Phân tích kết quả:
- **Không có thông tin đăng nhập:** server trả về `401 Unauthorized` kèm header `WWW-Authenticate: Basic realm="Restricted Admin Area"`. Header này yêu cầu client gửi thông tin xác thực.
- **Có thông tin đăng nhập đúng:** server trả về `200 OK` cùng trang `admin/index.html` (`Content-Length: 679`, đúng bằng kích thước file `admin-index.html`).

![curl kiểm tra Basic Auth](images/02-curl-test.png)

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| Cài `apache2-utils` | ✅ |
| `.htpasswd` đặt tại `/etc/nginx/`, không đặt trong `/var/www/` | ✅ |
| Tài khoản `admin_user`, mật khẩu được băm | ✅ |
| `location /admin` với `auth_basic` + `auth_basic_user_file` | ✅ |
| Không đăng nhập: `401 Unauthorized` + `WWW-Authenticate` | ✅ |
| Đăng nhập đúng: `200 OK` (không phải 401) | ✅ |
