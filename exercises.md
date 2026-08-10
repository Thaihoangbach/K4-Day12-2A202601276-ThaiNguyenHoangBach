# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Thái Nguyễn Hoàng Bách  Mã học viên: 2A202601276

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy app lên Railway nhưng tôi quên cấu hình `API_TOKEN`, việc `Settings()` bị lỗi ngay khi khởi động giúp tôi phát hiện vấn đề ngay trong lúc deploy. App không thể chạy với một cấu hình không an toàn.

Nếu `api_token` có giá trị mặc định là `"changeme"` thì app vẫn khởi động bình thường. Vì giá trị này nằm trong source code nên người khác có thể biết token và sử dụng nó để gọi `/chat`. Khi đó tôi có thể không phát hiện ra lỗi cho đến khi API phát sinh nhiều request hoặc chi phí lớn.

Vì vậy, fail fast giúp phát hiện lỗi cấu hình ngay từ đầu thay vì để một cấu hình sai tiếp tục chạy trong production.


---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> {"event": "service_started", "severity": "INFO", "ts": "2026-08-10T14:04:36.933645+00:00", "service": "day12-chat-service", "version": "1.0.0"} 
2 việc log JSON làm được mà print không làm: ví dụ lọc bằng jq/grep+grep -o theo trường client_id, đưa vào hệ thống như Datadog/CloudWatch để dựng dashboard/cảnh báo tự động theo severity.

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
| 1 stage (bản đầu) | 286 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Multi-stage image nhỏ hơn vì stage cuối chỉ copy những thành phần cần thiết
để chạy ứng dụng từ stage build. Các file phục vụ quá trình build như cache,
source hoặc các dependency/tool chỉ cần lúc build không cần xuất hiện trong
runtime image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

>  Nếu COPY . . đứng TRƯỚC pip install: sửa 1 dòng code bất kỳ cũng làm cache của pip install bị vô hiệu → cài lại toàn bộ dependency mỗi lần build, chậm hẳn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng trong code Python (vd RCE qua thư viện, deserialization) → kẻ tấn công thực thi lệnh với quyền của user chạy process → nếu đó là root, họ có quyền ghi mọi nơi trong container, và nếu có lỗ hổng container-escape (kernel/runtime bug) thì leo luôn ra host. USER appuser (uid 10001, không có quyền sudo, không sở hữu file hệ thống) cắt đứt chuỗi ở bước "quyền của process" — dù bị chiếm cũng chỉ có quyền hạn chế.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> WWW-Authenticate: Bearer là chuẩn HTTP (RFC 7235) — client (hoặc thư viện HTTP) dựa vào header này để biết cần xác thực kiểu gì mà tự động xử lý (vd trình duyệt hiện popup login, hoặc middleware tự retry với đúng scheme). Trả cùng 1 thông báo cho cả 3 lỗi: nếu nói rõ "sai token" vs "thiếu header", kẻ dò token (brute-force) biết được token gần đúng hay chưa từng thử — thông tin thừa giúp họ thu hẹp phạm vi dò.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Tự tính: capacity=10, sau 10 phút im lặng xô đầy lại đúng 10 (nhờ min(capacity,...)), nên gửi liên tiếp được đúng 10 request rồi request thứ 11 bị 429. Nếu bỏ min(...): xô tích tuyến tính không giới hạn theo thời gian im lặng — 10 phút × 10 token/phút = 100 token tích lũy, gửi được 100 request liên tiếp trước khi cạn. Bạn có thể tự verify bằng cách viết 1 test nhỏ gọi bucket.available() với/không có dòng min.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> $30/tháng: nếu sự cố xảy ra 2h sáng và không ai để ý tới sáng, thiệt hại tối đa có thể là toàn bộ $30 (nếu cả tháng ngân sách còn nguyên). $1/ngày: thiệt hại tối đa bị chặn ở $1 vì CostGuard.today() dùng key theo ngày UTC — client bị chặn ngay khi vượt $1, và tự "hồi phục" ngay 00:00 UTC hôm sau (key ngày mới, spent() trả về 0 lại) mà không cần ai can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp làm 1 và cho kiểm tra Redis: Redis mất kết nối 30 giây → endpoint đó trả 503 → cả 3 container coi là "liveness fail" (không phải chỉ "chưa sẵn sàng") → orchestrator (Docker/K8s) hiểu nhầm là process chết, ra lệnh restart cả 3 container cùng lúc dù process vẫn sống khỏe, chỉ là phụ thuộc ngoài bị lỗi tạm thời — gây gián đoạn thật sự trong lúc lẽ ra chỉ cần tạm ngừng nhận traffic mới (đúng việc của /readyz) rồi tự phục hồi khi Redis sống lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: Deploy Logs báo lặp lại Error: Invalid value for '--port': '$PORT' is not a valid integer.
Cách tìm ra nguyên nhân: xem tab Deploy Logs trên Railway, thấy uvicorn nhận literal chuỗi '$PORT' thay vì số cổng thật → hiểu ra startCommand trong railway.toml bị Railway chạy không qua shell nên biến môi trường không được giãn nở.
Cách sửa: xóa dòng startCommand khỏi railway.toml, để Railway dùng lại CMD ["sh","-c","uvicorn ... --port ${PORT:-8000}"] có sẵn trong Dockerfile — dòng này chạy qua sh -c nên $PORT được shell thay thế đúng.
