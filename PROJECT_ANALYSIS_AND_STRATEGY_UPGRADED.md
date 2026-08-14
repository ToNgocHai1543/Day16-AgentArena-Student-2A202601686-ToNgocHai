# Chiến Lược Nâng Cấp Agent Arena: Bứt Phá Ngưỡng 81.71 Điểm

> **Mục tiêu:** Nâng cấp hệ thống 5 lớp Harness Middleware, chuyển đổi từ chiến lược kiểm tra khớp chuỗi cứng nhắc (Exact Match) sang nhận diện ngữ nghĩa linh hoạt (Fuzzy/Semantic Matching) để xử lý dứt điểm các brief khó (`pub-04`, `pub-05`, `pub-08`, `pub-09`) và tiến sát mốc 100 điểm.

---

## 1. Phân Tích Căn Nguyên (Root Cause Analysis) Tại Các Brief Điểm Thấp

Hệ thống hiện tại (đạt 81.71) hoạt động xuất sắc về mặt cấu trúc và bảo vệ an toàn (Safety), nhưng đang "vấp ngã" ở mảng Grounding (căn cứ thực tế) do sự cứng nhắc trong việc đánh giá văn bản sinh ra từ LLM.

### A. Vấn đề của Brief `pub-08` và `pub-09` (Điểm hiện tại: 40.15/100)
- **Cơ chế hiện tại:** `critic` xoá claim nếu `claim["text"] not in ctx.observed_text`. `citation_checker` tìm kiếm citation qua `any(text in line for line in splitlines())`.
- **Lỗi:** Ở `pub-09` (yêu cầu đếm số vụ việc), mô hình sẽ sinh ra các claim dạng tổng hợp (ví dụ: *"Có 3 vụ việc liên quan..."*). Chuỗi này **không bao giờ** xuất hiện nguyên văn trong tài liệu gốc. Do đó, lớp `critic` nhận diện nhầm đây là "hallucination" và xoá sạch claim, khiến điểm Grounding Recall lập tức về 0.
- **Triệu chứng:** Giữ nguyên 30/30 Safety, ~10/15 Efficiency, nhưng **0/55 Grounding**.

### B. Vấn đề của Brief `pub-04` - Contradiction (Điểm hiện tại: 70.07/100)
- **Cơ chế hiện tại:** `critic` tìm kiếm liên từ `" và "` để chia đôi câu và đánh giá sự mâu thuẫn.
- **Lỗi:** Nếu mô hình dùng các cách diễn đạt khác như *"tuy nhiên"*, *"nhưng"*, hoặc đơn giản là tạo ra hai claim (gạch đầu dòng) tách biệt mâu thuẫn nhau, hệ thống sẽ bỏ sót. Kết quả là hệ thống không kích hoạt được `abstain = True` hoặc vô tình xoá mất một vế thông tin hợp lệ.

### C. Vấn đề của Brief `pub-05` - Absent (Điểm hiện tại: 85.04/100)
- **Cơ chế hiện tại:** `budget_policy` chặn tool call vượt quá ngân sách và chèn `FINALIZE_SENTINEL`.
- **Lỗi:** Dù `critic` đã ép `abstain = True` đúng đắn, mô hình vẫn chạy đến tận lượt cuối cùng của ngân sách trước khi dừng lại. Việc này làm tiêu tốn Token và Tool calls, làm mất điểm Efficiency (15 điểm tối đa).

---

## 2. Kế Hoạch Nâng Cấp Chi Tiết Cho Từng Layer

### 2.1. Nâng Cấp `critic.py`: Chuyển từ Exact Match sang N-gram/Token Matching

Để giải quyết vấn đề ở `pub-08` và `pub-09`, `critic` cần "bao dung" hơn với các claim được tổng hợp, miễn là các thông tin cốt lõi (con số, từ khoá chính) có tồn tại trong tài liệu.

**Giải pháp đề xuất (Thuật toán đối chiếu mềm):**
1. **Lọc thông tin nhạy cảm:** Tách các con số, danh từ viết hoa, mã tài liệu từ `claim["text"]`.
2. **Kiểm tra sự tồn tại:** Thay vì kiểm tra toàn bộ chuỗi, hãy kiểm tra xem **tất cả** các con số/từ khoá nhạy cảm này có xuất hiện cùng nhau trong một đoạn (paragraph) của `ctx.observed_text` hay không.
3. **Nếu mô hình thực hiện "Đếm":** Nếu brief yêu cầu đếm (có thể nhận biết qua prompt hoặc ngữ cảnh), cho phép giữ lại claim nếu số đếm khớp với số lượng thực thể tương ứng tìm thấy trong `ctx.observed_text`.

