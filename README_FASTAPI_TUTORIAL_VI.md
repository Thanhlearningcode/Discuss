# 🐍 FastAPI Digital Twin – Tutorial Dành Cho Newbie

> Mục tiêu: Sau khi đọc, bạn có thể: (1) Chạy server FastAPI, (2) Hiểu mọi middleware / route trong `fastapi_server.py`, (3) Gọi thử `/api/gpt`, (4) Mở rộng bảo mật / scale.

---
## 1. FastAPI là gì và vì sao dùng ở đây?
FastAPI là framework Python hiện đại giúp xây dựng API nhanh, cú pháp rõ ràng, hỗ trợ async (xử lý đồng thời) tốt hơn Flask. Trong dự án Digital Twin:
- Dễ tạo endpoint chuẩn REST (+ có thể nâng cấp lên WebSocket).
- Async gọi GPT (OpenAI) không chặn server.
- Tốc độ cao hơn trong tải lớn (khi nhiều request /api/gpt cùng lúc).
- Tự động docs (OpenAPI) nếu cần (truy cập `/docs`).

So sánh nhanh:
| Tiêu chí | Express (Node) | Flask | FastAPI |
|---------|----------------|-------|---------|
| Async intrinsic | ✔ (event loop) | ✖ (blocking) | ✔ (async/await) |
| Tốc độ gọi GPT song song | Tốt | Hạn chế | Tốt |
| Type hints | Tùy | Thủ công | Tích hợp mạnh |
| Auto docs | Cần plugin | Không mặc định | ✔ Tự động |

---
## 2. Sơ đồ luồng cơ bản
```
Trình duyệt (UI) ── cookie session ──> FastAPI
      |                               |
      | (fetch /api/gpt)              |-- httpx async --> OpenAI API
      |                               |
      | (fetch /api/me /api/login)    |  SessionMiddleware (cookie ký)
```
Thư mục tĩnh `/public` (chứa `login.html`, `app/index.html`) được mount để phục vụ giao diện.
`/app` yêu cầu đã đăng nhập (session tồn tại).

---
## 3. Chuẩn bị môi trường (Windows PowerShell)
```powershell
# Bước 1: Tạo virtual env
python -m venv .venv

# Bước 2: Kích hoạt
.venv\Scripts\activate

# Bước 3: Cài dependency
pip install -r requirements.txt

# (Tuỳ chọn) Kiểm tra phiên bản
python -V
pip list
```
Nếu chưa có `requirements.txt` đầy đủ, đảm bảo gồm: `fastapi`, `uvicorn`, `httpx`.

---
## 4. Chạy server
### Dev (hot reload)
```powershell
$env:OPENAI_API_KEY='sk-...'
$env:SESSION_SECRET='chuoi_bi_mat_doan_du'
uvicorn fastapi_server:app --host 0.0.0.0 --port 3000 --reload
```
### Production (ví dụ đơn giản)
```powershell
$env:NODE_ENV='production'
uvicorn fastapi_server:app --host 0.0.0.0 --port 3000 --workers 4
```
Triển khai thực tế thường đặt sau reverse proxy (Nginx/Apache/Cloudflare TLS). Server này không tự sinh chứng chỉ.

---
## 5. Cấu trúc chính của file `fastapi_server.py`
| Thành phần | Vai trò |
|------------|--------|
| Biến PATH / ENV | Xác định thư mục public, môi trường (NODE_ENV), secret session |
| SessionMiddleware | Quản lý cookie session đăng nhập |
| HTTPSRedirectMiddleware | Ép chuyển hướng HTTP → HTTPS nếu không phải localhost |
| HSTSMiddleware | Thêm header bảo mật HSTS ở production |
| StaticFiles mount `/public` | Phục vụ file tĩnh (HTML/JS/CSS) |
| Hàm `require_auth` | Guard yêu cầu user đăng nhập |
| Route auth (/api/login, logout, me) | Đăng nhập, kiểm tra trạng thái |
| Route bảo vệ `/app` | Trả về dashboard `index.html` nếu có session |
| Route `/api/gpt` | Proxy OpenAI Chat Completions async |
| Route `/health` | Kiểm tra nhanh tình trạng + host/protocol |
| Handler 404 custom | JSON lỗi rõ ràng |

---
## 6. Session – Cách hoạt động
Sử dụng `SessionMiddleware` của Starlette:
```python
app.add_middleware(SessionMiddleware,
    secret_key=SESSION_SECRET,
    same_site='lax',
    https_only=(NODE_ENV=='production'))
```
- `secret_key`: dùng để ký nội dung session (bắt buộc đổi khi lên production).
- Trình duyệt gửi lại cookie → server đọc và giải mã để lấy `request.session`.
- Khi login thành công: `request.session['user'] = { 'username': ... }`.

