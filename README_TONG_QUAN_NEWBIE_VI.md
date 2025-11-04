# 📘 TỔNG QUAN DỰ ÁN DIGITAL TWIN – BẢN DÀNH CHO NEWBIE

> Mục tiêu: Bạn chỉ cần đọc file này là hiểu bức tranh toàn bộ hệ thống, biết nên chạy gì trước, và tự test end‑to‑end (UI ↔ Điều khiển ↔ Dữ liệu ↔ AI Assistant ↔ GPT).

---
## 1. Dự án này làm gì?
Hệ thống mô phỏng (digital twin) cho phép bạn:
- Quan sát dữ liệu cảm biến / trạng thái từ thiết bị (hoặc dữ liệu giả lập) trên giao diện web.
- Điều khiển (ví dụ: START / STOP / SPEED) từ xa qua nhiều kênh (MQTT, WebSocket/HTTPS, TCP bridge…).
- Phân tích luồng dữ liệu realtime hiệu quả (tránh tải lại toàn bộ).
- Có trợ lý AI trong trình duyệt hỗ trợ giải thích – có thể gọi GPT nếu cần thông minh hơn.

---
## 2. Sơ đồ khái niệm (Tổng quan)
```
[ Thiết bị / Giả lập ]
        | (Serial / TCP / Script)
        v
  +------------------+
  |  Bridge (C++ /   |<-- Python fallback / MATLAB TCP client
  |  Python / MATLAB)|
  +------------------+
        | (MQTT publish trạng thái / nhận lệnh) hoặc (TCP JSON dòng)
        v
            (A) MQTT Broker
                |                 (B) HTTPS / WebSocket Server
                |                          |
                v                          v
                    [ Giao diện Web (public/app/index.html + js) ]
                                 |
                      +-------------------------+
                      |  AI Assistant (FAQ +   |
                      |  semantic + QA + GPT)  |
                      +-------------------------+
                                 |
                        /api/gpt  (Node / Flask / FastAPI proxy)
```

Có thể chọn 1 trong 2 chiến lược transport chính:
- Hybrid: MQTT (điều khiển) + HTTPS (ingest dữ liệu / API).
- Full HTTPS/WebSocket: bỏ MQTT, dùng WS để vừa push trạng thái vừa nhận lệnh.

---
## 3. Thành phần thư mục chính
| Thư mục / File | Vai trò ngắn gọn |
|----------------|------------------|
| `public/app/index.html` | Giao diện chính (UI dashboard + assistant) |
| `public/app/js/` | Thư viện Three.js và script liên quan (3D nếu dùng) |
| `assistant.js` (được tách) | Logic trợ lý AI (FAQ → semantic → QA → synthesis → GPT) |
| `public/api/*.php` | Endpoint PHP cũ: `data.php` (full), `stream.php` (incremental) |
| `server.js` | Backend Node/Express (auth, static, `/api/gpt`) |
| `python_server.py` | Backend Flask thay thế tương đương |
| `fastapi_server.py` | Backend FastAPI (hiệu năng hơn, async GPT) |
| `faq.json` | CSDL FAQ cho assistant |
| `C++ bridge` (main.cpp, CMake…) | Nhận MQTT → forward TCP JSON tới MATLAB |
| MATLAB scripts | Nhận TCP JSON, xử lý lệnh, gửi phản hồi |
| `requirements.txt` | Dependencies Python (Flask + FastAPI + httpx) |
| Các README_* khác | Giải thích chuyên sâu theo mảng (realtime data, assistant, bridge…) |

---
## 4. Dữ liệu realtime: 3 cấp độ
| Cách | Mô tả | Ưu | Nhược | Khi nào dùng |
|------|-------|----|-------|-------------|
| Full reload (`data.php`) | Mỗi lần tải lại toàn bộ bảng | Dễ hiểu | Tốn băng thông, chậm | Chỉ debug nhỏ |
| Incremental (`stream.php?after_id=`) | Chỉ lấy bản ghi mới hơn ID cuối | Nhanh, nhẹ | Cần nhớ ID cuối | Mặc định hiện tại |
| WebSocket push | Server chủ động đẩy bản ghi mới | Thời gian thực tốt nhất | Cần WS infra | Khi tối ưu cuối |

---
## 5. Điều khiển & MQTT
### 5.1 Topic chuẩn (ví dụ)
```
PREFIX/control/command   (UI → thiết bị: START, STOP, SPEED:1200)
PREFIX/status/state      (Bridge / thiết bị → UI: RUNNING / STOPPED)
PREFIX/status/speed      (Tốc độ hiện tại)
PREFIX/status/metrics    (Khác: nhiệt độ, dòng… nếu có)
```
### 5.2 Token & bảo mật
- Dùng `token` (chuỗi bí mật) kèm trong payload hoặc topic (tuỳ kiến trúc) để từ chối lệnh không hợp lệ.
- Broker nên bật user/password, ACL giới hạn prefix.
- Khi public test: dùng broker tạm + đổi token thường xuyên.

