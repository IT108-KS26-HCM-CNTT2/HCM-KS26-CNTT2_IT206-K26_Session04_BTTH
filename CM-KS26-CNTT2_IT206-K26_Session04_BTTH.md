


## 1. Bối cảnh và Thiết kế Giải pháp

### Bài toán thực tế
Em chọn bài toán **Xử lý và phản hồi đánh giá (review) của khách hàng trên sàn thương mại điện tử**. Mỗi ngày shop nhận hàng trăm đánh giá lẫn lộn: khen ngợi, góp ý sản phẩm, chửi mắng vô cớ, hoặc spam ký tự rác. Nếu để AI trả lời 1 lần duy nhất thì rất dễ bị "ngáo" (ví dụ khách spam ký tự rác mà AI vẫn cảm ơn rối rít, hoặc khách chê rách hàng mà AI chỉ trả lời cho có lệ).

### Kiến trúc chuỗi Prompt khép kín (Prompt Chaining)
Em thiết kế một quy trình chạy tuần tự 3 bước liên kết chặt chẽ với nhau qua định dạng JSON gọn nhẹ (để tối ưu token và không bị vỡ cấu trúc):

1. **Bước 1 (Tiền xử lý & Bắt lỗi dị biệt):** Lọc dữ liệu rác, kiểm tra xem review có phải tiếng Việt hợp lệ hay spam/ký tự lạ không. Nếu là dữ liệu dị biệt thì chặn lại ngay, không cho chạy tiếp để tránh làm crash luồng sau.
2. **Bước 2 (Phân tích Logic - Chain-of-Thought):** Đọc nội dung hợp lệ, suy luận từng bước để phân loại cảm xúc (Tích cực / Tiêu cực / Góp ý) và tìm ra đúng nguyên nhân vấn đề khách gặp phải.
3. **Bước 3 (Ra quyết định & Phản hồi chống đứt gãy):** Nhận kết quả JSON từ Bước 2 để xuất hành động tương ứng (Tự động cảm ơn nếu khen, tạo mã voucher xin lỗi nếu lỗi do shop, hoặc đẩy sang nhân viên thật nếu khách khiếu nại quá căng thẳng).

---

## 2. Hệ thống Prompt Thực Thi (Tối ưu Token)

### Prompt 1: Bộ lọc đầu vào & Chặn dữ liệu dị biệt
```text
Nhiệm vụ: Kiểm tra tính hợp lệ của đánh giá khách hàng.
Quy tắc:
- Nếu đầu vào rỗng, toàn ký tự đặc biệt, số vô nghĩa, hoặc ngôn ngữ không xác định -> gán "hop_le": false.
- Nếu có nội dung có nghĩa -> gán "hop_le": true và làm sạch khoảng trắng thừa.

Chỉ xuất JSON gọn:
{"hop_le": true/false, "noi_dung_sach": "...", "ly_do_loi": "..."}

Đánh giá gốc: "{{REVIEW_GOC}}"
```

### Prompt 2: Suy luận nguyên nhân (Chain-of-Thought)
```text
Nhiệm vụ: Phân tích đánh giá đã làm sạch. Nếu "hop_le" là false, trả về ngay {"trang_thai": "STOP"}.
Nếu "hop_le" là true, hãy suy luận từng bước trong mục "cot":
- Bước 1: Xác định sắc thái (KHEN / CHE / GOP_Y).
- Bước 2: Tìm lỗi cụ thể (giao chậm, sai màu, rách, hay không có lỗi).
- Bước 3: Đánh giá mức độ nghiêm trọng (THAP / CAO).

Chỉ xuất JSON gọn:
{
  "trang_thai": "OK",
  "cot": "tóm tắt 3 bước suy luận ngắn gọn",
  "sac_thai": "KHEN | CHE | GOP_Y",
  "nguyen_nhan": "...",
  "muc_do": "THAP | CAO"
}

Dữ liệu từ Bước 1: {{JSON_BUOC_1}}
```

### Prompt 3: Điều hướng xử lý & Xử lý đứt gãy quy trình
```text
Nhiệm vụ: Dựa vào JSON Bước 2 để xuất hành động cuối cùng:
- Nếu "trang_thai" == "STOP": Xuất "[BO_QUA_SPAM] Không phản hồi."
- Nếu "muc_do" == "CAO": Xuất "[CANH_BAO_NHAN_VIEN] Chuyển CSKH gọi điện trực tiếp vì lỗi nghiêm trọng."
- Các trường hợp còn lại: Viết câu phản hồi ngắn gọn (dưới 40 từ), lịch sự, xưng "shop" gọi "bạn", nhắc đúng tới "nguyen_nhan" ở Bước 2.

Dữ liệu từ Bước 2: {{JSON_BUOC_2}}
```

---

## 3. Kết quả chạy thử thực tế

Em chạy thử chuỗi 3 prompt trên với **2 tình huống thực tế** (1 tình huống dữ liệu dị biệt và 1 tình huống khách chê lỗi nghiêm trọng) để kiểm tra độ ổn định của hệ thống.

### Trường hợp 1: Bẫy dữ liệu dị biệt (Khách spam ký tự rác)
* **Input gốc:** `",,,...@@@ 123456 ???"`
* **Kết quả Bước 1:**
  ```json
  {"hop_le": false, "noi_dung_sach": "", "ly_do_loi": "Chuỗi chỉ chứa ký tự đặc biệt và số vô nghĩa"}
  ```
* **Kết quả Bước 2:**
  ```json
  {"trang_thai": "STOP"}
  ```
* **Kết quả Bước 3 (Đầu ra cuối):**
  > `[BO_QUA_SPAM] Không phản hồi.`
*(Hệ thống dừng an toàn, tiết kiệm token ở Bước 2 và 3, không bị lỗi)*

### Trường hợp 2: Khách đánh giá tiêu cực nghiêm trọng (Cần xử lý đứt gãy)
* **Input gốc:** `"   Giao hàng thì lâu mà mở hộp ra cái áo bị rách một đường dài ngay ngực, làm ăn chán thật sự!!!   "`
* **Kết quả Bước 1:**
  ```json
  {"hop_le": true, "noi_dung_sach": "Giao hàng thì lâu mà mở hộp ra cái áo bị rách một đường dài ngay ngực, làm ăn chán thật sự!!!", "ly_do_loi": ""}
  ```
* **Kết quả Bước 2:**
  ```json
  {
    "trang_thai": "OK",
    "cot": "B1: Khách chê gay gắt (CHE). B2: Lỗi giao chậm và áo bị rách ngay ngực. B3: Hàng hư hỏng nặng nên mức độ CAO.",
    "sac_thai": "CHE",
    "nguyen_nhan": "Giao hàng chậm và áo bị rách ở ngực",
    "muc_do": "CAO"
  }
  ```
* **Kết quả Bước 3 (Đầu ra cuối):**
  > `[CANH_BAO_NHAN_VIEN] Chuyển CSKH gọi điện trực tiếp vì lỗi nghiêm trọng.`

---
