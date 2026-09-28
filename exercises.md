# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Cảnh Duy  Mã học viên: 2A202602815

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: khi deploy lên Railway, tôi tạo service mới và quên set
> `AGENT_API_KEY` trong tab Variables. Nếu code có mặc định `"changeme"`, app
> vẫn khởi động, health check xanh, Railway báo deploy thành công — nhưng
> `/ask` đang mở công khai với một khóa mà ai đọc repo (hoặc đoán) cũng biết.
> Người lạ gọi `/ask` bằng `X-API-Key: changeme` là tiêu vào ngân sách LLM của
> tôi, và tôi chỉ phát hiện khi nhận hóa đơn. Không có mặc định thì container
> chết ngay lúc khởi động, deploy đỏ, log ghi rõ nguyên nhân — lỗi hiện ra
> *trước* khi có traffic thật.
>
> Khi kiểm tra thực tế, tôi phát hiện code ban đầu chưa fail fast thật: chạy
> `docker run -e REDIS_URL=fake:// agent:multi` (không có key) thì container
> vẫn `Up 5 minutes (healthy)`, vì `get_settings()` chỉ được gọi lần đầu ở
> request `/ask`. Tức là trên cloud nó vẫn "xanh" rồi trả 500 cho user đầu
> tiên. Tôi sửa bằng cách gọi `get_settings()` trong `lifespan` của
> `app/main.py`; sau đó container thoát ngay với
> `ValidationError: agent_api_key Field required` và
> `ERROR: Application startup failed. Exiting.` Bài học: khai báo trường bắt
> buộc chưa đủ, phải *đọc* config lúc khởi động thì mới là fail fast.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật khi chạy `docker compose up --scale agent=3` và gọi `/ask`:
>
> ```
> agent-3  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:34:31.244289+00:00", "user_id": "sv-ex-173430", "tokens_in": 88, "tokens_out": 44, "cost_usd": 3.96e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và gom theo trường.** Tôi đã chạy
>    `docker compose logs agent | Select-String sv-ex-173430` để lấy đúng 6
>    request của một user, và thấy chúng rải đều qua `agent-1/2/3`. Với JSON,
>    công cụ log (Railway, Datadog, Loki) còn làm được query kiểu
>    `user_id = "sv-01" AND level = "error"` hay tính tổng `cost_usd` theo
>    user/ngày để biết ai đang tiêu nhiều tiền nhất — `print` không có trường
>    nào để lọc.
> 2. **Cảnh báo và đo lường tự động.** Vì `tokens_in`, `cost_usd`,
>    `timestamp` là số/thời gian máy đọc được, có thể vẽ biểu đồ và đặt alert
>    ("cost_usd trong 1 giờ > 1$ thì báo"). Tôi còn thấy `tokens_in` tăng dần
>    1 → 43 → 88 → 134 → 179 → 231 qua 6 lượt, chứng tỏ lịch sử hội thoại được
>    gửi kèm prompt — một insight về chi phí mà dòng `print` không cho thấy.
>
> Ngoài ra trên Railway, log JSON một dòng được platform parse thành các
> trường (`event="ask_completed" user_id="sv-rl" cost_usd=...`) để tìm kiếm
> trực tiếp trên dashboard.

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
| 1 stage (bản đầu) | 1 728 MB (1.73 GB) |
| Multi-stage | 265 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi lấy lại Dockerfile gốc bằng `git show 1bf8ea5:Dockerfile`, build thành
> `agent:single`, so với `agent:multi` (Dockerfile hiện tại). Chênh lệch
> ~1.46 GB, gần như toàn bộ đến từ **base image**:
>
> | Thành phần | `agent:single` | `agent:multi` |
> |---|---|---|
> | Base image | `python:3.11` = 1 595 MB | `python:3.12-slim` = 177 MB |
> | Layer thư viện Python | 95.1 MB (`pip install`) | 68.6 MB (`COPY /install`) |
> | Code | 397 kB (`COPY . .`) | 61 kB `app` + 16 kB `utils` |
>
> - ~1.42 GB là phần base image đầy đủ có mà bản slim không có: bộ biên dịch
>   (`gcc`, `make` — tôi kiểm tra `which gcc` trong `agent:single` ra
>   `/usr/bin/gcc`), header C, thư viện `-dev`, git, curl… Những thứ này chỉ
>   cần khi *build* package, không cần khi *chạy* app.
> - ~27 MB ở layer thư viện: bản multi dùng `pip install --no-cache-dir` trong
>   stage `builder` rồi chỉ copy thư mục `/install` sang, nên không mang theo
>   cache của pip.
> - Layer code: `COPY . .` chép cả `tests/`, các file `.md`, `grade.py`,
>   `nginx/`… (tôi `ls /app` trong image thấy có cả file tài liệu nội bộ của
>   giảng viên). Bản multi chỉ copy `app/` và `utils/`.
>
> Image nhỏ hơn 6.5 lần nghĩa là pull/deploy nhanh hơn, tốn ít dung lượng
> registry hơn, và ít phần mềm hơn cho kẻ tấn công lợi dụng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một ký tự `#` vào cuối `app/main.py` rồi chạy
> `docker build --progress=plain -t agent:multi .`. Kết quả:
>
> ```
> #6 [builder 2/4] WORKDIR /app                 CACHED
> #7 [builder 3/4] COPY requirements.txt .      CACHED
> #8 [builder 4/4] RUN pip install ...          CACHED
> #9 [runtime 3/6] COPY --from=builder /install CACHED
> #10 [runtime 4/6] COPY app ./app              ← chạy lại
> #11 [runtime 5/6] COPY utils ./utils          ← chạy lại
> #12 [runtime 6/6] RUN useradd ...             ← chạy lại
> ```
>
> - **Dùng lại cache:** mọi layer đến trước `COPY app` — quan trọng nhất là
>   layer `pip install`, vì `requirements.txt` không đổi nên checksum của layer
>   `COPY requirements.txt` không đổi.
> - **Chạy lại:** `COPY app` (nội dung thay đổi) và *mọi layer sau nó* — Docker
>   cache theo chuỗi, một layer đổi thì toàn bộ layer phía sau bị vô hiệu,
>   kể cả `COPY utils` dù `utils/` không đổi.
>
> Nếu `COPY . .` đứng trước `RUN pip install` (đúng như Dockerfile gốc), sửa
> bất kỳ file nào cũng làm layer `COPY . .` đổi → `pip install` chạy lại từ
> đầu. Tôi đo thật với cùng thay đổi 1 ký tự:
>
> | Dockerfile | Thời gian build lại |
> |---|---|
> | Gốc (`COPY . .` trước `pip install`) | **126.8 giây** — cài lại toàn bộ thư viện |
> | Của tôi (requirements trước, code sau) | **1.3 giây** |
>
> Nhận xét thêm: có thể đưa `RUN useradd` lên trước `COPY app` để nó cũng
> được cache, vì bước tạo user không phụ thuộc vào code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy root:
>
> 1. Code Python có lỗ hổng cho phép thực thi lệnh — ví dụ một thư viện bị lỗi
>    deserialize, hoặc agent được cho quyền gọi tool chạy shell và bị
>    prompt injection.
> 2. Kẻ tấn công có shell trong container **với uid 0 (root)**.
> 3. Là root trong container, hắn đọc/ghi được mọi file, cài thêm công cụ
>    (`apt install`), đọc secret trong biến môi trường.
> 4. Từ root trong container thoát ra host: nếu container có mount
>    `/var/run/docker.sock` hay volume của host, hoặc chạy `--privileged`, hoặc
>    kernel có lỗ hổng container escape — mà uid 0 trong container mặc định
>    cũng là uid 0 trên host (nếu không bật user namespace) — thì hắn thành
>    root trên máy host, điều khiển mọi container khác.
>
> Lệnh `USER appuser` cắt chuỗi ở **bước 2**: shell kẻ tấn công lấy được chỉ có
> quyền của một user thường (uid 10001). Tôi kiểm tra
> `docker run --rm --entrypoint whoami agent:multi` → `appuser`, trong khi
> `agent:single` (Dockerfile gốc) → `root`. Với `appuser`, hắn không cài được
> package, không ghi được vào thư mục hệ thống, và các kỹ thuật escape cần
> quyền root phần lớn thất bại; file bị ghi đè trên volume host cũng chỉ mang
> uid 10001 chứ không phải root. Lỗ hổng vẫn còn, nhưng thiệt hại bị giới hạn
> trong phạm vi của app.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **Tối đa 20 request trong 2 giây.**
>
> Đếm theo phút đồng hồ nghĩa là bộ đếm reset về 0 đúng giây 00 mỗi phút:
>
> - 10:00:59 — gửi 10 request. Bộ đếm phút 10:00 = 10, vẫn "đúng luật".
> - 10:01:00 — sang phút mới, bộ đếm reset về 0, gửi tiếp 10 request.
>   Bộ đếm phút 10:01 = 10, vẫn "đúng luật".
>
> → 20 request trong khoảng 10:00:59–10:01:00, gấp đôi hạn mức, dồn vào cùng
> một thời điểm — đủ để tạo burst làm quá tải LLM backend.
>
> Sliding window chặn được vì nó đếm các request trong **60 giây ngay trước
> thời điểm hiện tại**, không phụ thuộc ranh giới phút. Ở 10:01:00, cửa sổ là
> 10:00:00–10:01:00 và vẫn chứa 10 request lúc 10:00:59 → request thứ 11 bị
> 429. Trong code, `zremrangebyscore(key, 0, now - 60)` xóa các timestamp cũ
> hơn 60 giây rồi `zcard` đếm phần còn lại. Khi test trên Railway, 15 request
> liên tiếp cho đúng `200×10` rồi `429×5`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> | | Rate limit | Cost guard |
> |---|---|---|
> | Đo cái gì | **Số request** trong 60 giây gần nhất | **Số tiền** đã tiêu trong tháng |
> | Cửa sổ | Ngắn (60 s, trượt) | Dài (theo tháng, key `cost:<user>:2026-09`) |
> | Bảo vệ khỏi | Burst/spam, quá tải hệ thống | Hóa đơn LLM vượt ngân sách |
> | Mã lỗi | 429 Too Many Requests (thử lại sau `Retry-After`) | 402 Payment Required (đợi tháng sau hoặc nâng hạn mức) |
> | Redis | Sorted set, score = timestamp | Một số thực, cộng dồn bằng `incrbyfloat` |
>
> **Rate limit cho qua nhưng cost guard chặn:** một user gửi đều 5 request/phút
> (dưới hạn mức 10), nhưng mỗi câu hỏi dán kèm tài liệu dài hàng chục nghìn
> token, chạy suốt nhiều ngày. Không lần nào vượt tốc độ, nhưng tổng
> `cost_usd` tích lũy vượt `MONTHLY_BUDGET_USD=10` → phải trả 402. Tôi cũng
> thấy trong log là `tokens_in` tăng theo độ dài lịch sử hội thoại, nên cùng
> một số request, chi phí vẫn có thể tăng dần.
>
> **Cost guard cho qua nhưng rate limit chặn:** một script lỗi gửi 15 request
> "test" liên tiếp trong vài giây. Mỗi request chỉ tốn ~0.00002 $, tổng chưa
> tới 0.001 $ — ngân sách còn gần nguyên. Nhưng từ request thứ 11 là 429 —
> đúng những gì tôi quan sát khi test trên Railway (`200×10` rồi `429×5`).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp làm một endpoint có kiểm tra Redis:
>
> 1. **t = 0 s** — Redis mất kết nối (ví dụ restart, mạng chập chờn).
> 2. Cả 3 container gọi `ping()` thất bại **cùng lúc**, vì chúng dùng chung
>    một Redis → endpoint trả 503 trên cả 3.
> 3. Load balancer thấy cả 3 instance lỗi → rút cả 3 khỏi vòng xoay → mọi
>    request của user nhận 502/503, kể cả những request không cần Redis.
> 4. Vì endpoint này đồng thời là **liveness probe**, sau vài lần thất bại
>    (ví dụ 3 lần × 10 s) orchestrator kết luận process "chết" → **restart cả
>    3 container**.
> 5. **t = 30 s** — Redis sống lại, nhưng 3 container đang khởi động lại
>    (vài giây đến vài chục giây); nếu Redis vẫn chưa ổn định lúc container
>    start, chúng lại fail và bị restart tiếp → vòng lặp restart.
> 6. Kết quả: sự cố 30 giây của Redis biến thành downtime toàn hệ thống dài
>    hơn nhiều, và mọi request đang xử lý dở bị cắt khi container bị kill.
>
> Tách ra thì: `/ready` 503 → LB tạm ngừng gửi traffic, **không restart**;
> `/health` vẫn 200 → container giữ nguyên. Redis về là `/ready` lại 200 ngay,
> không mất thời gian khởi động.
>
> Tôi quan sát được thực tế với 3 instance sau nginx: khi
> `docker compose stop redis`, lần gọi đầu `/ready` = 503 còn `/health` = 200
> (app đã tách đúng). Nhưng lần gọi thứ hai, **cả `/ready` lẫn `/health` qua
> nginx đều 502 Bad Gateway** — vì `nginx.conf` có
> `proxy_next_upstream ... http_503`, nginx coi 503 là upstream hỏng, thử cả 3
> instance, cả 3 đều 503 → đánh dấu cả cụm "down" trong một khoảng thời gian.
> Đó chính là bước 3 ở trên xảy ra ở tầng load balancer. Sau
> `docker compose start redis`, `/ready` trở lại 200 mà không container nào
> phải restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy `docker compose up -d --scale agent=3` (có nginx làm load balancer
> round-robin ở cổng 8000), rồi gọi `/ask` 6 lần với cùng
> `X-User-Id: sv-ex-173430`. Kết quả thật:
>
> | Lượt | Container xử lý (theo log) | `history_length` |
> |---|---|---|
> | 1 | agent-2 | 0 |
> | 2 | agent-1 | 2 |
> | 3 | agent-3 | 4 |
> | 4 | agent-2 | 6 |
> | 5 | agent-1 | 8 |
> | 6 | agent-3 | 10 |
>
> Mỗi lượt rơi vào một container khác, nhưng `history_length` vẫn tăng đều
> +2 (1 message user + 1 message assistant), vì cả 3 container cùng đọc/ghi
> key `history:sv-ex-173430` trong một Redis.
>
> Nếu lưu trong dict Python, mỗi container có dict riêng trong RAM của nó.
> Với cùng thứ tự round-robin trên, con số sẽ là:
>
> | Lượt | Container | `history_length` với dict |
> |---|---|---|
> | 1 | agent-2 | 0 |
> | 2 | agent-1 | 0 ← agent-1 chưa biết gì |
> | 3 | agent-3 | 0 |
> | 4 | agent-2 | 2 ← chỉ nhớ lượt 1 |
> | 5 | agent-1 | 2 |
> | 6 | agent-3 | 2 |
>
> Tức là thay vì 0-2-4-6-8-10, con số đứng yên hoặc nhảy lung tung tùy
> request rơi vào đâu — agent "mất trí nhớ" ngẫu nhiên. Tệ hơn, restart hay
> deploy lại một container là mất sạch lịch sử trên container đó, và mỗi
> instance tự cộng dồn rate limit/chi phí riêng nên hạn mức thực tế bị nhân 3.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** sau khi deploy lên Railway, `/health` và `/ready` đều 200, gọi
> `/ask` không có key ra 401 đúng, nhưng lệnh gọi `/ask` **có** API key lại trả:
>
> ```
> {"detail":[{"type":"json_invalid","loc":["body",12],"msg":"JSON decode error",
>   "ctx":{"error":"Unterminated string starting at"}}]} -> 422
>  -> 000
> ```
>
> **Tìm nguyên nhân:**
>
> 1. Ban đầu tôi nghĩ bản deploy lỗi. Nhưng `railway logs` cho thấy request tới
>    server là `"POST /ask HTTP/1.1" 422 Unprocessable Entity` — server vẫn
>    chạy, chỉ là nhận **body JSON hỏng** (lỗi ở vị trí 12, chuỗi chưa đóng).
> 2. Dòng thứ hai ` -> 000` nghĩa là curl gọi thêm một "URL" không kết nối
>    được — dấu hiệu tham số bị tách ra thành nhiều mảnh.
> 3. So sánh: vòng lặp rate limit dùng cùng cú pháp `-d '{\"question\":\"test\"}'`
>    thì chạy tốt (200×10, 429×5). Khác biệt duy nhất: câu hỏi
>    `"Deploy la gi?"` **có dấu cách**, còn `"test"` thì không.
> 4. Kết luận: Windows PowerShell 5.1 truyền tham số chứa `\"` và khoảng trắng
>    sang chương trình ngoài (`curl.exe`) sai cách — chuỗi JSON bị cắt tại dấu
>    cách: curl gửi `{"question":"Deploy` làm body (→ 422) và coi `la`, `gi?"}`
>    là URL khác (→ 000). Server trên Railway hoàn toàn không có lỗi.
>
> **Sửa:** không truyền JSON qua dòng lệnh curl nữa mà dùng cmdlet của
> PowerShell, để nó tự serialize body:
>
> ```powershell
> $body = @{ question = "Deploy la gi?" } | ConvertTo-Json
> Invoke-RestMethod -Method Post -Uri "$URL/ask" -ContentType "application/json" `
>   -Headers @{ "X-API-Key" = $KEY; "X-User-Id" = "sv-test" } -Body $body
> ```
>
> (Cách khác: ghi JSON ra file và dùng `curl.exe -d "@body.json"`.)
>
> **Lỗi thứ hai tránh được nhờ đọc tài liệu platform:** `railway.toml` ban đầu
> có `startCommand = "uvicorn ... --port $PORT"`. Với build từ Dockerfile,
> Railway chạy start command ở exec form, không qua shell, nên `$PORT` không
> được thay giá trị. Tôi bỏ dòng này để Railway dùng `CMD ["sh", "-c", "... --port
> ${PORT:-8000}"]` trong Dockerfile. Log sau khi deploy xác nhận
> `Uvicorn running on http://0.0.0.0:8080` — Railway gán `PORT=8080` và app
> đọc đúng.
>
> Bài học: khi kiểm tra một bản deploy, phải tách được lỗi của **công cụ phía
> client** khỏi lỗi của **server**. Log phía server (`railway logs`) là nơi
> phân xử.
