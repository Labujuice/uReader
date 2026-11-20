# uReader — Web Serial UBX / NMEA GNSS Analyzer

**Live Page:** https://labujuice.github.io/uReader/

**uReader** is a single-file, zero-dependency HTML5 frontend tool for reading, analyzing, and debugging **UBX** and **NMEA 0183** GNSS serial streams (`/dev/ttyACMx`, `/dev/ttyUSBx`, COM ports) directly in the browser via the **Web Serial API**.

Inspired by u-blox u-center, it works with any standard NMEA 0183 receiver and provides full bidirectional UBX monitoring and configuration for both **u-blox Gen 9/10** (`CFG-VALSET` / `VALGET` / `VALDEL`, e.g. M9, F9, M10) and **u-blox Gen 8 / 7** (legacy `CFG-PRT` / `CFG-MSG` / `CFG-RATE` / `CFG-NAV5` / `CFG-CFG`, e.g. M8N).

---

## Features

- **Single Standalone File (`index.html`)**
  - All HTML, CSS, and JavaScript are inlined with zero external JS/CSS dependencies.
  - Works offline (Position Map automatically falls back to a coordinate grid when offline).
- **Web Serial Connection & Autobaud**
  - Select standard baudrates (`4800` – `921600`) or enter a custom baudrate.
  - **Autobaud** scans common rates (`38400`, `9600`, `115200`, `230400`, `460800`, `57600`, `19200`, `4800`, `921600`) until valid UBX/NMEA frames are detected.
  - Live `DTR` / `RTS` signal controls and automatic USB disconnect handling.
- **u-center-Like Telemetry & Analysis Panels**
  - **Data & Fix Status**: Latitude/Longitude, MSL & ellipsoid altitude, geoid separation, ground speed, heading, NED velocity, `hAcc`/`vAcc`/`sAcc`/`headAcc`, DOPs (`GDOP`/`PDOP`/`HDOP`/`VDOP`/`TDOP`), TTFF, uptime, and `UBX-MON-VER` firmware/hardware identity.
  - **Skyview**: Polar azimuth/elevation plot color-coded by constellation (GPS, Galileo, BeiDou, GLONASS, QZSS, SBAS), Used vs. Tracked indicators, optional sky trails, and hover tooltips.
  - **Signal Level (C/N0)**: Multi-signal bar chart from `UBX-NAV-SIG` (with fallback to `UBX-NAV-SAT` / NMEA `GSV`) and mean C/N0 readout.
  - **Deviation Map**: Local ENU meter scatter plot with `CEP50`, `R95`, max drift, `Lock Ref`, and `Mean Ref`.
  - **Position Map**: Free Google Maps tile layers (**Google Road**, **Google Hybrid**, **Google Satellite**, **Google Terrain**, plus **OpenStreetMap** fallback) with live track polyline, `hAcc` accuracy circle, pan/zoom, and `Follow` mode.
  - **Time-Series Telemetry**: Rolling strip charts for C/N0, `hAcc`/`vAcc`, `PDOP`/`HDOP`, and `SVs Used`/`Tracked`.
  - **RF Monitor & Spectrum Analyzer**:
    - `UBX-MON-RF` (Gen 9/10) & `UBX-MON-HW` (Gen 8): Jamming State, CW Jamming Indicator, AGC Count, Noise/ms, Antenna status/power, I/Q metrics.
    - `UBX-MON-SPAN`: 256-bin RF spectrum analyzer with Max-Hold trace and frequency/power crosshair readout.
- **Dual-Generation UBX Receiver Configuration (`Gen: Auto / Gen 9/10 / Gen 8`)**
  - Auto-detects receiver generation from `UBX-MON-VER` (`PROTVER`) and `ACK-NAK` fallback.
  - Toolbar quick controls: **Poll MON-VER**, **Enable UBX**, **Stop NMEA**, **Save CFG** (`UBX-CFG-CFG` on Gen 8), and **Hot / Warm / Cold Start** (`UBX-CFG-RST`).
  - **Receiver Configuration Center**:
    - Gen 9/10: `UBX-CFG-VALSET` / `VALGET` / `VALDEL` across RAM / BBR / Flash layers (`CFG-RATE-*`, `CFG-NAVSPG-*`, `CFG-PM-OPERATEMODE`, `CFG-SIGNAL-*`, `CFG-UART1-*`, `CFG-MSGOUT-*`, and a Raw 32-bit Key-Value editor).
    - Gen 8 Legacy: `CFG-PRT` (UART1 & USB `portID=3`), `CFG-MSG`, `CFG-RATE`, `CFG-NAV5`, and `CFG-CFG`.
- **Packet Inspector & Serial Console**
  - **Packet Tree**: Decoded UBX & NMEA packet list with field-by-field JSON inspection and raw hex payload.
  - **Text Console**: Live NMEA sentence stream.
  - **Hex View**: Formatted RX/TX hex + ASCII dump.
  - **TX Builder**: Send custom UBX frames (auto-calculates `0xB5 0x62` header and Fletcher-8 checksum) or NMEA sentences (auto-appends `*XX\r\n`).

---

## Requirements & Quick Start

> **Important:** Browser security rules restrict the **Web Serial API** to **Chromium-based browsers** (Chrome, Edge, Opera) running in a **secure context** (`https://` or `http://localhost`). Opening `index.html` directly via `file://` will disable Web Serial.

- **Option A — Online (GitHub Pages):** Open **https://labujuice.github.io/uReader/** directly in Chrome or Edge.
- **Option B — Local Server:**
  1. Start a local static HTTP server in this directory:
     ```bash
     python3 -m http.server 8080
     ```
  2. Open **`http://localhost:8080`** in Chrome or Edge.

Then click **Connect Serial** and select your receiver port (e.g. `/dev/ttyACM0`, `/dev/ttyUSB0`).

### Linux Serial Permissions (`/dev/ttyACM*` / `/dev/ttyUSB*`)

If Chromium cannot open the serial port due to permissions, add your user to the `dialout` group and re-login:

```bash
sudo usermod -aG dialout "$USER"
```

*(On some distributions, check `ls -l /dev/ttyACM0 /dev/ttyUSB0` to confirm the owning group.)*

---

## Supported UBX & NMEA Messages

| Protocol | Messages |
| :--- | :--- |
| **UBX-NAV** | `NAV-PVT` (`0x01 0x07`), `NAV-SAT` (`0x01 0x35`), `NAV-SIG` (`0x01 0x43`), `NAV-DOP` (`0x01 0x04`), `NAV-STATUS` (`0x01 0x03`) |
| **UBX-MON** | `MON-VER` (`0x0A 0x04`), `MON-RF` (`0x0A 0x38`), `MON-SPAN` (`0x0A 0x31`), `MON-HW` (`0x0A 0x09`) |
| **UBX-CFG** | `CFG-VALSET` (`0x06 0x8A`), `CFG-VALGET` (`0x06 0x8B`), `CFG-VALDEL` (`0x06 0x8C`), `CFG-PRT` (`0x06 0x00`), `CFG-MSG` (`0x06 0x01`), `CFG-RST` (`0x06 0x04`), `CFG-RATE` (`0x06 0x08`), `CFG-CFG` (`0x06 0x09`), `CFG-NAV5` (`0x06 0x24`) |
| **UBX-ACK** | `ACK-ACK` (`0x05 0x01`), `ACK-NAK` (`0x05 0x00`) |
| **NMEA 0183** | `GGA`, `RMC`, `GSA`, `GSV`, `GST` (with multi-constellation talker IDs `GP`, `GA`, `GB`/`BD`, `GL`, `GQ`/`QZ`, `GN`) |

---

## Typical Workflow

1. Choose the receiver baudrate (or click **Autobaud** after connecting).
2. On connection, uReader polls `UBX-MON-VER` and `UBX-CFG-PRT` to detect the receiver protocol generation (`Gen 9/10` vs. `Gen 8`).
3. Click **Enable UBX** to turn on binary `NAV-PVT`, `NAV-SAT`, `NAV-DOP`, `NAV-STATUS`, and `MON-RF`/`MON-SPAN` (or `MON-HW` on Gen 8).
4. Optionally click **Stop NMEA** to switch the active port to UBX-only output.
5. On Gen 8 devices, click **Save CFG** to persist settings to BBR/Flash; on Gen 9/10 devices, select `RAM + BBR` or `RAM + BBR + Flash` as the target layer before applying changes.
