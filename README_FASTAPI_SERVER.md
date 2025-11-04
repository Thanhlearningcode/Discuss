# FastAPI Server for Digital Twin

This README explains `fastapi_server.py` – a FastAPI implementation replacing the original Node `server.js` (and alternative to the Flask version).

## 1. Mục tiêu
Replicate core features:
- HTTPS redirect (+ optional disable) & HSTS in production
- Session-based auth (login/logout/me)
- Protected `/app` route serving `public/app/index.html`
- Static assets under `/public`
- Health check `/health`
- GPT proxy `/api/gpt` using OpenAI Chat Completions

## 2. Thư viện
| Library | Purpose |
|---------|---------|
| fastapi | High-performance web framework (built on Starlette) |
| uvicorn | ASGI server to run FastAPI |
| httpx   | Async HTTP client for calling OpenAI |
| starlette.middleware.sessions | Signed cookie session storage |

## 3. Cấu trúc
```
fastapi_server.py
public/...
requirements.txt
```

## 4. Chạy Development (Windows PowerShell)
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:OPENAI_API_KEY = 'sk-...'
$env:NODE_ENV = 'development'
uvicorn fastapi_server:app --host 0.0.0.0 --port 3000 --reload
```
Visit: http://localhost:3000

## 5. Biến môi trường
| Name | Ý nghĩa | Mặc định |
|------|---------|----------|
| PORT | Cổng chạy server | 3000 |
| NODE_ENV | `development` hoặc `production` | development |
| OPENAI_API_KEY | Key gọi OpenAI API | (bắt buộc) |
| OPENAI_MODEL | Model (vd: gpt-4o-mini) | gpt-4o-mini |
| SESSION_SECRET | Secret ký cookie session | change_this_secret |
| DISABLE_HTTPS_REDIRECT | `1` để tắt redirect HTTPS | 0 |

## 6. Mô tả mã nguồn
### 6.1 Khởi tạo app & session
```python
app = FastAPI(title="Digital Twin FastAPI Server")
app.add_middleware(SessionMiddleware, secret_key=SESSION_SECRET, same_site='lax', https_only=(NODE_ENV=='production'))
```
- SessionMiddleware: tạo cookie ký để lưu `session`.
- Truy cập bằng `request.session`.

### 6.2 HTTPS Redirect Middleware
```python
class HTTPSRedirectMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # kiểm tra X-Forwarded headers và redirect 308 nếu chưa HTTPS
```
- Giống logic Express trước: bỏ qua localhost, ACME challenge.

### 6.3 HSTS Middleware
Thêm header `Strict-Transport-Security` khi production.

### 6.4 Static Files
```python
app.mount('/public', StaticFiles(directory=str(PUBLIC_DIR)), name='public')
```
- File như `public/login.html` có thể truy cập `/public/login.html`.

### 6.5 Auth Routes
- POST `/api/login`: kiểm tra user demo, ghi vào `request.session['user']`.
- POST `/api/logout`: `session.clear()`.
- GET `/api/me`: trả trạng thái đăng nhập.

### 6.6 Protected `/app`
```python
@app.get('/app')
async def protected_app(user=Depends(require_auth)):
    return FileResponse(str(APP_INDEX))
```
- Dùng `Depends(require_auth)` để chặn nếu chưa login.

### 6.7 Health Check
Trả về `{ok, protocol, host}` dựa vào headers proxy.

### 6.8 GPT Proxy
```python
async with httpx.AsyncClient(timeout=20) as client:
    resp = await client.post('https://api.openai.com/v1/chat/completions', ...)
```
- Input: `{question}` JSON.
- Output: `{answer}`.
- Timeout: 20s, trả 504 nếu quá lâu.

### 6.9 404 Handler
Trả JSON `{error: 'Not found'}` thay vì HTML mặc định.

## 7. So sánh Express vs FastAPI
| Aspect | Express | FastAPI |
|--------|---------|--------|
| Routing | `app.get('/path')` | Decorators `@app.get('/path')` |
| Middleware | `app.use(fn)` | Custom `BaseHTTPMiddleware` / dependency injection |
| Session | `express-session` | `SessionMiddleware` (signed cookie) |
| Static | `express.static()` | `StaticFiles` mount |
| JSON Parsing | body-parser | Automatic JSON deserialization |
| Type hints | Optional | Core (Pydantic models, validation) |

## 8. Kiểm thử nhanh
Checklist:
- [ ] `/api/login` với đúng user trả `{ok:true}`
- [ ] `/api/login` sai trả 401
- [ ] `/api/me` phản ánh trạng thái
- [ ] `/app` trả index sau login, redirect /login.html khi chưa login
- [ ] `/api/gpt` trả answer khi có OPENAI_API_KEY
- [ ] `/health` trả ok

## 9. Rate Limit (tùy chọn)
Cài đặt: `pip install slowapi`
```python
from slowapi import Limiter
from slowapi.util import get_remote_address
limiter = Limiter(key_func=get_remote_address)
@app.post('/api/gpt')
@limiter.limit('15/minute')
async def api_gpt(...):
    ...
```
Add middleware integration per slowapi docs.

## 10. Production Gợi ý
- Dùng `uvicorn --workers 4` hoặc `gunicorn -k uvicorn.workers.UvicornWorker`.
- Đặt `NODE_ENV=production`, `SESSION_SECRET` mạnh.
- Reverse proxy thêm header `X-Forwarded-Proto=https`.

## 11. Bảo mật đề xuất
- Regenerate session ID (tự xóa + tạo mới trên login).
- Sử dụng HTTPS mọi nơi (HSTS đã hỗ trợ).
- Kiểm tra độ dài input, chặn question > 1000 ký tự.
- Thêm CSP header (Content-Security-Policy) để hạn chế script injection.

## 12. Mở rộng tiếp
- Chuyển sang JWT thay cho session nếu muốn stateless.
- WebSocket real-time: `websockets` hoặc `fastapi` builtin (via `WebSocket` route).
- Tách router: `auth_router`, `gpt_router`.

## 13. Troubleshooting
| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| 401 `/api/gpt` | Chưa login | Gọi `/api/login` trước |
| 500 Missing OPENAI_API_KEY | Chưa set biến | Export env KEY |
| 308 redirect lặp | Proxy không gửi header | Tạm thời đặt DISABLE_HTTPS_REDIRECT=1 |
| 504 GPT timeout | Mạng chậm hoặc model delay | Thử lại / tăng timeout |

## 14. Chạy Production ví dụ
```bash
export OPENAI_API_KEY=sk-...
export NODE_ENV=production
export SESSION_SECRET="super_long_random_secret"
uvicorn fastapi_server:app --host 0.0.0.0 --port 3000 --workers 4
```

## 15. Tham khảo
- FastAPI: https://fastapi.tiangolo.com/
- Uvicorn: https://www.uvicorn.org/
- httpx: https://www.python-httpx.org/
- OpenAI API: https://platform.openai.com/docs/api-reference

---
FastAPI phiên bản này sẵn sàng thay thế hoàn toàn cho Node hoặc Flask tùy lựa chọn deploy của bạn.
