# Báo Cáo Cảm Nhận Cá Nhân (Reflection) - Lab 18
**Học viên:** Trần Chí Vĩ
**MSSV:** 2A202602968
**Lớp:** K4 - Track 3B

---

## 1. Ứng dụng Bài giảng vào Quá trình Lập trình

- **Kỹ thuật Semantic Chunking (M1):**
  Hàm `chunk_semantic()` sử dụng nhúng vector và độ đo Cosine để nhóm các câu liên quan về mặt ngữ nghĩa. Ngưỡng 0.85 tỏ ra rất hiệu quả trong việc nhận diện đúng thời điểm văn bản chuyển ý để ngắt đoạn.

- **Mô hình Cha-Con (Hierarchical Chunking) (M1):**
  Đây là điểm mấu chốt quyết định thành công của bài Lab. Việc dùng đoạn con (256 ký tự) để tăng tốc độ tìm kiếm, sau đó nạp toàn bộ đoạn cha (2048 ký tự) vào LLM giúp bảo toàn bức tranh tổng thể. Nhờ vậy, chỉ số Context Recall đã đạt mức xuất sắc 0.9750.

- **Cắt đoạn theo cấu trúc (Structure-Aware Chunking) (M1):**
  Hàm `chunk_structure_aware()` bóc tách tài liệu thông qua các thẻ H1, H2, H3. Nhờ cơ chế này, cấu trúc bảng phân quyền nghỉ phép không bị phá vỡ, giúp LLM trả lời chính xác câu hỏi "nghỉ 20 ngày cần ai duyệt" là CEO.

- **Tìm kiếm lai (Hybrid Search) với BM25 & Dense (M2):**
  Việc áp dụng thư viện `underthesea` tách từ tiếng Việt giúp BM25 bắt chuẩn xác các từ khóa viết tắt như "PVI", "MFA", "CEO". Việc kết hợp tìm kiếm ngữ nghĩa `bge-m3` và dung hòa qua thuật toán RRF đã đẩy Context Precision lên mức 0.9400.

- **Tối ưu xếp hạng bằng Cross-Encoder (M3):**
  Bộ lọc Hybrid Search cung cấp Top 20 tài liệu. Tiếp đó, Cross-Encoder phân tích chéo từng cặp (câu hỏi, đoạn văn) để chắt lọc ra Top 3 tối ưu nhất. Chính bước này đã giúp loại bỏ hiệu quả tài liệu chính sách mật khẩu v1.0 lỗi thời.

- **Đánh giá bằng RAGAS (M4):**
  Bộ đo lường 4 tiêu chí cung cấp cái nhìn rõ ràng để chẩn đoán lỗi xảy ra ở khâu truy xuất (retrieval) hay tạo văn (generation). Khi thấy Faithfulness rớt xuống 0 và LLM báo "Không tìm thấy", có thể khẳng định chắc chắn nguyên nhân đến từ luồng retrieval.

- **Làm giàu dữ liệu (Enrichment) (M5):**
  Sử dụng LLM tóm tắt, tạo câu hỏi giả định và thêm thẻ metadata `version` trước khi tiến hành nhúng vector. Thao tác này giúp hệ thống tách biệt rạch ròi giữa văn bản hiện hành (v2.0) và văn bản cũ.

---

## 2. Các Sự Cố Trong Quá Trình Thực Hiện

**Sự cố 1: Lỗi ghi đè Parent ID (Nghiêm trọng nhất)**
- **Hiện tượng:** Trong lần chạy đầu tiên, điểm RAGAS tụt giảm thê thảm. Faithfulness giảm từ 0.9308 xuống 0.5875; Context Recall giảm từ 0.9850 xuống 0.6333. Hệ thống trả lời sai hoặc "Không tìm thấy" ở 10/20 câu hỏi.
- **Nguyên nhân:** Hàm tạo chunk gán ID chung chung như `parent_0`. Khi quét qua toàn bộ 26 files, các ID này đè lên nhau, làm toàn bộ ngữ cảnh bị tráo đổi sang file cuối cùng.
- **Giải quyết:** Gắn tiền tố là tên file vào `parent_id` (ví dụ `quy_che_luong_parent_0`). Lỗi được khắc phục ngay lập tức.

**Sự cố 2: Bảng biểu bị cắt vụn do Chunking**
- **Hiện tượng:** Truy vấn liên quan đến nghỉ phép 20 ngày bị sai lệch do số "20 ngày" và thẩm quyền "CEO duyệt" rơi vào 2 chunk khác nhau.
- **Giải quyết:** Loại bỏ Basic Chunking, chuyển sang Structure-Aware Chunking dựa trên Header.