Đổi secret: đặt biến môi trường `SESSION_SECRET`. Ví dụ PowerShell: `$env:SESSION_SECRET='my-long-random'`.

---
## 7. Middleware HTTPS Redirect
Mục tiêu: Nếu truy cập bằng HTTP (trên domain thực), chuyển sang HTTPS 308.
Logic bỏ qua:
- Localhost / 127.0.0.1.
- Đường dùng cho Let's Encrypt `/.well-known/acme-challenge`.
Điều kiện phát hiện HTTPS dựa trên header reverse proxy (`x-forwarded-proto`, `x-forwarded-port`...).
Tắt redirect (ví dụ chạy sau proxy test): đặt `DISABLE_HTTPS_REDIRECT=1`.

---
## 8. Middleware HSTS
Thêm header:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
Chỉ khi `NODE_ENV=production`. Giúp trình duyệt buộc dùng HTTPS các lần sau.

---
## 9. Các Route quan trọng
### 9.1 `/api/login` (POST)
Input JSON:
```json
{ "username": "admin", "password": "123456" }
```
Nếu đúng → lưu session + trả `{ ok: true, username }`. Sai → 401.
### 9.2 `/api/logout` (POST)
Xóa session, trả `{ ok: true }`.
### 9.3 `/api/me` (GET)
Trả trạng thái: `{ loggedIn: true, user: {...} }` hoặc `{ loggedIn: false }`.
### 9.4 `/app` (GET)
Yêu cầu đã login. Trả file `public/app/index.html`.
### 9.5 `/public/...`
File tĩnh (truy cập trực tiếp JS/CSS/images).
### 9.6 `/api/gpt` (POST – bảo vệ)
Proxy câu hỏi tới OpenAI. Body:
```json
{ "question": "Giải thích pipeline assistant?" }
```
Response:
```json
{ "answer": "..." }
```
Các kiểm tra:
- Thiếu `OPENAI_API_KEY` → 500.
- Empty question / quá dài (>1000) → 400.
- Timeout GPT → 504.
### 9.7 `/health`
Kiểm tra host + protocol (hữu ích xem reverse proxy có gửi header đúng không).

---
## 10. Gọi thử bằng PowerShell
```powershell
# Login
Invoke-RestMethod -Uri http://localhost:3000/api/login -Method Post -Body '{"username":"admin","password":"123456"}' -ContentType 'application/json' -SessionVariable sess

# Me (dùng session đã lưu)
Invoke-RestMethod -Uri http://localhost:3000/api/me -WebSession $sess

# GPT (giả sử đã có OPENAI_API_KEY)
Invoke-RestMethod -Uri http://localhost:3000/api/gpt -Method Post -Body '{"question":"Hello GPT"}' -ContentType 'application/json' -WebSession $sess
```

---
## 11. Gọi thử bằng curl (Linux/macOS hoặc Git Bash)
```bash
curl -c cookie.txt -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"123456"}'

curl -b cookie.txt http://localhost:3000/api/me

curl -b cookie.txt -X POST http://localhost:3000/api/gpt \
  -H 'Content-Type: application/json' \
  -d '{"question":"Nêu điểm khác nhau giữa Flask và FastAPI"}'
```

---
## 12. Thêm Rate Limiting (Gợi ý – chưa code sẵn)
Bạn có thể dùng thư viện `slowapi`:
```powershell
pip install slowapi
```
Ví dụ tích hợp (pseudo):
```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi.responses import JSONResponse

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, lambda r, e: JSONResponse({'error':'Too Many Requests'}, status_code=429))

@app.post('/api/gpt')
@limiter.limit('15/minute')
async def api_gpt(...):
    ...
```
Hoặc tự viết token bucket: lưu dict {ip: [timestamps]} và lọc.

---
## 13. Bảo mật nâng cao đề xuất
| Mục | Hiện tại | Đề xuất |
|-----|----------|--------|
| SESSION_SECRET | Chuỗi mặc định | Dùng random dài >32 ký tự |
| HTTPS | Via proxy | Bật HSTS + chặn HTTP hoàn toàn |
| GPT endpoint | Cơ bản bảo vệ login | Thêm rate limit + kiểm tra từ khóa cấm (nếu cần) |
| Logging lỗi | Mặc định (print stack) | Dùng logging module + rotate file |
| Input SPEED, lệnh khác | Chưa trong FastAPI file | Validate min/max ở endpoint lệnh riêng |
| Env vars | Thủ công | Dùng `python-dotenv` trong dev |

