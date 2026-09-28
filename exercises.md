# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay thế các dòng placeholder mẫu bằng câu trả lời chi tiết.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Quốc Hiệp  Mã học viên: 2A202602755

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên môi trường staging hoặc production, kỹ sư vô tình quên cấu hình biến môi trường `AGENT_API_KEY` (hoặc cấu hình sai tên biến trên dashboard). Nếu hệ thống gán mặc định `"changeme"`, ứng dụng vẫn khởi động trơn tru và báo healthy, nhưng bất kỳ ai trên Internet cũng có thể đoán ra chuỗi `"changeme"` để truy cập API trái phép, làm rò rỉ dữ liệu hoặc tiêu cạn hạn mức ngân sách LLM của hệ thống. Với cơ chế fail fast không có giá trị mặc định, app lập tức dừng lại với ngoại lệ `ValidationError` ngay lúc nạp cấu hình, buộc lập trình viên phải cung cấp khóa hợp lệ trước khi service có thể nhận bất kỳ request nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được trong thực tế:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:51:24.123456Z", "user_id": "sv-test", "tokens_in": 43, "tokens_out": 47, "cost_usd": 3.465e-05}
```

Hai việc làm được với log JSON:
1. **Lọc và truy vấn có cấu trúc (Structured Querying)**: Các hệ thống phân tích log tập trung (Datadog, Grafana Loki, ELK, CloudWatch) có thể bóc tách tự động các trường key-value, cho phép lập tức tìm kiếm chính xác theo điều kiện: `user_id == "sv-test" AND cost_usd > 0.0001` hoặc truy vết phiên làm việc mà không cần viết các biểu thức chính quy (regex) phức tạp và dễ gãy.
2. **Tổng hợp số liệu và cảnh báo định lượng (Metrics Aggregation & Alerting)**: Có thể trực tiếp tính toán tổng chi phí (`sum(cost_usd)`), lượng token tiêu thụ trung bình theo từng user, và thiết lập cảnh báo tự động khi phát hiện chi phí hoặc số token đột biến trong khoảng thời gian ngắn.

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
| 1 stage (bản đầu) | 1190 MB |
| Multi-stage | 220 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch gần 970 MB bao gồm:
1. **Hệ điều hành cơ sở đầy đủ so với bản slim**: Bản 1 stage dùng `python:3.11` chứa đầy đủ thư viện C/C++, trình biên dịch gcc, build-essential, gdb, tài liệu trợ giúp và các tiện ích hệ điều hành Debian đầy đủ.
2. **Tách biệt môi trường build và runtime**: Bản multi-stage tách riêng stage `builder` để chạy `pip install`, sau đó chỉ sao chép các package đã cài hoàn chỉnh sang stage runtime `python:3.11-slim`. Toàn bộ cache của pip, file object trung gian `.o` và header biên dịch đều bị loại bỏ hoàn toàn khỏi image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi chỉ sửa một ký tự trong `app/main.py`: Các layer trước đó gồm `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install`, layer tạo user và copy thư viện `/install` đều giữ nguyên checksum nên được tận dụng 100% từ Docker cache (`CACHED`). Chỉ các layer từ `COPY app ./app` trở đi mới bị cache invalidation và chạy lại, toàn bộ quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` trước `RUN pip install`: Bất kỳ một chỉnh sửa nhỏ nào ở mã nguồn cũng làm thay đổi checksum của bước `COPY . .`, khiến Docker hủy cache của tất cả các layer phía sau. Hậu quả là mỗi lần đổi code dù chỉ 1 dòng, Docker đều buộc phải tải và cài đặt lại toàn bộ thư viện trong `requirements.txt` từ đầu, gây lãng phí băng thông và thời gian build hàng chục lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo thang đặc quyền:
  1. Kẻ tấn công phát hiện một lỗ hổng RCE (Remote Code Execution) hoặc Command Injection trong ứng dụng Python hoặc qua một dependency độc hại.
  2. Mã độc được thực thi bên trong container với tư cách user sở hữu tiến trình. Nếu container không chỉ định `USER`, tiến trình chạy bằng `root` (UID 0).
  3. Kẻ tấn công có toàn quyền root trong namespace container. Nếu máy chủ host hoặc container runtime tồn tại lỗ hổng container breakout (như lỗi kernel, mount nhạy cảm `/var/run/docker.sock` hoặc privilege escalation), kẻ tấn công với UID 0 có thể phá vỡ ranh giới cô lập để truy cập trực tiếp filesystem và tiến trình của host máy chủ với quyền root.
