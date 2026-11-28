# uReader — Web Serial UBX / NMEA GNSS Analyzer

**Version:** `v2.0.0` ([Release Log / 更新日誌](LOG.md))  
**Live Page:** https://labujuice.github.io/uReader/

**uReader** is a single-file, zero-dependency HTML5 frontend tool for reading, analyzing, and debugging **UBX** and **NMEA 0183** GNSS serial streams (`/dev/ttyACMx`, `/dev/ttyUSBx`, COM ports) directly in the browser via the **Web Serial API**.

Inspired by u-blox u-center, it works with any standard NMEA 0183 receiver and provides full bidirectional UBX monitoring, full-field message inspection, rate control, and production configuration backup/burn/verify for both **Modern KV (`PROTVER >= 27.0`: M9 / M10 / F9 / F11 / F20)** and **Legacy CFG (`PROTVER < 27.0`: u-blox 8 / M8)** receivers.

---

## Features

- **Single Standalone File (`index.html`)**
  - All HTML, CSS, and JavaScript are inlined with zero external JS/CSS dependencies.
  - Works completely offline (Position Map falls back to an offline coordinate grid when tiles are unreachable).
- **Messages Explorer & Full-Field Unpacker (`Messages` Workspace)**
  - Covers **7 categories / 86 official messages** (`UBX-NAV`, `UBX-MON`, `UBX-RXM`, `UBX-TIM`, `UBX-INF`, `UBX-CFG`, and `NMEA-Standard`).
  - Keyword & hex Class/ID search, `Active Only` filtering, RX blink indicator, active-port `Rate` badge, and 2-second sliding-window `Live Hz` display.
  - Unpacks every schema field (including reserved fields, `X1/X2/X4` bitfield sub-rows, repeated blocks, and `CFG-VALGET`/`CFG-VALSET` KV entries) plus Raw Hex/ASCII inspection, 1-click Copy Hex, Poll (`UBX` / `$EIGPQ`), and per-message `NAV Rate` configuration.
- **Persistent Rate Control & Bidirectional Protocol Switches**
  - Toolbar **Rate Control** strip with `Time Ref: UTC`, bidirectional `Meas Period (ms)` ↔ `Freq (Hz)` conversion, quick presets (`1/5/10/20/25 Hz`), `Nav Cycles (1..127)`, effective output rate readout, and automatic low-baudrate bandwidth overload warning.
  - Bidirectional `[NMEA]` and `[UBX]` port-aware switches with `Active (Custom)` partial state detection and `🔇 靜默模式` indicator.
- **Persistence, Production Backup & Batch Burn/Verify**
  - **Save to Flash** (M8 `CFG-CFG` / Modern `CFG-VALSET 0x07`, with ROM-only `ACK-NAK` BBR `0x03` fallback prompt) and **Set Default** (with baudrate-reset warning modal).
  - **Read & Save CFG**: Paginated full-receiver KV / M8 configuration export (`{MOD}_{PROTVER}_{YYYYMMDD_HHMMSS}.json`, `schemaVersion: "uReader-cfg-v1"`).
  - **Load & Burn CFG**: Pre-flight generation guard & cross-model bisect/skip, automatic baud-change last-scheduling & serial reconnect, read-back PASS/FAIL diff banner, and Factory Batch Mode (`⚡ Burn Next Device`).
- **Web Serial Connection & Autobaud**
  - Select standard baudrates (`4800` – `921600`) or enter a custom baudrate, plus `USB` / `UART1` active-port auto-detection and `Autobaud` scanning.
  - Throttled UI pipeline & sliced read-loop yielding keep the UI responsive even under high-rate `/dev/ttyACM0` streams.
- **u-center-Like Telemetry & Analysis Panels**
  - **Data & Fix Status**, **Skyview** (constellation filters & trails), **Signal Level (C/N0)**, **Deviation Map** (`CEP50`/`R95`), **Position Map** (Google Maps / OSM + offline grid), **Time-Series Telemetry**, **RF Monitor & `MON-SPAN` Spectrum Analyzer**, **Config Center** (incl. F11 dual-band L5/E5a/B2a signals), and **Packet / Text / Hex Console & TX Builder**.

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

## Supported UBX & NMEA Messages (7 Categories · 86 Messages)

| Category | Coverage |
| :--- | :--- |
| **UBX-NAV** (23) | `PVT`, `POSLLH`, `POSECEF`, `HPPOSLLH`, `HPPOSECEF`, `VELNED`, `VELECEF`, `STATUS`, `DOP`, `SAT`, `SIG`, `TIMEUTC`, `TIMEGPS`, `TIMEBDS`, `TIMEGAL`, `TIMEGLO`, `CLOCK`, `ORB`, `EOE`, `RELPOSNED`, `PL`, `COV`, `SVIN` |
| **UBX-MON** (14) | `VER`, `RF`, `SPAN`, `HW`, `HW2`, `HW3`, `COMMS`, `SYS`, `IO`, `MSGPP`, `RXBUF`, `TXBUF`, `PATCH`, `GNSS` |
| **UBX-RXM** (7) | `RAWX`, `SFRBX`, `RTCM`, `MEASX`, `COR`, `RLM`, `PMP` |
| **UBX-TIM** (5) | `TP`, `TM2`, `VRFY`, `SMEAS`, `VCOCAL` |
| **UBX-INF** (5) | `ERROR`, `WARNING`, `NOTICE`, `TEST`, `DEBUG` |
| **UBX-CFG** (20) | `PRT`, `MSG`, `INF`, `RST`, `RATE`, `CFG`, `ANT`, `SBAS`, `NMEA`, `ODO`, `NAVX5`, `NAV5`, `TP5`, `ITFM`, `PM2`, `GNSS`, `TMODE3`, `VALSET`, `VALGET`, `VALDEL` |
| **NMEA-Standard** (12) | `GGA`, `RMC`, `GSA`, `GSV`, `GST`, `VTG`, `ZDA`, `GLL`, `GBS`, `GRS`, `DTM`, `TXT` |

*(Plus `UBX-ACK-ACK` / `ACK-NAK` status decoding.)*

---

## Typical Workflow

1. Choose the receiver baudrate (or click **Autobaud** after connecting).
2. On connection, uReader queries `UBX-MON-VER` (and `CFG-PRT` on M8) plus USB/UART1 detection to set **Gen** (`Modern KV` vs `Legacy M8`) and **Port** (`USB` vs `UART1`) automatically.
3. Use the **Rate Control** strip to adjust measurement rate (`1–25+ Hz`) and nav cycles, or toggle **`[NMEA]`** / **`[UBX]`** on the active port.
4. Open the **Messages** tab to inspect all 86 messages with full field unpacking, live Hz, Poll (`UBX`/`$EIGPQ`), and per-message `NAV Rate`.
5. Click **Save to Flash** (falls back to `RAM+BBR` on ROM-only modules) or use **Read & Save CFG** / **Load & Burn CFG** for full-receiver backup and verified batch flashing.
