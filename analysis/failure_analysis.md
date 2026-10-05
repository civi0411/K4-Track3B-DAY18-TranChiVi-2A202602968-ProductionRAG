# Báo cáo Phân tích Lỗi & Khắc phục (Failure Analysis) - Lab 18
**Học viên:** Trần Chí Vĩ (MSSV: 2A202602968) - Lớp: K4 - Track 3B

---
## 1. Kết quả RAGAS (RAGAS Scores)

| Chỉ số RAGAS | Baseline (Naive) | Production (Trước khi fix) | Production (Sau khi fix) | Biến động so với Baseline |
|---|---|---|---|---|
| Faithfulness      | 0.9308 | 0.5875 | 0.9458 | Tăng 0.0150 |
| Answer Relevancy  | 0.7211 | 0.5780 | 0.7850 | Tăng 0.0639 |
| Context Precision | 0.9150 | 0.7833 | 0.9400 | Tăng 0.0250 |
| Context Recall    | 0.9850 | 0.6333 | 0.9750 | Giảm 0.0100 (Không đáng kể) |

*Đánh giá chung:* 
Trong phiên bản Production đầu tiên, toàn bộ 4 chỉ số đều tụt giảm nghiêm trọng (10/20 câu trả lời sai hoặc không tìm thấy) do lỗi trùng lặp `parent_id` khiến dữ liệu ngữ cảnh bị ghi đè. Sau khi tinh chỉnh và áp dụng trọn vẹn 5 module (Từ M1 đến M5 cùng Prompt), hệ thống đã vượt qua Baseline ở 3/4 chỉ số. Đặc biệt, mảng Failures hiện tại đã trống (không còn câu nào trả lời sai).

---
## 2. Nhật ký lỗi và Cách khắc phục (Bug Log)

### Lỗi 1: Trùng lặp ID Parent (Parent-Child Collision) - Nghiêm trọng nhất
- **Hiện tượng:** Faithfulness giảm từ 0.9308 xuống 0.5875. Rất nhiều câu hỏi cơ bản LLM đều trả lời "Không tìm thấy".
- **Ví dụ điển hình:**
  - "Phụ cấp ăn trưa hàng tháng là bao nhiêu?" -> Không tìm thấy.
  - "Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu?" -> Không tìm thấy.
- **Nguyên nhân:** Thuật toán chia chunk sinh ra `parent_id` theo dạng `parent_0`, `parent_1`. Do không có tiền tố phân biệt, dữ liệu của các file sau đã ghi đè lên file trước, làm ngữ cảnh bị sai lệch hoàn toàn sang file "An ninh thông tin".
- **Khắc phục (M1):** Đưa tên file gốc vào làm tiền tố cho `parent_id` (ví dụ: `quy_che_luong_parent_0`). Sau khi sửa, 7/10 câu lỗi đã được trả lời chính xác như Ground Truth.

### Lỗi 2: Thuật toán Chunking cắt vỡ bảng biểu
- **Hiện tượng:** Câu hỏi "Nghỉ phép không lương 20 ngày cần ai phê duyệt?" trả lời sai vì cụm từ "20 ngày" và "CEO duyệt" bị cắt sang 2 chunk riêng biệt.
- **Nguyên nhân:** Cơ chế Basic Chunking ngắt đoạn cứng nhắc theo số lượng ký tự, không hiểu được cấu trúc tài liệu.
- **Khắc phục (M1):** Triển khai Structure-Aware Chunking dựa trên các thẻ H1, H2, H3. Nhờ vậy, cấu trúc của bảng phân quyền nghỉ phép được giữ nguyên vẹn.
- **Kết quả:** LLM đã trích xuất đúng thông tin: "Nghỉ 16-30 ngày cần CEO duyệt".

### Lỗi 3: Dense embedding bỏ qua các từ viết tắt
- **Hiện tượng:** Truy vấn "Bảo hiểm PVI" không trả về được tài liệu mong muốn.
- **Nguyên nhân:** Vector Dense thường không làm tốt việc tìm kiếm các từ viết tắt ngắn hoặc mã định danh.
- **Khắc phục (M2):** Tích hợp thêm Hybrid Search (BM25) và tinh chỉnh bộ tách từ `underthesea` để giữ các cụm từ như "PVI", "MFA", "CEO".
- **Kết quả:** BM25 bắt chính xác từ khóa, thuật toán RRF dung hòa kết quả tốt, đưa tài liệu đúng lên top 1. 

### Lỗi 4: Xung đột phiên bản tài liệu (Version Conflict)
- **Hiện tượng:** 
  - Trả lời "8 ký tự" (chính sách v1.0) cho câu hỏi độ dài mật khẩu thay vì "12 ký tự" (chính sách v2.0).
  - Trả lời chu kỳ đổi mật khẩu là "90 ngày" thay vì "120 ngày".
- **Nguyên nhân:** Cả hai phiên bản tài liệu đều được đưa vào ngữ cảnh. BM25 ưu tiên bản v1.0 do xuất hiện trước, làm LLM thiên vị thông tin cũ.
- **Khắc phục (M3, M5, Prompt):**
  1. Trích xuất siêu dữ liệu `version` (M5).
  2. Bổ sung Cross-Encoder (M3) để rerank lại kết quả.
  3. Thêm quy tắc vào Prompt: "Nếu có nhiều phiên bản, luôn ưu tiên sử dụng phiên bản mới nhất."
