# IRIS 2.0 — Intelligent Retinal Interpretation System
IRIS 2.0 runs entirely offline on a **Raspberry Pi 5**, helping visually impaired users navigate their environment safely.
It detects nearby objects, measures obstacle distance, monitors for falls, and streams structured data to a companion mobile app (which handles text-to-speech announcements, as well as it also handle Online Version of IRIS 2.0).
No internet connection required.

---

## What It Does

| Feature | How |
|---|---|
| Object detection | YOLOv8n on CPU at 8–12 FPS |
| Distance measurement | HC-SR04 ultrasonic at 10 Hz |
| Fall detection | MPU9250 IMU via I2C at 50 Hz |
| JSON stream | WebSocket server on port 8765 |

---

## Hardware Required

- Raspberry Pi 5
- Pi Camera Module (any CSI camera — Camera Module v2/v3, HQ Camera, ArduCam)
- HC-SR04 Ultrasonic Sensor
- MPU9250 IMU breakout board
- 1kΩ and 2kΩ resistors (for HC-SR04 ECHO voltage divider)
- Official 27W USB-C PSU

---

## Wiring

### HC-SR04 — Ultrasonic Distance Sensor

| HC-SR04 Pin | Connection | Raspberry Pi Pin | Notes |
|-------------|------------|------------------|---------|
| **VCC** | → | **5V** (Pin 2 or 4) | Power supply |
| **GND** | → | **GND** (Pin 6) | Ground |
| **TRIG** | → | **GPIO23** (Pin 16) | Trigger (3.3V output from Pi is OK) |
| **ECHO** | ⚠️ | **GPIO24** (Pin 18) | **MUST USE VOLTAGE DIVIDER!** |

The HC-SR04 ECHO pin outputs 5V. The Pi GPIO only tolerates 3.3V.
Build this simple voltage divider between ECHO and GPIO24:

```
HC-SR04 ECHO ─── 1kΩ ─── GPIO24 (Pin 18)
                           │
                          2kΩ
                           │
                         GND
```

### MPU9250 — 9-Axis IMU (I2C)

| MPU9250 Pin | Connection | Raspberry Pi Pin | Notes |
|-------------|------------|------------------|---------|
| **VCC** | → | **3.3V** (Pin 1) | ⚠️ 3.3V ONLY! Do not use 5V |
| **GND** | → | **GND** (Pin 6) | Ground |
| **SDA** | → | **GPIO2 (SDA)** (Pin 3) | I2C Data |
| **SCL** | → | **GPIO3 (SCL)** (Pin 5) | I2C Clock |
| **AD0** | ↓ | GND or Float | I2C address (GND = 0x68, VCC = 0x69) |

### Pi Camera (CSI)

Connect via CSI-2 ribbon cable to the **CAM0** port on Pi 5. No GPIO needed.
Works with any Pi CSI camera: Camera Module v2/v3, HQ Camera, ArduCam, etc.

```
iris_offline/
├── main.py               # starts everything, keeps it running
├── vision.py             # camera capture + YOLOv8n inference
├── ultrasonic.py         # HC-SR04 distance polling
├── fall_detection.py     # MPU9250 I2C fall detection
├── server.py             # WebSocket JSON stream server (port 8765)
├── utils.py              # shared state, data models, helpers
├── requirements.txt      # Python packages
├── iris_offline.service  # systemd service file (manual start)
├── setup.sh              # full install + re-setup script (idempotent)
└── update.sh             # quick update: git pull → sync → restart
```

---

## Architecture Overview

