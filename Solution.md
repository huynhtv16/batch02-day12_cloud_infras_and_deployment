# Bài Làm - Day 12 Code Lab

## Part 1 - Localhost vs Production

### Exercise 1.1 - Các anti-pattern trong `01-localhost-vs-production/develop/app.py`

1. API key bị hardcode trực tiếp trong source code.
2. Database URL chứa username/password cũng bị hardcode.
3. Bật `DEBUG` và `reload=True`, không phù hợp production.
4. Port bị cố định là `8000`, không đọc từ biến môi trường `PORT`.
5. Host bind vào `localhost`, không phù hợp khi chạy trong container/cloud.
6. Không có endpoint `/health` hoặc `/ready` để platform kiểm tra trạng thái.
7. Dùng `print()` thay vì logging chuẩn, thậm chí log cả secret.
8. Không có cơ chế graceful shutdown khi app bị dừng.

### Exercise 1.3 - So sánh Basic và Advanced

| Feature | Basic | Advanced | Tại sao quan trọng? |
|---|---|---|---|
| Config | Hardcode trong code | Đọc từ environment variables | Cùng một code/image có thể chạy ở dev, staging, production. |
| Secrets | Lưu trong source code | Lấy từ biến môi trường | Tránh lộ key khi push code lên GitHub. |
| Host/port | `localhost:8000` | `0.0.0.0` và `PORT` env | Bắt buộc khi deploy Docker, Railway, Render. |
| Health check | Không có | Có `/health` và `/ready` | Platform biết app còn sống hay sẵn sàng nhận traffic. |
| Logging | `print()` | Structured JSON logging | Dễ đọc log, search, parse và monitor. |
| Shutdown | Tắt đột ngột | Có lifespan/SIGTERM | Cho request đang chạy hoàn thành trước khi process thoát. |

## Part 2 - Docker Containerization

### Exercise 2.1 - Dockerfile cơ bản

1. Base image là `python:3.11`.
2. Working directory là `/app`.
3. Copy `requirements.txt` trước để Docker cache layer cài dependencies. Khi chỉ sửa code, Docker không phải cài lại thư viện.
4. `CMD` là command mặc định khi container chạy và có thể override. `ENTRYPOINT` thường dùng để cố định executable chính của container.

### Exercise 2.2 - Build và run basic image

Lệnh đã chạy:

```bash
docker build -f 02-docker/develop/Dockerfile -t lab02-agent-develop .
docker run -p 18000:8000 lab02-agent-develop
```

Kết quả kiểm tra:

```text
POST /ask?question=What%20is%20Docker -> 200
Image size: khoảng 1.66GB
```

Nhận xét: image basic chạy được nhưng khá lớn vì dùng `python:3.11` full image và single-stage build.

### Exercise 2.3 - Multi-stage build

Dockerfile production có 2 stage:

| Stage | Nhiệm vụ |
|---|---|
| Builder | Cài dependencies và build package cần thiết. |
| Runtime | Chỉ copy dependencies đã cài và source code cần để chạy app. |

Kết quả:

```text
lab02-agent-production image size: khoảng 236MB
```

Image production nhỏ hơn vì dùng `python:3.11-slim` và không giữ lại build tools trong runtime image.

### Exercise 2.4 - Docker Compose stack

Các service trong stack:

| Service | Vai trò |
|---|---|
| `nginx` | Reverse proxy/load balancer, expose ra host port `8081`. |
| `agent` | FastAPI AI agent, chạy nội bộ port `8000`. |
| `redis` | Cache/session service cho stack production-style. |

Sơ đồ:

```text
Client -> Nginx :8081 -> Agent :8000
                         |
                         -> Redis :6379
```

Kết quả test:

```text
GET /health qua Nginx -> 200
POST /ask qua Nginx -> 200
```

Checkpoint 2:

- Hiểu cấu trúc Dockerfile.
- Biết lợi ích của multi-stage build.
- Hiểu Docker Compose orchestration.
- Biết dùng `docker logs`, `docker exec`, `docker ps`, `docker images`.

## Part 3 - Cloud Deployment

### Exercise 3.1 - Railway

App Railway đọc port từ biến môi trường:

```python
port = int(os.getenv("PORT", 8000))
```

Điều này quan trọng vì Railway tự inject biến `PORT`.

Kết quả smoke test local:

```text
GET /health -> 200
POST /ask -> 200
```

Các bước deploy Railway:

```bash
railway login
railway init
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
railway up
railway domain
```

### Exercise 3.2 - Render

So sánh `render.yaml` và `railway.toml`:

| Railway | Render |
|---|---|
| `railway.toml` chủ yếu cấu hình build/deploy cho service. | `render.yaml` mô tả service, runtime, region, plan, env vars, health check, add-on. |
| Workflow thiên về CLI: `railway init`, `railway up`. | Workflow thiên về GitHub Blueprint: push repo, connect Render, Render đọc `render.yaml`. |
| Env vars set bằng CLI hoặc dashboard. | Env vars khai báo trong Blueprint hoặc để `sync: false`. |

### Exercise 3.3 - GCP Cloud Run

`cloudbuild.yaml` mô tả pipeline build/deploy.  
`service.yaml` mô tả Cloud Run service như image, env vars, port, scaling và runtime config.

Checkpoint 3:

- Đã hiểu cách set environment variables trên cloud.
- Đã hiểu health check path khi deploy.
- Đã có cấu hình Render để push/deploy.

Public URL:

```text
https://batch02-day12cloudinfrasanddeployment-production-20e4.up.railway.app/
```

API URL:

```text
https://batch02-day12cloudinfrasanddeployment-production-20e4.up.railway.app/
```

