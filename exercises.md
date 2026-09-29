# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Mai Văn Trung  Mã học viên: 2A202602513

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: lúc tạo service `agent` trên Railway, mình phải tự khai báo từng
biến trong Variables. Giả sử mình quên `AGENT_API_KEY`:

- **Có mặc định `"changeme"`:** build xanh, container chạy, `/health` trả 200,
  Railway báo "Deployment successful". Mình tưởng mọi thứ ổn. Nhưng chuỗi
  `changeme` nằm công khai trong repo, nên ai đọc code cũng gọi được `/ask` với
  key đó và tiêu ngân sách LLM. Mình chỉ phát hiện ra khi xem hóa đơn.
- **Không có mặc định:** `Settings()` ném `ValidationError` ngay lúc import.
  Mình đã thử `Settings(_env_file=None)` khi không có biến và thấy lỗi xuất hiện
  lập tức. Deploy sẽ đỏ, log ghi rõ thiếu trường `agent_api_key`, nên mình sửa
  trước khi có bất kỳ request nào.

Ở máy local, mình còn chặn sớm thêm một tầng trong `docker-compose.yml` bằng
`${AGENT_API_KEY:?...}`: thiếu biến thì `docker compose up` báo lỗi trước cả
khi tạo container.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thật lấy từ `docker compose logs agent`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:28:03.066927+00:00", "user_id": "sv02-live", "tokens_in": 400, "tokens_out": 45, "cost_usd": 8.7e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được:

1. **Lọc và tổng hợp theo field.** Ví dụ lọc `event == "ask_completed"` và
   `user_id == "sv02-live"` rồi cộng `cost_usd` để biết một user tiêu bao nhiêu
   tiền hôm nay. Chuỗi `print` không có `user_id` hay chi phí, và kể cả có thì
   phải viết regex để tách ra.
2. **Đặt cảnh báo và vẽ biểu đồ.** Ví dụ cảnh báo khi `tokens_in` vượt ngưỡng.
   Mình thấy `tokens_in` tăng lên 400 vì history được gửi kèm vào prompt. Có
   thể đếm số `ask_completed` mỗi phút làm metric traffic. Railway đã tự parse
   log JSON của mình thành dạng `[INFO] event="service_started" ...`, nên lọc
   được theo field ngay trên dashboard.

Nó còn phải nằm trên đúng một dòng: log JSON của mình xen giữa log text của
uvicorn (`INFO: ... "POST /ask HTTP/1.1" 200 OK`) mà vẫn tách riêng được, vì
platform gom log theo dòng.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Mình xem từng layer bằng `docker history`:

- Layer `pip install` ở cả hai bản gần như bằng nhau (95.1 MB so với 96 MB), và
  source code chỉ khoảng 140 KB. Vậy thư viện và code không phải phần chênh.
- Khoảng 1.4 GB chênh lệch gần như toàn bộ nằm ở **base image**. `python:3.11`
  bản đầy đủ mang theo gcc, header C, git, các gói `-dev` phục vụ việc build
  thư viện. Lúc chạy service không cần những thứ đó. `python:3.11-slim` bỏ
  chúng đi.
- Multi-stage giúp giữ image runtime sạch. Nếu có thư viện phải biên dịch, việc
  biên dịch chỉ xảy ra ở stage `builder`, còn stage runtime chỉ copy
  `/opt/venv` sang.

Mình cũng nhận ra Dockerfile gốc dùng `COPY . .` trong khi `.dockerignore` gốc
chỉ có `.git`, nên `.venv` và cả file `.env` chứa secret đều bị nướng vào image.
Image còn có thể nhỏ hơn nữa, vì `requirements.txt` đang gộp cả pytest,
fakeredis, httpx (chỉ dùng cho test) vào image runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình thử trên một bản copy: thêm một dòng comment vào `app/main.py` rồi build
lại với `--progress=plain`:

- **Dùng lại cache (CACHED):** `python -m venv`, `COPY requirements.txt`,
  `RUN pip install -r requirements.txt` (khoảng 96 MB không phải tải và cài
  lại), `groupadd/useradd`, `WORKDIR`, `COPY --from=builder /opt/venv`.
- **Chạy lại:** `COPY app/`, vì nội dung file đổi nên checksum đổi. Thêm
  `COPY utils/` ngay sau đó, vì khi một layer mất cache thì mọi layer phía sau
  nó cũng phải build lại, dù `utils/` không đổi.