IRIS 2.0 is built as a **multi-threaded real-time sensor fusion system** running on a single Raspberry Pi 5:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  IRIS 2.0 Offline Module — 5 Threads + Central Shared State                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │ VisionThread         │  │ UltrasonicThread    │  │ FallDetection       │  │
│  │ • PiCamera2          │  │ • GPIO23 (TRIG)     │  │ • I2C bus 1         │  │
│  │ • YOLOv8n 10 FPS     │  │ • GPIO24 (ECHO)     │  │ • MPU9250 @ 50 Hz   │  │
│  │ (640×480)            │  │ • 10 Hz polling     │  │                     │  │
│  └─────────────┬────────┘  └─────────────┬───────┘  └──────────┬──────────┘  │
│                │                         │                     │             │
│                └─────────────────────────┼─────────────────────┘             │
│                                          ▼                                   │
│                       ┌───────────────────────────────────────┐              │
│                       │ SharedState (Thread-safe RLock)       │              │
│                       │                                       │              │
│                       │ • detections[]   • fps                │              │
│                       │ • distance_feet  • system_status      │              │
│                       │ • fall.status    • errors[]           │              │
│                       └────────────────┬──────────────────────┘              │
│                                        │                                     │
│                       ┌────────────────▼─────────────────┐                   │
│                       │ ServerThread                     │                   │
│                       │ (WebSocket port 8765)            │                   │
│                       │                                  │                   │
│                       │ Pushes JSON every 500ms to all   │                   │
│                       │ connected clients                │                   │
│                       └──────────────────────────────────┘                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Design Principles:**
- **Single Source of Truth** — All modules write to/read from SharedState
- **Thread Safety** — RLock prevents partial updates from being observed
- **Graceful Degradation** — If any sensor fails, others continue running
- **Non-blocking** — Each thread is independent; no single point waits for another
- **Low Latency** — WebSocket pushes at 500ms; no buffering or batching

**Why No Local Display?**

The Vision thread stores annotated frames (with YOLO bounding boxes), but there is **no local display thread**. This design choice:

- ✅ Reduces CPU/RAM overhead on the Pi (headless operation)
- ✅ Allows monitoring from any client (web/mobile app, SSH session, Python script)
- ✅ Avoids display dependency (works over SSH without X11/Wayland)
- ✅ Enables future integration of streamed video endpoints if needed

To monitor live detections: connect to the WebSocket endpoint and visualize on your client device.

---

## Setup

> Run all commands below directly in your Raspberry Pi terminal.
> Connect via SSH, TigerVNC, or a keyboard+monitor.

---

### Step 1 — Flash the OS

Use **Raspberry Pi Imager** on your PC:
- OS: **Raspberry Pi OS Lite 64-bit (Bookworm)**
- Enable SSH in Imager settings (Ctrl+Shift+X) so you can log in remotely

---

### Step 2 — Enable I2C and Camera

```bash
sudo raspi-config nonint do_i2c 0
sudo raspi-config nonint do_camera 0
sudo reboot
```

---

### Step 3 — Verify Hardware

```bash
sudo i2cdetect -y 1            # MPU9250 should show as 68 (or 69)
rpicam-still --list-cameras    # Pi camera should be listed
rpicam-still -t 0              # opens live camera preview — press Ctrl+C to exit
```

---

### Step 4 — Run the Setup Script

Clone the repo and run `setup.sh` in one go:

```bash
sudo apt install -y git
git clone https://github.com/thekaushal01/Offline-Module-v2-for-IRIS2.0.git ~/iris_offline_src \
    || git -C ~/iris_offline_src pull
sudo bash ~/iris_offline_src/iris_offline/setup.sh https://github.com/thekaushal01/Offline-Module-v2-for-IRIS2.0.git
```

The script handles everything automatically:
- Installs all system and Python dependencies
- Enables I2C, camera, GPU memory in `config.txt`
- Creates the `iris` system user
- Sets up a Python virtual environment
- Downloads the YOLOv8n model
- Installs the systemd service file
- Applies Pi 5 performance tuning

> The first run takes 10–15 minutes (mostly pip + model download). Subsequent re-runs take under a minute.

When the script finishes, reboot to activate I2C and camera changes:

```bash
sudo reboot
```

After reboot, run IRIS from the terminal (as the `pi` user — no `sudo -u iris` needed):

```bash
/opt/iris_offline/venv/bin/python /opt/iris_offline/main.py
```

The WebSocket server will be live on port 8765.
Press `Ctrl+C` to stop.

**Verify it's working** (open a second terminal while IRIS is running):

