# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tạ Đăng Dương |
| Mã học viên | 2A202603018 |
| Repo | https://github.com/duong004/K4-L3B-DAY12-TaDangDuong-2A202603018-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-zg2d.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value (kết nối nội bộ qua Blueprint) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo

```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness  
HTTP/2 200
date: Tue, 29 Sep 2026 04:48:50 GMT
content-type: application/json
cf-cache-status: DYNAMIC
rndr-id: 65273441-6df5-4e1f
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a4285b99e8e15dee-HKG
alt-svc: h3=":443"; ma=86400

# 2. Readiness  
HTTP/2 200
date: Tue, 29 Sep 2026 04:49:43 GMT
content-type: application/json
cf-cache-status: DYNAMIC
rndr-id: 181e9ab8-d422-4069
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a4285ce4fbf6043e-HKG
alt-svc: h3=":443"; ma=86400

# 3. Không có API key  
HTTP/2 401
date: Tue, 29 Sep 2026 04:49:58 GMT
content-type: application/json
cf-cache-status: DYNAMIC
rndr-id: 3b908d3e-74e9-4c39
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a4285d419b3ef57a-HKG
alt-svc: h3=":443"; ma=86400

# 4. Có API key  
HTTP/2 200
date: Tue, 29 Sep 2026 04:53:58 GMT
content-type: application/json
cf-cache-status: DYNAMIC
rndr-id: b2ddd41c-ac28-4968
server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-ray: a42863223d21ddc9-HKG
alt-svc: h3=":443"; ma=86400

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

# 5. Rate limit — gọi 15 lần  
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429

```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

* `screenshots/dashboard.png` — trang quản lý service trên platform
* `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl