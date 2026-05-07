# Quizploit - picoCTF 2026

- **Category:** Binary Exploitation
- **Difficulty:** Easy
- **Author:** ADITYA SUDHANSU

## Description
Solve the quiz.
Download the source code to answer questions here.
Download the binary to answer questions here.
Additional details will be available after launching your challenge instance.

---

## Phân tích bài toán (Analysis)
Bài toán cung cấp cho chúng ta source code `vuln.c` và file thực thi `vuln`. Khác với các bài pwn thông thường yêu cầu chúng ta phải tự viết script exploit để lấy shell hoặc đọc file, challenge này lại được thiết kế dưới dạng một bài **Quiz** (trắc nghiệm) tương tác. Khi kết nối vào server, chúng ta sẽ phải trả lời 13 (0xD) câu hỏi liên quan đến kiến thức cơ bản về lỗi Buffer Overflow và thông tin về file binary. Nếu trả lời đúng toàn bộ, chương trình sẽ tự động cấp Flag.

### Phân tích Source code `vuln.c` (Dựa trên `image-1.png`)
```c
#include <stdio.h>
#include <stdlib.h>

/*
This is not the challenge, just a template to answer the questions.
To get the flag, answer all the questions.
There are no bugs in the quiz.
There are 0xD questions in total.
*/

void win(){
    system("cat flag.txt");
}

void vuln(){
    char buffer[0x15] = {0};
    fprintf(stdout, "\nEnter payload: ");
    fgets(buffer, 0x90, stdin);
}

void main(){
    vuln();
}
```
Từ mã nguồn trên, ta có thể rút ra một số thông tin quan trọng:
- Hàm `win()` thực thi `system("cat flag.txt");` nhưng **không được gọi** ở bất kỳ đâu trong quá trình chạy bình thường của chương trình.
- Trong hàm `vuln()`, mảng `buffer` được khởi tạo với kích thước là `0x15` (21 bytes).
- Chương trình sử dụng hàm `fgets(buffer, 0x90, stdin);` để lấy đầu vào từ người dùng, nhưng lại đọc tới `0x90` (144 bytes). Điều này chắc chắn dẫn đến lỗi **Buffer Overflow**.

## Lời giải (Solution)
Dựa vào các hình ảnh kết quả do bạn cung cấp, dưới đây là đáp án cho 13 câu hỏi của Quizploit để có được flag:

1. **Question 0x1:** What's the architecture of the binary?
   - Đáp án: **`64-bit`** (Dựa vào hint và kiến trúc hệ thống hiện tại).
2. **Question 0x2:** What's the linking of the binary? (e.g. static, dynamic)
   - Đáp án: **`dynamic`** (File sử dụng các hàm chuẩn của thư viện C mà không được biên dịch tĩnh).
3. **Question 0x3:** Is the binary 'stripped' or 'not stripped'?
   - Đáp án: **`not stripped`** (File giữ lại các symbols để debug).
4. **Question 0x4:** Looking at the vuln() function, what is the size of the buffer in bytes? (e.g. 0x10)
   - Đáp án: **`0x15`** (Như trong khai báo `char buffer[0x15]`).
5. **Question 0x5:** How many bytes are read into the buffer? (e.g. 0x10)
   - Đáp án: **`0x90`** (Tham số trong hàm `fgets()`).
6. **Question 0x6:** Is there a buffer overflow vulnerability? (yes/no)
   - Đáp án: **`yes`** (Vì số lượng byte nhập vào lớn hơn kích thước của buffer).
7. **Question 0x7:** Name a standard C function that could cause a buffer overflow in the provided C code.
   - Đáp án: **`fgets`**
8. **Question 0x8:** What is the name of function which is not called any where in the program?
   - Đáp án: **`win`**
9. **Question 0x9:** What type of attack could exploit this vulnerability? (e.g. format string, buffer overflow, etc.)
   - Đáp án: **`buffer overflow`**
10. **Question 0xa:** How many bytes of overflow are possible? (e.g. 0x10)
    - Đáp án: **`0x7B`** (Tính toán: 0x90 - 0x15 = 144 - 21 = 123 bytes = 0x7B).
11. **Question 0xb:** What protection is enabled in this binary?
    - Đáp án: **`NX`** (No eXecute - Chống thực thi code trực tiếp trên stack).
12. **Question 0xc:** What exploitation technique could bypass NX? (e.g. shellcode, ROP, format string)
    - Đáp án: **`ROP`** (Kỹ thuật Return-Oriented Programming thường dùng khi NX được kích hoạt).
13. **Question 0xd:** What is the address of 'win()' in hex? (e.g. 0x4011eb)
    - Đáp án: **`0x401176`** (Có thể dùng lệnh `objdump -d ./vuln | grep win` như trong `image-12.png` để lấy địa chỉ).

Sau khi vượt qua toàn bộ 13 câu hỏi, hệ thống chúc mừng người chơi đã đạt `PERFECT SCORE!` và xuất flag.

### Flag
```text
picoCTF{my_bIn@4y_3xpl0it_fL@g_58c7b379}
```