## Part 4 - API Security

### Exercise 4.1 - API Key authentication

API key được kiểm tra trong hàm `verify_api_key()` thông qua header:

```text
X-API-Key
```

Kết quả:

| Trường hợp | Response |
|---|---|
| Không có key | `401 Unauthorized` |
| Sai key | `403 Forbidden` |
| Đúng key | `200 OK` |

Cách rotate key:

1. Đổi biến môi trường `AGENT_API_KEY`.
2. Restart/redeploy service.
3. Client dùng key mới trong header `X-API-Key`.

Kết quả test:

```text
Không có key -> 401
Có key đúng -> 200
```

### Exercise 4.2 - JWT authentication

JWT flow:

1. User gửi username/password đến `/auth/token`.
2. Server xác thực thông tin đăng nhập.
3. Server tạo JWT có `sub`, `role`, `iat`, `exp`.
4. Client gọi API với header `Authorization: Bearer <token>`.
5. Server verify token và lấy thông tin user/role.

Kết quả test:

```text
POST /auth/token -> 200
POST /ask với Bearer token -> 200
GET /me/usage với Bearer token -> 200
```

### Exercise 4.3 - Rate limiting

Algorithm được dùng: sliding window bằng `deque` lưu timestamp request.

Limit:

| Role | Limit |
|---|---|
| user | 10 requests/phút |
| admin | 100 requests/phút |

Admin không bypass hoàn toàn, nhưng được dùng limiter riêng có quota cao hơn (`rate_limiter_admin`).

Khi vượt limit, API trả:

```text
429 Too Many Requests
```

### Exercise 4.4 - Cost guard

Trong bản demo production, `CostGuard` đang dùng in-memory để:

1. Kiểm tra budget trước khi gọi mock LLM.
2. Ước lượng token input/output.
3. Ghi nhận chi phí sau mỗi request.
4. Trả `402` nếu user vượt budget.
5. Trả `503` nếu global budget vượt giới hạn.

Trong production thật, nên chuyển sang Redis để nhiều instance dùng chung budget:

```text
budget:{user_id}:{YYYY-MM}
```

Logic Redis đề xuất:

```python
month_key = datetime.now().strftime("%Y-%m")
key = f"budget:{user_id}:{month_key}"
current = float(r.get(key) or 0)

if current + estimated_cost > 10:
    return False

r.incrbyfloat(key, estimated_cost)
r.expire(key, 32 * 24 * 3600)
return True
```

Checkpoint 4:

- Đã implement/test API key authentication.
- Đã hiểu JWT flow.
- Đã implement/test rate limiting.
- Đã hiểu cost guard và cách chuyển sang Redis.

## Part 5 - Scaling & Reliability

### Exercise 5.1 - Health checks

Các endpoint:

| Endpoint | Vai trò |
|---|---|
| `/health` | Liveness probe: process còn sống không. |
| `/ready` | Readiness probe: app sẵn sàng nhận traffic chưa. |

Trong bản production Part 5, `/ready` kiểm tra Redis trước khi báo ready.

### Exercise 5.2 - Graceful shutdown

App dùng FastAPI lifespan và signal handler. Khi shutdown:

1. App đổi trạng thái không ready.
2. Uvicorn xử lý graceful shutdown.
3. Log lại quá trình shutdown.
4. Đóng tài nguyên cần thiết.

### Exercise 5.3 - Stateless design

Anti-pattern:

```python
conversation_history = {}
```

Vấn đề: khi scale nhiều instance, mỗi instance có memory riêng, user có thể mất session nếu request sau đi vào instance khác.

Cách đúng:

```text
Lưu session/history trong Redis
```

Nhờ đó bất kỳ instance nào cũng đọc được cùng một session.

### Exercise 5.4 - Load balancing

Stack Part 5:

```text
Client -> Nginx :8080 -> Agent replica 1
                    | -> Agent replica 2
                    | -> Agent replica 3
                    |
                    -> Redis
```

Lệnh đã chạy:

```bash
docker compose -p lab05scale -f 05-scaling-reliability/production/docker-compose.yml up -d --build --scale agent=3
```

### Exercise 5.5 - Test stateless

Kết quả từ `test_stateless.py`:

```text
3 agent instances khác nhau đã xử lý request.
Session history vẫn được giữ nguyên trong Redis.
Tổng số message trong history: 10.
```

Checkpoint 5:

- Đã implement/test health và readiness checks.
- Đã có graceful shutdown.
- Đã refactor thành stateless bằng Redis.
- Đã chạy Nginx load balancing.
- Đã test session vẫn giữ khi request đi qua nhiều instance.

## Part 6 - Final Project

Project cá nhân/nhóm được đặt tại:

```text
06-lab-complete/codebase
```

Tên project:

```text
An - Conversational Symptom Triage Assistant
```

Các phần productionization đã làm:

| Requirement | Trạng thái |
|---|---|
| Frontend app | Hoàn thành, dùng React + Vite. |
| Backend REST API | Hoàn thành, có `/triage` và `/health`. |
| Dockerized | Hoàn thành, multi-stage Dockerfile. |
| Config từ env vars | Hoàn thành, dùng `PORT`, provider/model env, API key env. |
| Health endpoint | Hoàn thành, `GET /health`. |
| Serve frontend trong production container | Hoàn thành, backend serve thư mục `dist`. |
| Render config | Hoàn thành, có `render.yaml` ở root repo. |
| Public/API URL note | Để trống để điền sau khi deploy. |

Kết quả Docker test:

```text
Image: an-symptom-triage-render
GET /health -> 200
GET / -> 200 text/html
```

Public URL:

```text

```

API URL:

```text

```
