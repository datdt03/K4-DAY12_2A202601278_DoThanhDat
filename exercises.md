# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thành Đạt  Mã học viên: 2A202601278

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi deploy lên Railway nhưng quên tạo `API_TOKEN` cho service `chat`. Nếu token mặc định là `"changeme"`, ứng dụng vẫn báo healthy và người ngoài có thể đoán token này để gọi `/chat`, làm phát sinh chi phí mà tôi không nhận ra ngay. Khi `api_token` không có mặc định, deployment dừng ngay với lỗi cấu hình; tôi sửa biến môi trường trước khi URL được đưa vào sử dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được có dạng: `{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T09:15:00+00:00","client_id":"sv01","prompt_tokens":4,"completion_tokens":31,"usd_cost":0.0000192}`. Với JSON này, tôi có thể lọc và đếm tỷ lệ sự kiện theo `severity` hoặc khoảng `ts`; đồng thời có thể nhóm theo `client_id` và cộng `usd_cost` để tìm client tiêu nhiều tiền. Dòng `print("đã trả lời xong")` không chứa các trường có cấu trúc để thực hiện hai việc đó.

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
| 1 stage (bản đầu) | khoảng 1.800 MB (số của bản khởi đầu; image cũ không còn trên máy) |
| Multi-stage | 270 MB (đo bằng `docker images`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch khoảng 1,53 GB chủ yếu là base image Python đầy đủ, công cụ build/compiler, cache cài đặt và các file trung gian chỉ cần lúc build. Multi-stage chỉ chép thư viện đã cài từ stage `builder` sang `python:3.11-slim`, nên stage runtime không mang theo các công cụ và layer trung gian đó.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer lấy base image, `COPY requirements.txt`, `pip install`, chép dependency từ builder và tạo `appuser` vẫn được dùng lại từ cache. Layer `COPY app ./app` và các layer sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source làm layer copy đổi, kéo theo layer cài dependency mất cache và phải tải/cài lại toàn bộ thư viện dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với UID của process trong container. Khi process là root, chúng có quyền sửa file hệ thống trong container; nếu runtime hoặc mount/capability bị cấu hình yếu, quyền này có thể được dùng để đọc secret, sửa volume hoặc khai thác tiếp ra host. `USER appuser` cắt chuỗi tại bước đầu: mã khai thác chỉ nhận UID 10001 với quyền tối thiểu, không có quyền root để thay đổi phần hệ thống được bảo vệ.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` cho client biết tài nguyên yêu cầu cơ chế xác thực Bearer theo chuẩn HTTP, nhờ đó client có thể phản ứng đúng với lỗi 401. Tôi dùng cùng thông báo cho thiếu header, sai scheme và sai token để không biến API thành công cụ dò: nếu trả lỗi chi tiết khác nhau, kẻ tấn công biết phần nào đã đúng và thu hẹp dần không gian đoán token.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Xô có sức chứa 10 nên sau khi im lặng 10 phút nó vẫn chỉ có tối đa 10 token; client gửi được 10 request liên tiếp và request thứ 11 bị 429. Nếu bỏ `min(capacity, ...)`, tốc độ 10 token/phút trong 10 phút làm nó tích thêm 100 token; nếu trước khi chờ xô đang đầy thì phép tính thành 110 token, nên có thể gửi 110 request trước khi bị chặn. Đây là burst vượt xa sức chứa thiết kế.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, sự cố từ 2 giờ sáng có thể đốt tối đa phần còn lại của cả 30 USD và service chỉ tự có ngân sách lại khi sang tháng mới. Với hạn mức 1 USD/ngày, thiệt hại trong ngày bị chặn ở khoảng 1 USD; key ngân sách đổi theo ngày UTC nên service tự nhận request lại vào ngày kế tiếp. Hạn mức ngày vì vậy giới hạn blast radius và thời gian gián đoạn nhỏ hơn nhiều.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint chung kiểm tra Redis, khi Redis mất kết nối cả ba container đồng thời trả 503. Orchestrator hiểu đây là lỗi liveness và restart cả ba; các container mới khởi động vẫn gặp Redis đang lỗi nên tiếp tục fail healthcheck và bị restart lặp lại. Khi Redis hồi phục sau 30 giây, có thể chưa còn instance ổn định nào sẵn sàng phục vụ, biến lỗi dependency ngắn thành gián đoạn toàn cụm. Tách `/healthz` giúp process vẫn báo sống, còn `/readyz` chỉ yêu cầu load balancer tạm ngừng gửi traffic.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Railway, healthcheck liên tục báo `service unavailable` và log Uvicorn ghi `Invalid value for '--port': '$PORT' is not a valid integer`. Tôi đọc log deployment và kiểm tra `railway.toml`, từ đó thấy `startCommand` truyền `$PORT` ở exec form nên biến không được shell mở rộng. Tôi xóa custom `startCommand` để Railway dùng `CMD ["sh", "-c", "... --port ${PORT:-8000}"]` trong Dockerfile, sau đó deploy lại. Container khởi động thành công và `GET /healthz` trả 200.
