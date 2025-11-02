# MQTT Integration Guide (Full)

This document consolidates everything you need to: (1) run a local MQTT broker, (2) understand how this source uses MQTT, (3) connect MATLAB (or Python fallback) to receive commands and publish status, (4) test end-to-end quickly, and (5) harden for production.

## 1. Overview
Web UI (in `public/app/index.html`) publishes motor control commands & periodic status over MQTT. External clients (MATLAB script or Python bridge) subscribe to command topic, act (or simulate), and publish status back. No original business/data logic in the project was changed; MQTT is an add‑on communication layer.

```
[Browser UI] <--pub/sub--> [MQTT Broker] <--pub/sub--> [MATLAB or Python Bridge] --> (Serial) STM32
```
python .\python\motor_mqtt_tcp_bridge.py --broker test.mosquitto.org --prefix digitwin --token DT123 --tcp-port 5055
Key topics (prefix = `digitwin` by default):
- Commands: `digitwin/motor/cmd`
- Status:   `digitwin/motor/status`
- (Future) Telemetry: `digitwin/motor/telemetry`

Payload examples:
```json
// Command
{ "type": "speed", "value": 60, "t": 1697040000000, "token": "DT123" }
{ "type": "start", "t": 1697040000500, "token": "DT123" }
{ "type": "stop",  "t": 1697040000900 }
// Status
{ "running": true, "speed": 60, "t": 1697040005000, "token": "DT123" }
```

