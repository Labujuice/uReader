# Changelog — uReader

所有人工發布版號定義於 [index.html](index.html) 頂端常數（`APP_VERSION` / `APP_RELEASE_DATE`），並於本檔記錄各版本異動明細。

---

## [v2.0.0] - 2026-11-28

Major feature & production tooling upgrade per [uReader v2.0 PRD](req_doc/ureader_v2_0.md), along with high-rate serial streaming and UI responsiveness fixes.

### Added（新增功能）
- **人工版號管理 (`APP_VERSION = 'v2.0.0'`)**：
  - 於單一核心常數管理版本號，同步顯示於網頁標題、左上角版本徽章與匯出 JSON 組態檔的 `appVersion` 欄位。
  - 新增本更新日誌檔案 `LOG.md`。
- **`Messages` 全量訊息檢視器與雙向拆解器（FR-1）**：
  - 獨立雙欄工作區（左側 35% 訊息樹、右側 65% 解譯檢視與底部 Hex/ASCII），收錄 **7 大分類、86 筆官方 UBX / NMEA 訊息**（含 `UBX-NAV` 23、`UBX-MON` 14、`UBX-RXM` 7、`UBX-TIM` 5、`UBX-INF` 5、`UBX-CFG` 20、`NMEA-Standard` 12）。
  - 支援關鍵字與 `0xCC 0xII` 搜尋、`Active Only` 篩選、即時 RX 閃爍提示、Port-Aware 啟用倍頻徽章與 2 秒滑動視窗 `Live Hz`。
  - 支援單次 `Poll`（含 UBX 查詢幀、`$EIGPQ,{TYPE}`、`CFG-VALGET` 核心 RAM 輪詢與指令型訊息保護）、`NAV Rate`（`0..255` 與 `Off/1/2/5/10` 快捷寫入及回讀驗證）、完整 Declarative Schema 全欄位拆解（含 `reserved*`、Repeated Group、`X1/X2/X4` Bitfield 縮排子列與 `CFG-VALGET`/`VALSET` 動態 KV 拆解）及一鍵 `Copy Hex`。
- **常駐雙階速率控制條 `Rate Control`（FR-2）**：
  - 固定 `Time Ref: UTC`（`timeRef = 0`），提供 `Meas Period (ms)` ↔ `Freq (Hz)` 雙向連動換算、`1/5/10/20/25 Hz` 快捷鍵、`Nav Cycles (1..127)` 與即時有效輸出頻率計算，並支援 `UBX-CFG-RATE`（Legacy M8）與 `CFG-VALSET` RAM（Modern KV）跨世代寫入。
  - 當有效輸出頻率 `>= 10 Hz` 且序列埠鮑率 `<= 38400 bps` 時，顯示低鮑率頻寬超載黃色警示。
- **雙向協定總開關 `[NMEA]` / `[UBX]` 與靜默模式（FR-3）**：
  - 針對目前作用埠（`USB` / `UART1`）執行基礎封包一鍵啟閉，支援 `Active (Custom)` 部分開啟偵測；當 NMEA 與 UBX 週期輸出皆關閉時，狀態列顯示 `🔇 靜默模式（週期輸出已手動關閉）`。
- **一鍵固化 `Save to Flash` 與恢復預設 `Set Default`（FR-4）**：
  - `Save to Flash` 支援 M8 `CFG-CFG` 與 Modern `CFG-VALSET (layers=0x07)`；針對無外掛 Flash 之 ROM 版模組（如 M10 ROM）收到 `ACK-NAK` 時，明確警示並提供降級改寫 `RAM + BBR (0x03)` 確認對話框。
  - `Set Default` 執行前跳出鮑率重置警示對話框，依世代執行 `CFG-CFG` 或 `CFG-VALDEL (0x06)` + `CFG-RST`。
- **全機備份 `Read & Save CFG` 與產線燒錄驗證 `Load & Burn CFG`（FR-5）**：
  - 支援 Modern 分頁掃描 `CFG-VALGET`（`position += 64` 覆蓋全群組）與 Legacy M8 全組態匯出，自動下載 `{MOD}_{PROTVER}_{YYYYMMDD_HHMMSS}.json`（`schemaVersion: "uReader-cfg-v1"`）。
  - 載入燒錄前進行 Pre-Flight 檢查（阻擋跨世代設定檔、警告跨型號並以二分搜尋隔離跳過不支援 Key）；支援無 Flash 自動降級 BBR、主動埠鮑率變更最後排程並自動重連新鮑率、高對比 PASS/FAIL 橫幅與差異比對清單，以及工廠大量燒錄浮動操作列（`⚡ Burn Next Device`）。
- **F11 雙頻與新世代擴充**：
  - `GNSS Signals` 頁籤補入 `GPS L5 (0x10310004)`、`Galileo E5a (0x10310009)`、`BDS B2a (0x10310028)`，支援 `PROTVER >= 27.0` 全系列（M9 / M10 / F9 / F11 / F20）。

### Fixed & Improved（修復與效能最佳化）
- **修復 `/dev/ttyACM0` @ `115200` 高流量連線畫面卡住問題**：
  - 將 `Packet Tree`、`Text Console`、`Hex View`、RF Blocks、`CFG-VALGET` 表格與協定狀態切換改為 dirty-flag 合併節流更新（上限 10 FPS），並移除 `offsetParent` 強制同步 reflow 與熱路徑全幀重組，大幅降低高流量 CPU 負載。
  - `startReadLoop()` 加入 `4096 B` 切片與 `8 ms` 事件迴圈讓出預算，確保高頻或緩衝突發下仍能流暢操作與斷線。
- **修復 `Messages` 左側訊息樹難以點擊切換右側欄的問題**：
  - `renderMsgTree()` 改為結構比對快取與原位 DOM 更新，避免背景重繪於 `mousedown`–`mouseup` 間銷毀節點而吞掉點擊事件；支援 `mousedown` 即時切換右側 Inspector。

---

## [v1.0.0] - 2026-11-20

Initial standalone release (`ce0010e` – `658b453`).

### Added
- 單一零外部相依 `index.html` Web Serial GNSS 分析工具（支援 Chrome / Edge / Opera）。
- Web Serial 連線、`4800–921600` 及自訂鮑率、Autobaud 自動掃描與 DTR/RTS 控制。
- 多視窗儀表板：Overview Data、Skyview（含衛星軌跡與多星系過濾）、C/N0 長條圖、Deviation Map（CEP50/R95）、Position Map（Google Maps / OSM 圖磚與離線網格 fallback）、Time-Series 趨勢圖、RF Monitor (`MON-RF`/`MON-HW`) 與 `MON-SPAN` 頻譜分析。
- Gen 9/10 (`VALSET`/`VALGET`/`VALDEL`) 與 Gen 8 (`CFG-PRT`/`CFG-MSG`/`CFG-RATE`/`CFG-NAV5`/`CFG-CFG`) 基礎設定中心與 Packet / Text / Hex Console。