---
## 6. Bridge C++ / Python / MATLAB
| Thành phần | Chức năng |
|------------|-----------|
| C++ Bridge (Paho C) | Subscribe lệnh MQTT → gửi qua TCP tới MATLAB; nhận phản hồi có thể publish lại |
| Python fallback | Thử nghiệm khi chưa build được C++ (logic tương tự) |
| MATLAB TCP client | Kết nối tới bridge, đọc JSON từng dòng, thực thi (gọi motor / mô phỏng) |
| MATLAB no-serial mode | Cho phép demo không cần phần cứng; chỉ log & giả lập state |

### 6.1 Định dạng JSON dòng (ví dụ)
```
{"type":"command","cmd":"SPEED","value":1200}\n
{"type":"command","cmd":"STOP"}\n
```

---
## 7. Transport: Hybrid vs Full HTTPS/WebSocket
| Chiến lược | Mô tả | Khi phù hợp |
|------------|-------|-------------|
| Hybrid | Ingest dữ liệu qua HTTPS/ PHP / streaming; điều khiển = MQTT | Đã có broker, muốn tận dụng ổn định MQTT |
| Full HTTPS/WS | Một server WebSocket xử lý cả push dữ liệu + lệnh | Môi trường bị hạn chế port/broker hoặc muốn đơn giản triển khai |

---
## 8. Backend lựa chọn
| Lựa chọn | Ưu | Khi chọn |
|----------|----|---------|
| `server.js` (Express) | Dễ sửa, quen thuộc | Môi trường Node sẵn trên cPanel |
| `python_server.py` (Flask) | Đơn giản, sync | App nhẹ, ít tải |
| `fastapi_server.py` (FastAPI) | Hiệu năng, async GPT | Mong muốn scale hoặc nhiều request GPT |

Tất cả đều có `/api/gpt`, auth cơ bản (session), redirect HTTPS, HSTS.

---
## 9. AI Assistant (Chạy trong trình duyệt)
Pipeline theo thứ tự giảm chi phí / tăng thông minh:
1. Exact FAQ match (so khớp normalize tiếng Việt – bỏ dấu).
2. Keyword + synonyms expansion (speed/rpm, websocket/ws...).
3. Suggestion (gợi ý câu gần nhất khi gõ).
4. Semantic embeddings (MiniLM đa ngôn ngữ) → ranking top N.
5. QA model (DistilBERT) extract câu trả lời từ đoạn FAQ ghép.
6. Smart Synthesis: Ghép 2–3 hit thành đoạn tự nhiên.
7. Deep Explanation Mode: Hiển thị vì sao chọn (scores, top hits).
8. GPT Fallback: Gọi `/api/gpt` nếu người dùng bấm nút GPT hoặc local pipeline score thấp.

### 9.1 Lưu trữ
- Embeddings FAQ cache trong IndexedDB → lần sau mở nhanh hơn.
- Lịch sử hội thoại hiển thị trong panel (có nút xóa & copy).

### 9.2 Khi nào dùng GPT?
- Câu hỏi mở rộng vượt phạm vi FAQ.
- Người dùng muốn văn phong tự nhiên hoặc giải thích thêm.

---
## 10. GPT Endpoint (`/api/gpt`)
- Chạy server-side để giữ kín `OPENAI_API_KEY`.
- Body gửi: `{ "question": "..." }`.
- Trả về: `{ answer: "..." }`.
- Timeout nội bộ (nên ~15s). (Rate limit dự kiến: 15 req/phút/IP – TODO nếu chưa thêm).

### 10.1 Cấu hình API Key
Tạo file `.env` (local):
```
OPENAI_API_KEY=sk-xxx
```
Triển khai cPanel: đặt vào biến môi trường thay vì .env.

---
## 11. Quy trình khởi động nhanh (Quick Start)
### Bước 1: Chọn backend
- Nhanh nhất: Node `server.js` nếu môi trường đã có.
- Muốn async + Python: dùng `fastapi_server.py`.

### Bước 2: Chuẩn bị GPT (tùy chọn)
- Có key: export hoặc `.env`.
- Không có key: Assistant vẫn chạy local (FAQ + semantic + QA).

### Bước 3: Dữ liệu thử
- Dùng PHP `stream.php` nếu đã có DB.
- Hoặc tạo script giả lập publish MQTT / hoặc push WS.

### Bước 4: Điều khiển
- UI nhập SPEED / START / STOP.
- Kiểm tra broker nhận lệnh (sub log) hay TCP bridge log.

### Bước 5: Assistant
- Gõ câu hỏi sẵn có trong FAQ để thấy trả lời nhanh.
- Thử câu mở rộng → bấm GPT.

---
## 12. Demo không cần phần cứng
1. Bật chế độ `CONFIG.ENABLE_SERIAL=false` trong MATLAB script.
2. Bridge chỉ log lệnh, vẫn cập nhật state giả lập.
3. UI hiển thị có phản hồi → xác nhận pipeline hoạt động.