Nếu đặt `COPY . .` lên trước `RUN pip install`, thì chỉ cần sửa một ký tự ở bất
kỳ file nào, checksum của layer COPY cũng đổi. Khi đó `pip install` và mọi layer
sau nó đều phải chạy lại, mỗi lần build mất thêm thời gian tải và cài lại toàn
bộ thư viện. Tách `COPY requirements.txt` riêng giúp layer cài thư viện chỉ chạy
lại khi `requirements.txt` thật sự thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:

1. Code có lỗ hổng, ví dụ một thư viện parse input bị lỗi hoặc một chỗ gọi
   `eval`/`subprocess` với dữ liệu người dùng. Kẻ tấn công chạy được lệnh tùy ý
   (RCE) bên trong process Python.
2. Lệnh đó chạy với quyền của process. Image 1-stage không có `USER`
   (`docker image inspect` cho `Config.User` rỗng), nên process là **root**
   trong container.
3. Là root, kẻ tấn công ghi đè được mọi file (sửa code, cài công cụ), đọc được
   secret trong biến môi trường, và nếu container có mount volume, mount
   `docker.sock` hay chạy `--privileged`, thì root trong container tác động
   thẳng lên host.
4. Root trong container cũng chính là uid 0 trên host (nếu không bật user
   namespace). Chỉ cần thêm một lỗ hổng kernel hoặc container runtime là thoát
   ra được với quyền root trên host.

`USER app` cắt chuỗi ở **bước 2**. Trong image của mình, `id` ra
`uid=999(app) gid=999(app)`, user không có shell đăng nhập. RCE chỉ có quyền của
một user thường: không cài được gói, không ghi được file hệ thống, file do
volume của root sở hữu cũng không động vào được. Bước leo thang lên host khó hơn
rất nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request trong 2 giây**, gấp đôi hạn mức:

- Lúc 10:00:59, gửi 10 request. Bộ đếm của phút 10:00 lên 10/10, hợp lệ.
- Lúc 10:01:00, bộ đếm reset về 0. Gửi tiếp 10 request, bộ đếm phút 10:01 lên
  10/10, vẫn hợp lệ.

Tức là 20 request trong khoảng 1–2 giây mà không bị chặn.

Với sliding window, mình đếm số request trong đúng 60 giây **gần nhất** tính từ
thời điểm hiện tại (Redis Sorted Set, score là timestamp, xóa entry cũ hơn
`now - 60`). 10 request lúc :59 vẫn nằm trong cửa sổ khi request thứ 11 tới lúc
:00, nên request thứ 11 bị 429. Mình đã quan sát được trên máy local: gọi liên
tục 12 lần với cùng user thì được `200` × 10, sau đó `429` kèm
`retry-after: 60`. Trên Railway cũng vậy: 15 lần gọi cho ra 9 lần 200 (1 lượt
đã dùng từ lệnh trước) rồi 6 lần 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

| | Rate limit | Cost guard |
|---|---|---|
| Đo cái gì | **Số request** trong 60 giây | **Số tiền** đã tiêu trong tháng |
| Key Redis | `ratelimit:<user>` (Sorted Set) | `cost:<user>:2026-09` (số thực) |
| Mã lỗi | 429 Too Many Requests | 402 Payment Required |
| Bảo vệ khỏi | Spam, bot, quá tải trong thời gian ngắn | Hóa đơn tích lũy trong thời gian dài |

**Rate limit cho qua nhưng cost guard chặn:** một user gửi đều 5 request/phút,
luôn dưới hạn mức 10, nhưng mỗi câu hỏi rất dài và lịch sử hội thoại được gửi
kèm. Mình thấy chi phí một lượt tăng từ khoảng 2.3e-05 USD lên 8.7e-05 USD khi
`tokens_in` lên 400. Gửi đều như vậy cả tháng thì tổng chi phí vượt
`MONTHLY_BUDGET_USD = 10`, và từ đó cost guard trả 402. Test
`test_qua_http_thi_tra_402` cũng cho thấy: đặt chi phí đã tiêu = 999 thì ngay
request đầu tiên đã bị 402, dù rate limit chưa bị dùng lượt nào.

**Cost guard cho qua nhưng rate limit chặn:** đầu tháng, user mới tiêu gần
như 0 USD, nhưng một script lỗi gọi `/ask` 50 lần trong vài giây. Ngân sách còn
dư rất nhiều, nhưng đến request thứ 11 trong 60 giây thì rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối. Cả 3 container gọi `ping()` thất bại cùng lúc, nên
   endpoint gộp trả 503 ở cả 3.