## 2. Local Broker (Mosquitto) Quick Build
### 2.1 Windows (Temporary Dev)
1. Download Mosquitto: https://mosquitto.org/download/
2. Install (include service if you want auto-start).
3. Create a working folder (e.g. inside repo): `digitwinbkdn\broker` already contains `mosquitto.conf` sample.
4. Open PowerShell in that folder and create a password file (optional now):
```powershell
mosquitto_passwd -c .\passwd digitwinuser
```
5. Run broker referencing config:
```powershell
"C:\Program Files\mosquitto\mosquitto.exe" -c .\mosquitto.conf
```
6. Enabled listeners (in sample):
   - TCP: 1883 (mqtt://)
   - WebSocket: 8083 (ws://) — For browser if you host UI locally.
   - WSS (8084) commented; needs certs.

### 2.2 Ubuntu / Debian
```bash
sudo apt update && sudo apt install -y mosquitto mosquitto-clients
sudo systemctl enable mosquitto && sudo systemctl start mosquitto
# Edit config
sudo nano /etc/mosquitto/mosquitto.conf
# (Copy content from broker/mosquitto.conf, adjust paths)
# Create password file (optional)
sudo mosquitto_passwd -c /etc/mosquitto/passwd digitwinuser
sudo systemctl restart mosquitto
```
Enable WebSockets by adding:
```
listener 8083
protocol websockets
```
Restart service.

### 2.3 Docker
Minimal run:
```bash
docker run -d --name mosq -p 1883:1883 -p 8083:8083 \
  -v $(pwd)/broker/mosquitto.conf:/mosquitto/config/mosquitto.conf \
  eclipse-mosquitto:2
```
Add volumes for persistence (passwd, data) as needed.

### 2.4 Public Test (Skip Build)
Use `wss://test.mosquitto.org:8081` (no auth, only for quick demo). **Do not** use for sensitive data.

## 3. Source Code Integration Points
File: `public/app/index.html` contains an MQTT init block which:
- Adds settings modal (broker URL, prefix, token, auto-connect, remote control).
- Dynamically loads MQTT JS client if needed.
- Publishes on speed slider changes and start/stop events.
- Publishes heartbeat status every ~5s.
- Applies remote commands (if remote control enabled) after token check.
No existing PHP/DB/data acquisition logic altered.

Config is stored via `localStorage` so user settings persist page reloads.

## 4. MATLAB Client (If mqttclient Available)
File: `matlab/motor_mqtt_client.m` (script style).
Highlights:
- CONFIG block at top (BROKER_URL, TOPIC_PREFIX, AUTH_TOKEN, etc.)
- Auto-reconnect with fixed delay.
- Simulation mode when `CONFIG.ENABLE_SERIAL = false`.
- Frame helpers prepared for future binary serial protocol.
- Publishes status (heartbeat) at `CONFIG.PUBLISH_INTERVAL`.

Run (MATLAB Command Window):
```matlab
motor_mqtt_client
```
If your MATLAB version errors on `mqttclient`, install the MQTT Support Package or use Python fallback below.

## 5. Python Fallback Bridge
File: `python/motor_mqtt_bridge.py`
Install deps:
```powershell
pip install paho-mqtt
# optional serial
pip install pyserial
```
Run:
```powershell
python .\python\motor_mqtt_bridge.py --broker test.mosquitto.org --prefix digitwin --token DT123
```
Options:
- `--port COM6` to enable serial passthrough (needs pyserial).
- `--heartbeat 5` change status interval.

## 6. UI Configuration Steps (Browser)
Open settings (🛰 button):
1. Broker URL: (e.g.) `wss://test.mosquitto.org:8081` or your own `wss://host:8084/mqtt`.
2. Prefix: `digitwin` (match clients).
3. Token: `DT123` (optional shared secret).
4. Remote Control: ON (to echo remote commands locally).
5. Auto-Connect: ON (reconnect logic active).
Save → Connect.

## 7. End-to-End Test (Without Hardware)
1. Start Python bridge OR MATLAB script in simulation mode.
2. UI connects (pill shows MQTT ON).
3. Change speed slider: Bridge logs `[CMD] SPEED -> value`.
4. Click Start / Stop: Bridge logs `[CMD] START` / `[CMD] STOP`.
5. Observe periodic `[STATUS]` lines from bridge.
6. (Optional) Use an external MQTT client and subscribe `digitwin/motor/#` to verify messages.

## 8. Switching to Real STM32
- Set `CONFIG.ENABLE_SERIAL = true` in MATLAB (or run Python with `--port COMx`).
- Implement the actual binary frame protocol if needed inside Python `_send_serial` or MATLAB send* functions (already scaffolded in MATLAB version).
- Optionally extend telemetry parse to publish to `digitwin/motor/telemetry`.

## 9. Security & Hardening
| Layer | Action | Benefit |
|-------|--------|---------|
| Broker | Disable anonymous, use password_file | Blocks random users |
| Topics | Use unique prefix per deployment | Avoid collisions |
| Transport | Use TLS (wss:// / tls://) | Protect in transit |
| Access | ACL file restrict read/write | Least privilege |
| Payload | Validate speed 0–100, known types | Prevent malformed commands |
| Rate Limit | Debounce UI publishes, limit commands/sec | Avoid floods |
| Token | Shared secret field in payload | Basic filtering |
| Retain | Retained status (optional) | Late joiners get last state |
| Presence | LWT (Last Will) presence topics | Detect unexpected disconnect |

Example ACL (conceptual):
```
user webclient
pattern write digitwin/motor/cmd
pattern read  digitwin/motor/status

user bridge
pattern read  digitwin/motor/cmd
pattern write digitwin/motor/status
```

## 10. Tunneling Local Broker (If UI Remote, Broker Local)
- Ngrok: `ngrok tcp 1883` (for MQTT), or `ngrok http 8083` (WebSocket) then use provided URL.
- Cloudflared: `cloudflared tunnel --url http://localhost:8083`.
- Caution: Add auth/token; tunnels are public entry points.

## 11. Common Troubleshooting
| Symptom | Cause | Fix |
|---------|-------|-----|
| UI pill stays OFF | Wrong wss URL or broker down | Verify port & protocol (ws vs wss) |
| Python bridge no commands | Prefix/token mismatch | Align settings |
| MATLAB mqttclient undefined | Older MATLAB / missing support package | Install MQTT support or use Python bridge |
| Flood of speed messages | Rapid slider events | Add debounce / throttle in UI |
| Status missing in another client | Not subscribed or heartbeat interval long | Subscribe correct topic / lower interval |
| Token mismatch warnings | Different token values | Synchronize token |
| Public broker random messages | Shared global namespace | Move to private or change prefix |

## 12. Production Checklist
- [ ] Private broker (VPS or managed cloud) with TLS
- [ ] Password + ACL rules
- [ ] Unique deployment prefix
- [ ] Retained status enabled (if desired)
- [ ] Presence via LWT topics
- [ ] Logging & monitoring (broker logs, metrics)
- [ ] Backup / rotate credentials
- [ ] UI minified & remove test console logs

## 13. Minimal Quick Commands Recap (Windows)
```powershell
# Run Python fallback bridge
pip install paho-mqtt
python .\python\motor_mqtt_bridge.py --broker test.mosquitto.org --prefix digitwin --token DT123

# (Optional) Install Mosquitto locally and run with provided config
"C:\Program Files\mosquitto\mosquitto.exe" -c .\broker\mosquitto.conf
```

## 14. Next Extensions
- QoS 1 for critical commands
- Telemetry aggregation & charts via a backend collector
- WebSocket compression / payload minimization
- Multi-motor namespace: `<prefix>/motor/<id>/cmd`

---
If you need a localized (Vietnamese) summary or a script to auto-generate password + ACL, let me know.
