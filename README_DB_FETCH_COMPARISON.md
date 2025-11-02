# So sánh Phương Pháp Lấy Dữ Liệu Từ Database Lên Web GUI

Tài liệu này so sánh phương pháp CŨ (full reload qua `data.php`) với phương pháp MỚI (incremental streaming qua `stream.php` và tùy chọn WebSocket push) về kiến trúc, hiệu năng, tài nguyên trên cPanel, độ phức tạp và lộ trình triển khai.

## 1. Ba Phương Án
| Phương án | Mô tả | Trạng thái | Mục tiêu sử dụng |
|-----------|-------|-----------|------------------|
| Full Reload (`data.php`) | Mỗi chu kỳ gọi `data.php?file=...&limit=N` lấy lại N bản ghi (thường cũ + mới) | Cũ | Dễ code ban đầu, thử nghiệm nhỏ |
| Incremental Poll (`stream.php`) | Lần đầu lấy cửa sổ, sau đó chỉ lấy hàng mới `id > lastId` | Mới (đã có) | Realtime nhẹ, ít tải, dễ scale moderate |
| WebSocket Push | Server theo dõi insert và đẩy thẳng row mới qua WS | Tùy chọn (chưa triển khai) | Latency thấp hơn nữa, nhiều client đồng thời |

## 2. Luồng Dữ Liệu Tổng Quan
```
Local ingest (JSON/binary pack) --> ingest.php --> MySQL filtered_readings
                                              --> (Query đọc) --> GUI
```
GUI Options:
1. Full reload: Lặp lại SELECT ORDER BY ... LIMIT.
2. Incremental: SELECT WHERE id > lastId.
3. Push (tương lai): Trigger/binlog → WS broadcast.

## 3. Chi Tiết Kỹ Thuật Mỗi Phương Án
### 3.1 Full Reload (`data.php`)
**Query**: `SELECT id, created_at, AmpH_C1_nX, AmpV_C2_nX FROM filtered_readings WHERE file_id=? ORDER BY created_at ASC LIMIT ?`

**Đặc điểm**:
- Dễ implement (đã có sẵn file `data.php`).
- Mỗi lần nhận cả block dữ liệu → trùng lặp.
- Tốn băng thông và CPU PHP + MySQL khi bảng lớn.

**Nhược điểm chính**:
- Chi phí tăng tuyến tính với lịch sử.
- Cập nhật chậm nếu cần nhiều lần/giây.
- Đẩy nặng trình duyệt khi cứ phải diff nhiều mẫu.

### 3.2 Incremental Poll (`stream.php`)
**Query ban đầu**: `ORDER BY id ASC/DESC LIMIT L` (mode oldest hoặc latest).

**Query delta**: `SELECT ... WHERE file_id=? AND id > :after_id ORDER BY id ASC LIMIT :limit`

**Yêu cầu Index**: `(file_id, id)` để truy vấn nhanh phần cuối.

**Ưu điểm**:
- Tải theo tốc độ ingest (chỉ row mới) → O(k) thay vì O(n).
- Ít băng thông, đáp ứng realtime (1–2 s).
- Không cần thay đổi ingest hiện tại.

**Nhược điểm**:
- Cần giữ biến `lastId` phía client.
- Phải xử lý trường hợp “không có dữ liệu mới” (backoff interval).
- Không tự đẩy; vẫn là pull định kỳ.

### 3.3 WebSocket Push (Tương lai)
**Ý tưởng**: Khi insert, một tiến trình (Node/PHP daemon) lấy row mới và làm `ws.send(JSON)` cho các client đã subscribe.

**Ưu điểm**:
- Latency thấp hơn poll (sub-second).
- Băng thông tối ưu hơn khi nhiều client (một broadcast thay vì nhiều HTTP request).

**Nhược điểm**:
- Tăng độ phức tạp (phải có kênh theo dõi insert: polling tail id, binlog, hoặc trigger + queue).
- Cần daemon lâu dài (không phải request/response PHP truyền thống).
- Quản lý reconnect + auth WS.

## 4. So Sánh Hiệu Năng (Định Tính)
| Tiêu chí | Full Reload | Incremental Poll | WS Push |
|----------|-------------|------------------|--------|
| Băng thông | Cao dần theo lịch sử | Thấp theo delta | Rất thấp (chỉ delta) |
| CPU MySQL | Nhiều (quét nhiều row) | Ít (index + phần cuối) | Ít (chỉ truy vấn mới) |
| Độ trễ cảm nhận | Vừa (phụ thuộc chu kỳ) | Thấp (1–2 s) | Rất thấp (<1 s) |
| Độ phức tạp code | Thấp | Thấp-Trung bình | Cao |
| Khả năng scale client | Vừa (sớm nghẽn) | Tốt hơn | Tốt nhất |
| Stateful frontend | Không (chỉ render lại) | Có (lastId) | Có (socket state) |

