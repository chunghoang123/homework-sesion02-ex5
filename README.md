# Bài 5: Phân quyền thư mục Web (chown & chmod)

## 1. Mục tiêu
- User `devops` sửa được mã nguồn mà không cần `sudo`.
- Nginx (chạy dưới user/group `www-data`) vẫn đọc được web, không lỗi 403.

## 2. Hiện trạng ban đầu
```text
$ ls -la /var/www/ptit-web/
drwxr-xr-x 3 www-data www-data ... .
drwxr-xr-x 2 www-data www-data ... html

$ ls -la /var/www/ptit-web/html/
-rwxr-xr-x 1 www-data www-data ... index.html
```
Toàn bộ thuộc `www-data` -> `devops` không ghi được nếu thiếu `sudo`.

## 3. Lệnh đã chạy (bằng user devops + sudo một lần duy nhất)

```bash
# Đổi owner -> devops, group -> www-data (đệ quy)
sudo chown -R devops:www-data /var/www/ptit-web

# Thư mục: 755 (devops rwx, www-data + other r-x để Nginx traverse)
sudo find /var/www/ptit-web -type d -exec chmod 755 {} +

# File: 644 (devops rw-, www-data + other r-- để Nginx đọc)
sudo find /var/www/ptit-web -type f -exec chmod 644 {} +
```

Kết quả `ls -la`:
```text
$ ls -la /var/www/ptit-web/
drwxr-xr-x 3 devops www-data ... .
drwxr-xr-x 2 devops www-data ... html

$ ls -la /var/www/ptit-web/html/
-rw-r--r-- 1 devops www-data ... index.html
```

Giải thích quyền `-rw-r--r--`:
- Owner `devops`: `rw-` (đọc + ghi, không cần sudo).
- Group `www-data`: `r--` (Nginx đọc được file tĩnh).
- Other: `r--` (cho phép đọc công khai, đúng tính chất web tĩnh; nếu muốn kín hơn dùng 640/750 thì Nginx vẫn chạy vì nó thuộc group www-data).

## 4. Kiểm tra (đăng nhập user devops, KHÔNG dùng sudo)

```bash
devops@test:~$ whoami
devops
devops@test:~$ echo "Update" >> /var/www/ptit-web/html/index.html && echo WRITE_OK
WRITE_OK
devops@test:~$ tail -3 /var/www/ptit-web/html/index.html
</body>
</html>
Update
```

Chú ý: lệnh trên chạy với `BatchMode=yes` (SSH key, không password, không sudo) mà vẫn `WRITE_OK` -> chứng minh `devops` đã có quyền ghi.

Kiểm tra Nginx không 403:
```text
$ curl -s -o /dev/null -w 'LOCAL:%{http_code}\n' http://localhost/
LOCAL:200
$ curl http://221.121.1.244/  (từ máy cá nhân)
EXT:200 + nội dung mới có dòng "Update"
```

=> **Đạt**: ghi file thành công bằng user thường, web hiển thị nội dung mới, HTTP 200 (không 403).

## 5. File nộp
```
homework/session_02/ex5/
  README.md
```
