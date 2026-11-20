# u-Reader M10 — Web Serial GNSS & u-blox Analyzer

A single-file, zero-dependency, u-center-inspired HTML5 frontend tool for connecting to GNSS receivers (`/dev/ttyACMx`, `/dev/ttyUSBx`, COM ports) directly from a browser via the **Web Serial API**.

Built primarily for **u-blox M10** (Gen 9/10 `CFG-VALSET` interface) with automatic detection and backward-compatible support for **u-blox M8 / M7** (Gen 8 legacy `CFG-PRT` / `CFG-MSG` / `CFG-RATE` / `CFG-NAV5` / `CFG-CFG`), plus standard **NMEA 0183**.

---

## Features

- **Single Standalone File (`index.html`)**
  - All HTML, CSS, and JavaScript are inlined with zero external JS/CSS dependencies.
  - Works completely offline (Position Map falls back to a coordinate grid when offline).
- **Web Serial Connection & Autobaud**
  - Select standard baudrates (`4800` – `921600`) or enter a custom baudrate.
  - **Autobaud** scans common rates (`38400`, `9600`, `115200`, `230400`, `460800`, `57600`, `19200`, `4800`, `921600`) until valid NMEA/UBX frames are detected.
  - Live `DTR` / `RTS` signal toggles and USB disconnect handling.
- **u-center-Like Telemetry & Analysis Panels**
  - **Data & Fix Status**: Lat/Lon, MSL & ellipsoid altitude, geoid separation, ground speed, heading, NED velocity, `hAcc`/`vAcc`/`sAcc`/`headAcc`, DOPs (`GDOP`/`PDOP`/`HDOP`/`VDOP`/`TDOP`), TTFF, uptime, and `UBX-MON-VER` firmware/hardware info.
  - **Skyview**: Polar azimuth/elevation plot color-coded by constellation (GPS, Galileo, BeiDou, GLONASS, QZSS, SBAS), Used vs. Tracked markers, optional sky trails, and hover tooltips.
  - **Signal Level (C/N0)**: Multi-signal bar chart from `UBX-NAV-SIG` (with fallback to `UBX-NAV-SAT` / NMEA `GSV`) and mean C/N0 readout.
  - **Deviation Map**: Local ENU meter scatter plot with `CEP50`, `R95`, max drift, `Lock Ref`, and `Mean Ref`.
  - **Position Map**: Free Google Maps tile layers (**Google Road**, **Google Hybrid**, **Google Satellite**, **Google Terrain**, plus **OpenStreetMap** fallback) with live breadcrumb track, `hAcc` accuracy circle, pan/zoom, and `Follow` mode.
  - **Time-Series Telemetry**: Rolling strip charts for C/N0, `hAcc`/`vAcc`, `PDOP`/`HDOP`, and `SVs Used`/`Tracked`.
  - **RF Monitor & Spectrum Analyzer**:
    - M10 `UBX-MON-RF` & M8 `UBX-MON-HW` (Jamming State, CW Jamming Indicator, AGC Count, Noise/ms, Antenna status/power, I/Q metrics).
    - M10 `UBX-MON-SPAN` 256-bin RF spectrum analyzer with Max-Hold trace and frequency/power crosshair readout.
- **Dual-Generation Receiver Configuration (`Gen: Auto / M10 / M8`)**
  - Auto-detects receiver generation from `UBX-MON-VER` (`PROTVER`) and `ACK-NAK` fallback.
  - Toolbar quick controls: **Poll MON-VER**, **Enable UBX**, **Stop NMEA**, **Save CFG** (`UBX-CFG-CFG` on M8), and **Hot / Warm / Cold Start** (`UBX-CFG-RST`).
  - **M10 Configuration Center**: `UBX-CFG-VALSET` / `VALGET` / `VALDEL` across RAM / BBR / Flash layers (`CFG-RATE-*`, `CFG-NAVSPG-*`, `CFG-PM-OPERATEMODE`, `CFG-SIGNAL-*`, `CFG-UART1-*`, `CFG-MSGOUT-*`, and a Raw 32-bit Key-Value editor).
  - **M8 Legacy Compatibility**: `CFG-PRT` (UART1 & USB `portID=3`), `CFG-MSG`, `CFG-RATE`, `CFG-NAV5`, and `CFG-CFG`.
- **Packet Inspector & Serial Console**
  - **Packet Tree**: Decoded UBX & NMEA packet list with field-by-field JSON inspection and raw hex payload.
  - **Text Console**: Live NMEA sentence stream.
  - **Hex View**: Formatted RX/TX hex + ASCII dump.
  - **TX Builder**: Send custom UBX frames (auto-calculates `0xB5 0x62` header and Fletcher-8 checksum) or NMEA sentences (auto-appends `*XX\r\n`).

---

## Requirements & Quick Start

> **Important:** Browser security rules restrict the **Web Serial API** to **Chromium-based browsers** (Chrome, Edge, Opera) running in a **secure context** (`http://localhost` or `https://`). Opening `index.html` directly via `file://` will disable Web Serial.

1. Start a local static HTTP server in this directory:
   ```bash
   python3 -m http.server 8080
   ```
2. Open **`http://localhost:8080`** in Chrome or Edge.
3. Click **Connect Serial** and select your receiver port (e.g. `/dev/ttyACM0`, `/dev/ttyUSB0`).

### Linux Serial Permissions (`/dev/ttyACM*` / `/dev/ttyUSB*`)

If Chromium cannot open the serial port due to permissions, add your user to the `dialout` group and re-login:

```bash
sudo usermod -aG dialout "$USER"
```

*(On some distributions, also check `ls -l /dev/ttyACM0 /dev/ttyUSB0` to confirm the owning group.)*

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

## Typical Workflow (M10 / M8N)

1. Choose the receiver baudrate (or click **Autobaud** after connecting).
2. On connection, the tool polls `UBX-MON-VER` and `UBX-CFG-PRT` to identify the receiver generation (`M10` vs. `M8`).
3. Click **Enable UBX** to turn on binary `NAV-PVT`, `NAV-SAT`, `NAV-DOP`, `NAV-STATUS`, and `MON-RF`/`MON-SPAN` (or `MON-HW` on M8).
4. Optionally click **Stop NMEA** to switch the active port to UBX-only output.
5. On M8 devices, click **Save CFG** to persist settings to BBR/Flash; on M10 devices, select `RAM + BBR` or `RAM + BBR + Flash` as the target layer before applying changes.
