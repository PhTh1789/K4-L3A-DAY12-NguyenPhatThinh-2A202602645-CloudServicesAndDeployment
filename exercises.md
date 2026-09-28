# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Phát Thịnh Mã học viên: 2A202602645

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Trong môi trường production, nếu kỹ sư deploy một phiên bản mới nhưng quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard hoặc file cấu hình hệ thống:

- Nếu để giá trị mặc định (như `"changeme"`): Service vẫn khởi động thành công và báo trạng thái healthy. Khi đó, endpoint `/ask` bị mở toang với một khóa công khai mặc định mà bất kỳ ai hay bot quét internet nào cũng biết. Kẻ tấn công có thể spam liên tục làm tiêu sạch ngân sách token LLM, hoặc người dùng hợp lệ dùng khóa thật thì bị từ chối truy cập. Đây là lỗi diễn ra âm thầm (silent failure) và cực kỳ khó truy vết.
- Khi "chết sớm" (Fail fast): Do `agent_api_key` không có giá trị mặc định, Pydantic `BaseSettings` sẽ ném ngoại lệ `ValidationError` ngay lúc khởi động tiến trình, làm ứng dụng dừng ngay lập tức với exit code 1. Nền tảng cloud/orchestrator (như Docker, Kubernetes, Railway) sẽ lập tức phát hiện container không khởi động được, ngăn chặn việc rollout phiên bản lỗi ra công chúng, và phát cảnh báo đến đội ngũ kỹ thuật để khắc phục trước khi có bất kỳ rủi ro bảo mật nào xảy ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được khi gọi `/ask`:

```json
{
  "event": "ask_completed",
  "level": "info",
  "service": "day12-agent",
  "timestamp": "2026-09-28T15:09:41.123456Z",
  "user_id": "cp5-test",
  "tokens_in": 43,
  "tokens_out": 47,
  "cost_usd": 0.00003465
}
```

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không thể làm được:

1. **Truy vấn, lọc và cảnh báo tự động trên các hệ thống giám sát tập trung (ELK Stack, Datadog, Grafana Loki, CloudWatch):** Máy tính và parser có thể tự động bóc tách từng trường (field) để thực hiện các bộ lọc chính xác cao như: tìm các request có `cost_usd > 0.001`, lọc lỗi theo `level = "error"`, hoặc nhóm theo `user_id` để phát hiện tài khoản có hành vi bất thường mà không cần viết regex phức tạp.
2. **Xây dựng Dashboard phân tích chi phí và hiệu năng (FinOps & APM Metrics):** Dữ liệu JSON có kiểu số rõ ràng cho phép hệ thống phân tích tổng hợp các chỉ số định lượng như tính tổng chi phí `sum(cost_usd)`, tính trung bình token tiêu thụ theo thời gian thực, hoặc vẽ biểu đồ biến động chi phí token theo từng giờ/ngày, phục vụ quản trị ngân sách và thiết lập alert tự động khi chi phí tăng đột biến.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản               | Dung lượng |
| ----------------- | ---------- |
| 1 stage (bản đầu) | 1020 MB    |
| Multi-stage       | 271 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~749 MB) bao gồm:

1. **Bộ công cụ biên dịch C và header hệ thống (Build Tools):** Bản base image đầy đủ (`python:3.11`) chứa cả `gcc`, `g++`, `make`, các thư viện phát triển `*-dev`, bộ mã nguồn test suite của Python và nhiều tiện ích hệ điều hành Debian không cần thiết trong quá trình runtime. Bản `python:3.11-slim` đã loại bỏ hoàn toàn các gói thừa này.
2. **Cơ chế tách rời của Multi-stage build:** Toàn bộ cache bánh nướng của pip (`~/.cache/pip`), các file build trung gian, mã nguồn các package tạm thời chỉ tồn tại trong stage `builder` và bị vứt bỏ hoàn toàn. Stage `runtime` cuối cùng chỉ copy đúng kết quả đã cài đặt (`/install` sang `/usr/local`) và source code ứng dụng, giúp image đạt kích thước tối ưu (content size nén khi đẩy lên registry chỉ còn ~63.9 MB).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Những layer được dùng lại từ cache và layer phải chạy lại:**
  - Các layer trước đó: `FROM python:3.11-slim ...`, `WORKDIR /app`, `COPY requirements.txt .`, và `RUN pip install ...` đều được **dùng lại 100% từ Docker cache** vì file `requirements.txt` và các câu lệnh trước đó hoàn toàn không thay đổi.
  - Chỉ có layer `COPY app ./app` và các bước kế tiếp (`USER appuser`, `CMD`) là bị vô hiệu hóa cache và phải chạy lại. Nhờ đó, thời gian build lại cực kỳ nhanh (chỉ mất chưa đầy 1-2 giây).

- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Bất kỳ khi nào sửa đổi dù chỉ một ký tự trong `app/main.py` hay bất kỳ file nào trong source code, layer `COPY . .` sẽ bị thay đổi checksum, dẫn tới toàn bộ cache từ layer đó trở đi bị vô hiệu hóa (cache bust).
  - Kết quả là lệnh `RUN pip install` bắt buộc phải tải và cài đặt lại toàn bộ các thư viện từ đầu trong mỗi lần build, khiến thời gian build tăng từ vài giây lên vài phút, lãng phí tài nguyên CPU và băng thông mạng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện từ lỗ hổng code đến chiếm quyền máy host:**
  1. Ứng dụng Python tồn tại lỗ hổng (ví dụ: thực thi mã từ xa Remote Code Execution thông qua `pickle.loads` không an toàn, hoặc Command Injection qua `os.system` / `subprocess`).
  2. Kẻ tấn công gửi payload khai thác thành công và chiếm được quyền thực thi interactive shell bên trong container.
  3. Nếu container chạy với quyền mặc định là `root` (UID 0), kẻ tấn công trong container chính là root (UID 0). Họ có toàn quyền ghi đè các file hệ thống nhạy cảm, cài thêm công cụ tấn công bằng `apt`, và quan trọng nhất là có thể khai thác các lỗ hổng bảo mật của Linux Kernel (như Dirty COW, cgroup release agent breakout) hoặc docker.sock được mount để vượt rào (container breakout) ra ngoài hệ điều hành máy host.
  4. Sau khi thoát ra ngoài host với UID 0, kẻ tấn công chiếm toàn quyền kiểm soát máy chủ vật lý/cloud của hệ thống.

- **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
  Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay tại **bước 3**. Kẻ tấn công khi có shell chỉ mang quyền hạn của một người dùng thông thường không có đặc quyền (unprivileged user): không thể ghi vào các thư mục hệ thống như `/usr` hay `/bin`, không thể chạy lệnh `apt/sudo`, không thể truy cập tài nguyên của tiến trình khác, và không đủ quyền hạn (capabilities) để khai thác các lỗ hổng leo thang đặc quyền hay container breakout sang máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức 10 request/phút, nếu dùng cách đếm theo phút đồng hồ (Fixed Window reset lúc giây 00):

- **Số request tối đa:** Người dùng có thể gửi tối đa **20 request** chỉ trong vòng 2 giây liên tiếp.
- **Cách đạt được:**
  - Vào giây thứ `00:59` (giây cuối cùng của phút thứ nhất), người dùng gửi dồn dập 10 request. Hệ thống kiểm tra thấy chưa vượt quá 10 req trong phút này nên cho qua toàn bộ (10/10).
  - Ngay khi đồng hồ chuyển sang giây `01:00` (giây đầu tiên của phút thứ hai), bộ đếm Fixed Window bị reset về 0. Người dùng lập tức gửi tiếp 10 request nữa. Hệ thống thấy phút mới chưa có request nào nên tiếp tục cho qua toàn bộ (10/10).
  - Như vậy, trong khoảng thời gian chỉ 2 giây (từ 00:59 đến 01:01), hệ thống đã phải chịu tải tới 20 request (gấp đôi hạn mức cho phép), có thể gây nghẽn hoặc sập các dịch vụ phía sau (spike traffic).
