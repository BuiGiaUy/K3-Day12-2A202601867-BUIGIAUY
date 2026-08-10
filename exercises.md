# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng mẫu bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bui Gia Uy  Mã học viên: 2A202601867

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ khi deploy lên Railway, nếu quên đặt `AGENT_API_KEY`, `Settings` sẽ báo lỗi validation ngay lúc container khởi động và deployment bị unhealthy. Điều này giúp phát hiện lỗi trước khi service nhận request. Nếu đặt mặc định là `changeme`, app vẫn online với một secret ai cũng biết; kẻ khác có thể gọi `/ask`, làm phát sinh chi phí và truy cập dữ liệu. Vì vậy fail fast an toàn hơn nhiều.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log tôi quan sát được sau khi gọi /ask là:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-08-10T05:01:28.570366+00:00","user_id":"cp5-test","tokens_in":45,"tokens_out":45,"cost_usd":0.00003375}
```

Với dòng này, tôi có thể lọc và đếm request theo user_id, event hoặc khoảng thời gian để tìm lỗi và tải cao. Tôi cũng có thể cộng cost_usd/token theo user để cảnh báo hoặc kiểm tra ngân sách. print("đã trả lời xong") không có schema ổn định nên khó query và aggregate tự động.

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
| 1 stage (bản đầu) | Chưa đo lại được: Dockerfile ban đầu đã bị thay trong quá trình làm CP2 |
| Multi-stage | 183 MB (image day12-agent:cp2-test) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Trong Dockerfile hiện tại, COPY requirements.txt . và RUN pip install nằm trước COPY app và COPY utils. Khi chỉ sửa một ký tự trong app/main.py, các layer từ base đến pip install vẫn được dùng lại; Docker chỉ chạy lại layer copy source và các layer sau đó. Nếu đặt COPY . . trước RUN pip install, mọi thay đổi code sẽ làm layer copy đổi, khiến pip install mất cache và cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng trong Python có thể cho phép attacker thực thi lệnh trong container. Nếu container chạy root, tiến trình bị khai thác có quyền root trong container và có thể đọc/ghi các tài nguyên host mà Docker đã cấp, hoặc tiếp tục khai thác container runtime để nâng quyền trên host. USER appuser trong Dockerfile cắt chuỗi này: tiến trình ứng dụng chỉ chạy với user thường, nên quyền của attacker bị giới hạn dù code bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây. Ví dụ user gửi 10 request lúc 10:00:59 và thêm 10 request lúc 10:01:00. Bộ đếm theo phút đồng hồ reset ở giây 00, nên nhóm đầu thuộc phút trước và nhóm sau thuộc phút mới, dù thực tế chỉ cách nhau khoảng một giây. Sliding window 60 giây không có khe hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lần gọi trong một khoảng thời gian, còn cost guard giới hạn tổng tiền đã tiêu theo từng user trong tháng. Một user có thể gửi một request rất lớn khi quota request vẫn còn; rate limit cho qua nhưng cost guard chặn vì chi phí dự kiến vượt ngân sách. Ngược lại, user có thể gửi nhiều request nhỏ, mỗi request rất rẻ và tổng chi phí vẫn dưới budget; cost guard cho qua nhưng rate limiter chặn khi vượt số request/phút.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Khi Redis mất kết nối, cả 3 container đều trả trạng thái không sẵn sàng nếu endpoint kiểm tra Redis. Load balancer thấy cả 3 unhealthy và ngừng gửi traffic; nếu dùng chung /health, orchestrator có thể hiểu đó là lỗi sống còn rồi restart cả 3 container. Sau 30 giây, Redis quay lại nhưng cả cụm có thể vừa bị restart đồng thời, biến lỗi Redis ngắn thành sự cố toàn hệ thống. Tách /health liveness khỏi /ready readiness giúp chỉ rút instance khỏi traffic mà không restart toàn bộ cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu history nằm trong dict Python, mỗi container có một bản dict riêng. Request thứ nhất có thể vào container A nên trả history_length=0, request kế tiếp vào B cũng có thể trả lại 0 thay vì 2; con số sẽ tăng hoặc reset tùy load balancing. Với Redis, mọi container đọc cùng một list history:<user_id>, nên history được dùng lại ổn định và độ dài tăng theo các request trước đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi tôi gặp khi deploy Railway là Uvicorn báo Invalid value for '--port': '$PORT' is not a valid integer. Tôi đọc log deployment và thấy railway.toml đang override start command bằng chuỗi --port $PORT, nhưng Railway truyền chuỗi đó literal thay vì shell-expand biến. Tôi bỏ startCommand khỏi railway.toml để dùng CMD của Dockerfile, nơi sh -c expand ${PORT:-8000}, rồi redeploy. Sau đó service chuyển sang Online và /health trả 200.