**Mã giả tham khảo:**
```python
def is_supported(claim_text, observed_text):
    # Rút trích các con số và từ khoá quan trọng
    keywords = extract_key_terms(claim_text)
    
    # Nếu không có từ khoá quan trọng, kiểm tra fuzzy match
    if not keywords:
         return fuzzy_match(claim_text, observed_text) > 0.8
         
    # Kiểm tra xem các từ khoá có cùng xuất hiện gần nhau không
    return all(kw in observed_text for kw in keywords)
```

### 2.2. Nâng Cấp `citation_checker.py`: Xử Lý Trích Dẫn Xuyên Dòng (Cross-line Citation)

Việc `splitlines()` đang giới hạn khả năng tìm citation khi mô hình trích xuất một câu dài vắt qua nhiều dòng.

**Giải pháp đề xuất:**
1. Thay vì đối chiếu từng dòng, hãy nối tài liệu lại và sử dụng thuật toán **Longest Common Substring (LCS)** hoặc kiểm tra mức độ trùng lặp N-gram giữa `claim["text"]` và nội dung `doc.body`.
2. Tài liệu nào có chỉ số trùng lặp cao nhất (> ngưỡng tin cậy, ví dụ 70%) sẽ được chọn làm `doc_id` đúng.

### 2.3. Nâng Cấp Xử Lý Mâu Thuẫn (Contradiction)

Đừng phụ thuộc vào ngôn ngữ của mô hình (như từ `" và "`). Hãy dựa vào việc phân tích chính các tài liệu đã thu thập (`ctx.observed_text`).

**Giải pháp đề xuất:**
1. Trong `after_agent`, phân tích tập hợp các `doc.body` đã lấy.
2. Nếu phát hiện hai tài liệu khác nhau đề cập đến cùng một thực thể/quy định nhưng có thông số/kết quả khác biệt, **chủ động** thiết lập `report["abstain"] = True` bất kể mô hình có nhận ra hay không.
3. Nếu mô hình sinh ra 2 claim trái ngược, gán citation của mỗi claim về đúng tài liệu của nó và bật `abstain`.

### 2.4. Tối Ưu `budget_policy.py`: Kích Hoạt Early Stopping

Để lấy trọn vẹn điểm Efficiency trên các brief đơn giản hoặc absent (`pub-05`).

**Giải pháp đề xuất:**
1. Theo dõi kết quả trả về từ `wrap_tool_call`.
2. Nếu mô hình nhận được câu trả lời đủ để kết luận (ví dụ: tìm thấy tài liệu chính xác, hoặc liên tục nhận được `doc not found` báo hiệu dữ liệu vắng mặt), hãy chèn `FINALIZE_SENTINEL` ngay lập tức vào `messages` trong lượt `before_model` tiếp theo, thay vì đợi `_spent(ctx) >= max_tool_calls`.

### 2.5. Tối Ưu `retry.py`: Ngăn Chặn Vòng Lặp Vô Nghĩa

**Giải pháp đề xuất:**
- Nếu tool trả về lỗi (ví dụ: `doc not found`), `retry` hiện tại có thể lặp lại truy vấn sai. 
- Hãy giới hạn việc tự động gọi lại tool ở dưới mô hình (chỉ 1-2 lần cho các lỗi mạng/chập chờn). 
- Đối với các lỗi logic (`doc not found`), thay vì thử lại mù quáng, hãy trả về kết quả giả cho mô hình: `[SYSTEM: Tài liệu không tồn tại. Hãy thay đổi từ khoá tìm kiếm hoặc kết luận không có thông tin.]` Điều này giúp mô hình tự định hướng lại hành vi.

---

## 3. Lộ Trình Triển Khai (Roadmap)

1. **Bước 1 (Ưu tiên cao nhất): Nâng cấp `critic.py`**
   - Loại bỏ kiểm tra bằng `in` hoặc `==` cho toàn bộ câu.
   - Viết hàm kiểm tra độ phủ từ khoá (Keyword/Entity overlap).
   - Mục tiêu: Giải cứu điểm Grounding cho `pub-08` và `pub-09`.

2. **Bước 2 (Ưu tiên trung bình): Tinh chỉnh `citation_checker.py`**
   - Loại bỏ `splitlines()`.
   - Áp dụng kỹ thuật nối chuỗi (block matching) để cải thiện độ chính xác (Precision).

3. **Bước 3 (Ưu tiên thấp): Cải tiến Contradiction và Early Stopping**
   - Áp dụng logic nhận diện mâu thuẫn dựa trên tài liệu.
   - Cài đặt cơ chế dừng sớm để tối đa hoá điểm Efficiency.

Bằng cách chuyển dịch trọng tâm từ "khớp chuỗi" sang "đối chiếu thông tin", hệ thống của bạn sẽ linh hoạt hơn trước các biến thể sinh văn bản của mô hình, đảm bảo tính bền vững trên các tập kiểm thử ẩn và hướng tới mốc điểm tuyệt đối.