## 5. Ưu / Nhược Khi Chạy Trên cPanel
| Yếu tố | Full Reload | Incremental | WebSocket Push |
|--------|-------------|-------------|---------------|
| Shared hosting hạn chế CPU | Dễ quá tải nếu dataset lớn | Phù hợp hơn | Cần daemon Node → phải cấu hình kỹ |
| Memory PHP | Mỗi request giải phóng nhưng payload lớn | Nhỏ, chỉ delta | Không dùng PHP nếu đẩy qua Node |
| Giới hạn concurrent requests | Nhiều poll “full” gây nghẽn | Delta nhẹ nên ổn | Ít HTTP, chủ yếu giữ WS sockets |
| Dễ triển khai | Đã có | Chỉ thêm `stream.php` | Cần service chạy 24/7 |
| Logging / Debug | Đơn giản | Đơn giản | Phức tạp hơn (kết nối trạng thái) |

## 6. Lộ Trình Khuyến Nghị
1. (Đang) Dùng Incremental Poll (`stream.php`).
2. Thêm Backoff: Khi 3 lần liên tiếp không có dữ liệu mới → tăng interval.
3. Trim buffer client: Giữ tối đa 10k mẫu.
4. Nếu ingest tần suất cao (>20 row/s) hoặc có >50 client → cân nhắc WS push.
5. Bổ sung auth Bearer cho `stream.php` (tương tự `ingest.php`).

## 7. Đo Lường Hiệu Năng Đề Xuất
Frontend:
```js
performance.mark('pollStart');
fetch(url).then(r=>r.json()).then(j=>{
  performance.mark('pollEnd');
  performance.measure('pollLatency','pollStart','pollEnd');
  console.log(performance.getEntriesByName('pollLatency').pop());
});
```

Server (Node WebSocket):
```js
setInterval(()=>{
  const {rss,heapUsed} = process.memoryUsage();
  console.log('[MEM]', (rss/1048576).toFixed(1)+'MB', 'heap', (heapUsed/1048576).toFixed(1)+'MB');
}, 60000);
```

MySQL (nếu có CLI access):
```sql
SHOW GLOBAL STATUS LIKE 'Queries';
SHOW PROCESSLIST; -- kiểm tra xem có nhiều SELECT full reload không
```

## 8. Mã Tối Giản Incremental Poll (Nhắc lại)
```js
let lastId=null, currentFile='A_10_00.csv';
function poll(){
  const base='/public/api/stream.php?file='+encodeURIComponent(currentFile);
  const url= lastId? base+'&after_id='+lastId : base+'&mode=latest';
  fetch(url).then(r=>r.json()).then(j=>{
    if(!j.ok) return setTimeout(poll,3000);
    if(j.data.length){ j.data.forEach(row=>appendSample(row)); lastId=j.meta.max_id; }
    setTimeout(poll, j.meta.poll_hint_ms||1500);
  }).catch(()=> setTimeout(poll,3000));
}
poll();
```

## 9. Khi Nào Nên Chuyển Sang WS Push
- Ingest trung bình > 30 row/s hoặc spike cao.
- Nhiều client (≥ 100) gây nhiều request poll.
- Cần hiển thị ngay lập tức (latency < 500 ms) cho phân tích thời gian thực.

## 10. Rủi Ro & Giảm Thiểu
| Rủi ro | Phát sinh ở | Giảm thiểu |
|--------|-------------|------------|
| Full reload chậm | Dataset lớn | Chuyển incremental + index |
| Poll rỗng liên tục | Thời gian nhàn rỗi | Backoff tăng interval |
| Bộ nhớ client phình | Lưu quá nhiều samples | Trim / decimate |
| WS push mất kết nối | Mạng không ổn định | Reconnect exponential + buffer tạm |
| Auth bypass stream | Endpoint mở | Thêm Bearer check giống ingest |

## 11. Mẹo Tối Ưu Thêm
- Dùng `gzip` (thường auto) nếu batch > vài KB.
- Truyền token auth qua header (fetch) tránh làm lộ trong URL log.
- Tách logic vẽ chart & logic poll (module design) để dễ giới hạn tần suất redraw.
- Pre-aggregate: nếu cần RMS/KTS thời gian thực, có thể gửi kèm trong mỗi delta thay vì truy vấn riêng.

## 12. Kết Luận Ngắn
Incremental `stream.php` là bước nhảy lớn về hiệu năng so với full reload: tuyến tính theo “delta” thay vì “lịch sử”, thân thiện với cPanel (CPU/RAM thấp), dễ mở rộng và đủ nhanh cho hầu hết nhu cầu điều khiển/giám sát hiện tại. WebSocket push chỉ cần khi số client và tốc độ ingest vượt mức trung bình.

---
Nếu bạn muốn mình tạo khung WebSocket push (prototype) hoặc thêm auth cho `stream.php`, hãy yêu cầu tiếp theo và mình sẽ thực hiện.
