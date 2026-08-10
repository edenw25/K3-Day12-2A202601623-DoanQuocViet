# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng  *Câu trả lời của bạn* bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Quốc Việt  Mã học viên: 2A202601623

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: tôi deploy lên Railway và quên set `AGENT_API_KEY` trong tab
> Variables của dashboard.
>
> **Nếu có mặc định `"changeme"`:** app khởi động bình thường, `/health` trả
> 200, platform báo deploy thành công, URL public sống. Không có dấu hiệu nào
> cho thấy có vấn đề. Nhưng khóa đang dùng là một chuỗi ai cũng đoán được —
> `"changeme"` là giá trị mặc định phổ biến nhất trong các repo mẫu, và repo
> của tôi lại là public nên người ta đọc `config.py` là biết ngay. Bất kỳ ai
> cũng gọi được `/ask` bằng khóa đó. Vì `verify_api_key` trả về
> `ANONYMOUS_USER` khi không gửi `X-User-Id`, mọi kẻ lạ sẽ dùng chung một
> user_id, tức chung một hạn mức rate limit và chung một ví ngân sách với
> người dùng thật của tôi. Tôi chỉ phát hiện khi hóa đơn LLM về cuối tháng,
> hoặc khi người dùng thật bắt đầu bị trả 429/402 mà không hiểu vì sao.
>
> **Vì không có mặc định:** `Settings()` ném `ValidationError` ngay trong
> `get_settings()`, uvicorn không lên nổi, container exit. Health check của
> platform fail nên Railway giữ nguyên bản cũ đang chạy tốt thay vì đẩy bản
> hỏng ra production. Tôi thấy lỗi ngay trong deploy log, sửa mất 30 giây.
>
> Điểm mấu chốt: cả hai trường hợp tôi đều mắc cùng một lỗi. Khác nhau ở chỗ
> lỗi đó lộ ra sau **30 giây** hay sau **30 ngày** — và trong trường hợp thứ
> hai thì nó đã kịp thành một sự cố bảo mật, không còn là một lỗi cấu hình.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `docker compose logs agent`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:24:05.394927+00:00", "user_id": "sv-test", "tokens_in": 48, "tokens_out": 46, "cost_usd": 3.48e-05}
> ```
>
> **Việc 1 — tổng hợp chi phí theo từng user mà không cần đọc log bằng mắt.**
> Vì mỗi dòng là một JSON object có `user_id` và `cost_usd`, tôi lọc và cộng
> dồn được bằng một câu lệnh:
>
> ```bash
> docker compose logs --no-log-prefix agent \
>   | grep ask_completed | jq -s 'group_by(.user_id)
>   | map({user: .[0].user_id, total: map(.cost_usd) | add})'
> ```
>
> Trên cloud thì đây chính là thứ Datadog/CloudWatch dùng để dựng biểu đồ chi
> phí và đặt cảnh báo kiểu "user nào tiêu quá 1 USD/ngày thì báo tôi".
> `print("đã trả lời xong")` không mang theo `user_id` lẫn `cost_usd`, nên
> điều duy nhất tôi đếm được là *số dòng* — không biết ai tiêu và tiêu bao
> nhiêu.
>
> **Việc 2 — truy vết một sự cố theo dòng thời gian.** `timestamp` là ISO-8601
> UTC nên sắp xếp và cắt theo khoảng thời gian được, còn `user_id` cho phép
> lọc ra đúng một người. Khi có người báo "lúc 11h tôi bị 429", tôi lọc đúng
> user đó trong khoảng thời gian đó và thấy ngay họ đã gọi bao nhiêu lần, mỗi
> lần tốn bao nhiêu token. Điều này còn quan trọng hơn khi chạy nhiều
> instance: log của 3 container đổ chung vào một nơi, không có `user_id` và
> `timestamp` chuẩn thì không cách nào ghép lại thành một câu chuyện. Log dạng
> chữ tự do thì mỗi lần muốn hỏi một câu mới lại phải viết regex mới, còn
> JSON thì chỉ là đổi tên trường.

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
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng **1.46 GB**, đến từ hai nguồn:
>
> **1. Base image (khoảng 900 MB).** Bản đầu dùng `python:3.11` đầy đủ, bản
> sau dùng `python:3.11-slim`. Bản đầy đủ mang theo cả một môi trường phát
> triển C: `gcc`, `g++`, `make`, header của Python và của các thư viện hệ
> thống, cộng thêm `git`, `curl`, và bộ tài liệu/locale của Debian. Đó là
> những thứ cần để *biên dịch* thư viện, không phải để *chạy* nó. Ảnh `slim`
> giữ lại đúng Python runtime và vài tiện ích tối thiểu.
>
> **2. Dấu vết của bước cài đặt (phần còn lại).** Ở bản 1 stage, `pip` để lại
> cache các file `.whl` và `.tar.gz` đã tải trong `~/.cache/pip`, cộng với
> file tạm sinh ra lúc build. Cách viết multi-stage loại bỏ chúng theo hai
> tầng: `--no-cache-dir` bảo pip đừng giữ file tải về, và quan trọng hơn là
> stage runtime chỉ `COPY --from=builder /install /usr/local` — tức là chỉ
> lấy **kết quả** cài đặt, còn toàn bộ hệ thống file của stage builder (kể cả
> compiler và mọi file rác sinh ra trong lúc build) bị vứt đi, không bao giờ
> xuất hiện trong image cuối.
>
> Điều tôi thấy thú vị: một layer đã tạo ra thì không xoá được. Nếu tôi chỉ
> thêm `RUN rm -rf ~/.cache/pip` vào cuối Dockerfile 1 stage, image vẫn không
> nhỏ đi — file cache vẫn nằm trong layer trước đó, chỉ là bị layer sau che
> lại. Multi-stage giải quyết được vì nó không xoá gì cả, nó bắt đầu lại từ
> một base sạch và chỉ mang sang thứ cần thiết.
>
> Về mặt thực tế, 1.46 GB đó là thời gian: mỗi lần deploy phải đẩy và kéo
> thêm chừng ấy dữ liệu qua mạng, và mỗi container mới scale lên đều phải đợi
> tải xong mới khởi động được.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> **Layer được dùng lại từ cache (toàn bộ phần đắt tiền):**
>
> - Cả stage `builder`: `FROM python:3.11-slim AS builder`, `WORKDIR /app`,
>   `COPY requirements.txt .`, `RUN pip install --no-cache-dir --prefix=/install`.
>   Chúng được giữ vì `requirements.txt` không đổi — Docker so sánh nội dung
>   file được COPY chứ không so sánh ngày sửa.
> - Trong stage runtime: `FROM`, `WORKDIR /app`, và
>   `COPY --from=builder /install /usr/local`.
>
> **Layer phải chạy lại:** từ `COPY . .` trở xuống — `COPY . .`,
> `RUN useradd ... && chown -R`, `USER appuser`, `EXPOSE`, `HEALTHCHECK`,
> `CMD`. Đây là quy tắc chung: một layer hỏng cache thì mọi layer sau nó cũng
> hỏng theo, vì mỗi layer được xây trên kết quả của layer trước.
>
> May là các layer này đều rẻ — chúng chỉ copy vài trăm KB source code và tạo
> một user. Build lại sau khi sửa code chỉ mất khoảng **3–4 giây**, so với
> khoảng một phút cho lần build đầu.
>
> **Nếu đặt `COPY . .` lên trước `RUN pip install`:** mọi thay đổi trong bất
> kỳ file nào của repo — kể cả sửa một dấu chấm trong README — đều làm hỏng
> cache của layer `COPY`, kéo theo `pip install` phải chạy lại từ đầu: tải và
> cài lại fastapi, uvicorn, pydantic, redis... Mỗi lần build mất thêm khoảng
> một phút thay vì 3 giây.
>
> Nguyên tắc rút ra: **xếp các layer theo tần suất thay đổi, ít đổi lên
> trước.** Danh sách thư viện vài tuần mới đổi một lần, còn source code thì
> đổi vài phút một lần — nên `requirements.txt` phải được copy riêng và đứng
> trước. Đó cũng là lý do `COPY requirements.txt .` phải là một dòng riêng
> chứ không gộp vào `COPY . .`.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Chuỗi sự kiện khi container chạy bằng root:**
>
> 1. Code Python của tôi có một lỗ hổng cho phép thực thi lệnh — ví dụ tôi
>    truyền input người dùng vào `os.system`/`eval`, hoặc dùng một thư viện
>    có CVE dạng deserialization/RCE mà tôi chưa vá.
> 2. Kẻ tấn công gửi một request được dựng riêng và đạt được khả năng chạy
>    lệnh tuỳ ý **bên trong container**, với quyền của tiến trình uvicorn.
> 3. Vì tiến trình đó là root, kẻ tấn công đọc được mọi file trong container:
>    biến môi trường của process (`/proc/1/environ` — nơi có `AGENT_API_KEY`
>    và `REDIS_URL` kèm mật khẩu), mọi secret được mount vào, toàn bộ source.
> 4. Vẫn với quyền root, họ cài thêm công cụ (`apt-get install`,
>    `pip install`), ghi đè file hệ thống trong `/usr/local`, và cắm một
>    backdoor vào chính code đang chạy.
> 5. Bước cuối — thoát ra host — cần thêm một điều kiện, và đây là chỗ quyền
>    root thật sự nguy hiểm: nếu container được chạy với `--privileged`, hoặc
>    được mount `/var/run/docker.sock` hay một thư mục của host, hoặc kernel
>    có lỗ hổng thoát container, thì root-trong-container biến thành
>    root-trên-host. Lý do là mặc định Docker **không bật user namespace
>    remapping**: UID 0 trong container chính là UID 0 của host, chỉ bị giới
>    hạn bởi namespace và capabilities. Chẳng hạn chỉ cần `docker.sock` được
>    mount là kẻ tấn công tạo một container mới có `-v /:/host` và đọc/ghi
>    toàn bộ đĩa của host — không cần lỗ hổng kernel nào cả.
>
> **`USER appuser` cắt chuỗi ở bước 3.** Từ dòng đó trở đi, uvicorn chạy bằng
> một UID thường, nên khi kẻ tấn công đạt được thực thi lệnh ở bước 2, họ kế
> thừa đúng quyền hạn chế đó:
>
> - không ghi được vào `/usr/local` hay bất kỳ thư mục hệ thống nào, nên
>   không cài được công cụ và không sửa được thư viện;
> - không đọc được các file chỉ root mới đọc được;
> - nếu có thoát được ra host qua một lỗ hổng nào đó, họ hạ cánh xuống một
>   UID không đặc quyền chứ không phải root, tức là còn phải tìm thêm một lỗ
>   hổng leo thang đặc quyền nữa.
>
> Nói cách khác, `USER` không ngăn được bước 1 và 2 — lỗ hổng vẫn là lỗ hổng.
> Nó biến một sự cố "mất toàn bộ host" thành "một tiến trình bị chiếm quyền
> trong một container dùng một lần". Đây là phòng thủ nhiều lớp: giả định
> lớp trước sẽ thủng, và làm cho hậu quả nhỏ nhất có thể.
>
> Trong Dockerfile của tôi, thứ tự cũng quan trọng: `useradd` và `chown -R`
> phải chạy **sau** `COPY . .` (lúc còn là root) rồi mới `USER appuser`. Đặt
> `USER` lên sớm hơn thì các lệnh COPY phía sau không ghi được vào `/app`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **Tối đa 20 request trong 2 giây** — gấp đôi hạn mức.
>
> Cách đạt được: bám vào đúng ranh giới giữa hai phút đồng hồ.
>
> - Từ 10:00:59.0 đến 10:00:59.9 — gửi 10 request. Bộ đếm của phút `10:00`
>   tăng từ 0 lên 10, vừa chạm hạn mức nhưng chưa vượt, nên cả 10 đều được
>   cho qua.
> - Đúng 10:01:00.0 — bộ đếm reset về 0 vì đã sang phút mới.
> - Từ 10:01:00.0 đến 10:01:00.9 — gửi tiếp 10 request nữa. Bộ đếm của phút
>   `10:01` chạy từ 0 lên 10, cũng hợp lệ.
>
> Kết quả: 20 request trong khoảng chưa tới 2 giây, mà xét theo luật thì
> không phút nào bị vượt quá 10. Với LLM, 20 request dồn trong 2 giây có thể
> đủ làm nghẽn worker và đội chi phí gấp đôi so với dự tính.
>
> Sliding window trong `rate_limiter.py` không có lỗ hổng này vì nó không hỏi
> "phút đồng hồ hiện tại là phút nào", mà hỏi "trong **60 giây vừa qua** có
> bao nhiêu request". Tại thời điểm 10:01:00,
> `zremrangebyscore(key, 0, now - 60)` chỉ xoá những entry cũ hơn 10:00:00,
> nên 10 request lúc 10:00:59 vẫn nằm nguyên trong ZSET và `zcard` vẫn trả về
> 10 → request thứ 11 bị chặn ngay.
>
> Cái giá phải trả là bộ nhớ: cửa sổ trượt phải lưu timestamp của **từng**
> request (mỗi user một ZSET) thay vì chỉ một con số đếm. Đó là lý do phải
> có `expire(key, WINDOW_SECONDS)` để Redis tự dọn, và member của ZSET phải
> là chuỗi duy nhất (`f"{now}:{uuid4().hex}"`) — nếu chỉ dùng timestamp làm
> member thì hai request đến trong cùng một thời điểm sẽ ghi đè lên nhau và
> tôi đếm thiếu.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau ở đại lượng được giới hạn và ở khung thời gian.** Rate limit đếm
> *số request* trong 60 giây gần nhất; nó bảo vệ tài nguyên máy chủ khỏi bị
> dồn dập. Cost guard cộng dồn *số tiền* trong cả tháng; nó bảo vệ ví. Một
> request là một đơn vị với rate limit, nhưng với cost guard thì một request
> có thể đắt gấp trăm lần request khác tuỳ độ dài prompt và câu trả lời.
> Chúng cũng khác về mã lỗi: 429 (Too Many Requests, kèm `Retry-After` — thử
> lại sau là được) so với 402 (Payment Required — thử lại cũng vô ích cho tới
> khi sang tháng mới hoặc nâng ngân sách).
>
> **Rate limit cho qua nhưng cost guard chặn:** một user hỏi đều đặn 5
> request/phút, luôn dưới hạn mức 10/phút nên không bao giờ chạm 429. Nhưng
> mỗi câu hỏi của họ dài và hội thoại đã tích đủ 20 message lịch sử, nên mỗi
> lần gọi phải gửi kèm toàn bộ ngữ cảnh đó — số token đầu vào lớn gấp nhiều
> lần một câu hỏi thường. Duy trì nhịp này vài giờ là tiêu hết 10 USD ngân
> sách tháng. Cost guard trả 402 dù người này chưa từng gọi quá nhanh một
> giây nào.
>
> **Cost guard cho qua nhưng rate limit chặn:** một user mới toanh, `spent()`
> trả về 0.0 nên còn nguyên ngân sách. Nhưng script phía họ bị lỗi vòng lặp và
> bắn 50 request trong 10 giây. Rate limit chặn từ request thứ 11 với 429,
> trong khi cost guard hoàn toàn im lặng vì tổng chi tiêu vẫn gần bằng 0.
>
> Trong `/ask` tôi gọi `limiter.check()` trước rồi mới `guard.check()`, và cả
> hai đều đứng trước `ask_llm()`. Thứ tự này quan trọng vì tiền chỉ mất ở
> bước gọi LLM: chặn sau khi đã gọi thì tôi vừa trả tiền cho câu trả lời, vừa
> trả về lỗi cho người dùng — mất cả đôi đường.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Giả sử endpoint gộp đó vừa được dùng làm liveness probe (Docker
> `HEALTHCHECK`, orchestrator quyết định restart) vừa làm readiness probe
> (nginx quyết định gửi traffic).
>
> 1. **Giây 0** — Redis ngừng phản hồi.
> 2. **Giây ~10** — đến nhịp probe (`--interval=10s`), cả 3 container cùng
>    gọi `store.ping()`, cùng nhận exception, cùng trả 503. Cả cụm chuyển sang
>    unhealthy **đồng thời**, vì cả 3 phụ thuộc chung một Redis.
> 3. **Ngay sau đó** — nginx thấy cả 3 upstream đều lỗi nên rút hết khỏi vòng
>    round-robin. Không còn instance nào nhận traffic: 100% request trả
>    502/503, **kể cả `/health` và những request vốn không cần Redis**.
> 4. **Giây ~30–40** — sau đủ số lần fail liên tiếp (`--retries=3`),
>    orchestrator kết luận cả 3 container đã hỏng và **restart cả 3 cùng lúc**.
>    Đây là thiệt hại nặng nhất: process Python hoàn toàn khoẻ mạnh bị giết vì
>    một dịch vụ bên ngoài gặp sự cố.
> 5. **Giây ~40–55** — container mới khởi động. Nếu Redis vẫn chưa lên, probe
>    lại fail, lại bị restart — cụm rơi vào **crash loop**. Trạng thái
>    in-memory (kết nối, cache nội bộ) mất sạch sau mỗi vòng.
> 6. **Giây 30** — Redis thật ra đã hồi phục. Nhưng cụm thì chưa: các container
>    đang ở giữa chu kỳ khởi động lại, phải đợi uvicorn lên, đợi qua
>    `start_period`, đợi probe xanh trở lại rồi nginx mới đưa vào vòng.
>
> **Kết quả: Redis chết 30 giây, nhưng service chết lâu hơn thế nhiều** — và
> phần lớn thời gian chết thêm là do chính cơ chế tự phục hồi gây ra.
>
> Khi tách hai endpoint như trong bài làm của tôi, cùng kịch bản đó diễn ra
> khác hẳn:
>
> 1. Giây 0 — Redis ngừng phản hồi.
> 2. `/health` không chạm Redis nên vẫn trả 200 → **không container nào bị
>    restart**, cả 3 process vẫn sống và giữ nguyên trạng thái.
> 3. `/ready` gọi `store.ping()`, nhận `False`, trả 503 → nginx ngừng đẩy
>    request mới vào. Người dùng thấy lỗi, nhưng đó là lỗi trung thực: service
>    thật sự chưa phục vụ được.
> 4. Giây 30 — Redis lên lại. `ping()` trả `True` ngay ở nhịp probe kế tiếp,
>    `/ready` trả 200, nginx đưa cả 3 instance trở lại vòng.
> 5. Tổng thời gian gián đoạn ≈ 30 giây + một nhịp probe, **không có lần
>    restart nào**.
>
> Đây là lý do hai câu hỏi phải tách bạch: `/health` trả lời "có cần
> **restart** tiến trình này không?", `/ready` trả lời "có nên **gửi request**
> vào tiến trình này lúc này không?". Trộn hai câu hỏi lại là để một sự cố
> tạm thời ở dependency kích hoạt một hành động không thể hoàn tác.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi phải sửa `docker-compose.yml` trước mới chạy được lệnh này: ban đầu tôi
> map cứng `"8000:8000"` nên khi scale lên 3, container thứ hai chết ngay với
> `Bind for 0.0.0.0:8000 failed: port is already allocated` — ba container
> không thể cùng chiếm một cổng của host. Tôi đổi thành `- "8000"` để Docker
> tự cấp cổng ngẫu nhiên, rồi thêm service `nginx` giữ cổng 8000 và chia tải
> xuống ba instance.
>
> **Kết quả đo được với 6 request cùng `X-User-Id: sv-scale`:**
>
> | Lần gọi | 1 | 2 | 3 | 4 | 5 | 6 |
> |---|---|---|---|---|---|---|
> | `history_length` | 0 | 2 | 4 | 6 | 8 | 10 |
>
> Đếm log của từng container thì 6 request được chia đều **2/2/2** cho
> agent-1, agent-2, agent-3. Nghĩa là dãy số tăng đều 0, 2, 4, 6, 8, 10 đó
> được tạo ra bởi ba tiến trình khác nhau, trên ba container khác nhau, mà
> không tiến trình nào biết gì về hai tiến trình kia. Mỗi lượt tăng 2 vì một
> lượt hỏi ghi 2 message: một của `user`, một của `assistant`.
>
> **Nếu lịch sử nằm trong một dict Python:** mỗi container có vùng RAM riêng
> nên có một dict riêng, hoàn toàn độc lập. Với round-robin, tôi sẽ thấy:
>
> | Lần gọi | 1 | 2 | 3 | 4 | 5 | 6 |
> |---|---|---|---|---|---|---|
> | Container | 1 | 2 | 3 | 1 | 2 | 3 |
> | `history_length` | 0 | 0 | 0 | 2 | 2 | 2 |
>
> Con số không tăng đều mà **giậm chân theo cụm 3**, vì mỗi container chỉ đếm
> được những lượt do chính nó xử lý. Từ phía người dùng, agent nhớ được đúng
> một phần ba câu chuyện, và nhớ nhầm phần nào thì tuỳ load balancer.
>
> Hai hệ quả nữa mà bảng trên chưa thể hiện:
>
> - **Restart là mất trắng.** Container bị restart (deploy bản mới, OOM,
>   crash) thì dict biến mất cùng process. Với Redis, tôi restart cả 3
>   container mà lịch sử vẫn còn, vì nó nằm ngoài vòng đời của process.
> - **Không scale ra được.** Mỗi instance thêm vào làm chất lượng tệ đi chứ
>   không tốt lên — thêm container nghĩa là thêm một bản ký ức rời rạc nữa.
>
> Đó chính là ý nghĩa của "stateless": bản thân process không giữ gì cả, nên
> ba container là ba bản sao thay thế được cho nhau. Test
> `test_state_khong_nam_trong_process` mô phỏng đúng điều này bằng cách tạo
> hai `ConversationStore` khác nhau trên cùng một Redis và kiểm tra cái thứ
> hai đọc được dữ liệu cái thứ nhất vừa ghi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi deploy lên **Railway** và mất bốn lần thử mới xanh. Kể lại cả bốn vì mỗi
> lần lỗi ở một tầng khác nhau, và chính điều đó dạy tôi cách khoanh vùng.
>
> ---
>
> **Nấc 1 — `Healthcheck failure` sau 30 giây.**
>
> Dashboard báo Build ✓, Deploy ✓, nhưng `Network › Healthcheck` ✗. Nhìn từ
> ngoài thì giống hệt "app chậm", nhưng Deploy Logs nói khác:
>
> ```
> Usage: uvicorn [OPTIONS] APP
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
>
> Chuỗi `'$PORT'` xuất hiện **nguyên văn, có cả dấu `$`** — nghĩa là biến
> không rỗng mà là *không được giãn*. Nguyên nhân nằm ở `railway.toml`:
>
> ```toml
> startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"
> ```
>
> Railway chạy `startCommand` dạng exec, không qua shell, nên `$PORT` được
> truyền tới uvicorn như một chuỗi 5 ký tự. Tệ hơn, `startCommand` **ghi đè**
> `CMD` trong Dockerfile, nên bản `sh -c ... ${PORT:-8000}` mà tôi đã viết
> đúng ở CP2 hoàn toàn bị vô hiệu. Đây đúng là cái bẫy tôi từng gặp ở CP2,
> chỉ chuyển chỗ xảy ra.
>
> **Sửa:** xoá hẳn dòng `startCommand` để Railway dùng `CMD` trong Dockerfile.
>
> ---
>
> **Nấc 2 — `404 Application not found`.**
>
> Gọi vào domain thì nhận 404, nhưng header mới là chỗ đáng đọc:
>
> ```
> Server: railway-hikari
> x-railway-fallback: true
> {"status":"error","code":404,"message":"Application not found"}
> ```
>
> Không có header `server: uvicorn` — nghĩa là request chưa từng chạm tới app
> của tôi, nó chết ngay ở router biên của Railway. Router không tìm thấy
> container nào để chuyển tiếp.
>
> Nguyên nhân hoá ra rất tầm thường: Railway không áp dụng ngay khi tôi thêm
> biến môi trường, nó gom lại thành *staged changes* và chờ bấm nút **Deploy**.
> Tôi thêm 5 biến rồi đi làm việc khác, nên không có deployment nào sống cả.
>
> ---
>
> **Nấc 3 và 4 — `502 Application failed to respond`, do lệch cổng.**
>
> Sau khi bấm Deploy, mã lỗi đổi từ 404 sang 502. Bản thân sự thay đổi này đã
> là thông tin: router giờ **đã tìm thấy** container, nhưng gõ cửa mà không ai
> trả lời. Deploy Logs cho thấy app hoàn toàn khoẻ mạnh:
>
> ```
> INFO:     Application startup complete.
> INFO:     Uvicorn running on http://0.0.0.0:8080
> ```
>
> Uvicorn nghe **8080**, trong khi lúc Generate Domain tôi đã gõ tay target
> port là **8000**. Railway cấp `PORT=8080`, app đọc đúng biến đó và bind
> 8080 — tức là phần `${PORT:-8000}` hoạt động chính xác — nhưng router lại
> đẩy traffic vào 8000.
>
> **Sửa:** đặt tường minh biến `PORT=8000` trên dashboard, để cả ba chỗ cùng
> một con số: biến môi trường, target port của domain, và `EXPOSE 8000` trong
> Dockerfile.
>
> ---
>
> **Kết quả sau khi sửa:**
>
> ```
> /health         → 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
> /ready          → 200 {"status":"ready","redis":true}
> /ask không key  → 401 {"detail":"invalid or missing API key"}
> /ask có key     → 200, history_length 0 → 2 giữa hai request
> 13 lần liên tiếp → 200 ×10 rồi 429 429 429
> ```
>
> ---
>
> **Điều tôi rút ra.** Mã lỗi HTTP và header cho biết request chết ở **tầng
> nào**, và đó là thứ quyết định nên đi đọc log ở đâu:
>
> | Triệu chứng | Chết ở tầng | Đọc gì |
> |---|---|---|
> | 404 + `x-railway-fallback: true` | router biên | trạng thái deployment |
> | 502 `failed to respond` | giữa router và container | cổng app đang nghe |
> | `server: uvicorn` + lỗi 4xx/5xx | bên trong app | Deploy Logs, traceback |
>
> Ba lần đầu tôi đều suýt sửa mò. Lần nào đọc log trước cũng tìm ra nguyên
> nhân trong chưa tới một phút, còn đoán thì tốn cả chục phút và một lần
> deploy vô ích. Đây cũng là lý do `log_event` ở CP1 đáng giá: dòng
> `{"event": "service_started", ...}` xuất hiện trong Deploy Logs chính là
> ranh giới rõ ràng giữa "app chưa khởi động nổi" và "app sống, lỗi nằm ở
> tầng mạng".
