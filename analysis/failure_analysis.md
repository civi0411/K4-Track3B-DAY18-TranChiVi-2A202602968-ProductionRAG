# Báo cáo Phân tích Lỗi (Failure Analysis) - Lab 18
**Học viên:** Trần Chí Vĩ (MSSV: 2A202602968) - Lớp: K4 - Track 3B

---
## 1. Kết quả RAGAS (RAGAS Scores)

| Chỉ số RAGAS | Baseline (Nguyên bản) | Production (Trước khi fix) | Mức chênh lệch |
|---|---|---|---|
| Faithfulness | 0.8308 | 0.2875 | Giảm 0.5433 |
| Answer Relevancy | 0.5211 | 0.2780 | Giảm 0.2431 |
| Context Precision | 0.9250 | 0.3833 | Giảm 0.5417 |
| Context Recall | 0.9250 | 0.4333 | Giảm 0.4917 |

*Đánh giá chung:* Hệ thống gặp sự cố nghiêm trọng khiến cả 4 chỉ số đều tụt dốc. Vấn đề lớn nhất nằm ở việc thiết kế sai ID của Parent Chunk, dẫn đến việc các tài liệu bị đè lên nhau. LLM không nhận được đúng ngữ cảnh nên liên tục báo "Không tìm thấy".

---
## 2. Top 5 Trường hợp Lỗi Nặng Nhất (Bottom-5)

### Lỗi 1: Truy vấn về phụ cấp ăn trưa
- **Câu hỏi đầu vào:** Phụ cấp ăn trưa hàng tháng là bao nhiêu?
- **Kỳ vọng:** Phụ cấp là 1 triệu VNĐ, trả cùng kỳ lương.
- **Thực tế:** Không tìm thấy.
- **Chỉ số thấp nhất:** Faithfulness (0.0) và Context Recall (0.0).
- **Phân tích luồng lỗi:** Kết quả sai -> Ngữ cảnh bị sai lệch (bị tráo sang nội dung an ninh thông tin) -> Câu hỏi rõ ràng -> Cần fix ở bước M1 (Chunking).
- **Nguyên nhân chính:** Khi code chia nhỏ file, các ID của đoạn cha bị trùng lặp (`parent_0`, `parent_1`). Do đó, dữ liệu của file sau đè lên file trước.
- **Giải pháp đề xuất:** Nối thêm tên gốc của file vào `parent_id` để tạo mã định danh độc nhất.

### Lỗi 2: Truy vấn tiền phạt tạm ứng
- **Câu hỏi đầu vào:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Kỳ vọng:** Phạt trễ 5 ngày, tính 2%/tháng trên 15 triệu = 300.000đ/tháng (chia ra khoảng 50.000đ cho 5 ngày).
- **Thực tế:** Không tìm thấy.
- **Chỉ số thấp nhất:** Faithfulness (0.0), Context Precision (0.0).
- **Phân tích luồng lỗi:** Kết quả sai -> Ngữ cảnh sai -> Quá trình tìm kiếm bị sót -> Fix ở M1 & M2.
- **Nguyên nhân chính:** Vẫn là lỗi trùng ID. Ngoài ra, câu hỏi yêu cầu tính toán logic (20 - 15 = 5) nên cần ngữ cảnh cực kỳ chi tiết.
- **Giải pháp đề xuất:** Sau khi fix ID, cần bổ sung tìm kiếm từ khóa BM25 và cấu hình lại Prompt LLM để có bước lập luận (Chain-of-Thought).

### Lỗi 3: Truy vấn bảo hiểm PVI
- **Câu hỏi đầu vào:** Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên?
- **Kỳ vọng:** Hạn mức 200 triệu VNĐ/năm (nội/ngoại trú, nha khoa).
- **Thực tế:** Không tìm thấy.
- **Chỉ số thấp nhất:** Faithfulness (0.0) và Context Recall (0.0).
- **Phân tích luồng lỗi:** Kết quả sai -> Ngữ cảnh sai -> Thiếu từ khóa "PVI" -> Fix ở M1 & M2.
- **Nguyên nhân chính:** Từ "PVI" là từ viết tắt, mô hình vector (Dense) không nhận diện tốt nếu không có BM25 hỗ trợ. Hơn nữa, lỗi Parent ID vẫn đang làm sai lệch context.
- **Giải pháp đề xuất:** Cải thiện hàm tách từ tiếng Việt để giữ nguyên "PVI", kết hợp BM25 để bắt dính keyword này.

### Lỗi 4: Nghỉ phép 20 ngày
- **Câu hỏi đầu vào:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Kỳ vọng:** Nghỉ 16-30 ngày cần CEO duyệt. (Trên 14 ngày tự đóng bảo hiểm).
- **Thực tế:** Không tìm thấy.
- **Chỉ số thấp nhất:** Faithfulness (0.0) và Context Recall (0.0).
- **Phân tích luồng lỗi:** Kết quả sai -> Ngữ cảnh bị thiếu bảng phân quyền -> Fix ở M1 & M3.
- **Nguyên nhân chính:** Thuật toán chia chunk ngắt nhầm vào giữa bảng phân quyền, khiến đoạn văn chứa "20 ngày" và "CEO" bị rách đôi.
- **Giải pháp đề xuất:** Áp dụng Structure-Aware Chunking (chia theo tiêu đề Header) để giữ nguyên vẹn toàn bộ bảng biểu.

### Lỗi 5: Độ dài mật khẩu
- **Câu hỏi đầu vào:** Mật khẩu phải có tối thiểu bao nhiêu ký tự?
- **Kỳ vọng:** 12 ký tự (chính sách v2.0). Bản v1.0 (8 ký tự) đã cũ.
- **Thực tế:** Mật khẩu phải có tối thiểu 8 ký tự.
- **Chỉ số thấp nhất:** Faithfulness (0.0) và Context Precision (0.0).
- **Phân tích luồng lỗi:** Kết quả sai (trả lời theo bản cũ) -> Ngữ cảnh không đủ tốt (cả bản cũ và mới đều xuất hiện nhưng bản cũ lên top) -> Fix ở M5 và Prompt.
- **Nguyên nhân chính:** Xung đột phiên bản. BM25 ưu tiên bản v1.0, LLM thấy bản v1.0 nằm trước nên tin tưởng trả lời sai.
- **Giải pháp đề xuất:** Trích xuất Metadata (`version`) ở M5 để lọc. Bổ sung luật vào Prompt: luôn ưu tiên tài liệu có phiên bản mới nhất.

---
## 3. Phân Tích Chuyên Sâu (Case Study)

**Trường hợp:** Độ dài mật khẩu (Xung đột phiên bản).
**Quy trình phân tích:**
- LLM xuất ra thông tin lỗi thời (8 ký tự) thay vì chính sách mới (12 ký tự).
- Lý do: Search engine đưa cả 2 phiên bản vào context, nhưng xếp hạng v1.0 cao hơn. Người dùng không chỉ định rõ năm hay phiên bản trong câu hỏi.
**Hướng khắc phục:**
1. **Ở khâu Data:** Cần gán thẻ metadata (`version: v2.0`) khi xử lý file.
2. **Ở khâu Prompt:** Dạy LLM xử lý tình huống bằng câu lệnh: *"Nếu có nhiều phiên bản tài liệu khác nhau, hãy áp dụng phiên bản mới nhất"*.
**Nếu có thêm thời gian:** Tôi sẽ xây dựng bộ lọc siêu dữ liệu (Metadata Filtering) tự động nhận diện version từ tên file để loại bỏ các văn bản đã cũ khỏi kết quả tìm kiếm.
