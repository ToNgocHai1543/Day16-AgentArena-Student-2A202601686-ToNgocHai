# Chiến Lược Tối Ưu Agent Arena: Token Overlap & Hard Cutoff

> **Mục tiêu:** Bỏ qua các phương pháp kiểm tra chuỗi nguyên thủy (Exact Match), áp dụng kỹ thuật đếm điểm giao thoa từ vựng (Token Set Intersection) và can thiệp vòng lặp bạo lực (Hard Override) để tối đa hoá điểm số với số lượng code tối thiểu (dưới 62 dòng theo đúng chuẩn bài lab).

---

## 1. Vấn Đề Lớn Nhất Bị Phá Vỡ: Bẫy "Abstain Domino"

Ở chiến lược cũ, việc gắn chặt `abstain = True` vào điều kiện `len(claims) == 0` đã giết chết điểm số trên các brief có dữ liệu nhưng mô hình lại sinh văn bản theo kiểu tóm tắt (ví dụ `pub-09`).
- Mô hình tóm tắt thông tin -> Không khớp Exact Match -> `critic` xoá claim -> `claims` rỗng -> Kích hoạt nhầm `abstain` trên brief bình thường -> Scorer phạt 0 điểm.

---

## 2. Chiến Lược Cốt Lõi Cho Từng Layer

### A. `critic.py` & `citation_checker.py`: Kỹ Thuật "Token Overlap"

Thay vì đi tìm chuỗi con, hãy băm văn bản thành các tập hợp từ vựng và tính tỷ lệ bao phủ. Phương pháp này "miễn nhiễm" với lỗi xuống dòng, dư khoảng trắng, hoặc các từ nối ngẫu nhiên do LLM sinh ra.

**Hàm lõi cần tái sử dụng:**
```python
import re

def get_tokens(text):
    # Lọc lấy từ vựng, bỏ dấu câu và chuyển chữ thường
    return set(re.findall(r'\w+', str(text).lower()))

def check_support_level(claim_text, doc_body):
    claim_tokens = get_tokens(claim_text)
    if not claim_tokens: 
        return 0.0
    
    doc_tokens = get_tokens(doc_body)
    # Tính tỷ lệ từ của claim xuất hiện trong doc
    return len(claim_tokens.intersection(doc_tokens)) / len(claim_tokens)
```

- **Trong `critic.py`:** Duyệt qua các claims, dùng hàm trên để kiểm tra với toàn bộ `ctx.observed_text`. Nếu tỷ lệ `< 0.75` (hoặc một ngưỡng an toàn như 0.8), thì mới xoá claim đó.
- **Trong `citation_checker.py`:** Loại bỏ `splitlines()`. Gắn `doc_id` bằng cách tìm tài liệu có tỷ lệ `check_support_level` cao nhất (và phải >= 0.75). 

### B. `budget_policy.py`: Chiến Lược "Hard Cutoff"

Không tin tưởng vào việc LLM sẽ nghe lời Nudge nếu nó đang bận xử lý lỗi. Phải cắt đứt luồng công cụ ở mức middleware.

**Can thiệp tại `wrap_tool_call`:**
```python
def wrap_tool_call(self, ctx, call, name, args):
    # Giữ lại 1 lượt để model gọi tool submit
    if ctx.tools.calls >= ctx.max_tool_calls - 1:
        # Không chạy call() thật, ném thẳng lỗi giả mạo để ép LLM dừng
        return ToolResult(
            ok=False, 
            content="[SYSTEM FATAL ERROR: Budget depleted. BẮT BUỘC gọi 'submit' VỚI BÁO CÁO FINAL NGAY BÂY GIỜ.]"
        )
    
    # Cho phép chạy bình thường nếu ngân sách còn
    return call(name, args)
```

### C. Gỡ Rối Contradiction & Absent Bằng Suy Luận Chéo

Hãy để chính thuật toán `Token Overlap` giải quyết mâu thuẫn. Nếu một câu claim trộn 2 thông tin trái ngược từ 2 nguồn khác nhau, nó sẽ không đạt tỷ lệ 75% ở bất kỳ nguồn đơn lẻ nào, và sẽ tự động bị `critic` xoá.

**Cập nhật quy tắc Abstain (đặt ở cuối `after_agent` trong `critic`):**
Chỉ kích hoạt `abstain = True` nếu mô hình đã nỗ lực tìm kiếm nhưng mọi kết quả đều bị loại:
```python
if not report["claims"] and ctx.tools.calls > 2:
    report["abstain"] = True
    report["answer"] = "Không đủ thông tin đồng nhất hoặc không tìm thấy dữ liệu để trả lời."
```

---

## 3. Tổng Kết Các Ưu Điểm So Với Cách Làm Cũ

1. **Siêu Ngắn Gọn:** Code cho thuật toán Token Overlap chưa tới 10 dòng, đảm bảo nằm an toàn trong giới hạn quy chuẩn của lab.
2. **Kháng Nhiễu Tốt:** Không còn sợ LLM thay thế từ "và", "nhưng", "tuy nhiên", hay tự ý xuống dòng.
3. **Tiết Kiệm Điểm Efficiency:** Hard Cutoff đảm bảo không bao giờ bị vượt `max_tool_calls`, lấy trọn vẹn điểm thời gian thực thi.
