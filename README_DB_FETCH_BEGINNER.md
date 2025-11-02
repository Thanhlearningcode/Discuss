# HƯỚNG DẪN DỄ HIỂU: Lấy Dữ Liệu Từ DB Lên Web

Tài liệu này dành cho người MỚI (newbie) chưa biết gì. Mình sẽ giải thích cực kỳ đơn giản hai cách:
- Cách CŨ: Full reload bằng `data.php`
- Cách MỚI: Incremental (tăng dần) bằng `stream.php`

## 1. Bạn đang cố làm gì?
Bạn có dữ liệu các mẫu đo (ví dụ cảm biến) được ghi vào database MySQL. Bạn muốn hiển thị nó trên trang web thành biểu đồ cập nhật theo thời gian.

## 2. Bảng dữ liệu nhìn như thế nào?
Mỗi dòng (row) là một mẫu mới:
```
id | created_at        | file_id | AmpH_C1_nX | AmpV_C2_nX | ...
1  | 2025-11-02 10:00  |   7     |   ...      |   ...      | ...
2  | 2025-11-02 10:00  |   7     |   ...      |   ...      | ...
3  | 2025-11-02 10:01  |   7     |   ...      |   ...      | ...
...
```
`id` tăng dần khi có bản ghi mới.

## 3. Cách CŨ: Full Reload (`data.php`)
Mỗi lần cập nhật, web gọi:
```
GET /public/api/data.php?file=A_10_00.csv&limit=1000
```
Server trả về tối đa 1000 dòng đầu (hoặc cuối) tùy query.

Sau đó web:
1. Xóa dữ liệu đang vẽ.
2. Duyệt qua 1000 dòng vừa nhận.
3. Vẽ lại biểu đồ từ đầu.
4. Chờ vài giây rồi làm lại.

### Vấn đề của cách cũ
- Luôn tải lại cả đống dòng cũ đã thấy → lãng phí.
- Mỗi lần phải vẽ lại toàn bộ biểu đồ → chậm dần.
- Dữ liệu càng nhiều càng chậm (n tỉ lệ với tổng lịch sử).

### Code minh họa cách cũ
```js
function reloadAll(){
  fetch('/public/api/data.php?file=A_10_00.csv&limit=1000')
    .then(r=>r.json())
    .then(json=>{
      if(!json.ok) return;
      const rows = json.data; // chứa cả dữ liệu cũ
      chartReset();           // Xóa biểu đồ
      rows.forEach(r=>chartAddPoint(r)); // Vẽ lại toàn bộ
    })
    .finally(()=> setTimeout(reloadAll, 2000)); // Lặp lại sau 2s
}
reloadAll();
```

## 4. Cách MỚI: Incremental (`stream.php`)
Ý tưởng: Chỉ lấy NHỮNG DÒNG MỚI kể từ lần trước.

### Lần đầu (chưa biết gì)
Gọi:
```
GET /public/api/stream.php?file=A_10_00.csv&mode=latest
```
Server trả 1 "cửa sổ" dữ liệu mới nhất + thêm `meta.max_id` (id lớn nhất).
Bạn lưu `lastId = meta.max_id`.

### Các lần sau
Gọi:
```
GET /public/api/stream.php?file=A_10_00.csv&after_id=lastId
```
Server chỉ trả các dòng có `id > lastId` (delta). Nếu không có gì mới thì mảng `data` rỗng.
Bạn cập nhật biểu đồ bằng cách THÊM các điểm mới, không xóa cái cũ.

### Lợi ích
- Ít dữ liệu hơn rất nhiều (chỉ delta). 
- Biểu đồ không phải vẽ lại toàn bộ.
- Nhanh hơn, nhẹ hơn cho server và trình duyệt.

### Code minh họa cách mới
```js
let lastId = null; // chưa biết id cuối
function poll(){
  let url = '/public/api/stream.php?file=A_10_00.csv';
  if(lastId === null){
    url += '&mode=latest'; // lấy cửa sổ đầu tiên
  } else {
    url += '&after_id=' + lastId; // lấy delta mới
  }
  fetch(url)
    .then(r=>r.json())
    .then(json=>{
      if(!json.ok) return;
      const rows = json.data; // chỉ dòng mới
      rows.forEach(r=>chartAddPoint(r)); // Thêm vào biểu đồ
      // Cập nhật lastId nếu server có meta.max_id
      if(json.meta && json.meta.max_id){
        lastId = json.meta.max_id;
      }
    })
    .finally(()=> setTimeout(poll, 1500)); // lặp lại
}
poll();
```