**Sự cố 3: Xung đột phiên bản văn bản**
- **Hiện tượng:** Hệ thống cung cấp câu trả lời dựa trên chính sách mật khẩu cũ v1.0 thay vì v2.0.
- **Giải quyết:** Gắn metadata `version` ở khâu M5, kết hợp Cross-Encoder rerank ở khâu M3 và tinh chỉnh hệ thống Prompt.

**Sự cố 4: Lỗi 402 Token Limit từ OpenRouter**
- **Hiện tượng:** Lỗi 402 xuất hiện do mô hình mặc định xin cấp phát lên đến 16000 tokens.
- **Giải quyết:** Khai báo giới hạn `max_tokens=500`. Cấu hình này giúp API chạy ổn định và tiết kiệm tài nguyên.

**Sự cố 5: Giảm thiểu độ trễ khi chạy Unit Test**
- **Hiện tượng:** Việc khởi tạo mô hình 2.2GB trong mỗi vòng lặp test mất quá nhiều thời gian.
- **Giải quyết:** Lưu trữ cache mô hình ở cấp biến global để chỉ load một lần duy nhất, rút ngắn thời gian test xuống chỉ còn khoảng 30 giây.

---

## 3. Kết Quả Hệ Thống Cuối Cùng

Sau quá trình tối ưu toàn bộ 5 nhóm lỗi, phiên bản Production hiện tại ghi nhận kết quả:

| Chỉ số | Baseline | Trước tối ưu | Sau tối ưu | Đánh giá |
|---|---|---|---|---|
| Faithfulness | 0.9308 | 0.5875 | 0.9458 | Vượt baseline |
| Answer Relevancy | 0.7211 | 0.5780 | 0.7850 | Vượt baseline |
| Context Precision | 0.9150 | 0.7833 | 0.9400 | Vượt baseline |
| Context Recall | 0.9850 | 0.6333 | 0.9750 | Ổn định ở mức rất cao |

**Trạng thái hiện tại:** Không còn câu hỏi nào bị đánh lỗi (Failures = []). 

**Đúc kết:** Hơn 80% khiếm khuyết của RAG không xuất phát từ LLM mà bắt nguồn từ quy trình xử lý dữ liệu thô (cắt đoạn, đánh ID, làm giàu siêu dữ liệu). Tinh chỉnh kỹ thuật ở khâu dữ liệu mang lại hiệu suất cải thiện vượt bậc.

---

## 4. Kế Hoạch Ứng Dụng Vào Dự Án Cá Nhân

**Tên dự án:** Hệ Thống Hỏi Đáp Pháp Lý Doanh Nghiệp (Enterprise Legal QA)

**Tình trạng hiện tại:**
- Pipeline RAG sử dụng phương pháp ngắt đoạn tĩnh theo số lượng từ, chỉ áp dụng mô hình Dense Search thông thường.
- Vấn đề gặp phải: Khó tra cứu các nghị định có mã số ngắn (vd: NĐ 100), dữ liệu luật cũ đan xen với luật mới gây nhầm lẫn nghiêm trọng.

**Kế hoạch nâng cấp:**
1. **Khâu Chunking:** Ứng dụng Hierarchical và Structure-Aware Chunking để bảo vệ nguyên vẹn các bảng biểu, phụ lục và cấu trúc điều khoản luật.
2. **Khâu Search:** Tích hợp Hybrid Search (BM25 + Dense) và thuật toán RRF để bắt dính số hiệu văn bản pháp lý.
3. **Khâu Reranking:** Sử dụng Cross-Encoder để lọc các điều khoản sát nghĩa nhất với truy vấn pháp lý.
4. **Metadata & Enrichment:** Đính kèm siêu dữ liệu về `ngày ban hành` và `tình trạng hiệu lực` để loại bỏ các văn bản luật hết hạn.
5. **Đánh giá:** Triển khai RAGAS sau mỗi vòng lặp nâng cấp dữ liệu.

**Lộ trình thực hiện:**
- **Tuần 1:** Chuẩn hóa toàn bộ dữ liệu văn bản luật, xây dựng luồng Chunking mới.
- **Tuần 2:** Tích hợp Hybrid Search và Cross-Encoder, đo lường độ trễ truy xuất.
- **Tuần 3:** Khai thác tính năng Metadata Enrichment để lọc văn bản hết hiệu lực.
- **Tuần 4:** Chạy tập test 50 câu hỏi pháp lý với RAGAS, vá các lỗi tồn đọng và triển khai lên môi trường máy chủ.