2. Orchestrator dùng endpoint này làm **liveness probe** (Docker `HEALTHCHECK`,
   Railway `healthcheckPath`). Sau vài lần fail liên tiếp, nó kết luận cả 3
   container "chết".
3. Orchestrator **restart cả 3 container** gần như cùng lúc. Các request đang xử
   lý dở bị cắt, user nhận lỗi 502/503.
4. Container mới khởi động, nhưng nếu Redis vẫn chưa về thì probe tiếp tục fail
   và chúng lại bị restart, thành vòng lặp restart. Trong lúc đó không có
   instance nào phục vụ được, kể cả những việc không cần Redis.
5. Redis quay lại sau 30 giây, nhưng cụm còn phải chờ container khởi động lại
   và qua `start_period` của healthcheck, nên thời gian gián đoạn thực tế dài
   hơn 30 giây.

Khi tách riêng như mình làm: `/health` không nhận dependency nào nên vẫn 200,
không container nào bị restart. `/ready` trả 503 `{"redis": false}` nên load
balancer chỉ tạm ngừng gửi traffic. Redis về thì `/ready` trả 200 lại và traffic
chảy tiếp, không mất 30 giây khởi động lại. Lỗi nằm ở Redis thì không nên
"chữa" bằng cách restart app.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Mình chạy `AGENT_HOST_PORT=8000-8002 docker compose up -d --scale agent=3`
(dùng dải cổng vì 3 container không cùng map được vào cổng 8000), rồi gọi lần
lượt vào từng container với cùng `X-User-Id: sv-scale`:

| Gọi vào | `history_length` |
|---|---|
| container qua cổng 8000 | 0 |
| container qua cổng 8001 | 2 |
| container qua cổng 8002 | 4 |

Ba container khác nhau nhưng con số tăng đều 0 → 2 → 4 (mỗi lượt thêm 1 message
user và 1 message assistant), vì cả ba đọc và ghi cùng key `history:sv-scale`
trong Redis.

Nếu lưu trong dict Python, mỗi container có dict riêng trong RAM của nó, nên
kết quả sẽ là **0 → 0 → 0**. Mỗi container chỉ thấy lượt mình tự xử lý. Qua
load balancer, con số còn nhảy lung tung (0, 2, 0, 4, 2…) tùy request rơi vào
container nào, và agent "mất trí nhớ" giữa chừng. Container restart là mất sạch
lịch sử. Với Redis, mình có thể dừng hẳn một container (thử `docker compose stop`
thấy tắt sạch trong 1 giây, exit 0) mà lịch sử của user vẫn còn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lần deploy Railway của mình build và chạy thành công ngay lần đầu
(BUILDING → DEPLOYING → SUCCESS trong khoảng 40 giây), nhờ đã chạy thử với
`docker compose` ở máy trước. Có hai vấn đề thật mình gặp và xử lý:

**1. Domain vừa tạo chưa sẵn sàng.** Ngay sau `railway domain`, CLI báo
`Sync status: CREATING`, nghĩa là domain công khai chưa trỏ xong vào service.
Mình không gọi test ngay mà kiểm tra `railway logs --deployment` trước, và thấy
app đã chạy bình thường (`Uvicorn running on http://0.0.0.0:8080`). Như vậy nếu
gọi URL mà lỗi thì nguyên nhân nằm ở domain đang đồng bộ, không phải ở app. Mình
viết vòng lặp gọi `/health` mỗi 10 giây cho tới khi nhận được 200 rồi mới chạy
các lệnh kiểm tra. Dòng log đó cũng xác nhận Railway cấp `PORT=8080` chứ không
phải 8000. Vì Dockerfile đọc `${PORT:-8000}` nên app nghe đúng cổng. Nếu mình cố
định 8000, proxy của Railway sẽ gọi vào cổng không có ai nghe và health check
timeout.

**2. Secret bị in ra terminal.** Khi tạo service bằng
`railway add --service agent --variables "AGENT_API_KEY=..."`, CLI in lại
nguyên dòng `Enter a variable AGENT_API_KEY=<giá trị thật>` ra output. Secret
có thể lọt vào lịch sử terminal hoặc log CI. Mình phát hiện khi đọc lại output
của lệnh. Cách xử lý: bản deploy đã dùng key riêng, khác key local, nên key lộ
không mở được service chạy ở máy. Việc cần làm tiếp là đổi key mới trong tab
Variables trên dashboard (Railway sẽ tự redeploy) và cập nhật `DEPLOY_API_KEY`
trong `.env`. Lần sau mình sẽ set secret trực tiếp trên dashboard thay vì truyền
qua tham số dòng lệnh.