---
## 13. Bảo mật cơ bản (Hiện tại & Kế hoạch)
| Mục | Đã có | Kế hoạch / Gợi ý |
|-----|-------|------------------|
| Session auth | Có (cookie) | Thêm timeout ngắn, regenerate ID |
| HTTPS redirect | Có | Thêm tự động HSTS (đã có) |
| HSTS | Node/Python đã set | Kiểm tra header trên prod |
| Rate limit GPT | Chưa / Đang TODO | Dùng token bucket theo IP |
| MQTT | Token + có thể user/pass | ACL prefix + TLS nếu public |
| WebSocket | Cơ bản | Thêm check token, close idle |
| Input validation | Cơ bản | Sanitize lệnh tốc độ (max/min) |

---
## 14. Troubleshooting nhanh
| Vấn đề | Nguyên nhân phổ biến | Cách xử lý |
|--------|----------------------|-----------|
| UI không nhận dữ liệu mới | Gọi nhầm `data.php` thay vì `stream.php` | Chắc chắn hàm fetch incremental chạy định kỳ và giữ `lastId` |
| Lệnh MQTT không tác dụng | Sai prefix hoặc thiếu token | Kiểm tra biến cấu hình UI và broker log |
| GPT báo lỗi | Thiếu API key hoặc timeout | Kiểm tra biến môi trường, thử lại câu ngắn hơn |
| Assistant trả lời lệch | FAQ chưa đủ bao phủ | Mở rộng FAQ hoặc dùng GPT mode |
| Bridge không kết nối | Port TCP bị chặn | Đổi port, kiểm tra firewall |
| MATLAB script treo | Chưa flush newline JSON | Đảm bảo mỗi JSON kết thúc `\n` |

---
## 15. Một số lệnh / thao tác (tham khảo)
> Tùy môi trường, có thể khác; phần này minh họa.

Chạy FastAPI (dev):
```
uvicorn fastapi_server:app --reload --port 8000
```
Chạy Flask (đơn giản):
```
python python_server.py
```
Build C++ bridge (ví dụ – đã có script PowerShell):
```
./build_bridge.ps1
```

---
## 16. Roadmap ngắn gọn
- ✅ Hoàn thiện assistant đa tầng & GPT fallback.
- 🔄 Thêm rate limiting /api/gpt & WS.
- 🔄 README WS bridge chi tiết (bảo mật, token handshake).
- 🔄 Hợp nhất doc GPT vào một README chung (nếu cần tách).
- 🧪 Thêm test tự động cho semantic ranking (so sánh score).
- 🚀 WebSocket push full (thay polling incremental) khi cần độ trễ thấp.

---
## 17. FAQ siêu nhanh (Meta)
| Hỏi | Đáp ngắn |
|-----|----------|
| Không có GPT key có sao không? | Không – vẫn chạy local pipeline. |
| Dùng server nào? | Ưu tiên FastAPI nếu muốn async / hiệu năng. |
| Nên bắt đầu kiểm tra gì? | 1) UI load 2) stream incremental 3) gửi lệnh 4) assistant FAQ 5) GPT (nếu có key). |
| Mở rộng FAQ thế nào? | Thêm mục vào `faq.json`, reload; embeddings sẽ cache lại (lần đầu tính). |
| Tại sao tốc độ chưa update? | Bridge chưa publish status/speed hoặc đang ở demo no-serial. |

---
## 18. Ghi chú mở rộng
- Toàn bộ thiết kế ưu tiên từng lớp tăng dần độ phức tạp → bạn có thể tạm dừng ở bất kỳ mức nào vẫn dùng được.
- Tách GPT proxy server-side tránh lộ khóa và cho phép thêm rate limit.
- Kiến trúc giữ backward compatibility: vẫn có PHP endpoints cũ song song chiến lược mới.

---
## 19. Bạn nên làm gì ngay (Checklist 10 phút)
1. Chọn backend (FastAPI nếu cài Python chuẩn). 
2. Mở `public/app/index.html` trong trình duyệt qua server (đừng mở file://). 
3. Kiểm tra console không lỗi. 
4. Xem panel Assistant – hỏi câu trùng FAQ. 
5. Thử hỏi câu lạ → bấm GPT (nếu có key). 
6. Chạy bridge (C++ hoặc Python fallback) ở chế độ demo. 
7. Gửi START / SPEED trên UI và kiểm tra log. 
8. Xác nhận `stream.php` trả về bản ghi mới (Network tab). 

Hoàn thành = bạn đã nắm cơ bản. 🎉

---
## 20. Cần thêm? / Góp ý
Nếu muốn thêm README tốc độ nâng cao, bảo mật chuyên sâu, hoặc tích hợp RAG nội bộ: bổ sung roadmap và triển khai dần.

---
### Kết
Bạn đã có **toàn cảnh**: dữ liệu → transport → điều khiển → bridge → AI trợ lý → GPT. Bắt đầu từ đơn giản, bật tính năng nâng cao khi cần. Chúc bạn thành công!
