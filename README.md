
## Phần A. Báo cáo phân tích

### A1. Bài toán

MartX nhận đơn qua Zalo mỗi ngày. Nhân viên phải đọc tin, tính tiền rồi nhắn xác nhận lại cho khách. Khúc tính tiền hay sai nhất vì có giảm giá và phí giao. Mình thiết kế một chuỗi 3 prompt để AI làm giúp, và nếu dữ liệu có vấn đề thì chuỗi dừng lại để nhân viên xử lý, không chạy tiếp.

Hai kỹ thuật dùng trong bài:

- Chain-of-Thought (CoT): bắt AI viết từng bước suy luận trước khi ra kết quả. Mình dùng ở bước tính tiền, vì để AI trả lời thẳng một con số thì dễ sai mà khó biết sai ở đâu. Viết từng bước thì đọc lướt là kiểm tra được.
- Prompt Chaining: chia việc to thành nhiều prompt nhỏ, kết quả prompt trước là đầu vào của prompt sau. Mỗi prompt làm một việc nên dễ viết, dễ sửa.

### A2. Thiết kế Input / Output

Chuỗi có 3 prompt:

1. Prompt 1 (trích xuất). Input: tin nhắn của khách. Output: JSON gồm trang_thai ("ok" hoặc "loi"), ly_do_loi, ten_khach, dia_chi, san_pham (ten, so_luong, don_vi).
2. Prompt 2 (tính tiền, dùng CoT). Input: JSON của Prompt 1, bảng giá, quy tắc khuyến mãi. Output: các bước tính, rồi một dòng KET QUA gồm trang_thai, ly_do_loi, tam_tinh, giam_gia, phi_giao, tong_thanh_toan.
3. Prompt 3 (nhắn xác nhận). Input: thông tin đơn và kết quả tính tiền. Output: tin nhắn 3 câu gửi khách.

Bảng giá và quy tắc là dữ liệu giả định do mình tự đặt:

- Cà chua 30.000đ/kg, trứng gà 35.000đ/chục, sữa tươi 12.000đ/hộp, hành 5.000đ/bó.
- Tạm tính từ 100.000đ thì giảm 10%.
- Phí giao 15.000đ, miễn phí nếu tổng sau giảm từ 150.000đ.

```
Tin nhắn khách
   |
[Prompt 1: Trích xuất] -- loi --> DỪNG, báo nhân viên
   | ok
[Prompt 2: Tính tiền]  -- loi --> DỪNG, báo nhân viên
   | ok
[Prompt 3: Nhắn xác nhận] --> gửi khách
```

### A3. Bẫy dữ liệu và xử lý đứt gãy

Trong chuỗi, nếu một bước sai mà bước sau vẫn chạy thì lỗi lan đi, và kết quả cuối nhìn vẫn rất hợp lý nên khó phát hiện. Vì vậy mỗi bước đều có trường trang_thai. Gặp lỗi thì ghi "loi" kèm lý do và dừng.

Các tình huống mình lường trước:

- Số lượng mơ hồ như "vài hộp": Prompt 1 báo lỗi, không đoán số.
- Thiếu địa chỉ: Prompt 1 báo lỗi.
- Sản phẩm không có trong bảng giá: Prompt 2 dừng, không tự bịa giá.
- Đơn sát ngưỡng khuyến mãi (ví dụ sau giảm còn 150.300đ, chỉ hơn mốc miễn phí giao 300đ): Prompt 2 có bước kiểm tra riêng cho điều kiện này.
- Bước sau nhận dữ liệu lỗi: Prompt 3 có luật, thấy "loi" thì không viết tin, chỉ trả lời "DUNG CHUOI".

## Phần B. Prompt thực thi

Cách dùng: chạy Prompt 1, xem trang_thai. Nếu "ok" thì dán kết quả vào Prompt 2, rồi tiếp tục sang Prompt 3.

### Prompt 1: Trích xuất

````
Bạn là nhân viên nhập đơn hàng của Siêu thị MartX.
Đọc tin nhắn của khách trong thẻ <data>...</data> và trả về một JSON với các key: trang_thai, ly_do_loi, ten_khach, dia_chi, san_pham (mảng gồm ten, so_luong, don_vi).

Quy tắc:
1. Thông tin không có trong tin nhắn thì ghi null, không được tự đoán.
2. ten_khach bỏ xưng hô (chị, anh, em), chỉ giữ tên.
3. Nếu thiếu địa chỉ, hoặc có sản phẩm mà số lượng không phải số cụ thể (như "vài", "ít"), đặt trang_thai là "loi" và ghi ly_do_loi bằng một câu ngắn. Còn lại thì trang_thai là "ok", ly_do_loi là null.

Chỉ xuất JSON, không chào hỏi, không giải thích.

<data>
(dán tin nhắn của khách vào đây)
</data>
````

### Prompt 2: Tính tiền (CoT)

````
Bạn là nhân viên tính tiền của Siêu thị MartX.
Đơn hàng nằm trong <don_hang>, bảng giá trong <bang_gia>, quy tắc trong <quy_tac>.

Hãy suy luận từng bước, mỗi bước một dòng, có phép tính rõ ràng:
Bước 1: tính thành tiền từng sản phẩm.
Bước 2: cộng thành tạm tính.
Bước 3: kiểm tra điều kiện giảm giá, tính tiền giảm.
Bước 4: kiểm tra điều kiện phí giao.
Bước 5: tính tổng thanh toán.

Nếu trang_thai trong đơn là "loi", hoặc có sản phẩm không có trong bảng giá, thì dừng tính, không tự đoán giá, đặt trang_thai là "loi" và ghi ly_do_loi.

