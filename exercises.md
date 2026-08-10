# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Hoàng Gia Bảo  Mã học viên: 2A202601375

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường production/cloud, nếu quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng không có giá trị mặc định sẽ báo lỗi `ValidationError` và dừng khởi động ngay lập tức (fail-fast). Điều này giúp phát hiện lỗi cấu hình ngay trong quá trình deployment. Ngược lại, nếu đặt giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công nhưng sẽ chạy với một API key yếu/mặc định, tạo ra lỗ hổng bảo mật nghiêm trọng cho phép kẻ tấn công lợi dụng để truy cập hệ thống và làm tiêu tốn chi phí tài nguyên.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON mẫu thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T09:30:00+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0001}`

Hai việc làm được với dòng log JSON trên:
1. **Lọc và phân tích tự động bằng công cụ (Query & Indexing)**: Các hệ thống quản lý log (như Datadog, ELK, Grafana Loki) có thể parse các trường JSON để lọc ra các request của một `user_id` cụ thể hoặc tính tổng chi phí `cost_usd` theo từng mốc thời gian.
2. **Cảnh báo và tự động hóa (Automated Alerting)**: Có thể thiết lập quy tắc tự động gửi cảnh báo (alert) dựa trên trường `level` hoặc giá trị `cost_usd` khi đạt ngưỡng rủi ro, điều mà chuỗi văn bản đơn thuần từ `print()` không hỗ trợ máy đọc một cách đáng tin cậy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 165 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~855 MB) bao gồm các công cụ biên dịch (gcc, build-essential), bộ thư viện phát triển (python dev headers), bộ nhớ đệm pip (pip cache) và các file công cụ không cần thiết cho quá trình vận hành (runtime). Nhờ Multi-stage build, chỉ những thư viện Python đã biên dịch/cài đặt và mã nguồn ứng dụng được copy sang stage `runtime` tối giản (`python:3.11-slim`), loại bỏ toàn bộ dung lượng dư thừa từ `builder`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Các layer được dùng lại từ cache: `FROM python:3.11-slim AS builder`, `COPY requirements.txt .`, `RUN pip install ...` và `FROM python:3.11-slim AS runtime`.
- Các layer phải chạy lại: `COPY . .` (ở stage runtime) và các lệnh đặt phía sau nó.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Việc sửa 1 ký tự trong `app/main.py` sẽ làm mất hiệu lực cache (cache invalidation) của layer `COPY . .`. Do đó, Docker sẽ buộc phải chạy lại lệnh `RUN pip install` bên dưới từ đầu, tải và cài đặt lại toàn bộ thư viện python, làm kéo dài thời gian build ứng dụng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong ứng dụng Python -> Thực thi lệnh hệ thống bên trong container với quyền `root` (UID 0) -> Lợi dụng lỗ hổng escape container hoặc các socket/volume mounted từ host -> Nâng quyền và chiếm toàn bộ máy host do UID 0 trong container tương đương root trên host.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ bước thứ 2: Khi code Python bị khai thác, các lệnh thực thi trái phép chỉ có quyền hạn hạn chế của user thường (`appuser`), không thể sửa đổi hệ thống container hay thực thi các thao tác nâng quyền để thoát ra máy host.


---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời của bạn*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