```bash
# Check the WebSocket port is open
ss -tlnp | grep 8765

# Quick JSON frame via wscat (install once: npm i -g wscat)
wscat -c ws://localhost:8765
# Each message is a JSON payload — press Ctrl+C to disconnect

# Get your Pi's IP to connect from the companion app
hostname -I
# App connects to:  ws://<pi_ip>:8765

# Or test with Python
python3 << 'EOF'
import asyncio, websockets, json
async def test():
    async with websockets.connect('ws://localhost:8765') as ws:
        msg = await ws.recv()
        data = json.loads(msg)
        print(f"FPS: {data['fps']}, Distance: {data['distance_feet']}ft, Fall: {data['fall_detection']['status']}")
asyncio.run(test())
EOF
```

**Typical Terminal Output:**

```bash
$ /opt/iris_offline/venv/bin/python /opt/iris_offline/main.py
============================================================
IRIS 2.0 Offline Module — Starting up
============================================================
Started UltrasonicThread
Started FallDetectionThread
Started VisionThread
Started ServerThread
All systems running. Press Ctrl+C or SIGTERM to stop.
WebSocket JSON stream: ws://localhost:8765

  FPS=11.2  dist=3.45ft  fall=normal  [person(89%)  backpack(72%)]  status=clear
  FPS=11.0  dist=3.41ft  fall=normal  [person(91%)  backpack(75%)]  status=clear
```

---

## WebSocket Server Behavior

The server is **event-agnostic** and pushes a complete snapshot every 500ms regardless of whether data changed:

- **Camera offline:** `fps=0.0`, `vision=[]`, errors include "Camera read error"
- **I2C failing:** `fall_detection` fields included with last-known state
- **Distance unstable:** `distance_feet` includes raw reading; client can apply smoothing
- **Error auto-clear:** After 12 seconds of successful operation, transient errors are cleared from the errors list

This design allows clients to detect system health degradation without explicit "status" endpoints.

---

## Updating After Code Changes

Whenever you edit code on your PC, commit, and push — run this single command on the Pi:

```bash
sudo bash /opt/iris_offline/update.sh
```

This does: `git pull` → sync changed files → pip install (if needed). Takes under 30 seconds.
Then run IRIS again manually as usual.

**What is preserved across updates:**
- YOLOv8n model (`yolov8n.pt`) — never re-downloaded
- Python virtual environment — only updated if `requirements.txt` changed

**Typical workflow:**

```
Your PC (VS Code)              Raspberry Pi
─────────────────              ─────────────────────────────────────
Edit code
git add .
git commit -m "fix: ..."
git push
                     ──────→  sudo bash /opt/iris_offline/update.sh
                               ✓ pulled 3 files
                               ✓ code synced
                     run IRIS: /opt/iris_offline/venv/bin/python /opt/iris_offline/main.py
```

If you ever need to fully re-install from scratch (new Pi, or something broken):

```bash
sudo bash /opt/iris_offline/setup.sh
```

The same `setup.sh` does both full setup and re-setup. It skips steps that are already done (existing user, model, etc.).

---

## WebSocket API Reference

**Endpoint:** `ws://<pi_ip>:8765`

Connects with any WebSocket client. The server pushes a new JSON frame every **500ms** to all connected clients.

**Payload Schema:**

Each message is a complete snapshot of all currently active sensors:

```json
{
  "timestamp": "2026-02-25T10:30:45.123000Z",
  "fps": 10.2,
  "distance_feet": 2.4,
  "system_status": "warning",
  "fall_detection": {
    "status": "normal",
    "impact_g": 1.02
  },
  "vision": [
    { "name": "person", "confidence": 0.91 },
    { "name": "chair", "confidence": 0.87 }
  ],
  "errors": []
}
```

**Field Reference:**

| Field | Type | Description |
|---|---|---|
| `timestamp` | ISO-8601 string | UTC time when this snapshot was captured |
| `fps` | float | Current inference FPS (rounded to 1 decimal) |
| `distance_feet` | float | Distance from HC-SR04 sensor; -1.0 = unavailable |
| `system_status` | string | `clear`, `warning` (obstacle < 3ft), or `emergency` (fall detected) |
| `fall_detection.status` | string | `normal`, `impact_detected`, or `possible_fall` |
| `fall_detection.impact_g` | float | Current total acceleration in Gs |
| `vision[]` | array | Detected objects from YOLOv8n (max 20); sorted by confidence |
| `errors[]` | array | List of active error messages (max 10); auto-cleared after 12s startup period |

