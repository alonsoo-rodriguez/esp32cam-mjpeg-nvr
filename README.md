## Overview

This repository turns ESP32‑CAM–class boards into a full‑featured network video recorder and multi‑camera surveillance system.

There are two main components:

- **ESP32‑CAM firmware (`ESP32-CAM_MJPEG2SD`)**  
  Captures JPEG frames, writes motion‑triggered or time‑lapse AVI files to SD, exposes a browser UI for configuration and live MJPEG streaming, and integrates with peripherals (PIR, lamp, servos, battery monitor, microphone, telemetry sensors).

- **Python WebSocket surveillance backend (`python_backend`)**  
  Runs on a remote host, accepts multiple ESP32‑CAM clients over WebSockets, aggregates their video streams and status/motion events, and displays everything on a central control page.

The codebase is derived from an open‑source project and has been scrubbed of personal developer identifiers. References to individual usernames or personal repositories have been replaced with neutral placeholders.

## Key Capabilities

- **Recording & playback**
  - Capture JPEG frames and record **AVI** files with an indexed video stream and optional mono WAV audio.
  - Motion‑triggered recording with configurable sensitivity, minimum duration, and maximum frame count.
  - Independent **time‑lapse** capture stream (own interval, duration, and playback FPS).
  - Playback of stored AVI files as MJPEG over HTTP.

- **On‑device web UI (served by the ESP32)**
  - Single‑page app (`data/MJPEG2SD.htm` + `data/common.js`) for:
    - Camera controls: resolution, FPS, exposure, gain, color, lens correction, mirroring/flip, special effects.
    - Motion detection: sensitivity, thresholds, bands, and minimum sequence length.
    - Time‑lapse: enable/disable, interval, duration, playback rate.
    - Peripherals: lamp control, PIR, pan/tilt servos, battery monitor, telemetry, microphone.
    - Storage/file management: folder/file browser, playback selection, FTP upload, download, deletion, SD free‑space policy.
    - Logging and OTA: log viewer (RAM + SD), verbosity toggle, OTA upload for firmware and data files.
  - Persistent configuration stored via `configs.txt` + NVS, surfaced and edited through the UI.

- **Peripherals & telemetry**
  - Optional PIR sensor, lamp driver (PWM or WS2812), DS18B20 temperature sensor, ADC‑based battery monitoring, I²C‑based telemetry sensors, and pan/tilt servos.
  - External I2S or PDM microphone support that records audio into WAV, later muxed into the AVI file.
  - Telemetry module that periodically logs sensor data to CSV during recordings.

- **Networking & integrations**
  - Wi‑Fi Station + AP mode, MDNS, NTP time sync, ping‑based connectivity watchdog.
  - SD‑MMC or flash filesystem, with automatic cleanup and optional FTP offload of oldest data.
  - SMTP email alerts (e.g., motion alerts, IP‑change notifications), with frame attachments when configured.
  - MQTT integration for publishing motion and recording state changes.
  - IO‑extender mode over UART to move selected peripherals to a second ESP32 when GPIO pins are constrained.

- **Remote surveillance backend**
  - Tornado‑based Python WebSocket server (`python_backend/websockets_stream_server.py`).
  - ESP32‑CAM clients push JPEG frames plus metadata and motion events over WebSockets.
  - The server overlays timestamps and hostnames, and fans out data to browser clients.
  - `templates/index.html` + `js/main.js` + `css/main.css` implement a multi‑camera dashboard with:
    - Per‑camera video panes.
    - Motion indicators and overlays.
    - Simple per‑camera logs and status messages.

## Repository Layout (High‑Level)

**Firmware / Arduino side**

- `ESP32-CAM_MJPEG2SD.ino` – Main sketch; orchestrates startup: logging, storage, configuration, camera, Wi‑Fi, web/stream servers, peripherals, audio, telemetry.
- `appGlobals.h`, `globals.h`, `camera_pins.h`, `appSpecific.cpp` – Compile‑time configuration, pin mappings, global declarations, and app‑specific handlers.
- `mjpeg2sd.cpp`, `avi.cpp` – Core recording and playback pipeline (AVI building, indexing, and MJPEG streaming).
- `motionDetect.cpp` – Motion detection, motion maps for debugging, night/day determination.
- `webServer.cpp`, `streamServer.cpp`, `prefs.cpp` – HTTP API, WebSocket logging/status, MJPEG endpoints, configuration storage and JSON status.
- `utils.cpp`, `utilsFS.cpp`, `peripherals.cpp`, `uart.cpp`, `ftp.cpp`, `smtp.cpp`, `mic.cpp`, `telemetry.cpp`, `setupAssist.cpp` – Shared infrastructure: Wi‑Fi, NTP, logging, storage, peripherals, IO‑extender UART, FTP upload, SMTP alerts, audio capture, telemetry logging, and initial data‑file bootstrap.

