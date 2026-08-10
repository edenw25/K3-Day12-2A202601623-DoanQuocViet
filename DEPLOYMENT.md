# Thông Tin Deploy — Checkpoint 5

> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đoàn Quốc Việt |
| Mã học viên | 2A202601623 |
| Repo | https://github.com/edenw25/K3-Day12-2A202601623-DoanQuocViet |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k3-day12-2a202601623-doanquocviet-production.up.railway.app |
| Platform | Railway |
| Region | US West (edge Singapore — sin1) |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | ghim 8000 cho khớp target port của domain và `EXPOSE 8000` |
| `AGENT_API_KEY` | ✅ | sinh bằng `secrets.token_urlsafe(32)`, đặt trong dashboard Railway, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu service Redis của Railway bằng `${{Redis.REDIS_URL}}` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
URL=https://k3-day12-2a202601623-doanquocviet-production.up.railway.app

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i $URL/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i $URL/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DEPLOY_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 13 lần, những lần cuối phải trả 429
for i in $(seq 1 13); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $URL/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $DEPLOY_API_KEY" \
    -H "X-User-Id: sv-rate" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy lúc 2026-08-10, gọi từ máy cá nhân qua Internet:

```
# 1. /health
HTTP/1.1 200 OK
Content-Type: application/json
Server: railway-hikari
x-railway-edge: sin1

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. /ready
{"status":"ready","redis":true}        [HTTP 200]

# 3. /ask không có API key
{"detail":"invalid or missing API key"}   [HTTP 401]

# 4. /ask có API key — lượt hỏi thứ nhất
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến
môi trường, health check để orchestrator biết trạng thái, và giới hạn tài
nguyên.","user_id":"sv-test","history_length":0,"cost_usd":2.265e-05,
"tokens":{"in":3,"out":37}}

# 4b. /ask lượt thứ hai, cùng X-User-Id — lịch sử đọc lại từ Redis trên cloud
history_length : 2
cost_usd       : 3.57e-05
tokens         : in=46 out=48

# 5. Rate limit, 13 lần liên tiếp cùng một X-User-Id
200 200 200 200 200 200 200 200 200 200 429 429 429
```

Ba điểm đáng chú ý trong kết quả trên:

- `/ready` trả `redis: true` nghĩa là `store.ping()` gọi được tới Redis add-on
  của Railway qua biến tham chiếu, không phải Redis chạy trong cùng container.
- `history_length` tăng từ 0 lên 2 giữa hai request độc lập — state nằm ở
  Redis nên sống sót qua từng request, đúng yêu cầu stateless của CP4.
- Rate limit cắt đúng sau request thứ 10, khớp `RATE_LIMIT_PER_MINUTE=10`.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang Deployments trên Railway, deployment ở
  trạng thái ACTIVE với đủ 5 mốc Initialization / Build / Deploy / Network /
  Post-deploy
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt

## Sự Cố Gặp Phải Khi Deploy

Ghi lại để đối chiếu với câu 10 trong `exercises.md`. Bốn nấc, mỗi nấc một
tầng khác nhau của hệ thống:

1. **Healthcheck failure sau 30 giây.** Deploy Logs:
   `Error: Invalid value for '--port': '$PORT' is not a valid integer`.
   `startCommand` trong `railway.toml` được Railway chạy dạng exec, không qua
   shell, nên `$PORT` không giãn. Sửa: xoá `startCommand` để Railway dùng
   `CMD` trong Dockerfile — dòng đó đã bọc `sh -c` và có mặc định
   `${PORT:-8000}`.
2. **404 `Application not found`** kèm header `x-railway-fallback: true`.
   Router của Railway không tìm thấy container nào để chuyển tiếp: các biến
   môi trường vừa thêm còn ở trạng thái staged, chưa bấm nút Deploy.
3. **502 `Application failed to respond`.** Đã có container nhưng lệch cổng.
4. Deploy Logs cho thấy `Uvicorn running on http://0.0.0.0:8080` trong khi
   domain trỏ vào cổng 8000. Railway cấp `PORT=8080`, app đọc đúng biến đó,
   nhưng target port của domain đã bị ghim tay là 8000. Sửa: đặt tường minh
   `PORT=8000` để cả ba chỗ (biến môi trường, target port, `EXPOSE`) cùng
   một số.

Bài học chung: mã lỗi HTTP và header cho biết request chết ở tầng nào — 404
kèm `x-railway-fallback` là chết ở router, 502 là chết giữa router và
container, còn traceback trong Deploy Logs là chết bên trong app.
