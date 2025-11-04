# README — Full Code + Giải thích (newbie-friendly)

Kiến trúc mục tiêu:

```
(MQTT publishers: thiết bị/worker)
        │
        ▼
   EMQX (broker)
  1883 (TCP), 8083 (WebSocket)
        │                     
        ├── Ingestor (Python)  ← subscribe telemetry, ghi PostgreSQL
        │
        └── Flutter App  ← MQTT (WS) để realtime
             │
             ▼
        FastAPI (HTTP API)  ← JWT, đọc dữ liệu từ PostgreSQL
             │
             ▼
           PostgreSQL
```

> **Tư duy:** MQTT để realtime, PostgreSQL để lưu lịch sử, FastAPI để app/website truy cập, Ingestor là dịch vụ riêng đọc MQTT và ghi DB (giúp scale dễ dàng).

---

## 1) Yêu cầu & khởi động nhanh

* Cài **Docker** + **Docker Compose**.
* (Tùy chọn) Cài **Python 3.12+** nếu muốn chạy device giả lập ngoài docker.

**Chạy tất cả:**

```bash
docker compose up -d --build
```

* Mở Swagger: [http://localhost:8000/docs](http://localhost:8000/docs)
* EMQX Dashboard: [http://localhost:18083](http://localhost:18083)  (user/pass mặc định: admin/public nếu chưa đổi)
* MQTT WS endpoint: ws://localhost:8083/mqtt

**Gửi dữ liệu thử (thiết bị giả):**

```bash
python sim_device.py
```

Sau đó vào Swagger → `POST /auth/login` (email tuỳ ý) → lấy token → `GET /telemetry/dev-01` để xem dữ liệu.

**Chạy Flutter (web cho nhanh):**

```bash
cd flutter_app
flutter pub get
flutter run -d chrome
```

Bấm **Login REST** → **Connect MQTT** → xem logs.

> **Lưu ý:** Nếu chạy Flutter trên **Android/iOS** thật, đổi `localhost` trong code Flutter thành **IP LAN của máy** chạy EMQX (ví dụ `192.168.1.10`).

---

## 2) Cấu trúc thư mục dự án

```
repo/
├─ docker-compose.yml
├─ sim_device.py                 # thiết bị giả lập (paho-mqtt)
├─ backend/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  └─ app/
│     ├─ __init__.py
│     ├─ db.py
│     ├─ models.py
│     ├─ schemas.py
│     ├─ auth.py
│     ├─ mqtt_pub.py
│     └─ main.py
│  └─ ingestor/
│     └─ run.py
└─ flutter_app/
   ├─ pubspec.yaml
   └─ lib/
      └─ main.dart
```

---

## 3) docker-compose.yml (hạ tầng + dịch vụ)

```yaml
version: "3.8"
services:
  emqx:
    image: emqx/emqx:5
    container_name: emqx
    ports:
      - "1883:1883"    # MQTT TCP
      - "8083:8083"    # MQTT WebSocket (ws://localhost:8083/mqtt)
      - "18083:18083"  # Dashboard
    environment:
      - EMQX_ALLOW_ANONYMOUS=true   # DEV ONLY (prod: tắt và bật auth)

  db:
    image: postgres:16
    environment:
      - POSTGRES_DB=app
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secret
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      retries: 10

  api:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    environment:
      - DATABASE_URL=postgresql+asyncpg://app:secret@db:5432/app
      - JWT_SECRET=devsecret
      - MQTT__HOST=emqx
      - MQTT__WS_URL=ws://emqx:8083/mqtt
    depends_on:
      - db
      - emqx
    ports: ["8000:8000"]

  ingestor:
    build: ./backend
    command: python -m ingestor.run
    environment:
      - DATABASE_URL=postgresql+asyncpg://app:secret@db:5432/app
      - MQTT__HOST=emqx
      - MQTT__PORT=1883
      - MQTT__TOPIC=t0/devices/+/telemetry
    depends_on:
      - db
      - emqx
```

**Giải thích:**

* **emqx**: mở các cổng cần thiết. `EMQX_ALLOW_ANONYMOUS=true` để dev nhanh.
* **db**: Postgres với user/password đơn giản.
* **api**: build từ `backend/`, expose HTTP 8000.
* **ingestor**: chạy service lắng nghe MQTT và ghi DB.

---

## 4) backend/Dockerfile (chuẩn build theo context `./backend`)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
ENV PYTHONPATH=/app
```

**Giải thích:**

* Sử dụng **context** là thư mục `backend/`, nên `COPY requirements.txt` và `COPY .` là đúng.
* `PYTHONPATH=/app` để module `app.*` và `ingestor.*` import được lẫn nhau.

---

## 5) backend/requirements.txt (thư viện chính)

```text
fastapi==0.115.0
uvicorn[standard]==0.30.6
sqlalchemy==2.0.35
asyncpg==0.29.0
alembic==1.13.2
pydantic==2.8.2
python-jose[cryptography]==3.3.0
asyncio-mqtt==0.16.2
```

**Giải thích ngắn:**

* FastAPI + Uvicorn = HTTP API.
* SQLAlchemy (ORM) + asyncpg (driver Postgres) + Alembic (migrations).
* Pydantic = validate JSON.
* python-jose = JWT.
* asyncio-mqtt = client MQTT async cho Ingestor/API khi cần publish.

---

## 6) backend/app/db.py (kết nối DB)

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
import os

DATABASE_URL = os.getenv("DATABASE_URL")
engine = create_async_engine(DATABASE_URL, echo=False, pool_pre_ping=True)
SessionLocal = sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

class Base(DeclarativeBase):
    pass

async def get_db():
    async with SessionLocal() as session:
        yield session
```

**Giải thích:** tạo `engine` (kết nối DB), `SessionLocal` (sinh session cho mỗi request), `Base` để các model kế thừa. `get_db()` là dependency dùng trong FastAPI route.

---

## 7) backend/app/models.py (bảng ORM)

```python
from sqlalchemy import Column, BigInteger, String, Text, JSON, UniqueConstraint, ForeignKey, DateTime
from sqlalchemy.sql import func
from sqlalchemy.dialects.postgresql import UUID
from .db import Base

class User(Base):
    __tablename__ = "users"
    id = Column(BigInteger, primary_key=True)
    email = Column(String, unique=True, nullable=False)
    password_hash = Column(Text, nullable=False)
    role = Column(String, default="user")
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Device(Base):
    __tablename__ = "devices"
    id = Column(BigInteger, primary_key=True)
    device_uid = Column(String, unique=True, nullable=False)
    name = Column(String)
    tenant = Column(String, default="t0")
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class Telemetry(Base):
    __tablename__ = "telemetry"
    id = Column(BigInteger, primary_key=True)
    device_uid = Column(String, ForeignKey("devices.device_uid"), nullable=False)
    msg_id = Column(UUID(as_uuid=False), nullable=False)
    payload = Column(JSON, nullable=False)
    ts = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    __table_args__ = (UniqueConstraint("device_uid", "msg_id", name="uq_device_msg"),)
```

**Giải thích:** `Telemetry` có **unique (device_uid, msg_id)** để chống ghi trùng (idempotency). `payload` lưu JSON gốc.

---

## 8) backend/app/schemas.py (I/O models cho API)

```python
from pydantic import BaseModel, EmailStr
from typing import Any, Dict

class LoginIn(BaseModel):
    email: EmailStr
    password: str

class TokenOut(BaseModel):
    access_token: str
    token_type: str = "bearer"

class TelemetryOut(BaseModel):
    device_uid: str
    payload: Dict[str, Any]
    ts: str
```

**Giải thích:** Pydantic đảm bảo JSON vào/ra đúng định dạng, giúp Swagger tự sinh docs đẹp.

---

## 9) backend/app/auth.py (JWT tối giản)

```python
from fastapi import HTTPException, Depends
from jose import jwt
from datetime import datetime, timedelta
import os

JWT_SECRET = os.getenv("JWT_SECRET", "devsecret")
ALGO = "HS256"

def create_token(sub: str, minutes: int = 60):
    now = datetime.utcnow()
    payload = {"sub": sub, "iat": now, "exp": now + timedelta(minutes=minutes)}
    return jwt.encode(payload, JWT_SECRET, algorithm=ALGO)

from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
security = HTTPBearer()

def require_user(token: HTTPAuthorizationCredentials = Depends(security)):
    try:
        data = jwt.decode(token.credentials, JWT_SECRET, algorithms=[ALGO])
        return data["sub"]
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")
```

**Giải thích:** tạo/kiểm tra JWT bằng HS256. Demo cho dev; production cần thêm refresh token, revoke list, v.v.

---

## 10) backend/app/mqtt_pub.py (publish lệnh từ API – tuỳ chọn)

```python
import os, json
from asyncio_mqtt import Client

MQTT_HOST = os.getenv("MQTT__HOST", "emqx")
MQTT_PORT = int(os.getenv("MQTT__PORT", "1883"))

async def publish_command(device_uid: str, payload: dict):
    topic = f"t0/devices/{device_uid}/commands"
    async with Client(MQTT_HOST, MQTT_PORT) as client:
        await client.publish(topic, json.dumps(payload), qos=1)
```

**Giải thích:** ví dụ hàm publish lệnh tới thiết bị. (Trong demo chưa bật endpoint gọi hàm này.)

---

## 11) backend/app/main.py (FastAPI routes)

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from .db import Base, engine, get_db
from .models import User, Device, Telemetry
from .schemas import LoginIn, TokenOut, TelemetryOut
from .auth import create_token, require_user
from typing import List

from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="FastAPI + MQTT + Postgres")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # DEV ONLY: mở CORS để Flutter Web gọi đc
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.on_event("startup")
async def on_startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

@app.post("/auth/login", response_model=TokenOut)
async def login(body: LoginIn, db: AsyncSession = Depends(get_db)):
    # DEMO: chấp nhận mọi email/pass, tạo user nếu chưa có
    res = await db.execute(select(User).where(User.email == body.email))
    user = res.scalar_one_or_none()
    if not user:
        user = User(email=body.email, password_hash="demo")
        db.add(user)
        await db.commit()
    return TokenOut(access_token=create_token(sub=body.email))

@app.get("/devices", dependencies=[Depends(require_user)])
async def list_devices(db: AsyncSession = Depends(get_db)):
    res = await db.execute(select(Device))
    return [
        {"device_uid": d.device_uid, "name": d.name, "tenant": d.tenant}
        for d in res.scalars().all()
    ]

@app.get("/telemetry/{device_uid}", response_model=List[TelemetryOut], dependencies=[Depends(require_user)])
async def get_telemetry(device_uid: str, db: AsyncSession = Depends(get_db)):
    res = await db.execute(
        select(Telemetry)
        .where(Telemetry.device_uid == device_uid)
        .order_by(Telemetry.id.desc())
        .limit(100)
    )
    rows = res.scalars().all()
    return [
        {"device_uid": r.device_uid, "payload": r.payload, "ts": r.ts.isoformat()}
        for r in rows
    ]
```

**Giải thích:**

* `startup` tạo bảng (dev). Prod dùng Alembic.
* `/auth/login` trả JWT (demo). `/devices` & `/telemetry/{uid}` yêu cầu JWT.
* Thêm **CORS** để Flutter Web gọi HTTP từ trình duyệt không bị chặn.

---

## 12) backend/ingestor/run.py (dịch vụ đọc MQTT → ghi DB)

```python
import os, json, uuid, asyncio
from asyncio_mqtt import Client, MqttError
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.db import SessionLocal
from app.models import Telemetry, Device

MQTT_HOST = os.getenv("MQTT__HOST", "emqx")
MQTT_PORT = int(os.getenv("MQTT__PORT", "1883"))
TOPIC = os.getenv("MQTT__TOPIC", "t0/devices/+/telemetry")

async def ensure_device(db: AsyncSession, device_uid: str):
    res = await db.execute(select(Device).where(Device.device_uid == device_uid))
    d = res.scalar_one_or_none()
    if not d:
        db.add(Device(device_uid=device_uid, name=device_uid))
        await db.commit()

async def handle_message(topic: str, payload: bytes):
    parts = topic.split("/")  # t0 devices {uid} telemetry
    device_uid = parts[2]
    try:
        body = json.loads(payload.decode())
    except Exception:
        body = {"raw": payload.decode(errors="ignore")}
    msg_id = body.get("msg_id") or str(uuid.uuid4())

    async with SessionLocal() as db:
        await ensure_device(db, device_uid)
        db.add(Telemetry(device_uid=device_uid, msg_id=msg_id, payload=body))
        try:
            await db.commit()
        except Exception:
            await db.rollback()  # duplicate? bỏ qua trong demo

async def main():
    reconnect_delay = 3
    while True:
        try:
            async with Client(MQTT_HOST, MQTT_PORT) as client:
                await client.subscribe(TOPIC, qos=1)
                async with client.unfiltered_messages() as messages:
                    async for m in messages:
                        await handle_message(m.topic, m.payload)
        except MqttError:
            await asyncio.sleep(reconnect_delay)

if __name__ == "__main__":
    asyncio.run(main())
```

**Giải thích:**

* Subscribe wildcard `t0/devices/+/telemetry`.
* Parse JSON, đảm bảo `msg_id` (tự sinh nếu thiếu), **commit** vào DB.
* Nếu trùng unique → rollback và bỏ qua.
* Có vòng reconnect khi broker tạm rơi.

---

## 13) sim_device.py (thiết bị giả lập – chạy ngoài docker)

```python
import json, time, random
from paho.mqtt import client as mqtt

BROKER_HOST = 'localhost'  # nếu chạy trên máy khác: đổi sang IP LAN của host
BROKER_PORT = 1883
uid = 'dev-01'

c = mqtt.Client()
c.connect(BROKER_HOST, BROKER_PORT, 60)

# publish status retained (để Flutter thấy ngay)
import time as _t
c.publish(f't0/devices/{uid}/status', json.dumps({'online': True, 'ts': int(_t.time())}), qos=1, retain=True)

for i in range(20):
    payload = {
        'msg_id': f'{i:04d}',
        'ts': int(time.time()),
        'data': { 'temp': round(20 + random.random()*5, 2) }
    }
    c.publish(f't0/devices/{uid}/telemetry', json.dumps(payload), qos=1)
    time.sleep(1)
```

**Giải thích:** mỗi giây gửi một bản ghi nhiệt độ, **status** được retained để app nhận ngay khi subscribe.

---

## 14) flutter_app/pubspec.yaml

```yaml
name: mqtt_fastapi_demo
environment:
  sdk: ">=3.4.0 <4.0.0"
dependencies:
  flutter: { sdk: flutter }
  dio: ^5.5.0
  mqtt_client: ^10.2.0   # hoặc mqtt5_client nếu muốn MQTT v5
```

---

## 15) flutter_app/lib/main.dart (demo tối giản – HTTP + MQTT)

```dart
import 'package:flutter/material.dart';
import 'package:dio/dio.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

void main() => runApp(const MyApp());

class MyApp extends StatefulWidget { const MyApp({super.key}); @override State<MyApp> createState() => _MyAppState(); }

class _MyAppState extends State<MyApp> {
  final dio = Dio(BaseOptions(baseUrl: 'http://localhost:8000'));
  String? token;
  MqttServerClient? client;
  String logs = '';

  Future<void> login() async {
    final res = await dio.post('/auth/login', data: { 'email': 'demo@example.com', 'password': 'x' });
    token = res.data['access_token'];
    dio.options.headers['Authorization'] = 'Bearer $token';
    setState(() => logs += '\nHTTP: Login OK');
  }

  Future<void> connectMqtt() async {
    // Nếu chạy trên mobile device thật: đổi 'localhost' thành IP LAN của máy chạy EMQX
    client = MqttServerClient.withPort('localhost', 'flutter_client', 8083);
    client!.useWebSocket = true; // EMQX WS: ws://host:8083/mqtt
    client!.logging(on: false);
    client!.keepAlivePeriod = 20;
    client!.onDisconnected = () => setState(() => logs += '\nMQTT: Disconnected');

    final connMess = MqttConnectMessage()
        .withClientIdentifier('flutter_${DateTime.now().millisecondsSinceEpoch}')
        .startClean()
        .withWillQos(MqttQos.atLeastOnce);
    client!.connectionMessage = connMess;

    try {
      await client!.connect();
      setState(() => logs += '\nMQTT: Connected');
      const topic = 't0/devices/+/status';
      client!.subscribe(topic, MqttQos.atLeastOnce);
      client!.updates!.listen((events) {
        final recMess = events[0].payload as MqttPublishMessage;
        final pt = MqttPublishPayload.bytesToStringAsString(recMess.payload.message);
        setState(() => logs += '\n[status] $pt');
      });

      // (Tuỳ chọn) subscribe luôn telemetry của dev-01 cho dễ thấy
      const topic2 = 't0/devices/dev-01/telemetry';
      client!.subscribe(topic2, MqttQos.atLeastOnce);
      client!.updates!.listen((events) {
        final recMess = events[0].payload as MqttPublishMessage;
        final pt = MqttPublishPayload.bytesToStringAsString(recMess.payload.message);
        setState(() => logs += '\n[telemetry] $pt');
      });

    } catch (e) {
      setState(() => logs += '\nMQTT error: $e');
      client!.disconnect();
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Flutter + FastAPI + MQTT')),
        body: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(children: [
                ElevatedButton(onPressed: login, child: const Text('Login REST')),
                const SizedBox(width: 12),
                ElevatedButton(onPressed: connectMqtt, child: const Text('Connect MQTT')),
              ]),
              const SizedBox(height: 12),
              Expanded(child: SingleChildScrollView(child: Text(logs)))
            ],
          ),
        ),
      ),
    );
  }
}
```

**Giải thích:**

* `login()` gọi FastAPI → lấy JWT để gọi các API khác (demo chỉ log).
* `connectMqtt()` mở WebSocket đến EMQX, subscribe `status` và `telemetry` của `dev-01` → in ra logs.

---

## 16) Quy trình test đầy đủ (end-to-end)

1. `docker compose up -d --build` (chạy EMQX + Postgres + API + Ingestor).
2. `python sim_device.py` (thiết bị giả gửi dữ liệu + status retained).
3. Swagger: [http://localhost:8000/docs](http://localhost:8000/docs)

   * `POST /auth/login` (email tuỳ ý) → token
   * Authorize → dán token
   * `GET /telemetry/dev-01` → thấy dữ liệu đã ghi DB
4. Flutter Web: `flutter run -d chrome` → **Login REST** → **Connect MQTT** → thấy log `[status]` và `[telemetry]`.

---

## 17) Ghi chú Production (khi triển khai thật)

* **EMQX**: tắt anonymous, bật xác thực JWT/HTTP, kiểm soát quyền publish/subscribe theo topic. Bật TLS (1883→8883; 8083→8084).
* **FastAPI**: bỏ auto-create table, dùng **Alembic** migrations. Thêm rate limit, logging JSON, metrics.
* **Ingestor**: chạy nhiều replica + **shared subscriptions** `$share/ingestors/t0/devices/+/telemetry` để load-balance.
* **PostgreSQL**: tạo index theo truy vấn; dùng connection pool hợp lý; cân nhắc partition nếu dữ liệu lớn.
* **Flutter**: cấu hình reconnect/backoff MQTT, offline cache, đổi `localhost` thành domain/IP thật.

---

## 18) Lỗi thường gặp & cách xử lý

* **Flutter không nhận message**: sai host (`localhost` vs IP LAN), sai topic, WS path không `/mqtt`.
* **CORS khi gọi API**: bật `CORSMiddleware` (đang bật DEV). Prod nên cấu hình danh sách origin cụ thể.
* **Trùng bản ghi telemetry**: đảm bảo mỗi message có `msg_id` ổn định; DB đã có `UNIQUE(device_uid,msg_id)`.
* **Broker rớt**: Ingestor có vòng reconnect; kiểm tra logs và network.

---

> **Kết luận:** README này chứa **toàn bộ code chạy được** và giải thích từng phần. Từ đây anh có thể mở rộng: thêm endpoint gửi lệnh (server → device), device ACK, dashboards, và auth nâng cao cho broker.
