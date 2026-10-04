# Báo Cáo Cảm Nhận Cá Nhân (Reflection) - Lab 18
**Học viên:** Trần Chí Vĩ
**MSSV:** 2A202602968
**Lớp:** K4 - Track 3B

---

## 1. Kết nối Bài giảng vào Thực tế Code

- **Kỹ thuật Semantic Chunking (M1):** 
  Thay vì cắt đoạn một cách vô tri (như Basic chunking), hàm `chunk_semantic()` sử dụng nhúng vector và độ đo Cosine để giữ các câu liên quan ở lại với nhau. Ngưỡng 0.85 giúp ngắt dòng đúng lúc đoạn văn chuyển ý.
- **Mô hình Cha-Con (Hierarchical Chunking) (M1):** 
  Đây là điểm mấu chốt. Dùng đoạn con (256 ký tự) để tìm kiếm nhanh và chính xác, nhưng khi nạp vào LLM thì dùng đoạn cha (2048 ký tự) để lấy bức tranh toàn cảnh, tối ưu hóa Context Recall.
- **Cắt theo cấu trúc Markdown (Structure-Aware) (M1):** 
  Hàm `chunk_structure_aware()` bóc tách tài liệu dựa vào H1, H2, H3. Nhờ đó, các danh sách và bảng biểu không bao giờ bị xé lẻ.
- **Tìm kiếm lai (Hybrid Search) với BM25 & Dense (M2):** 
  Dùng thư viện `underthesea` tách từ tiếng Việt để BM25 hoạt động tốt, kết hợp mô hình `bge-m3` để bắt ngữ nghĩa. Kế đó, thuật toán RRF dung hòa hai thang điểm hoàn hảo.
- **Tối ưu xếp hạng bằng Cross-Encoder (M3):** 
  Sau khi Hybrid Search lọc ra Top 20, Cross-Encoder nhảy vào chấm điểm trực tiếp từng cặp câu hỏi - đoạn văn, giúp chắt lọc ra Top 3 chính xác nhất, bỏ đi những đoạn dễ gây nhầm lẫn.
- **Bộ công cụ đánh giá RAGAS (M4):** 
  Đo lường chi tiết 4 thang điểm. Qua đó, ta dễ dàng chẩn đoán lỗi nằm ở đâu (retrieval hay generation).
- **Làm giàu dữ liệu (Enrichment) (M5):** 
  Dùng LLM tóm tắt, tạo câu hỏi giả định và thêm ngữ cảnh vào từng đoạn văn trước khi nhúng. Cách này giảm thiểu tình trạng truy xuất thất bại đáng kể.

---

## 2. Vấn đề Gặp Phải và Cách Giải Quyết

**Sự cố 1: Lỗi đè ID Parent (Parent-Child Collision)**
- **Vấn đề:** Khi chạy, điểm số RAGAS rớt thảm hại, LLM toàn báo "Không tìm thấy".
- **Nguyên nhân:** Hàm tạo chunk chỉ đặt ID là `parent_0`, `parent_1`. Khi duyệt 26 files, các ID này đè lên nhau, khiến toàn bộ context bị tráo thành file "An ninh thông tin".
- **Giải quyết:** Tôi gắn thêm tên file vào trước ID (ví dụ: `quy_che_luong_parent_0`). Lỗi lập tức biến mất.

**Sự cố 2: OpenRouter báo lỗi 402 Token Limit**
- **Vấn đề:** Trả về lỗi 402 vì model cố gắng xin cấp phát 16000 tokens mặc định.
- **Giải quyết:** Chỉ định rõ ràng `max_tokens=500` vào tham số cấu hình. Model chạy mượt mà và tiết kiệm chi phí.

**Sự cố 3: Máy chậm khi chạy check_lab.py**
- **Vấn đề:** Chạy mất quá nhiều thời gian vì model nặng 2.2GB phải load đi load lại.
- **Giải quyết:** Sử dụng caching cấp biến global để chỉ load mô hình 1 lần duy nhất trong bộ Unit Test, thời gian chạy giảm xuống chỉ còn khoảng 30s.

---

## 3. Kế hoạch Áp Dụng cho Dự án Cá Nhân

**Tên dự án:** Hệ Thống Hỏi Đáp Pháp Lý Doanh Nghiệp (Enterprise Legal QA)

**Hiện trạng:** Pipeline cũ chỉ dùng Naive RAG (cắt chữ cứng ngắc, vector thuần). Thường xuyên trả lời sai số hiệu văn bản và không phân biệt được các nghị định mới/cũ.

**Kế hoạch nâng cấp:**
1. **Chia đoạn (Chunking):** Chuyển sang mô hình Hierarchical + Structure-Aware để không bao giờ làm đứt gãy các bảng phụ lục hay điều khoản luật pháp.
2. **Truy xuất (Search):** Áp dụng Hybrid Search (BM25 + Dense) để vừa hiểu ý nghĩa, vừa không bỏ sót số liệu chính xác. Dùng RRF để gộp điểm.
3. **Chấm điểm lại (Reranking):** Triển khai mô hình Cross-Encoder để ưu tiên các điều khoản khớp hoàn toàn với câu hỏi.
4. **Metadata & Enrichment:** Bổ sung siêu dữ liệu như "ngày ban hành", "tình trạng hiệu lực" để LLM tự động loại bỏ luật đã cũ.
5. **Kiểm thử tự động:** Tích hợp RAGAS để đo đạc chất lượng sau mỗi bản cập nhật.

**Tiến độ dự kiến:**
- **Tuần 1:** Chuẩn bị dữ liệu pháp lý và viết lại cấu trúc Chunking.
- **Tuần 2:** Chạy thử Hybrid Search và Reranker, kiểm tra độ trễ.
- **Tuần 3:** Tích hợp Metadata Enrichment, hoàn thiện bộ lọc thời gian hiệu lực.
- **Tuần 4:** Đo lường 50 câu hỏi bằng RAGAS, vá lỗi và đóng gói lên server.