- **Kết quả:** Mọi câu hỏi liên quan đến chính sách bảo mật đều được trả lời chuẩn xác theo phiên bản v2.0.

### Lỗi 5: Hiện tượng bịa thông tin (Hallucination)
- **Hiện tượng:** Truy vấn về mức hoàn trả phí đào tạo (25 triệu) khi nghỉ việc sớm, LLM đưa ra số tiền đúng nhưng lại giải thích sai lệch (bịa ra quy định "công ty chỉ chi tối đa 30 triệu").
- **Nguyên nhân:** Do ngữ cảnh trả về bị khuyết đoạn nói về "điều kiện cam kết 1 năm", nên LLM tự suy diễn lý do.
- **Khắc phục:** 
  1. Đặt `temperature = 0`.
  2. Yêu cầu LLM phải trích dẫn đúng bối cảnh tài liệu trước khi đưa ra kết luận.
- **Kết quả:** Trả lời đúng căn cứ pháp lý: "Nghỉ sau 8 tháng vi phạm cam kết 1 năm, phải hoàn trả 100% (25.000.000 VNĐ)".

---
## 3. Bảng đối chiếu Trước / Sau khắc phục

| Câu hỏi | Tình trạng trước fix | Tình trạng sau fix |
|---|---|---|
| Phụ cấp ăn trưa | Báo lỗi "Không tìm thấy" | Trả lời đúng: 1.000.000 VNĐ/tháng |
| Phạt tạm ứng 15tr/20 ngày | Báo lỗi "Không tìm thấy" | Tính toán đúng pro-rata: ~50.000đ |
| Bảo hiểm PVI | Báo lỗi "Không tìm thấy" | Trích xuất đúng hạn mức 200.000.000 VNĐ/năm |
| Nghỉ phép 20 ngày | Báo lỗi "Không tìm thấy" | Trả lời đúng thẩm quyền: CEO duyệt |
| Tài trợ khóa học 25tr | Trả lời đúng số tiền, sai lý do | Đưa ra đúng lý do: Vi phạm cam kết 1 năm |
| MFA | Không nêu rõ phiên bản | Nhấn mạnh yêu cầu bắt buộc theo v2.0 |
| Mật khẩu tối thiểu | Trả lời sai: 8 ký tự | Trả lời đúng: 12 ký tự |
| Chu kỳ đổi mật khẩu | Trả lời sai: 90 ngày | Trả lời đúng: 120 ngày |
| Nghỉ phép năm thử việc | Báo lỗi "Không tìm thấy" | Trả lời đúng: Không được nghỉ phép năm |
| Mentor và buddy | Báo lỗi "Không tìm thấy" | Trả lời đúng: Phải là hai người khác nhau |

---
## 4. Phân Tích Chuyên Sâu (Case Study)

**Trường hợp nghiên cứu:** Xung đột phiên bản chính sách bảo mật (v1.0 và v2.0).

**Quy trình chẩn đoán:**
1. LLM trả lời "8 ký tự", sai lệch so với Ground Truth (12 ký tự).
2. Kiểm tra chỉ số `context_precision` = 0.0, chứng tỏ tài liệu đúng không lọt được lên Top 1.
3. Truy vết tìm kiếm: Thuật toán BM25 ưu tiên tài liệu v1.0 vì văn bản này nằm phía trên trong quá trình gộp file.
4. LLM có xu hướng tin tưởng tài liệu đầu tiên nó đọc được, dẫn đến việc sinh ra câu trả lời sai.

**Hướng xử lý 3 lớp:**
- **Tầng dữ liệu (M5):** Gắn metadata `version` rõ ràng khi ingest file.
- **Tầng truy xuất (M2 + M3):** Sử dụng Cross-Encoder để rerank, ưu tiên các chunk có độ khớp ngữ nghĩa cao nhất với câu hỏi.
- **Tầng prompt:** Hard-code quy tắc xử lý xung đột vào system prompt ("Luôn áp dụng phiên bản mới nhất").

**Định hướng cải tiến tương lai:** Xây dựng hệ thống phân tích siêu dữ liệu (Metadata Filtering) từ tên file (ví dụ `*_v2.0.md`) để có thể chủ động loại bỏ văn bản cũ khỏi quá trình vector search ngay từ bước đầu.

---
## 5. Kết luận

Sau quá trình tối ưu và triển khai toàn vẹn 5 module, hệ thống Production đã đạt được trạng thái lý tưởng:
- **Xóa sổ toàn bộ lỗi** (`failures = []`).
- **Cải thiện 3/4 chỉ số RAGAS so với Baseline**, trong đó Context Precision đạt 0.9400.
- Khắc phục triệt để các dạng câu hỏi phức tạp đòi hỏi tính toán, suy luận logic hoặc phân biệt phiên bản.

**Bài học cốt lõi:** Phần lớn (hơn 80%) các lỗi truy xuất RAG không xuất phát từ việc LLM yếu kém, mà đến từ tầng chuẩn bị dữ liệu (cắt chunk, gán ID, xử lý metadata). Tối ưu tốt khâu dữ liệu đầu vào sẽ giúp toàn bộ hệ thống tăng tốc và chính xác hơn đáng kể mà không cần phải thay đổi mô hình 언어.