- **Ngược lại với Sliding Window (Sorted Set):** Thuật toán luôn tính chính xác trong khoảng thời gian trượt liên tục `[now - 60s, now]`, nên tại bất kỳ thời điểm nào nhìn về 60 giây trước đó, số lượng request không bao giờ vượt quá 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác nhau giữa hai cơ chế:**
  - **Rate Limit:** Bảo vệ **hạ tầng và tính sẵn sàng** của hệ thống trong **ngắn hạn** (cửa sổ 60 giây). Đo lường theo tần suất cuộc gọi (Request per Minute - RPM) để chống nghẽn mạng, chống tấn công DoS/Brute-force. Khi vi phạm trả về mã lỗi `429 Too Many Requests`.
  - **Cost Guard:** Bảo vệ **ngân sách tài chính** của doanh nghiệp trong **dài hạn** (chu kỳ tháng). Đo lường theo tổng chi phí tích lũy (USD) được tính toán dựa trên số lượng token thực tế (input + output) tiêu thụ qua các mô hình AI/LLM. Khi vi phạm trả về mã lỗi `402 Payment Required`.

- **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
  User trong cả tháng đã tương tác rất nhiều và chạm ngưỡng ngân sách $10.0 của tháng đó. Sau một thời gian nghỉ (hơn 1 tiếng), user gửi một câu hỏi duy nhất. Tần suất là 1 request/phút (hoàn toàn nằm trong hạn mức 10 req/phút của Rate Limit -> Rate Limit cho qua), nhưng Cost Guard kiểm tra thấy tổng chi tiêu đã là $10.0 (hết ngân sách) nên lập tức chặn lại và trả về lỗi `402 Payment Required`.

- **Tình huống Cost Guard cho qua nhưng Rate Limit chặn:**
  User mới bắt đầu chu kỳ tháng, ngân sách còn nguyên $10.0 chưa tiêu đồng nào. Trong vòng 5 giây đầu tiên, user gửi liên tiếp 11 request ngắn. Tổng chi phí của 11 request này chỉ tốn khoảng $0.0003 (hoàn toàn nằm trong ngân sách -> Cost Guard cho qua), nhưng vì gọi quá nhanh vượt quá 10 req/phút, Rate Limit sẽ lập tức chặn từ request thứ 11 và trả về lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp hai endpoint làm một và cho `/health` kiểm tra kết nối Redis, khi Redis mất kết nối trong 30 giây, các sự kiện sẽ diễn ra theo đúng thứ tự sau:

1. **Redis gặp sự cố:** Tiến trình Redis bị restart, nghẽn mạng hoặc mất kết nối tạm thời trong 30 giây.
2. **Health check đồng loạt thất bại:** Bộ giám sát (orchestrator như Kubernetes / Docker / Cloud platform) định kỳ gọi vào `/health` của cả 3 container agent. Do `/health` kiểm tra Redis và Redis đang chết, cả 3 container đều trả về HTTP 503.
3. **Orchestrator giết toàn bộ container:** Vì `/health` đóng vai trò là Liveness Probe ("tiến trình còn sống để phục hồi không?"), orchestrator hiểu rằng cả 3 container agent đều đã bị treo/hỏng nghiêm trọng và tiến hành **Restart (SIGKILL và tạo mới)** đồng loạt cả 3 container agent.
4. **Vòng xoáy khởi động lại (CrashLoopBackOff):** Các container mới được bật lên, tiếp tục thực hiện health check khi Redis vẫn chưa kịp phục hồi sau 30 giây. Chúng lại tiếp tục trả về 503 và lại bị orchestrator giết liên tục.
5. **Sập toàn bộ hệ thống (Cascading Failure):** Mọi kết nối của người dùng đang được xử lý dở dang bị ngắt đột ngột (lỗi 502 Bad Gateway). Khi Redis phục hồi xong, toàn bộ cụm agent vẫn đang loay hoay trong quá trình reboot hoặc bị áp đặt cơ chế backoff chờ khởi động. Một sự cố tạm thời ở tầng phụ thuộc (dependency) đã bị khuếch đại thành sự cố sập toàn diện hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Nếu lưu lịch sử trong một `dict` Python trong RAM (Stateful):**
  - Khi scale lên 3 container (`agent=3`) sau Load Balancer, mỗi container là một tiến trình riêng biệt với vùng nhớ RAM độc lập. Load Balancer sẽ phân phối các request của cùng một người dùng luân phiên qua các container theo thuật toán Round-Robin:
    - **Lượt 1 (vào Container A):** `history_length = 0`. Container A lưu câu hỏi và câu trả lời vào `dict` trong RAM của mình.
    - **Lượt 2 (vào Container B):** `history_length = 0`! Container B nhận request nhưng trong RAM của B chưa từng lưu thông tin user này, dẫn đến agent bị "mất trí nhớ".
    - **Lượt 3 (vào Container C):** `history_length = 0`! Tương tự, RAM của C hoàn toàn trống.
    - **Lượt 4 (vào Container A):** `history_length = 2`! Vì request tình cờ rơi lại trúng Container A, nơi có lưu dữ liệu từ lượt 1.
  - Người dùng sẽ thấy hiện tượng độ dài lịch sử nhảy loạn xạ (`0 -> 0 -> 0 -> 2 -> 2...`) và bot lúc nhớ lúc quên tùy theo request rơi trúng container nào. Ngoài ra nếu container bị restart thì toàn bộ lịch sử biến mất.

- **Khi dùng Redis (Stateless):**
  Cả 3 container đều chỉ là worker không trạng thái (stateless), mọi thao tác đọc/ghi lịch sử đều hướng về một cụm Redis tập trung. Dù Load Balancer đẩy request vào bất kỳ container nào, container đó cũng đọc được đúng lịch sử hiện tại từ Redis. Kết quả là `history_length` luôn tăng đều đặn và chuẩn xác: `0 -> 2 -> 4 -> 6 -> 8...`.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Trong quá trình triển khai hệ thống lên Railway ở Checkpoint 5, lỗi thực tế gặp phải là:

- **Thông báo lỗi:** Khi deploy thành công container và gọi thử endpoint `/ready`, server phản hồi `HTTP 500 Internal Server Error`.
- **Cách tìm ra nguyên nhân:**
  1. Kiểm tra endpoint `/health` thấy vẫn trả về `200 OK`, chứng tỏ container Docker và tiến trình Uvicorn vẫn đang sống bình thường.
  2. Xem xét mã nguồn trong `app/main.py`: endpoint `/ready` có dependency `store: ConversationStore = Depends(get_store)`. Hàm `get_redis_client()` sử dụng lệnh `redis.from_url(url)`.
  3. Kiểm tra biến môi trường `REDIS_URL` trên Railway: ban đầu biến được nhập dưới dạng chuỗi tham chiếu `${{Redis.REDIS_URL}}` hoặc bị trống, khiến thư viện `redis-py` ném ngoại lệ `ValueError: Redis URL must specify one of the following schemes (redis://, rediss://, unix://)`. Do ngoại lệ xảy ra trong dependency injection trước khi vào khối try/except của hàm `ready`, FastAPI đã chuyển thành lỗi 500.
- **Cách sửa:**
  1. Truy cập vào dashboard Railway, bấm vào service `Redis` và vào tab Variables để lấy chuỗi kết nối trực tiếp (`redis://default:mật_khẩu@redis.railway.internal:6379`).
  2. Quay lại service `Agent`, dán chuỗi kết nối chuẩn này vào biến môi trường `REDIS_URL`.
  3. Sau khi Railway tự động khởi động lại service với cấu hình mới, endpoint `/ready` lập tức phản hồi thành công `HTTP 200 OK` với nội dung `{"status": "ready", "redis": true}`.
