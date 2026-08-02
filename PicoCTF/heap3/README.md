# picoCTF - heap3 Writeup

## Thông tin bài

- **Tên bài:** heap3
- **Mảng kiến thức:** Binary Exploitation / Heap Exploitation
- **Flag:** `picoCTF{now_thats_free_real_estate_79173b73}`

---

## Mục tiêu

Chương trình cho phép thao tác với vùng nhớ trên heap. Nhiệm vụ là lợi dụng lỗi quản lý bộ nhớ để làm cho chương trình in ra flag.

Ý tưởng chính của bài là khai thác lỗi **use-after-free**: chương trình vẫn sử dụng lại con trỏ sau khi vùng nhớ đã bị `free()`.

---

## Kiến thức cần biết

### Heap là gì?

Heap là vùng nhớ dùng cho cấp phát động, thường được dùng qua các hàm như:

```c
malloc(size);
free(ptr);
```

Khi gọi `malloc`, chương trình xin một vùng nhớ trên heap. Khi gọi `free`, vùng nhớ đó được trả lại cho bộ cấp phát, nhưng con trỏ cũ vẫn có thể còn tồn tại trong chương trình.

Nếu chương trình tiếp tục dùng con trỏ đó sau khi đã `free`, ta có lỗi **use-after-free**.

---

## Phân tích lỗi

Trong bài heap3, chương trình có logic tương tự như sau:

```c
char *x = malloc(...);
char *input_data = malloc(...);

free(x);
free(input_data);

char *flag = malloc(...);
```

Sau khi `x` và `input_data` bị giải phóng, chương trình vẫn còn thao tác với các vùng nhớ liên quan.

Điểm quan trọng là sau khi một chunk bị `free`, nếu ta gọi `malloc` lại với kích thước tương tự, bộ cấp phát heap có thể trả lại chính chunk vừa bị giải phóng.

Vì vậy ta có thể điều khiển dữ liệu mới ghi vào vùng nhớ cũ.

---

## Ý tưởng khai thác

Ta cần làm cho vùng nhớ chứa dữ liệu ta nhập vào trùng với vùng nhớ mà chương trình dùng để kiểm tra hoặc in flag.

Quy trình khai thác tổng quát:

1. Cấp phát các chunk ban đầu.
2. Giải phóng một số chunk.
3. Cấp phát lại chunk mới có cùng kích thước.
4. Do heap tái sử dụng chunk đã `free`, dữ liệu mới của ta sẽ nằm đè lên vùng nhớ cũ.
5. Khi chương trình kiểm tra điều kiện, ta đã điều khiển được dữ liệu nên có thể lấy flag.

Đây là lý do tên flag có cụm **free real estate**: vùng nhớ đã `free` nhưng ta vẫn tận dụng lại được.

---

## Quan sát heap

Khi chương trình in heap, ta thấy các chunk nằm gần nhau trên heap. Ví dụ dạng:

```text
Address       -> Value
0x...         -> dữ liệu chunk 1
0x...         -> dữ liệu chunk 2
```

Sau khi `free`, chunk không biến mất ngay. Nó được đưa vào danh sách chunk trống của allocator.

Nếu sau đó ta `malloc` một chunk có kích thước phù hợp, allocator thường lấy lại chunk vừa được `free` trước đó.

Điều này tạo điều kiện để ta ghi dữ liệu vào vị trí mà chương trình không còn kiểm soát đúng cách.

---

## Khai thác

Với bài này, ta chỉ cần thao tác menu theo đúng thứ tự để lợi dụng việc cấp phát lại chunk.

Kịch bản khai thác thường là:

```text
1. Free chunk cũ
2. Allocate chunk mới
3. Ghi dữ liệu mong muốn vào chunk mới
4. Gọi chức năng in flag
```

Nếu viết bằng `pwntools`, script có thể có dạng:

```python
from pwn import *

HOST = "saturn.picoctf.net"
PORT = 12345  # thay bằng port thật của bài

# Chạy local:
# p = process("./chall")

# Chạy remote:
p = remote(HOST, PORT)

def choose(choice):
    p.sendlineafter(b"Enter your choice:", str(choice).encode())

def write_data(data):
    choose(2)
    p.sendlineafter(b"Data for buffer:", data)

# Các bước menu tùy theo binary thực tế của bài heap3.
# Ý tưởng là lợi dụng chunk đã free rồi malloc lại để ghi đè dữ liệu quan trọng.

# Ví dụ:
# choose(<free option>)
# choose(<malloc/write option>)
# write_data(b"payload")
# choose(<print flag option>)

p.interactive()
```

Lưu ý: phần `HOST`, `PORT` và số thứ tự menu cần thay bằng thông tin thật khi chạy challenge trên picoCTF.

---

## Kết quả

Sau khi khai thác thành công, chương trình in ra flag:

```text
picoCTF{now_thats_free_real_estate_79173b73}
```

---

## Tổng kết

Bài heap3 giúp luyện các ý chính sau:

- `free()` không xóa sạch dữ liệu ngay lập tức.
- Con trỏ sau khi `free()` vẫn còn giá trị địa chỉ cũ.
- Dùng lại con trỏ sau `free()` gây lỗi **use-after-free**.
- Heap allocator có thể tái sử dụng chunk vừa bị giải phóng.
- Nếu kiểm soát được chunk được cấp phát lại, ta có thể ghi đè dữ liệu quan trọng để lấy flag.

Điểm mấu chốt của bài là hiểu rằng vùng nhớ đã `free` không có nghĩa là biến mất. Nó chỉ được đánh dấu là có thể tái sử dụng, và đây chính là thứ ta khai thác.
