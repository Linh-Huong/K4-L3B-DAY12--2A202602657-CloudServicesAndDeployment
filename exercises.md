# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng ghi chú mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Tiến Linh - Mã học viên: 2A202602657

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `"changeme"` (hoặc một khóa mặc định bất kỳ), khi deploy lên staging hoặc production mà quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường và probe `/health` vẫn trả về 200 OK. Hệ thống CI/CD hoặc orchestrator tưởng rằng deploy thành công. Tuy nhiên, API lúc này có thể bị kẻ lạ lợi dụng khóa mặc định `"changeme"` để gọi API miễn phí, tiêu tốn sạch hạn ngạch token LLM và gây thiệt hại tài chính lớn trước khi bạn kịp phát hiện qua hóa đơn. Ngược lại, cơ chế "fail fast" (không có giá trị mặc định) khiến Pydantic ném `ValidationError` ngay lúc ứng dụng vừa nạp config khi khởi động; quy trình deployment sẽ lập tức báo lỗi đỏ, container cũ không bị ghi đè và lập tức cảnh báo cho kỹ sư sửa ngay trước khi người dùng hay kẻ xấu kịp truy cập.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:10:58.123456+00:00", "user_id": "sv01", "tokens_in": 1, "tokens_out": 40, "cost_usd": 2.415e-05}
```

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không thể làm được:
1. **Truy vấn, lọc và tổng hợp định lượng (Aggregation & Metrics):** Các hệ thống thu thập log (như Datadog, Grafana Loki, CloudWatch) có thể tự động parse các trường JSON để tính tổng chi phí phát sinh theo từng `user_id`, vẽ biểu đồ phân phối token theo thời gian thực hoặc tìm ra những người dùng tiêu tốn nhiều chi phí nhất.
2. **Thiết lập cảnh báo tự động (Alerting):** Dễ dàng cấu hình các rule cảnh báo tự động (ví dụ: cảnh báo khi `cost_usd > 0.05` trong 1 request, hoặc cảnh báo khi tỷ lệ log có `level: "error"` vượt quá 5% trong 5 phút), điều không thể làm được với chuỗi văn bản thuần vô cấu trúc từ `print()`.

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
| 1 stage (bản đầu) | ~1015 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~744 MB) bao gồm:
1. Bộ công cụ biên dịch và các gói phát triển của Debian (gcc, g++, build-essential, make, git, header files C/C++) vốn có sẵn trong image đầy đủ `python:3.11` để phục vụ cài đặt package nhưng hoàn toàn không cần thiết cho quá trình chạy app.
2. Cache của package manager (`pip cache`, `apt cache`) và các build artifacts tạm thời tạo ra trong quá trình cài đặt wheel.
3. Trong Multi-stage build, toàn bộ việc biên dịch diễn ra ở stage `builder`, stage `runtime` chỉ sử dụng base image siêu gọn nhẹ `python:3.11-slim` và copy duy nhất thư mục kết quả `/install` sang, giúp loại bỏ toàn bộ compiler và dependencies thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Docker thực hiện cache theo từng layer. Khi chỉ sửa code trong `app/main.py`, file `requirements.txt` không thay đổi nên layer `COPY requirements.txt .`, `RUN pip install` và `COPY --from=builder` đều **được dùng lại từ cache (CACHED)**. Chỉ có layer `COPY app ./app` và các bước kế tiếp mới phải build lại, quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất cứ khi nào sửa dù chỉ 1 ký tự trong mã nguồn, layer `COPY . .` sẽ bị thay đổi hash, làm Docker hủy toàn bộ cache từ layer đó trở đi. Hệ quả là lệnh `RUN pip install` sẽ phải tải và cài đặt lại toàn bộ thư viện từ Internet mỗi lần build, khiến thời gian build kéo dài thêm vài phút một cách lãng phí.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python có một lỗ hổng bảo mật (ví dụ: Remote Code Execution qua deserialization `pickle`, hoặc một thư viện bên thứ ba bị zero-day).
2. Kẻ tấn công gửi payload kích hoạt lỗ hổng và chiếm quyền shell bên trong container. Do container chạy dưới user `root` (UID 0), kẻ tấn công sở hữu quyền root trong container.
3. Từ quyền root này, kẻ tấn công khai thác tiếp các lỗ hổng của Linux kernel, hoặc tận dụng các lỗ hổng mount volume (như docker socket `/var/run/docker.sock`, privileged mode) để thực hiện kỹ thuật container breakout (thoát khỏi container).
4. Do UID 0 bên trong container mặc định tương ứng với UID 0 (root) của hệ điều hành host, kẻ tấn công lập tức có toàn quyền kiểm soát trên chính máy chủ host.

Lệnh `USER appuser` cắt đứt chuỗi ở bước 2:
Khi chuyển sang chạy bằng user không đặc quyền (`appuser` với UID 10001), kẻ tấn công khi chiếm được shell chỉ có quyền của user thường. Không có quyền root trong container, kẻ tấn công không thể cài thêm công cụ, không thể thao tác các thiết bị hệ thống, không thể load kernel module và không thể thực thi các exploit để breakout ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Con số tối đa: **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  - Ở giây 10:00:59 (giây cuối cùng của phút thứ nhất), người dùng gửi dồn dập 10 request. Vì trong phút 10:00 người dùng chưa gửi request nào nên cả 10 request đều được cho qua.
  - Sang giây 10:01:00 (hoặc 10:01:01 - đầu phút thứ hai), bộ đếm đồng hồ tự động reset về 0. Người dùng lập tức gửi thêm 10 request nữa. Hệ thống tính đây là lượt request của khung phút mới nên tiếp tục cho qua cả 10 request.
  - Kết quả: Trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01), hệ thống phải nhận tới 20 request (gấp đôi hạn mức cho phép). Thuật toán Sliding Window giải quyết triệt để lỗi này bằng cách luôn tính chính xác tổng số request trong cửa sổ 60 giây trượt lùi tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau:** 
  - Rate limit kiểm soát **tốc độ / tần suất gọi** (số request trong khoảng thời gian ngắn: giây/phút) nhằm bảo vệ hạ tầng máy chủ không bị nghẽn tải hoặc tấn công DoS.
  - Cost guard kiểm soát **chi phí tài chính** (tổng số tiền USD / số lượng token tích lũy trong tháng) nhằm bảo vệ ngân sách của chủ sở hữu hệ thống.
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:**
  - Người dùng gửi 1 request mỗi 2 phút (tần suất rất thấp, rate limit 10 request/phút hoàn toàn cho qua). Tuy nhiên, mỗi request chứa tài liệu cực lớn dài 50.000 tokens khiến chi phí mỗi lần gọi rất cao. Chỉ sau một vài request, tổng chi tiêu chạm ngưỡng $10.00/tháng, lúc này Cost guard sẽ chặn ngay lập tức (HTTP 402) dù Rate limit không hề bị kích hoạt.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:**
  - Đầu tháng, người dùng mới tiêu tốn $0.05 / $10.00 ngân sách (ngân sách còn rất dồi dào). Tuy nhiên, người dùng chạy một script gửi 15 request chỉ trong vòng 5 giây. Cost guard kiểm tra thấy ngân sách vẫn đủ nhưng Rate limit sẽ lập tức chặn từ request thứ 11 trở đi (HTTP 429) để bảo vệ server khỏi bị quá tải đột ngột.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố hoặc gián đoạn mạng trong 30 giây.
2. Endpoint kiểm tra thấy Redis không phản hồi và trả về mã lỗi HTTP 503 cho probe kiểm tra liveness (`/health`).
3. Container orchestrator (Docker Compose, Kubernetes, Railway...) nhận thấy liveness probe thất bại, kết luận rằng process của container bị deadlock hoặc hỏng hóc, và **tiến hành restart đồng loạt cả 3 container**.
4. Cả 3 container được khởi động lại. Trong quá trình khởi động, chúng lại chạy probe kiểm tra Redis. Vì Redis vẫn chưa kết nối lại được, chúng lại tiếp tục trả về 503.
5. Cụm container rơi vào chu kỳ khởi động lại liên tục (CrashLoopBackOff). Toàn bộ hệ thống rơi vào trạng thái sập hoàn toàn (downtime 100%).
6. Khi Redis hoạt động trở lại sau 30 giây, các container vẫn đang dở dang ở các chu kỳ bị kill và khởi động lại, khiến thời gian phục hồi kéo dài và biến một sự cố dependency tạm thời thành sự cố sập toàn bộ dịch vụ (cascading failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (Stateless): Giá trị `history_length` tăng đều đặn và tuyến tính sau mỗi lượt hỏi: 0 -> 2 -> 4 -> 6 -> 8... cho cùng một `user_id`, vì cả 3 container đều đọc và ghi chung vào một nguồn dữ liệu tập trung duy nhất trên Redis.
- Nếu lưu trong một dict Python (In-memory stateful):
  - Do load balancer phân phối các request ngẫu nhiên hoặc luân phiên (round-robin) qua 3 container A, B và C, mỗi container chỉ nắm giữ lịch sử các request từng gửi tới nó.
  - Kết quả là `history_length` trong response sẽ nhảy gián đoạn và lộn xộn, ví dụ: 0 -> 0 -> 0 -> 2 -> 2 -> 4... Agent sẽ bị hiện tượng "mất trí nhớ ngẫu nhiên" (ví dụ: câu hỏi 1 gửi container A, câu hỏi 2 gửi container B thì container B hoàn toàn không biết câu hỏi 1 là gì).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi không nhận cổng động của Cloud Provider (Railway) dẫn đến Health Check Timeout khi deploy.
- **Thông báo lỗi:** `Healthcheck failed: Timed out waiting for container to become healthy on port 8000`
- **Cách tìm ra nguyên nhân:** 
  - Xem Deploy Logs trên dashboard Railway, nhận thấy Railway tự động gán một cổng ngẫu nhiên qua biến môi trường `$PORT` (ví dụ `PORT=7428`), trong khi file cấu hình Dockerfile ban đầu dùng lệnh khởi chạy cứng `--port 8000`. Khi uvicorn chỉ lắng nghe cổng 8000 còn Railway probe vào cổng `$PORT`, probe sẽ timeout.
- **Cách sửa:**
  - Cập nhật lệnh `CMD` trong `Dockerfile` để đọc biến `PORT` từ môi trường với fallback:
    `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  - Đồng thời bind vào `0.0.0.0` để có thể nhận traffic từ mạng ngoài container. Sau khi commit và deploy lại, service khởi động thành công và vượt qua healthcheck ngay lập tức.
