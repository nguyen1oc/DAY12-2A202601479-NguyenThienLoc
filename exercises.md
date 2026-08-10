# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng Câu trả lời của bạn bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thiện Lộc  Mã học viên: 2A202601479

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định là `"changeme"`, khi deploy lên cloud mà quên cấu hình biến `AGENT_API_KEY`, app vẫn khởi động bình thường. Kẻ xấu có thể quét và dùng key mặc định này để gọi API và tiêu tốn tiền LLM của bạn mà bạn không hề hay biết. "Chết sớm" bắt buộc ta phải cấu hình key hợp lệ ngay từ lúc deploy thì service mới chạy được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log mẫu:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:00:15.123456+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.0001}`

Hai việc làm được:
1. Các công cụ giám sát (Datadog, CloudWatch, Grafana) có thể tự động bóc tách các trường `cost_usd`, `tokens_in`, `tokens_out` để tính tổng chi phí và vẽ biểu đồ theo thời gian thực.
2. Tự động lọc, tìm kiếm theo `user_id` và thiết lập hệ thống cảnh báo (alert) tự động khi phát hiện chi phí tăng đột biến hoặc log có `level: "error"`.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~178 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch (~840MB) bao gồm base image Debian đầy đủ (thay vì slim), các công cụ build/compiler, pip cache, header files và các file test nội bộ của thư viện bên thứ ba phát sinh lúc cài đặt nhưng không cần thiết khi chạy ứng dụng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer cài đặt dependency (`COPY requirements.txt` và `RUN pip install`) được tái sử dụng hoàn toàn từ Docker cache. Chỉ các layer từ `COPY . .` trở đi mới phải build lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file code nào, cache của bước `COPY` sẽ bị mất, buộc Docker phải chạy lại lệnh `pip install` từ đầu, làm tăng thời gian build lên rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Kẻ tấn công khai thác lỗ hổng thực thi mã (RCE) trong app Python -> Chiếm quyền root bên trong container -> Tận dụng quyền root kết hợp lỗ hổng kernel của host để thoát khỏi container (container breakout) -> Chiếm quyền root trên máy host.
Lệnh `USER appuser` cắt đứt chuỗi ngay từ bước đầu: kẻ tấn công chỉ có quyền user thường, không thể sửa file hệ thống hay thực hiện các lệnh leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây.
Cách đạt được: Gửi 10 request vào giây 59 của phút trước (10:00:59). Ngay giây tiếp theo (10:01:00), đồng hồ bước sang phút mới và reset hạn mức về 0, người đó gửi tiếp 10 request nữa. Kết quả là 20 request được gửi từ giây 59 đến giây 00 chỉ trong 2 giây liên tiếp.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau: Rate limit giới hạn số lượng request trong một khoảng thời gian ngắn (chống spam/DDoS), còn Cost guard giới hạn tổng số tiền chi tiêu thực tế (USD/tháng) dựa trên lượng token LLM tiêu thụ.
- Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 1 request/phút nhưng request đó chứa prompt siêu dài ngốn 100k token làm vượt quá ngân sách tháng -> Cost guard chặn (402).
- Cost guard cho qua nhưng Rate limit chặn: User mới dùng đầu tháng còn nhiều ngân sách nhưng gửi liên tục 50 request trong 5 giây -> Rate limit chặn (429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối -> Endpoint health check kiểm tra Redis thất bại.
2. Orchestrator (Docker/Kubernetes) tưởng container bị treo nên tự động restart cả 3 container agent.
3. Khi khởi động lại, Redis vẫn đang mất kết nối -> container lại fail health check lúc khởi động.
4. Cả cụm 3 container rơi vào vòng lặp khởi động lại liên tục (CrashLoopBackOff) và chết hẳn, thay vì chỉ tạm ngừng nhận traffic và tự phục hồi khi Redis có lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python ở RAM, mỗi container agent sẽ có một bộ nhớ riêng biệt. Khi Nginx phân phối luân phiên các request vào 3 container khác nhau, ta sẽ thấy `history_length` bị nhảy lung tung, không đồng bộ (ví dụ: lượt 1 vào agent 1 có length=0, lượt 2 vào agent 2 lại thấy length=0, lượt 3 vào agent 1 mới thấy length=2).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `ModuleNotFoundError: No module named 'uvicorn'` khi container khởi động trên Render.
- Nguyên nhân: Lệnh `pip install --user` ở stage builder cài package vào thư mục của root, khi copy sang `appuser` ở stage runtime thì Python không tự động nhận diện đường dẫn site-packages trong `sys.path`.
- Cách sửa: Tạo một môi trường ảo độc lập `/opt/venv` ở stage builder, cài đặt các package vào đó rồi copy `/opt/venv` sang stage runtime và thêm vào biến môi trường `PATH`.
