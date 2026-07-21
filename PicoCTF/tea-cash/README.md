# tea-cash

## Challenge Info

| Field       | Details                  |
|-------------|--------------------------|
| **CTF**     | picoCTF 2026             |
| **Category**| Binary Exploitation      |
| **Author**  | Aditya Sudhansu          |
| **Points**  | 100                      |
| **Difficulty** | Medium               |

## Description

> You've stumbled upon a mysterious cash register that doesn't keep money — it keeps secrets in memory. Traverse the free list and find all the free chunks to get to the flag.

**Provided files:** `heapedit`, `Makefile.share`, `libc.so.6`, `heapedit.c`

---

## Solution

### Overview

Đây là một bài **Heap Exploitation** cơ bản, yêu cầu ta duyệt qua **tcache free list** để tìm tất cả các free chunk trong heap. Bài không yêu cầu exploit phức tạp — chỉ cần hiểu cơ chế tcache và điền đúng địa chỉ các chunk theo thứ tự.

### Tcache Free List

Trong glibc, **tcache (thread-local caching)** là một cơ chế quản lý heap để tái sử dụng các chunk nhỏ đã được `free()`. Các chunk trong tcache được tổ chức thành một **singly-linked list** (danh sách liên kết đơn).

Cấu trúc:
```
tcache_entry (head) -> chunk1 -> chunk2 -> ... -> NULL
```

Mỗi free chunk lưu con trỏ `next` trỏ tới chunk tiếp theo ngay tại đầu vùng dữ liệu của nó.

### Analysis

Kết nối đến server và quan sát output:

```
$ nc candy-mountain.picoctf.net 55825
tcache head (start of free list) -> 0x22ad9490
```

Server cho ta biết địa chỉ head của tcache free list. Nhiệm vụ là duyệt qua từng chunk một.

### Exploit

Nhận được địa chỉ đầu của free list từ server:

```
tcache head (start of free list) -> 0x22ad9490
```

Duyệt qua lần lượt 6 chunk theo thứ tự địa chỉ tăng dần (mỗi chunk cách nhau `0x90` bytes — kích thước của chunk bao gồm header):

| Chunk | Address       |
|-------|--------------|
| 1     | `0x22ad9490` |
| 2     | `0x22ad9520` |
| 3     | `0x22ad95b0` |
| 4     | `0x22ad9640` |
| 5     | `0x22ad96d0` |
| 6     | `0x22ad9760` |

Sau khi nhập đúng thứ tự tất cả 6 địa chỉ, server xác nhận traversal thành công và trả về flag.

### Result

```
$ nc candy-mountain.picoctf.net 55825
tcache head (start of free list) -> 0x22ad9490
Chunk 1 address: 0x22ad9490
Chunk 2 address: 0x22ad9520
Chunk 3 address: 0x22ad95b0
Chunk 4 address: 0x22ad9640
Chunk 5 address: 0x22ad96d0
Chunk 6 address: 0x22ad9760
Correct traversal! Flag: picoCTF{0fd522cb3e9905002631d25e21a4750b}
```

## Flag

```
picoCTF{0fd522cb3e9905002631d25e21a4750b}
```

## Key Takeaways

- **tcache** trong glibc lưu trữ các free chunk dưới dạng singly-linked list.
- Head của free list được cung cấp sẵn → chỉ cần đọc con trỏ `next` của từng chunk để duyệt danh sách.
- Bài này rèn luyện kỹ năng đọc hiểu cấu trúc heap — nền tảng cho các bài heap exploitation phức tạp hơn như **Use-After-Free**, **tcache poisoning**, v.v.
