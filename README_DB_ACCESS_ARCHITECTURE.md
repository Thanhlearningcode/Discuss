# KIẾN TRÚC TRUY XUẤT DỮ LIỆU TỪ DB LÊN WEB

Tài liệu này trình bày 3 giai đoạn đã/đang dùng để hiển thị dữ liệu đo (samples) từ MySQL lên giao diện Web GUI:
- Giai đoạn ban đầu: Full Reload (`data.php`)
- Giai đoạn hiện tại: Incremental (Cursor) Poll (`stream.php`)
- Giai đoạn nâng cấp tương lai: WebSocket Push (daemon)

## 1. Mục tiêu chung
Cập nhật biểu đồ và bảng số liệu gần thời gian thực (near realtime) với chi phí tài nguyên thấp và đường nâng cấp rõ ràng khi tải tăng.

## 2. Kiến trúc mức cao
```
Sensor/Ingest Source --> ingest.php --> MySQL(filtered_readings)
                                       ^
                                       |
                               (Truy xuất API)
                                |         |
                          data.php     stream.php
                              |             ^
                              v             |
                           Web GUI   (poll delta)\
                                            \----> Tương lai: WS daemon push
```

## 3. Giai đoạn Ban Đầu: Full Reload (data.php)
**Cách hoạt động:** Frontend cứ mỗi X giây gọi:
```
GET /public/api/data.php?file=<tên file>&limit=N
```
PHP thực hiện SELECT các dòng theo `file_id` và trả danh sách (cả cũ + mới). Frontend thường reset biểu đồ rồi vẽ lại toàn bộ.

**Ưu điểm:**
- Code cực kỳ đơn giản.
- Phù hợp prototype/số mẫu ít.

**Nhược:**
- Lặp lại dữ liệu cũ → băng thông & CPU tăng.
- Không scale khi lịch sử lớn.
- Render lại nhiều điểm gây lag UI.

## 4. Giai đoạn Hiện Tại: Incremental Poll (stream.php)
**Ý tưởng:** Chỉ lấy PHẦN MỚI sau lần cuối.

**Lần đầu:**
```
GET /public/api/stream.php?file=<tên file>&mode=latest
```
Trả về "cửa sổ" mới nhất + `meta.max_id`. Frontend lưu `lastStreamId = max_id`.

**Các lần sau:**
```
GET /public/api/stream.php?file=<tên file>&after_id=lastStreamId
```
Trả về chỉ những dòng có `id > lastStreamId`. Nếu không có, `data=[]`.

**Trên frontend:**
- Giữ biến `lastStreamId`.
- Append điểm mới vào chart & bảng, không reset toàn bộ.
- Backoff (tăng interval) nếu nhiều lần rỗng.
- Trim số điểm giữ cho RAM ổn định.

**Ưu điểm:**
- Chi phí mỗi poll ~ O(k) với k = số row mới (delta), không phụ thuộc tổng lịch sử n.
- Giảm mạnh băng thông & CPU so với full reload.
- Dễ mở rộng bảo mật (thêm Bearer token giống ingest).

**Nhược:**
- Cần state nhỏ ở client (`lastStreamId`).
- Chưa chủ động push (vẫn là pull).

## 5. Giai đoạn Tương Lai: WebSocket Push
**Ý tưởng:** Một daemon (Node.js) giám sát DB (poll nhẹ id cuối hoặc dùng binlog/CDC). Khi có row mới:
```
wsServer.broadcast({ type:'reading', id, ts, AmpH, AmpV })
```
Frontend nhận qua socket, cập nhật ngay lập tức (sub-second).

**Ưu điểm:**
- Latency thấp hơn poll.
- Tối ưu khi nhiều client đồng thời (một broadcast thay vì nhiều HTTP request). 

**Nhược:**
- Tăng độ phức tạp: giữ kết nối, reconnect, auth token, chống spam.
- Cần tiến trình nền chạy liên tục (không chỉ PHP request).