---

## Fall Detection Logic

The MPU9250 is polled at 50 Hz. Falls are detected in three phases:

```
NORMAL
  │  Total acceleration >= 2.5g (sudden impact)
  ▼
IMPACT_DETECTED
  │  Wait 0.5s for motion to settle
  ▼
IMMOBILITY_CHECK
  │  Acceleration variance < 0.05g for 3 consecutive seconds
  ▼
POSSIBLE_FALL  →  auto-clears after 5 seconds
```

If the person moves again before the 3-second immobility window completes, the state resets to `normal` — so bumping a table won't trigger a false alarm.

---

## Performance & Behavior

| Component | Rate | Notes |
|---|---|---|
| WebSocket push | 500ms | Every connected client receives a JSON snapshot |
| Vision (YOLO) | 8–12 FPS | YOLOv8n at 640×480 on Pi 5 CPU; max 20 detections per frame |
| Ultrasonic (distance) | 10 Hz | 5-sample median filter applied |
| Fall detection (IMU) | 50 Hz | Raw poll rate; state machine processes every 200ms |
| Terminal status line | 500ms | Live feedback in the terminal without log spam |
| Log file heartbeat | 30s | Periodic systemd journal entries for monitoring |
| Startup error clear | 12s | Transient I2C/camera init errors are auto-cleared if resolved |

**Memory & Resource Usage:**
- RAM usage: < 500 MB (mostly YOLO model)
- CPU usage: ~40–60% on Pi 5 (shared across 5 threads)
- Continuous uptime: 8+ hours (with proper cooling)

---

---

## Thermal Tips

The Pi 5 will throttle at 85°C. For 8-hour sessions:

- Attach the **official RPi 5 active cooler** (highly recommended)
- Use a **vented case**, not a sealed enclosure
- Use the **official 27W USB-C PSU** — underpowering causes both throttling and instability
- Keep ambient temperature below 35°C

Monitor temperature while running:
```bash
watch -n2 vcgencmd measure_temp
vcgencmd get_throttled   # 0x0 means no throttle has occurred
```

---

## Troubleshooting

### MPU9250 not showing on I2C scan
```bash
sudo i2cdetect -y 1   # check for 0x68 or 0x69
```
If not found: VCC must be 3.3V (not 5V), and AD0 should be tied to GND.

### Camera not found
```bash
rpicam-still --list-cameras   # should list at least one camera
```
If none appear: check ribbon cable is firmly seated in CAM0 port, and verify camera is enabled via `sudo raspi-config`.

### HC-SR04 giving wrong distance
Most common cause: **missing voltage divider on ECHO pin**. ECHO outputs 5V — the Pi GPIO will be damaged without the divider (1kΩ to GPIO, 2kΩ to GND).

### WebSocket not reachable
```bash
ss -tlnp | grep 8765               # confirm port is open
sudo journalctl -u iris_offline -f  # check for server errors

# Test with wscat (npm i -g wscat)
wscat -c ws://localhost:8765
# Should immediately start receiving JSON frames

# Or with Python:
python3 -c "
import asyncio, websockets
async def t():
    async with websockets.connect('ws://localhost:8765') as ws:
        print(await ws.recv())
asyncio.run(t())
"
```

### Low FPS
```bash
vcgencmd get_throttled        # 0x0 = fine; anything else = throttling occurred
vcgencmd measure_temp         # check current temperature
```
If throttled: attach active cooler, use vented case, check PSU is official 27W model.

### Check memory usage
```bash
ps aux | grep main.py | grep -v grep   # quick check of IRIS process
```

---

## YOLO Model Options

You can swap `yolov8n.pt` for a larger model if you need better accuracy:

| Model | File size | FPS on Pi 5 | |
|---|---|---|---|
| yolov8n | 6.3 MB | 8–12 FPS | **Use this one** |
| yolov8s | 22 MB | 4–6 FPS | more accurate, slower |
| yolov8m | 52 MB | 1–3 FPS | too slow for real-time |

---

## License
MIT — Internal deployment only. Not for redistribution.

---