**Web UI served by ESP32**

- `data/MJPEG2SD.htm` – Main HTML page for camera control, monitoring, and configuration.
- `data/common.js` – Shared browser logic (status polling, user interaction, logging, OTA upload).
- `data/configs.txt` – Configuration schema: keys, groups, types, labels used to drive the UI.

**Python backend**

- `python_backend/websockets_stream_server.py` – Tornado WebSocket server and HTTP endpoints.
- `python_backend/templates/index.html` – Remote multi‑camera control page template.
- `python_backend/js/main.js` – Browser‑side WebSocket client and UI controller.
- `python_backend/css/main.css` – Dashboard styling.
- `python_backend/requirements.txt`, `python_backend/install.txt` – Python dependencies and installation notes.

## Building & Flashing the ESP32‑CAM Firmware

The firmware targets ESP32‑class camera boards with PSRAM (e.g., AI Thinker, ESP32‑S3‑based camera boards).

### Prerequisites

- ESP32 board support installed in the Arduino IDE (or equivalent support in PlatformIO / ESP‑IDF + Arduino core).
- PSRAM enabled in board settings.
- SD‑MMC wiring consistent with the pin definitions in `appGlobals.h` / `camera_pins.h`.

### Arduino IDE workflow

1. Install the **ESP32** board package via the Boards Manager.
2. Open `ESP32-CAM_MJPEG2SD.ino`.
3. In `appGlobals.h`:
   - Select the correct `CAMERA_MODEL_*` define for your hardware.
   - Confirm `STORAGE` type and SD‑MMC pin assignments.
4. Choose a board and partition scheme that provides enough flash for program + filesystem (if using LittleFS/SPIFFS).
5. Compile and upload using your preferred programmer / USB‑to‑serial adapter.

On first boot, the device will either:

- Connect to a preconfigured Wi‑Fi network and expose the UI at `http://<hostname>.local`, or  
- Start in AP mode and present a minimal Wi‑Fi setup page; once STA credentials are provided and saved, it reboots into normal mode.

## Running the Python WebSocket Backend

The backend is optional but recommended for internet‑facing or multi‑site deployments where cameras connect out to a central server.

### Prerequisites

- Python **3.8+** on Linux or Windows.
- A reachable TCP port (default `9090`) for WebSocket and HTTP traffic.

### Installation

```bash
cd python_backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt
```

### Start the server

```bash
cd python_backend
python websockets_stream_server.py
```

By default, the dashboard is then available at:

```text
http://<server-hostname>:9090/
```

## Connecting ESP32‑CAM Devices to the Backend

1. Flash and boot each ESP32‑CAM device with this firmware.
2. Open the device’s local web UI.
3. In the configuration pages (e.g., `Edit config` → `Other` or equivalent):
   - Set the WebSocket server **host/IP** to your backend host.
   - Set the WebSocket **port** to match the backend (default `9090`).
   - Enable the WebSocket / “remote surveillance server” option.
4. Save settings and reboot the device.

Each camera will then connect outbound to the backend and stream:

- JPEG frames for video.
- Motion start/stop events when motion detection is active.
- Informational/status messages (e.g., frame size, lamp changes, FPS) that are displayed in the dashboard.

## Security & Privacy Considerations

- This copy of the repository intentionally **removes personal developer handles and direct personal repo URLs** from comments, UI text, and configuration defaults.
- The firmware and UI expose sensitive configuration (Wi‑Fi credentials, FTP/SMTP passwords, etc.). You should:
  - Enable HTTP authentication on the UI.
  - Prefer placing devices behind a VPN, reverse proxy, or firewall rather than exposing them directly to the internet.
  - Use application‑specific passwords or low‑privilege accounts for SMTP/FTP/MQTT.
- If you publish a public fork, update `GITHUB_URL` in `appGlobals.h` to point at your own content host, and audit any remaining links to ensure they refer to your infrastructure.

## Extensibility Guidelines

The project is organised so that:

- **Application‑specific logic** is isolated in `appSpecific.cpp` and `appGlobals.h`.
- **Shared infrastructure** (Wi‑Fi, filesystem, logging, peripherals, FTP/SMTP/MQTT) is reusable by other ESP32 projects.
- **Configuration UI** is mostly driven by `configs.txt`, allowing new settings to be surfaced with minimal front‑end work.

When extending or integrating:

- Add new configuration keys to `data/configs.txt`, then wire them through `prefs.cpp` and `appSpecific.cpp`.
- Keep board‑specific pinouts and hardware assumptions localised to `camera_pins.h` and `appGlobals.h`.
- For remote control, telemetry, or monitoring, prefer using the existing MQTT and WebSocket paths rather than introducing new ad‑hoc protocols.