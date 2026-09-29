# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> Họ và tên: Tạ Đăng Dương  Mã học viên: 2A202603018

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định `"changeme"`, khi deploy lên cloud mà người quản trị quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn âm thầm khởi động bình thường (silent failure). Khi đó, các bot quét tự động trên internet có thể thử khóa phổ biến `"changeme"` để gọi trái phép vào endpoint `/ask`, làm rò rỉ dữ liệu hoặc đốt sạch hạn mức token LLM. Cơ chế "fail fast" bằng cách bỏ giá trị mặc định buộc Pydantic văng lỗi `ValidationError` ngay lúc khởi động, làm container dừng ngay từ đầu và kích hoạt cảnh báo deploy failed trên dashboard, ngăn chặn một service không an toàn lộ diện ra môi trường production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "INFO", "timestamp": "2026-09-29T04:53:58.123456Z", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 0.00002145}

```

Hai việc làm được với log JSON:

1. **Lọc và truy vấn tự động (Log Aggregation):** Các hệ thống thu thập log tập trung (Datadog, Loki, CloudWatch, Elasticsearch) có thể tự động bóc tách (parse) từng trường để truy vấn, nhóm log theo `user_id` cụ thể hoặc theo dõi sự kiện `ask_completed` mà không cần viết các biểu thức chính quy (regex) phức tạp.
2. **Thiết lập cảnh báo và trực quan hóa chi phí (Alerting & Metrics):** Có thể trích xuất trực tiếp các trường số học như `cost_usd`, `tokens_in`, `tokens_out` để dựng dashboard theo dõi mức tiêu thụ tài nguyên theo thời gian thực và kích hoạt cảnh báo tự động khi phát hiện chi phí của một user tăng đột biến.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent

```

| Bản | Dung lượng |
| --- | --- |
| 1 stage (bản đầu) | 685 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~414 MB) bao gồm:

* Cache tải về của pip trong quá trình cài đặt package (`~/.cache/pip`).
* Các công cụ build tạm thời, header file C/C++ và các file wheel trung gian dùng để biên dịch các thư viện Python có extension native.
* Trong multi-stage build, toàn bộ dependencies được cài đặt vào thư mục `/install` ở stage `builder`. Stage `runtime` chỉ copy thành phẩm sang base image `python:3.11-slim` sạch và loại bỏ hoàn toàn các tệp tạm thời cũng như công cụ biên dịch không cần thiết khi chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

* Các layer được dùng lại từ cache: Toàn bộ các layer từ đầu cho đến stage `builder`: tải base image, tạo thư mục làm việc, `COPY requirements.txt .`, và quan trọng nhất là lệnh `RUN pip install ...` vì file `requirements.txt` không hề thay đổi.
* Layer phải chạy lại: Bắt đầu từ layer `COPY . .` ở stage runtime (do phát hiện thay đổi nội dung file nguồn làm bust cache) và các lệnh tiếp theo như `RUN chown ...`.
* Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất cứ khi nào sửa đổi dù chỉ một ký tự code trong `app/`, cache của layer `COPY . .` sẽ bị vô hiệu hóa. Kéo theo lệnh `RUN pip install` ngay phía sau cũng bị mất cache và buộc phải tải, cài đặt lại toàn bộ thư viện từ đầu, khiến thời gian build kéo dài thêm vài phút thay vì chỉ mất 1-2 giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

* Chuỗi sự kiện leo thang đặc quyền:
1. Ứng dụng Python tồn tại lỗ hổng bảo mật (ví dụ: Remote Code Execution, command injection hoặc lỗi đọc/ghi file tùy ý).
2. Kẻ tấn công gửi payload khai thác thành công để thực thi shellcode. Do tiến trình chạy dưới quyền mặc định của container là `root` (UID 0), kẻ tấn công chiếm toàn quyền kiểm soát bên trong container.
3. Kẻ tấn công tận dụng quyền root container để khai thác tiếp các lỗi thoát container (container breakout) như lỗi nhân Linux, quyền privileged, hoặc can thiệp vào các volume mount nhạy cảm (như docker socket `/var/run/docker.sock`) để chiếm quyền điều khiển trực tiếp máy host với đặc quyền root.


* Điểm cắt đứt của lệnh `USER`: Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay từ bước 2. Khi mã độc được kích hoạt, nó chỉ chạy dưới quyền của một user thông thường không có đặc quyền (unprivileged). Kẻ tấn công không thể sửa file hệ thống của container, không có quyền can thiệp vào các tiến trình khác và không đủ điều kiện đặc quyền để thực hiện các kỹ thuật breakout leo thang lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.

Giải thích:
Với cơ chế cửa sổ cố định (fixed window) reset vào giây 00 mỗi phút:

