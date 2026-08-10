# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trương Ái Linh  Mã học viên: 2A202601496

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên đặt `API_TOKEN` thì `Settings` báo lỗi ngay
> lúc service khởi động. Nhờ vậy tôi biết cấu hình cloud đang thiếu trước khi URL
> được đưa cho người khác dùng. Nếu có mặc định `"changeme"`, service vẫn chạy và
> bất kỳ ai đoán được token mặc định đều có thể gọi `/chat`, làm phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi nhận được là:
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:00:00+00:00", "client_id": "sv-cp5-proof", "prompt_tokens": 5, "completion_tokens": 35, "usd_cost": 0.00002175}`.
> Từ log này tôi có thể (1) lọc và đếm request theo `client_id` để tìm client gọi
> nhiều nhất, và (2) cộng `usd_cost` hoặc theo dõi tỷ lệ lỗi theo thời gian để đặt
> cảnh báo. Dòng `print("đã trả lời xong")` không chứa dữ liệu có cấu trúc để làm
> hai việc đó.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | Chưa đo được trên máy hiện tại |
| Multi-stage | Chưa đo được trên máy hiện tại |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi chưa ghi được số MB thật vì Docker daemon trên máy không phản hồi; hai test
> build Docker cũng được pytest bỏ qua. Về thành phần chênh lệch, bản một stage dùng
> image Python đầy đủ và giữ cả công cụ/layer cài đặt. Bản multi-stage dùng
> `python:3.11-slim`; stage runtime chỉ nhận thư viện đã cài từ builder và source
> cần chạy, nên không mang môi trường build và các file dự án không cần thiết.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, sửa `app/main.py` vẫn dùng lại các layer base image,
> `COPY requirements.txt` và `RUN pip install`; chỉ layer copy source cùng các
> layer phía sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi
> thay đổi source làm layer COPY đổi, kéo theo việc cài lại toàn bộ dependency dù
> `requirements.txt` không đổi, nên build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng Python có thể cho kẻ tấn công thực thi lệnh trong container. Nếu tiến
> trình chạy root, mã độc có toàn quyền trong container; kết hợp cấu hình nguy hiểm
> như mount Docker socket, mount thư mục host hoặc một lỗi container runtime, nó có
> thể sửa dữ liệu hay giành quyền cao trên host. `USER appuser` cắt chuỗi ở bước
> đầu: mã khai thác chỉ nhận quyền UID 10001, không mặc nhiên có quyền root. Đây là
> giảm thiệt hại, không thay thế việc vá lỗ hổng và cấu hình container an toàn.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` cho client biết cơ chế xác thực mà tài nguyên yêu cầu,
> đúng hợp đồng của response 401 và giúp thư viện HTTP xử lý chuẩn. Tôi trả cùng
> thông báo `invalid or missing bearer token` cho thiếu header, sai scheme và sai
> token để người đang dò không biết mình đã đoán đúng phần nào. Client hợp lệ vẫn
> có đủ thông tin để sửa request theo định dạng Bearer.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Client chỉ gửi được 10 request liên tiếp rồi request thứ 11 nhận 429, vì xô bị
> chặn ở sức chứa 10 dù đã im lặng lâu. Nếu bỏ `min(capacity, ...)`, sau 10 phút xô
> nạp thêm 100 token; giả sử trước khi im lặng xô có 0 token thì client bắn được
> 100 request. Nếu xô còn đủ 10 token ở mốc lưu cuối thì phép tính sai còn cho tới
> 110 request. Nguyên nhân là token tích lũy không còn bị giới hạn bởi capacity.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, sự cố lúc 2 giờ sáng có thể tiêu hết gần $30 trước khi bị
> chặn và chỉ tự có ngân sách lại khi sang tháng mới. Với hạn mức $1/ngày, thiệt
> hại tối đa của client trong ngày đó khoảng $1 và service tự nhận request lại khi
> sang ngày UTC mới. Giới hạn ngày vì thế thu nhỏ phạm vi thiệt hại và thời gian
> chờ hồi phục, dù tổng lý thuyết trong 30 ngày vẫn là $30.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503 cho cả ba container. Orchestrator hiểu
> nhầm rằng ba process đã chết nên lần lượt restart chúng, có thể khiến toàn bộ cụm
> cùng không phục vụ. Redis trở lại sau 30 giây nhưng các container vẫn đang khởi
> động hoặc tiếp tục restart, làm sự cố dependency lan thành downtime toàn hệ
> thống. Tách probe thì `/healthz` vẫn 200, còn `/readyz` 503 chỉ khiến load balancer
> tạm rút instance khỏi traffic mà không restart process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp là Blueprint báo `cannot have more than 1 free tier Key Value
> instance`, khiến việc tạo Redis thất bại và web service bị hủy theo. Tôi tìm ra
> nguyên nhân từ trạng thái Blueprint: bước tạo `day12-chat-redis` lỗi trước, còn
> web service ghi `canceled: another action failed`. Tôi xóa/tái sử dụng Key Value
> free đang có rồi Manual Sync lại Blueprint. Sau đó `day12-chat` báo Deployed,
> `day12-chat-redis` Available; kiểm tra thật cho `/healthz` và `/readyz` đều 200.