- Lệnh `USER appuser` cắt đứt chuỗi ngay tại Bước 2: Khi tiến trình bị giới hạn ở quyền người dùng thông thường (`UID 10001`), kẻ tấn công dù khai thác được RCE cũng chỉ có quyền đọc/ghi tối thiểu trong thư mục ứng dụng, không thể sửa đổi cấu hình hệ thống, không có quyền can thiệp vào kernel hay các socket nhạy cảm, ngăn chặn hoàn toàn việc escape ra ngoài máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được: Người dùng chờ đến giây `00:59` của phút thứ nhất và gửi dồn dập 10 request (đạt đúng hạn mức 10/phút của phút thứ nhất). Ngay 1 giây sau đó, đồng hồ bước sang giây `01:00` (đầu phút thứ hai), bộ đếm cố định reset về 0, người dùng gửi thêm 10 request nữa. Tổng cộng trong khoảng thời gian chỉ 2 giây (từ 00:59 đến 01:00), hệ thống đã phải gánh 20 request, gây ra hiện tượng bùng nổ lưu lượng ở ranh giới (boundary burst). Cửa sổ trượt (sliding window) khắc phục triệt để lỗi này bằng cách luôn tính toán tổng request trong đúng 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau: Rate limit kiểm soát vận tốc/tần suất yêu cầu trong một khoảng thời gian ngắn (ví dụ tối đa 10 req/phút) nhằm bảo vệ hạ tầng máy chủ khỏi nguy cơ quá tải và từ chối dịch vụ (DoS). Cost guard kiểm soát tổng lượng ngân sách tài chính tích lũy trong một chu kỳ dài (ví dụ tối đa 10.0 USD/tháng) nhằm bảo vệ ví tiền của tổ chức trước chi phí API LLM.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: Một người dùng gửi request đầu tiên trong ngày (tần suất 1 req/phút, hoàn toàn nằm trong hạn mức tốc độ), nhưng trong tháng tài khoản này đã tiêu hết 10.0 USD ngân sách định mức (`spent >= budget`). Cost guard sẽ chặn ngay lập tức và trả về mã lỗi `402 Payment Required`.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: Một người dùng mới toanh chưa tiêu đồng nào trong tháng (ngân sách còn nguyên 10.0 USD, chi phí cuộc gọi chỉ khoảng 0.00003 USD, Cost guard hoàn toàn đồng ý), nhưng người này dùng script bắn liên tục 15 request trong 3 giây. Từ request thứ 11, Rate limiter sẽ kích hoạt và trả về `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Khi Redis gặp sự cố mạng hoặc khởi động lại trong 30 giây, endpoint gộp sẽ trả về `503 Service Unavailable`.
2. Do kiểm tra liveness bị thất bại, bộ điều phối container (Docker daemon / Kubernetes kubelet) xác định rằng tiến trình app đã bị hỏng/deadlock.
3. Bộ điều phối lập tức gửi tín hiệu giết (SIGKILL/SIGTERM) và khởi động lại liên tục cả 3 container agent trong cụm, rơi vào vòng lặp khởi động chết chóc (CrashLoopBackOff).
4. Các container khởi động lại tốn CPU, giải phóng connection pool và không thể phục vụ ngay cả những tài nguyên tĩnh hay các tác vụ độc lập với Redis.
5. Sau 30 giây khi Redis hoạt động bình thường trở lại, các container vẫn đang dở dang trong quá trình khởi động lại hoặc restart backoff, kéo dài thời gian gián đoạn dịch vụ và có thể gây nghẽn cổ chai kết nối đồng loạt vào Redis (thundering herd).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (kiến trúc Stateless): Lịch sử hội thoại được chia sẻ tập trung. Dù request của người dùng được bộ cân bằng tải phân phối ngẫu nhiên hoặc xoay vòng (round-robin) đến container agent-1, agent-2 hay agent-3, cả 3 node đều đọc và ghi vào cùng một key Redis. Do đó `history_length` tăng tuần tự và nhất quán sau mỗi lượt hỏi đáp (2, 4, 6, 8...).
- Nếu lưu trong một dict Python nội bộ (kiến trúc Stateful): Bộ nhớ RAM của 3 tiến trình là hoàn toàn độc lập. Người dùng gửi request 1 vào agent-1 thì agent-1 ghi nhớ (`length=2`). Request 2 rơi vào agent-2, agent-2 không hề biết gì về request trước nên thấy lịch sử rỗng (`length=0`) và ghi mới (`length=2`). Request 3 rơi vào agent-3 cũng tương tự. Kết quả là `history_length` sẽ nhảy lộn xộn, lúc tăng lúc giảm về 0 tùy thuộc request rơi vào container nào, khiến ngữ cảnh hội thoại của người dùng bị phân mảnh và gián đoạn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi gặp phải: Khi containerize và khởi chạy môi trường Docker, quá trình build bị dừng lại với thông báo lỗi: `process "/bin/sh -c useradd --create-home --uid 10001 appuser" did not complete successfully: exit code: 9 (useradd: user 'appuser' already exists)` và ở bước cài đặt package builder xuất hiện lỗi `OSError: [Errno 13] Permission denied: '/install'`.
- Cách tìm ra nguyên nhân: Qua kiểm tra chi tiết cấu hình base image bằng `docker image inspect` và xem logs build, phát hiện base image runtime đã có sẵn định danh `appuser` từ trước và đang đặt quyền mặc định là user không có quyền quản trị, dẫn đến việc không thể tạo lại user trùng tên và không có quyền ghi thư mục `/install` ở stage builder.
- Cách khắc phục: Bổ sung chỉ thị `USER root` cho stage `builder` để đảm bảo quyền tạo thư mục `/install` khi chạy `pip install --prefix=/install`. Đồng thời tại stage runtime, cập nhật câu lệnh tạo user thành kiểm tra điều kiện tồn tại trước khi tạo: `RUN id -u appuser >/dev/null 2>&1 || useradd --create-home --uid 10001 appuser`. Sau điều chỉnh này, Docker image build hoàn toàn trơn tru và đạt chuẩn non-root khi chạy.