* Tại giây `10:00:59` (giây cuối cùng của phút thứ 10), người dùng gửi 10 request. Hệ thống ghi nhận 10/10 request trong phút 10 và cho phép toàn bộ.
* Ngay khi đồng hồ chuyển sang `10:01:00`, bộ đếm được reset về 0.
* Tại giây `10:01:01` (giây đầu tiên của phút 11), người dùng gửi tiếp 10 request nữa. Hệ thống ghi nhận 10 request này thuộc hạn mức của phút 11 nên tiếp tục cho phép qua.
Như vậy trong khoảng thời gian từ `10:00:59` đến `10:01:01` (chỉ vỏn vẹn 2 giây), người dùng đã đẩy vào hệ thống 20 request mà không bị chặn, gây đột biến tải gấp đôi cho server.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

* Khác biệt cơ bản:
* **Rate limit**: Giới hạn *tần suất/số lượng request* trong một khung thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ khả năng chịu tải và chống nghẽn dịch vụ.
* **Cost guard**: Giới hạn *tổng chi phí tài chính/lượng token* tiêu thụ trong chu kỳ dài (ví dụ: $10.0/tháng) để bảo vệ ngân sách trước chi phí gọi LLM.


* Rate limit cho qua nhưng Cost guard chặn: Người dùng chỉ gửi 1 request duy nhất trong ngày (hoàn toàn hợp lệ theo rate limit 10 req/phút), nhưng nội dung request gửi kèm một văn bản rất dài khiến chi phí ước tính vượt quá ngân sách tháng $10.0 còn lại $\to$ Cost guard chặn và trả về lỗi HTTP 402 Payment Required.
* Cost guard cho qua nhưng Rate limit chặn: Người dùng còn nguyên ngân sách tháng $10.0 và chỉ gửi các câu hỏi cực ngắn tốn $0.00002 mỗi lần (chi phí không đáng kể, Cost guard cho qua), nhưng gửi dồn dập 15 request chỉ trong vòng 3 giây $\to$ Rate limit phát hiện vượt quá 10 req/phút và trả về mã lỗi HTTP 429 Too Many Requests từ request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
2. Endpoint gộp chung (đóng vai trò cả liveness probe) kiểm tra thấy Redis không phản hồi nên trả về HTTP 503 (unhealthy).
3. Bộ điều phối (orchestrator như Docker Swarm/Kubernetes/Render) nhận tín hiệu liveness fail nên kết luận tiến trình đã bị chết hoặc deadlock, lập tức gửi SIGKILL để buộc dừng và restart cả 3 container của service.
4. Ba container mới khởi động lại và lập tức gọi kiểm tra Redis (lúc này vẫn chưa kết nối lại được) $\to$ Tiếp tục trả về 503 $\to$ Orchestrator lại tiếp tục tiêu diệt và khởi động lại container liên tục.
5. Xảy ra hiện tượng "bão khởi động lại" (restart storm / crashloop), lãng phí tài nguyên CPU/RAM và làm gián đoạn hoàn toàn mọi request đang xử lý dở thay vì chỉ tạm dừng điều hướng traffic mới (như đúng vai trò của readiness probe).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

* Khi lưu bằng Redis (stateless): Giá trị `history_length` tăng tuần tự và nhất quán qua từng request ($0 \to 2 \to 4 \to 6 \dots$) dù request được phân bổ tới bất kỳ instance nào trong cụm 3 container.
* Nếu lưu bằng dict trong RAM (stateful): Khi có 3 instance chạy song song và load balancer phân phối request xoay vòng (round-robin), mỗi container chỉ lưu giữ một dict độc lập trong bộ nhớ của nó. Người dùng sẽ thấy `history_length` nhảy bất thường và ngắt quãng (ví dụ: request 1 vào container A trả về 0, request 2 vào container B vẫn trả về 0, request 3 vào container C vẫn trả về 0; request 4 quay lại A mới tăng lên 2...). Hệ thống sẽ có biểu hiện "mất trí nhớ" vì các container không chia sẻ trạng thái bộ nhớ với nhau.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

* **Thông báo lỗi:** Khi chạy `pytest tests/test_cp5.py`, 4 test tài liệu bị FAILED với lỗi `AssertionError: DEPLOYMENT.md còn chỗ trống '(điền ...)' chưa hoàn thiện`.
* **Nguyên nhân:** Hàm kiểm tra `require_filled` trong file test sử dụng biểu thức chính quy quét chuỗi `re.search(r"\(điền", text)` để kiểm tra tài liệu. Mặc dù service trên Render đã triển khai thành công và vượt qua các test kết nối, file `DEPLOYMENT.md` vẫn còn sót lại chữ mẫu `(điền: Redis add-on của platform / Upstash / ...)` tại mục `REDIS_URL` và đoạn ghi chú `(điền lý do nếu dùng phương án dự phòng...)` ở cuối trang.
* **Cách sửa:** Chỉnh sửa lại bảng biến môi trường trong `DEPLOYMENT.md`, cập nhật nguồn của `REDIS_URL` thành `Render Key Value (kết nối nội bộ qua Blueprint)`, đồng thời xóa hoàn toàn phần `## Nếu Dùng Phương Án Dự Phòng` ở cuối tài liệu. Chạy lại `pytest tests/test_cp5.py` và tất cả các test tài liệu lẫn live check đều đạt trạng thái PASSED.