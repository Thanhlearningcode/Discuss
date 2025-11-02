# Realtime Data Retrieval from Database to Web GUI

This guide explains fast strategies to bring new rows from MySQL (`filtered_readings`) to the browser (`public/app/index.html`) for near real-time visualization, using the existing ingestion endpoints (e.g. `ingest.php`) and the new streaming endpoint `public/api/stream.php`.

## 1. Current Ingestion Flow
Local process -> POST `/digitwinbkdn_api/api/ingest.php` (or `ingest_filtered.php`) -> rows inserted into `filtered_readings`.

Rows schema fragments (simplified):
```
filtered_readings(id PK AUTO_INCREMENT,
  file_id INT,
  created_at DATETIME(3),
  AmpH_C1_nX FLOAT,
  AmpV_C2_nX FLOAT,
  Time_domain_C1 FLOAT NULL,
  Time_domain_C2 FLOAT NULL,
  ...)
```

## 2. Frontend Requirements
- Append new samples (AmpH/AmpV) continually to chart/table without reloading entire dataset.
- Avoid redundant transfers (only fetch rows not yet seen).
- Work across many files (switch file quickly).
- Handle first load (initial batch) efficiently (option to load newest or oldest segment).

## 3. Naïve Approach (What to Avoid)
Calling the full `data.php?file=...&limit=1000` every few seconds and diffing in JS wastes bandwidth and DB CPU. It scales poorly when row count grows.

## 4. Incremental Streaming Endpoint (`stream.php`)
Added endpoint: `public/api/stream.php`.

Query parameters:
- `file`: same mapping logic as `data.php` (can be filename, `FILE#<id>`, or numeric id).
- `after_id`: last row id you have (only return rows with id > after_id).
- `limit`: max rows to return (default 500, clamp 1..2000).
- `mode=latest`: on initial load (no `after_id`) fetch newest rows instead of oldest.

Response shape:
```json
{
  "ok": true,
  "data": [ {"id": 1234, "ts": "2025-11-02 10:15:20.123", "AmpH": 0.56, "AmpV": 0.42}, ... ],
  "meta": {
    "count": 120,
    "max_id": 1353,
    "file_id": 7,
    "source_file": "A_10_00.csv",
    "initial": false,
    "poll_hint_ms": 1500
  }
}
```

### 4.1 DB Index Recommendation
Ensure composite index for fast `WHERE file_id = ? AND id > ?`:
```sql
ALTER TABLE filtered_readings ADD INDEX idx_file_id_id (file_id, id);
```

### 4.2 Client Poll Algorithm
Pseudo-code:
```js
let lastId = null;
async function poll(){
  const base = '/public/api/stream.php?file=' + encodeURIComponent(currentFile);
  const url = lastId ? base + '&after_id=' + lastId : base + '&mode=latest';
  const res = await fetch(url);
  const j = await res.json();
  if(!j.ok) return;
  const rows = j.data;
  if(rows.length){
    // Append to chart/table
    rows.forEach(r => appendSample(r));
    lastId = j.meta.max_id;
  }
  setTimeout(poll, j.meta.poll_hint_ms || 1500);
}
poll();
```

### 4.3 Initial vs Continuous
- First call (no `lastId`) loads a window (oldest or newest depending on `mode`).
- Subsequent calls only bring deltas. This keeps traffic proportional to ingestion speed.

## 5. Handling Multiple Files
When user switches file:
1. Cancel current polling timer.
2. Reset `lastId = null`.
3. Clear charts.
4. Start `poll()` again for the new file.

## 6. Error & Gap Handling
- If network error: retry after backoff (e.g. 2s, then 4s, max 15s).
- If ingestion jumps (server inserted huge batch): `max_id` leaps forward; still fine because client only cares `id > lastId`.
- If out-of-order timestamps: sort by `id` (monotonic) rather than `created_at` for continuity.

## 7. Real-Time Push Alternatives (Future)
If true sub-second latency required or very high ingestion rate:
1. WebSocket push: server keeps connection and broadcasts new rows.
2. SSE (Server-Sent Events): simpler than WS when only one-way stream.
3. Binpack: send binary pack of floats like ingest uses to reduce size.

Current approach chosen for simplicity + compatibility with existing PHP hosting.

## 8. Performance Tips
- Keep `limit` moderate (100–1000) to avoid large JSON for first load.
- Use chart decimation (Chart.js has decimation plugin) if sample count grows.
- Trim in-memory arrays (e.g. keep last N = 10k samples) to prevent UI slowdown.
- Consider compression (gzip) if rows > few KB (usually server auto handles).

## 9. Security Considerations
- Reuse existing auth (e.g. require a Bearer token) for `stream.php` if data is sensitive (currently open by default—add check similar to `ingest.php`).
- Rate limit IP if necessary (simple count per minute).
```php
// Example minimal token check snippet (add near top):
// $auth = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
// if(!preg_match('/^Bearer\s+(\S+)/',$auth,$m) || !hash_equals($API_BEARER,$m[1])) json_err('Auth',401);
```

## 10. Frontend Integration Notes
In `index.html` you can create a data adapter object:
```js
window.digitwinData = {
  startStream(file){ currentFile=file; lastId=null; clearDisplay(); poll(); },
  stop(){ if(pollTimer) clearTimeout(pollTimer); }
};
```
Combine with existing motor UI without changing control logic.

## 11. Checklist
| Step | Done? |
|------|-------|
| Add stream.php | ✓ |
| Create DB index | (run manually) |
| Integrate polling in UI | pending |
| Add auth (optional) | pending |
| Optimize chart memory | pending |

## 12. Next Steps
1. Add client-side polling block to `index.html`.
2. Add Bearer auth (if required) to `stream.php`.
3. Provide small utility to map available files using `files.php` then choose and stream.

---
Questions or need a WebSocket push variant? Ask and we can extend with a `ws_stream` channel.