Cuối cùng xuất đúng một dòng bắt đầu bằng "KET QUA:" rồi tới JSON gồm: trang_thai, ly_do_loi, tam_tinh, giam_gia, phi_giao, tong_thanh_toan. Khi lỗi thì các số ghi null.

<bang_gia>
Cà chua: 30.000đ/kg
Trứng gà: 35.000đ/chục
Sữa tươi: 12.000đ/hộp
Hành: 5.000đ/bó
</bang_gia>

<quy_tac>
Tạm tính từ 100.000đ trở lên: giảm 10%.
Phí giao 15.000đ. Miễn phí nếu tổng sau giảm từ 150.000đ trở lên.
</quy_tac>

<don_hang>
(dán JSON của Prompt 1 vào đây)
</don_hang>
````

### Prompt 3: Nhắn xác nhận

````
Bạn là nhân viên chăm sóc khách hàng của Siêu thị MartX.
Viết tin nhắn xác nhận đơn gồm đúng 3 câu, giọng thân thiện, chỉ dùng số liệu trong <thong_tin>, không bịa thêm.
Câu 1: chào khách bằng tên, nhắc lại sản phẩm và địa chỉ giao.
Câu 2: nêu tiền giảm, phí giao và tổng thanh toán.
Câu 3: cảm ơn khách.

Nếu trang_thai là "loi" thì không viết tin nhắn, chỉ trả lời đúng một dòng: DUNG CHUOI - chuyen nhan vien xu ly.

<thong_tin>
(dán JSON của Prompt 1 và dòng KET QUA của Prompt 2 vào đây)
</thong_tin>
````

## Phần C. Kết quả chạy thử

Đây là kết quả điển hình khi chạy chuỗi. Lời văn mỗi lần có thể khác một chút, nhưng các con số phải giống.

### Tin 1: đơn hợp lệ

Tin nhắn: "Chị Lan đặt 2kg cà chua, 1 chục trứng gà, 6 hộp sữa tươi, giao 25 Lê Lợi Q1"

Prompt 1:

```json
{
  "trang_thai": "ok",
  "ly_do_loi": null,
  "ten_khach": "Lan",
  "dia_chi": "25 Lê Lợi, Q1",
  "san_pham": [
    {"ten": "cà chua", "so_luong": 2, "don_vi": "kg"},
    {"ten": "trứng gà", "so_luong": 1, "don_vi": "chục"},
    {"ten": "sữa tươi", "so_luong": 6, "don_vi": "hộp"}
  ]
}
```

Prompt 2:

```
Bước 1: Cà chua 2 x 30.000 = 60.000đ. Trứng gà 1 x 35.000 = 35.000đ. Sữa tươi 6 x 12.000 = 72.000đ.
Bước 2: Tạm tính = 60.000 + 35.000 + 72.000 = 167.000đ.
Bước 3: 167.000đ >= 100.000đ nên giảm 10%. Tiền giảm = 16.700đ. Sau giảm còn 150.300đ.
Bước 4: 150.300đ >= 150.000đ nên miễn phí giao, phí giao = 0đ.
Bước 5: Tổng thanh toán = 150.300đ.
KET QUA: {"trang_thai":"ok","ly_do_loi":null,"tam_tinh":167000,"giam_gia":16700,"phi_giao":0,"tong_thanh_toan":150300}
```

Prompt 3:

```
Chào chị Lan, MartX đã nhận đơn của chị gồm 2kg cà chua, 1 chục trứng gà và 6 hộp sữa tươi, giao tới 25 Lê Lợi, Q1. Đơn được giảm 16.700đ, miễn phí giao hàng, tổng thanh toán là 150.300đ. Cảm ơn chị đã mua sắm tại MartX!
```

Mình tính tay lại: 60.000 + 35.000 + 72.000 = 167.000. Giảm 10% là 16.700, còn 150.300, vừa qua mốc 150.000 nên miễn phí giao. Kết quả đúng.

### Tin 2: sản phẩm không có trong bảng giá

Tin nhắn: "Em Hoa đặt 3 bó rau muống giao 12 Nguyễn Huệ"

Prompt 1 vẫn trả "ok" vì tin đủ thông tin (tên Hoa, địa chỉ 12 Nguyễn Huệ, rau muống 3 bó). Đến Prompt 2:

```
Bước 1: Rau muống không có trong bảng giá, không thể tính thành tiền. Dừng tính, không đoán giá.
KET QUA: {"trang_thai":"loi","ly_do_loi":"Sản phẩm 'rau muống' không có trong bảng giá","tam_tinh":null,"giam_gia":null,"phi_giao":null,"tong_thanh_toan":null}
```

Nếu vẫn thử chạy Prompt 3 thì nó chỉ trả:

```
DUNG CHUOI - chuyen nhan vien xu ly.
```

Nhân viên nhận lý do lỗi và liên hệ khách, không có tin nhắn sai nào được gửi đi.

### Tin 3: số lượng mơ hồ

Tin nhắn: "Anh Minh đặt vài hộp sữa tươi giao 5 Hai Bà Trưng Q3"

Prompt 1:

```json
{
  "trang_thai": "loi",
  "ly_do_loi": "Số lượng sữa tươi không cụ thể ('vài hộp')",
  "ten_khach": "Minh",
  "dia_chi": "5 Hai Bà Trưng, Q3",
  "san_pham": [
    {"ten": "sữa tươi", "so_luong": null, "don_vi": "hộp"}
  ]
}
```

Chuỗi dừng ngay ở bước đầu, Prompt 2 và 3 không cần chạy. Nhân viên gọi hỏi khách lấy bao nhiêu hộp.