Cài `python-dotenv` (dev):
```powershell
pip install python-dotenv
```
Tải trong code (đầu file):
```python
from dotenv import load_dotenv
load_dotenv()
```

---
## 14. Mở rộng tương lai
| Tính năng | Mô tả |
|-----------|------|
| WebSocket (Realtime) | `websocket_route` để push dữ liệu thay polling |
| Background Tasks | Dùng `asyncio.create_task` hoặc `FastAPI BackgroundTasks` |
| Event Streaming SSE | Cho client nhận dòng dữ liệu (Server-Sent Events) |
| RAG nội bộ | Thêm vector store (FAISS/Chroma) + embed FAQ động |
| OAuth2 thay login demo | Dùng `fastapi.security.OAuth2PasswordBearer` |
| Metrics Prometheus | Expose `/metrics` qua `prometheus_client` |

---
## 15. Triển khai Production mẫu (Nginx reverse proxy)
Nginx snippet (tham khảo):
```
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl;
  server_name example.com;
  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port 443;
  }
}
```
Chạy uvicorn với nhiều workers: `uvicorn fastapi_server:app --workers 4`.

---
## 16. Troubleshooting nhanh
| Vấn đề | Nguyên nhân | Cách xử lý |
|--------|-------------|------------|
| 404 `/app` | Chưa login | Gọi `/api/login` trước, kiểm tra cookie |
| 500 GPT missing key | Không set OPENAI_API_KEY | Xuất biến môi trường hoặc `.env` đúng |
| Timeout GPT 504 | Mạng chậm | Giảm câu hỏi, kiểm tra firewall |
| Cookie không lưu | Trình duyệt chặn third-party | Test ở tab mới hoặc xem devtools Application/Cookies |
| Redirect vòng lặp | Proxy header sai | Thêm `X-Forwarded-Proto https` trong reverse proxy |
| Không load static | Sai đường dẫn mount | Kiểm tra tồn tại thư mục `public` |

---
## 17. Checklist siêu nhanh cho Newbie (10 bước)
1. Tạo và kích hoạt venv.
2. Cài `requirements.txt`.
3. Set `SESSION_SECRET` và `OPENAI_API_KEY` (nếu dùng GPT).
4. Chạy uvicorn (`--reload` khi dev).
5. Mở `http://localhost:3000/login.html`.
6. Đăng nhập (demo: admin / 123456).
7. Truy cập `/app` (dashboard index.html).
8. Gọi `/api/me` xác nhận session.
9. Thử `/api/gpt` với câu hỏi đơn giản.
10. Xem log/console → không có lỗi nghiêm trọng.

Hoàn thành = bạn đã hiểu cơ bản FastAPI server. ✅

---
## 18. FAQ nhanh
| Hỏi | Đáp |
|-----|-----|
| Có cần GPT để server chạy? | Không, chỉ /api/gpt fail nếu thiếu key. |
| Đổi cổng thế nào? | `uvicorn ... --port 5000` hoặc env `PORT`. |
| Tắt HTTPS redirect? | `DISABLE_HTTPS_REDIRECT=1`. |
| Thêm route mới? | Dùng decorator `@app.get('/new')`. |
| Chạy nhiều máy? | Dùng reverse proxy + scale workers (uvicorn). |

---
## 19. Gợi ý tiếp theo
- Thêm rate limit thực tế.
- Viết test đơn vị (pytest) cho `/api/login` & `/api/gpt` (mock OpenAI).
- Thêm WebSocket cho push dữ liệu sensor.
- Log chuẩn (module `logging` + file rotate).

---
## 20. Trích đoạn quan trọng từ mã (tham khảo nhanh)
```python
@app.post('/api/gpt')
async def api_gpt(request: Request, user=Depends(require_auth)):
    key = os.environ.get('OPENAI_API_KEY')
    if not key:
        raise HTTPException(status_code=500, detail='Missing OPENAI_API_KEY')
    ...
    async with httpx.AsyncClient(timeout=20) as client:
        resp = await client.post('https://api.openai.com/v1/chat/completions', ...)
```
Ý nghĩa: kiểm tra key, bảo vệ bằng đăng nhập, gọi OpenAI async, xử lý timeout & lỗi HTTP.

---
### Kết luận
Bạn đã có nền tảng vững để phát triển tiếp (bảo mật, realtime nâng cao, AI phong phú). Hãy bắt đầu nhỏ, kiểm tra từng route rồi mở rộng.

Chúc bạn học tốt và xây dựng Digital Twin mạnh mẽ với FastAPI! 🚀
