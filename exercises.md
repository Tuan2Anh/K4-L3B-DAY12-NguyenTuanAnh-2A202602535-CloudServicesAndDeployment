# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết vào từng mục.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Anh  Mã học viên: 2A202602535

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi triển khai lên môi trường staging hoặc production trên cloud, nếu ta sơ suất quên không thiết lập biến môi trường `AGENT_API_KEY`:
> - Nếu đặt giá trị mặc định (như `"changeme"`): App vẫn khởi động bình thường, probe báo healthy. Bất kỳ ai hoặc các bot quét API trên Internet đều có thể dùng key `"changeme"` để gọi vào `/ask` và âm thầm đốt sạch tiền token LLM trong tài khoản của ta cho đến khi nhận hóa đơn.
> - Nếu không có giá trị mặc định: Pydantic ném `ValidationError` ngay lúc nạp cấu hình khi vừa khởi động (fail-fast). Container lập tức exit và platform cloud báo deployment thất bại ngay trước mắt ta lúc deploy, buộc ta phải cấu hình secret chuẩn xác trước khi cho phép bất kỳ traffic nào đi vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thực tế thu được:
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:00:15.864000+00:00", "user_id": "sv-test", "tokens_in": 41, "tokens_out": 45, "cost_usd": 3.315e-05}
> ```
> Hai việc làm được với log có cấu trúc:
> 1. **Phân tích và thống kê tự động:** Các công cụ gom log (Datadog, Elasticsearch, CloudWatch) có thể truy vấn định lượng trực tiếp trên từng trường JSON: tính tổng chi phí (`SUM(cost_usd)`), tìm ra user nào tiêu thụ nhiều token nhất trong ngày (`GROUP BY user_id`), hoặc vẽ biểu đồ tương quan giữa prompt length (`tokens_in`) và thời gian phản hồi.
> 2. **Cảnh báo (Alerting) tự động theo thời gian thực:** Dễ dàng thiết lập các rule giám sát tự động để gửi cảnh báo Slack/PagerDuty khi tỷ lệ log có `level == "error"` vượt quá 5% trong 5 phút, hoặc cảnh báo khi có request bất thường có `cost_usd > 0.05`. Lệnh `print()` thô không thể bóc tách trường để máy phân tích tự động như vậy.

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
| Multi-stage | ~268 MB (content size: 63.9 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750 MB) bao gồm:
> 1. **Base image & công cụ biên dịch:** Bản đầy đủ `python:3.11` chứa toàn bộ trình biên dịch C/C++ (`gcc`, `g++`), công cụ build (`make`), thư viện phát triển Debian và các header files không cần thiết ở runtime. Bản `python:3.11-slim` đã lược bỏ toàn bộ các gói nặng này, chỉ giữ lại runtime tối thiểu.
> 2. **Cache và file tạm khi cài đặt thư viện:** Ở multi-stage, toàn bộ quá trình build wheels và cache của `pip` nằm lại ở stage `builder`. Stage `runtime` chỉ copy đúng thư mục kết quả `/install` sang `/usr/local` mà không mang theo cache tải về hay các file rác phát sinh khi cài đặt.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:** Docker cache các layer từ trên xuống dưới. Khi chỉ sửa `app/main.py`, các layer `COPY requirements.txt .` và `RUN pip install ...` ở stage builder cùng các layer cơ sở ở stage runtime đều được tái sử dụng hoàn toàn từ cache (`CACHED`). Chỉ có layer `COPY app ./app` và các bước sau nó phải chạy lại, việc build hoàn thành trong tích tắc (< 1 giây).
> - **Nếu đặt `COPY . .` trước `RUN pip install`:** Mỗi lần sửa một ký tự trong code, layer `COPY . .` bị thay đổi checksum khiến cache bị mất hiệu lực (invalidated). Docker sẽ bị buộc phải tải và cài đặt lại toàn bộ các gói trong `requirements.txt` từ đầu qua mạng, khiến mỗi lần sửa code phải chờ hàng phút build lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện tấn công:
> 1. Ứng dụng Python có lỗ hổng (ví dụ RCE qua pickle, command injection, hoặc một CVE từ thư viện dependencies).
> 2. Kẻ tấn công gửi payload kích hoạt RCE và chiếm được quyền thực thi shell bên trong container.
> 3. Nếu không có lệnh `USER`, tiến trình trong container chạy dưới quyền `root` (UID 0).
> 4. Kẻ tấn công với quyền root container có thể khai thác các lỗi container escape (lỗ hổng Linux kernel, mount Docker socket không an toàn, hoặc lạm dụng các Linux capabilities chưa drop) để thoát ra ngoài máy host.
> 5. Khi đã thoát ra ngoài host, vì UID trong container là 0 nên kẻ tấn công trực tiếp sở hữu quyền root (UID 0) trên máy host, nắm toàn bộ quyền kiểm soát máy chủ vật lý.
>
> **Lệnh `USER appuser` cắt đứt chuỗi này ngay tại bước 3:** Mã độc chỉ có quyền của user thường `appuser` (UID 10001) không có đặc quyền. Kẻ tấn công không thể ghi vào thư mục hệ thống, không có quyền can thiệp kernel hay thực hiện các thao tác đặc quyền cần thiết để vượt rào (container escape) ra host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.
>
> Giải thích:
> - Lúc `10:00:59` (giây cuối cùng của phút thứ nhất), người dùng gửi dồn 10 request. Bộ đếm phút 10:00 ghi nhận 10/10 request (vẫn hợp lệ, chưa vi phạm).
> - Đúng `10:01:00`, đồng hồ bước sang phút mới và bộ đếm tự động reset về 0.
> - Lúc `10:01:01` (giây đầu tiên của phút thứ hai), người dùng gửi tiếp 10 request. Bộ đếm phút 10:01 ghi nhận 10/10 request (vẫn hợp lệ).
> - Như vậy, trong khoảng thời gian chỉ vỏn vẹn 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải gánh tới 20 request — gấp đôi hạn mức 10 req/phút. Cửa sổ trượt (Sliding Window) giải quyết triệt để lỗi này bằng cách luôn xét chính xác khoảng `[now - 60, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Khác biệt cốt lõi:**
>   - *Rate limit* bảo vệ về mặt **tần suất và tài nguyên hạ tầng** trong khung thời gian ngắn (ví dụ: tối đa 10 request/phút) để ngăn server bị quá tải (DoS).
>   - *Cost guard* bảo vệ về mặt **tài chính và ngân sách** trong chu kỳ dài (ví dụ: tối đa 10.0 USD/tháng) để ngăn chặn việc cạn kiệt tiền API do các prompt quá dài hoặc sử dụng nhiều token.
> - **Rate limit cho qua nhưng Cost guard chặn:** User gửi 1 request duy nhất trong ngày (hoàn toàn thỏa mãn < 10 request/phút). Tuy nhiên, tổng chi tiêu tích lũy của user trong tháng đã đạt $10.0. Cost guard phát hiện vượt ngân sách tháng nên chặn với mã `402 Payment Required`.
> - **Cost guard cho qua nhưng Rate limit chặn:** User mới tạo tài khoản đầu tháng, ngân sách còn nguyên $10.0. User gửi 15 request liên tiếp trong 3 giây. Cost guard thấy số dư còn nhiều nên cho phép, nhưng Rate limiter phát hiện vượt quá 10 req/phút nên chặn từ request thứ 11 với mã `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
> 2. Probe gộp chung (vừa liveness vừa readiness) kiểm tra Redis và thấy lỗi kết nối, đồng loạt trả về `503` trên cả 3 container agent.
> 3. Orchestrator (Docker / Kubernetes) nhận thấy liveness probe thất bại, suy đoán rằng các process container đã bị treo/hỏng và lập tức tiến hành restart cả 3 container cùng lúc.
> 4. Quá trình restart đột ngột làm đứt ngang toàn bộ các request người dùng đang được xử lý dở dang (người dùng gặp lỗi 502 Bad Gateway).
> 5. Khi các container khởi động lại nhưng Redis vẫn chưa xong 30s sự cố, chúng tiếp tục fail probe và bị restart theo chu kỳ (crash-loop backoff).
> 6. Sự cố tạm thời của Redis (chỉ 30 giây) bị khuếch đại thành thảm họa sập toàn bộ dịch vụ (cascading failure). Nếu phân tách đúng, `/ready` trả 503 để Load Balancer tạm ngưng rót request, còn `/health` vẫn 200 để giữ container sống chờ Redis online lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong RAM bằng dict Python:
> Vì Load Balancer phân bổ tuần tự (round-robin) các request tới 3 container A, B, C độc lập:
> - Request 1 tới container A: A lưu vào RAM của A, trả về `history_length = 0`.
> - Request 2 tới container B: B có RAM riêng rỗng, không biết gì về A, trả về `history_length = 0`.
> - Request 3 tới container C: C cũng có RAM riêng rỗng, trả về `history_length = 0`.
> - Request 4 quay lại container A: A nhớ câu 1, trả về `history_length = 2`.
> Kết quả là `history_length` sẽ nhảy lộn xộn (0, 0, 0, 2, 0, 2...) tùy vào container tiếp nhận request, khiến agent như bị "mất trí nhớ ngẫu nhiên".
> Khi dùng Redis tập trung, mọi container cùng truy cập chung một nguồn dữ liệu nên `history_length` luôn tăng đều đặn: 0 -> 2 -> 4 -> 6...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải:** Khi Docker build chạy lệnh `RUN pip install --no-cache-dir --prefix=/install -r requirements.txt`, quá trình bị lỗi:
>   `ERROR: Could not find a version that satisfies the requirement fastapi>=0.110 (from versions: none)`
>   `socket.gaierror: [Errno -3] Temporary failure in name resolution`
> - **Nguyên nhân:** Môi trường mạng Wi-Fi trường VinUni (`VinUni.local`) chặn các DNS công cộng như `8.8.8.8` (DNS mặc định mà Docker bridge sử dụng khi host dùng `systemd-resolved` 127.0.0.53). Docker container bên trong không phân giải được tên miền `pypi.org` dẫn đến `pip install` không thể kết nối tới kho lưu trữ package.
> - **Cách sửa:**
>   1. Cấu hình DNS của Docker daemon trong `/etc/docker/daemon.json` trỏ tới DNS nội bộ của mạng VinUni:
>      ```json
>      {"dns": ["10.140.64.98", "10.140.64.99", "1.1.1.1"]}
>      ```
>   2. Bổ sung tham số retry và timeout dài hơn vào `Dockerfile` để chống chập chờn mạng:
>      `RUN pip install --no-cache-dir --default-timeout=100 --retries 5 --prefix=/install -r requirements.txt`
>   Sau khi sửa, container build ổn định và deploy thành công lên Render với kết nối nội bộ Redis hoạt động trơn tru.