## 6. So sánh Tổng quan
| Tiêu chí | Full Reload | Incremental Poll | WS Push |
|---------|-------------|------------------|--------|
| Latency cảm nhận | Vừa (chu kỳ) | Thấp (1–2 s) | Rất thấp (<500 ms) |
| Băng thông | Cao, lặp | Thấp, theo delta | Rất thấp (delta) |
| Phức tạp code | Thấp | Thấp–Trung bình | Cao |
| Scale client | Hạn chế | Tốt hơn | Tốt nhất |
| Yêu cầu hạ tầng | PHP + MySQL | PHP + MySQL | + Daemon WS/CDC |

## 7. Dữ liệu Lịch sử (RMS/KTS)
- Incremental KHÔNG làm mất dữ liệu cũ: DB vẫn lưu toàn bộ. Lần initial `mode=latest` lấy một window đủ lớn để tính RMS/Kurtosis nền.
- Khi cần sâu hơn nữa: có thể mở rộng `stream.php` với `before_id` cho backfill hoặc tái dùng `data.php` như endpoint lịch sử paging.
- RMS/KTS realtime cửa sổ trượt: Client duy trì buffer + công thức cập nhật. Khi tải tăng, chuyển sang precompute server (bảng aggregate).

## 8. Lộ trình nâng cấp
1. (Đã) Loại bỏ full reload → dùng incremental.
2. Thêm Bearer token & rate limit cho `stream.php`.
3. Thêm `before_id` (paging ngược) nếu người dùng cần xem xa hơn.
4. WS daemon prototype (poll tail id mỗi 300 ms → broadcast delta).
5. CDC / Kafka nếu ingest & số client vượt ngưỡng.
6. Bảng aggregate RMS/KTS server-side để giảm CPU client.

## 9. Quyết định chuyển phase
| Metric | Ngưỡng sang WS Push |
|--------|---------------------|
| Ingest rate (rows/s) | > 30 liên tục |
| Concurrent clients | > 50 thường xuyên |
| Latency yêu cầu | < 0.5 s |
| Poll empty ratio | > 70% (phí request) |
| CPU MySQL do poll | > 15% tổng CPU |

## 10. Bảo mật cơ bản (khuyến nghị)
- Thêm header Authorization: `Bearer <token>` vào `stream.php` (giống ingest). 
- Giới hạn `limit` chặt (đã clamp 1..2000) tránh tải quá lớn.
- Rate limit đơn giản (ví dụ tối đa 1 request / 500 ms / IP). 
- Log truy cập bất thường (nhiều after_id nhảy lùi hoặc brute force file).

## 11. Đo lường nhanh
Frontend:
```js
console.time('poll');
fetch('/public/api/stream.php?file=A_10_00.csv&after_id='+lastStreamId)
  .then(r=>r.json())
  .then(()=> console.timeEnd('poll'));
```
Server (Node WS daemon prototype):
```js
setInterval(()=>{
  const m = process.memoryUsage();
  console.log('[mem]', (m.rss/1048576).toFixed(1),'MB');
}, 60000);
```
MySQL:
```sql
SHOW GLOBAL STATUS LIKE 'Queries';
SHOW PROCESSLIST; -- xem có nhiều SELECT full reload không
```

## 12. Tóm tắt cho quản lý
Hiện sử dụng incremental cursor polling (thay full reload) → giảm chi phí rõ rệt, đủ realtime ~1–2 s. Sẵn sàng nâng cấp lên WebSocket push hoặc CDC khi các ngưỡng tải đạt mức đề ra, không cần viết lại phần hiển thị.

## 13. TL;DR cho nhân sự mới
- Trước kia: Cứ mỗi vài giây tải lại cả block dữ liệu → dư thừa.
- Nay: Lần đầu lấy window mới nhất, sau đó chỉ hỏi “có gì mới sau id X?” → nhẹ & nhanh.
- Tương lai: Server sẽ tự đẩy các dòng mới qua WebSocket.

---
Muốn bổ sung phần backfill (`before_id`) hoặc prototype WS daemon, tạo issue tiếp để thực hiện.
