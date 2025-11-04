# Python Server (Flask) for Digital Twin

This document explains the new `python_server.py` which replicates the behavior of the original Node `server.js` using Python + Flask.

## 1. Mục tiêu thay thế
| Node.js (server.js) | Python (python_server.py) |
|---------------------|---------------------------|
| Express middleware  | Flask decorators / hooks  |
| express-session     | Flask-Session (filesystem) |
| Static `public/`    | Flask static folder        |
| /api/login/logout/me| Same routes                |
| /app guard & send index.html | Same logic        |
| HTTPS redirect + HSTS | before_request + after_request |
| /health endpoint     | Same                      |
| /api/gpt (OpenAI proxy) | Same behavior          |

## 2. Thư viện sử dụng
- Flask: Framework HTTP nhẹ cho Python.
- Flask-Session: Lưu session server-side (filesystem). Tránh lộ thông tin trên cookie.
- requests: Gọi OpenAI API qua HTTP.

## 3. Cấu trúc file
```
python_server.py
requirements.txt
public/ (tĩnh)
```

## 4. Giải thích từng phần mã chính
### 4.1 Khởi tạo ứng dụng
```python
app = Flask(__name__, static_folder=str(PUBLIC_DIR), static_url_path='/public')
```
- `static_folder`: nơi Flask tự phục vụ file tĩnh (CSS, JS, images).
- Truy cập: `/public/...` tương đương Node trước đây.

### 4.2 Cấu hình session
```python
app.config['SESSION_TYPE'] = 'filesystem'
app.config['SESSION_COOKIE_SECURE'] = (NODE_ENV == 'production')
Session(app)
```
- Lưu session vào thư mục `.flask_session`.
- `SESSION_COOKIE_SECURE` buộc HTTPS khi production.

### 4.3 Middleware HTTPS redirect
```python
@app.before_request
def force_https():
    # Kiểm tra headers proxy và chuyển sang https nếu chưa
```
- Sử dụng `before_request` tương tự Express middleware.
- Cho phép tắt qua biến `DISABLE_HTTPS_REDIRECT=1`.

### 4.4 HSTS header
```python
@app.after_request
def apply_hsts(resp):
    if NODE_ENV == 'production':
        resp.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains; preload'
    return resp
```
- Giống logic Node: trình duyệt nhớ HTTPS.

### 4.5 Auth & session
- Login: POST `/api/login` nhận JSON `{username,password}`.
- Lưu `session['user']` nếu thành công.
- Logout: xoá session.
- Kiểm tra login: `/api/me`.

### 4.6 Bảo vệ route `/app`
```python
@app.route('/app')
def app_index():
    if 'user' not in session:
        return redirect('/login.html')
    return send_from_directory(str(APP_INDEX.parent), APP_INDEX.name)
```
- Tương đương `requireAuth` bên Node.

### 4.7 Static file & root redirect
- `/` → redirect `login.html`.
- File tĩnh khác served bởi Flask static.

### 4.8 Health check
```python
@app.get('/health')
```
- Trả về protocol và host (dựa header `X-Forwarded-*`).

### 4.9 GPT Proxy
```python
@app.post('/api/gpt')
```
- Kiểm tra session trước.
- Đọc `OPENAI_API_KEY` từ env.
- Gửi payload tới OpenAI Chat Completions.
- Trả về JSON `{answer}`.
- Timeout 20s, bắt lỗi và trả mã 504 nếu quá hạn.

### 4.10 Entry point
```python
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=PORT, debug=(NODE_ENV!='production'))
```
- Dùng server dev tích hợp. Production nên dùng `gunicorn` hoặc `waitress`.

## 5. Chạy thử (Windows PowerShell)
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:OPENAI_API_KEY = 'sk-your-key'
$env:NODE_ENV = 'development'
python python_server.py
```
Truy cập: http://localhost:3000

## 6. Chuyển sang production
- Dùng reverse proxy (Nginx/Apache) terminate TLS.
- Chạy: `gunicorn -w 4 -b 0.0.0.0:3000 python_server:app` (hoặc `waitress-serve --port=3000 python_server:app`).
- Set biến môi trường: `NODE_ENV=production`, `OPENAI_API_KEY=...`, `SESSION_SECRET=...`.

## 7. So sánh nhanh Express vs Flask
| Khía cạnh | Express | Flask |
|-----------|---------|-------|
| Routing   | `app.get('/path', cb)` | `@app.get('/path') def fn():` |
| Middleware| `app.use(fn)` | `@app.before_request / @app.after_request` |
| Session   | `express-session` | `Flask-Session` |
| Static    | `express.static()` | built-in static folder |
| JSON body | `body-parser.json()` | `request.get_json()` |
| Env vars  | `process.env.VAR` | `os.environ.get('VAR')` |

## 8. Bảo mật đề xuất bổ sung
- Rate limit (Flask-Limiter) cho `/api/gpt`.
- CSRF bảo vệ form login (Flask-WTF) nếu form POST không phải JSON.
- Regenerate session ID sau login để giảm session fixation.

## 9. Thêm rate limit (ví dụ tham khảo)
```python
# pip install flask-limiter
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
limiter = Limiter(get_remote_address, app=app, default_limits=["200 per hour"])
@limiter.limit("15/minute")
@app.post('/api/gpt')
```

## 10. Mở rộng tiếp
- Tách routes vào blueprint (`auth.py`, `gpt.py`).
- Thêm WebSocket (dùng `flask-sock` hoặc chuyển sang `FastAPI`).
- Logging có cấu trúc (json logger) phục vụ phân tích.

## 11. Checklist kiểm thử
- [ ] Login đúng trả `{ok:true}`
- [ ] Login sai trả 401
- [ ] `/api/me` phản ánh trạng thái
- [ ] `/app` redirect khi chưa login, trả index khi đã login
- [ ] `/api/gpt` trả lời khi có KEY, 401 khi chưa login
- [ ] HTTPS redirect hoạt động sau proxy (kiểm tra header)

## 12. Lỗi thường gặp
| Lỗi | Nguyên nhân | Cách khắc phục |
|-----|-------------|----------------|
| 500 Missing OPENAI_API_KEY | Chưa set biến môi trường | Export / set env trước khi chạy |
| 504 OpenAI timeout | Mạng chậm / model delay | Tăng timeout hoặc thử lại |
| Session không lưu | Quyền thư mục `.flask_session` | Tạo thư mục, chỉnh quyền hoặc dùng Redis |

## 13. Tắt HTTPS redirect khi dev
Set env:
```
DISABLE_HTTPS_REDIRECT=1
```

## 14. Tài liệu thêm
- Flask docs: https://flask.palletsprojects.com/
- Flask-Session: https://flask-session.readthedocs.io/
- OpenAI API: https://platform.openai.com/docs/api-reference

---
Nếu cần chuyển sang FastAPI hoặc thêm WebSocket real-time, có thể mở rộng từ nền tảng này.