## 5. So sánh nhanh
| Tiêu chí | Full Reload (cũ) | Incremental (mới) |
|----------|------------------|-------------------|
| Dữ liệu tải mỗi lần | Toàn bộ (limit) | Chỉ dòng mới |
| Tốc độ chậm dần | Có | Ít hơn nhiều |
| Tốn băng thông | Cao | Thấp |
| Vẽ lại biểu đồ | Luôn từ đầu | Chỉ append |
| Độ phức tạp code | Rất đơn giản | Hơi thêm biến `lastId` |
| Phù hợp lâu dài | Không | Có |

## 6. Khi nào dùng cái nào?
- Prototype rất nhỏ: Full reload cho nhanh.
- Muốn chạy thực tế, dữ liệu tăng liên tục: Dùng incremental.

## 7. Những lỗi newbie hay gặp với incremental
| Lỗi | Nguyên nhân | Cách xử lý |
|-----|-------------|-----------|
| Không thấy dữ liệu mới | Bạn chưa cập nhật `lastId` | Luôn set `lastId = meta.max_id` nếu có. |
| Biểu đồ trùng điểm | Append lại dữ liệu cũ | Chỉ append `rows` khi `after_id` đã dùng đúng. |
| Poll nhanh quá (spam) | Dùng interval 200ms | Dùng 1000–2000ms hoặc adapt nếu ít dữ liệu mới. |
| Mất dữ liệu khi restart trang | Mất biến `lastId` | Chấp nhận (reload initial). Để nâng cao có thể lưu localStorage. |

## 8. Mẹo cải thiện
- Nếu nhiều lần liên tiếp không có dữ liệu, tăng khoảng thời gian poll (backoff).
- Giới hạn số điểm biểu đồ (ví dụ giữ 10.000 điểm, cũ hơn thì bỏ). 
- Gộp nhiều điểm mới (nếu đến quá nhanh) rồi append theo batch mỗi 500ms để mượt hơn.

## 9. Câu hỏi thường gặp
**Hỏi:** Sao không lấy luôn tất cả?
**Đáp:** Càng nhiều dữ liệu cũ, mỗi lần tải lại càng nặng, khiến trang chậm.

**Hỏi:** Nếu có dữ liệu mới đến giữa lúc poll?  
**Đáp:** Lần poll sau sẽ lấy nó vì `id` lớn hơn `lastId`.

**Hỏi:** Có mất dữ liệu không?  
**Đáp:** Không, vì mỗi row đều có id tăng dần; nếu poll chậm hơn ingest, lần sau vẫn lấy được.

## 10. Kết luận
- Cách cũ (full reload) dễ hiểu nhưng không hiệu quả về lâu dài.
- Cách mới (incremental) chỉ cần thêm một biến `lastId` là tiết kiệm lớn về tốc độ và tài nguyên.

Bạn chỉ cần nhớ: "Hỏi lần đầu: Cho tôi phần mới nhất + id cuối; Hỏi về sau: Cho tôi mọi thứ sau id đó".

## 11. ĐÃ TÍCH HỢP TRONG SOURCE Ở ĐÂU?
Trong file `public/app/index.html` đã thêm khối mã có comment:
```
/* NEWBIE FRIENDLY INCREMENTAL STREAMING (DB) */
```
Phần này tạo các hàm:
- `startIncrementalStream(file)` khởi động poll delta.
- `stopIncrementalStream()` dừng.
- `appendStreamRows(rows)` thêm dữ liệu mới vào chart mà không reset.

Biến `ENABLE_INCREMENTAL = true` bật/tắt chế độ.

Muốn chạy:
```js
startIncrementalStream('A_10_00.csv');
```
Nếu bạn đổi file khác, gọi lại `startIncrementalStream('<file mới>')`.

## 12. Bật/Tắt Nhanh Cho Thử Nghiệm
Đặt `ENABLE_INCREMENTAL = false` để quay lại behavior cũ (các hàm incremental sẽ không chạy, bạn có thể dùng lại logic full reload nếu có).

## 13. Nâng Cấp Sau Này (Gợi ý)
- Thêm lưu `lastStreamId` vào `localStorage` để giữ cursor khi refresh trang.
- Thêm auth token cho `stream.php` giống như `ingest.php` (bảo mật hơn).
- Chuyển sang WebSocket push nếu cần realtime sub-second.
- Sampling / decimation nếu dataset quá dày (giảm điểm để chart mượt).

---
Muốn thêm phiên bản WebSocket (đẩy ngay không cần poll) hoặc kiểm tra bảo mật cho `stream.php`, cứ yêu cầu tiếp theo nhé.
