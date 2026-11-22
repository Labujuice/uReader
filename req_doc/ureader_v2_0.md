# 產品需求規格書 (PRD)：uReader v2.0 — 全系列指令支援、訊息樹狀總管與量產設定工具

| 項目 | 內容 |
| :--- | :--- |
| **產品名稱** | uReader — Web Serial UBX / NMEA GNSS Analyzer |
| **版本代號** | v2.0 (u-center 進階控制、全平台指令支援與量產配置升級) |
| **支援硬體平台** | u-blox **M8 / M9 / M10 / F9 / F11 / F20** 全系列 GNSS 模組 |
| **文件狀態** | 定稿 (Ready for Coding Agent Implementation) |
| **核心架構紅線** | **單一檔案 (`index.html`)、零外部 JS/CSS 依賴、支援純離線運行** |

---

## 1. 背景與產品目標

### 1.1 現況與痛點
`uReader` 現有版本已具備基礎 NMEA 0183 與 UBX 的連線、遙測面板與基本配置能力。為使其成為可完全替代原廠 `u-center` 並支援實驗室研發與產線快速燒錄的專業工具，本次改版解決以下五大核心缺口：
1. **缺少訊息樹狀總管與全欄位解包**：缺乏類似 `u-center` 的分類樹狀視圖，無法在單一分頁中瀏覽所有 UBX/NMEA 訊息、即時觀察各封包頻率、單次 Poll 或調整 NAV 分頻倍率，且缺少所有解包欄位的完整結構化展示。
2. **雙級採樣調速不直覺**：`Measurement Rate`（量測頻率）與 `Navigation Rate`（解算週期倍率）目前藏在深層設定中，缺乏獨立快捷調速介面與頻寬超載預警。
3. **協定開關為無狀態按鈕**：頂部 `Enable UBX` 與 `Stop NMEA` 無法直覺進行「全關靜默 -> 單點開啟特定封包」的精準除錯動線。
4. **非揮發性記憶體控制不完整**：缺少跨世代一致的 `Save to Flash`（設定固化）與 `Set Default`（恢復出廠預設）按鈕。
5. **缺少量產級設定檔讀寫與回讀驗證**：產線或測試人員無法一鍵將 Golden Sample 的完整設定讀出為檔案，也無法在插上新板子後一鍵載入設定檔、燒錄至 Flash 並自動執行回讀驗證（Read-Back Verification）。

---

## 2. 功能需求規格 (Functional Requirements)

### FR-1：獨立「Message View」樹狀總管與全欄位解包器

#### 2.1 介面佈局與導航
* **獨立分頁**：在主導航列新增獨立分頁 **`Messages`**（與現有 Telemetry、Map、RF Monitor 等分頁並列），避免與即時圖表產生版面擁擠。
* **左右雙欄分割視窗（Split-Pane）**：
  * **左欄（Message Tree，寬度約 35%）**：仿照 `u-center` Messages View 的樹狀階層結構，支援關鍵字搜尋框（依訊息名稱或 Class/ID 過濾）與「僅顯示活躍訊息（Active Only）」過濾開關。
  * **右欄（Message Inspector，寬度約 65%）**：顯示左側當前選取訊息的即時狀態、控制項與**全欄位完整解包表格**。

#### 2.2 左欄：訊息分類樹狀結構（Message Tree）
* **覆蓋範圍（依據附錄 A 全系列官方指令書收錄標準全分類）**：
  * `UBX-NAV`：`PVT`, `POSLLH`, `POSECEF`, `HPPOSLLH`, `HPPOSECEF`, `VELNED`, `VELECEF`, `STATUS`, `DOP`, `SAT`, `SIG`, `TIMEUTC`, `TIMEGPS`, `TIMEBDS`, `TIMEGAL`, `TIMEGLO`, `CLOCK`, `ORB`, `EOE`, `RELPOSNED`, `PL`, `COV`, `SVIN`
  * `UBX-MON`：`VER`, `RF`, `SPAN`, `HW`, `HW2`, `HW3`, `COMMS`, `SYS`, `IO`, `MSGPP`, `RXBUF`, `TXBUF`, `PATCH`, `GNSS`
  * `UBX-RXM`：`RAWX`, `SFRBX`, `RTCM`, `MEASX`, `COR`, `RLM`, `PMP`
  * `UBX-TIM`：`TP`, `TM2`, `VRFY`, `SMEAS`, `VCOCAL`
  * `UBX-INF`：`ERROR`, `WARNING`, `NOTICE`, `TEST`, `DEBUG`
  * `NMEA-Standard`：`GGA`, `RMC`, `GSA`, `GSV`, `GST`, `VTG`, `ZDA`, `GLL`, `GBS`, `GRS`, `DTM`, `TXT`
* **節點即時資訊顯示**：每個訊息節點右側須同步顯示兩個關鍵徽章（Badge）：
  1. **設定倍率（Rate）**：顯示當前設定的 NAV 分頻數（如 `Off`、`1x`、`5x`）。
  2. **實測頻率（Live Hz）**：根據過去 2 秒滑動視窗實際收到的封包數計算（如 `5.0 Hz`、`0.0 Hz`）；收到封包瞬間節點圖示須有短暫閃爍（Blink）反饋。
* **狀態自動同步**：展開分類節點或點擊特定訊息時，系統自動向接收機查詢該訊息在 **Active Port（自動識別 USB 或 UART1）** 的當前輸出倍率（Legacy M8 發送 `CFG-MSG` Poll；Modern M9/M10/F9/F11/F20 發送 `CFG-VALGET` 查詢對應 `CFG-MSGOUT-*` Key）。

#### 2.3 右欄：訊息控制與全欄位完整解包（Full Field Unpacking）
* **頂部控制列**：
  * 顯示訊息名稱、Class/ID（如 `0x01 0x07`）、封包長度、最後更新時間戳（UTC）與累計接收封包數（Count）。
  * **`Poll` 按鈕**：點擊（或雙擊左側樹狀節點）立即發送單次輪詢請求（空 Payload 的 UBX Frame 或 `$EIGPQ` NMEA Poll）。
  * **`NAV Rate` 設定器**：提供數值輸入（`0` ~ `255`）與快捷按鈕（`Off (0)` / `1` / `2` / `5` / `10`），變更後立即下發至 Active Port 並回讀確認。
* **全欄位完整解包表格（關鍵需求）**：
  * **不得省略欄位**：無論該訊息是否有專屬視覺化圖表，右側解包器**必須將官方規格書中的每一個欄位完整展開顯示**（包含保留欄位 `reserved`、重複區塊 `Repeated Group` 與 `X1/X2/X4` Bitfield 旗標）。
  * **表格欄位規範**：
    1. `Offset / Type`：位元組偏移量與資料型別（如 `0 [U4]`、`20 [I4]`、`11 [X1]`）。
    2. `Field Name`：官方規格書欄位名稱（如 `iTOW`, `fixType`, `flags`, `hAcc`）。
    3. `Raw Value`：未縮放的原始十進位或十六進位值。
    4. `Scaled Value & Unit`：套用比例尺後的物理數值與單位（如 `25.0781234 deg`、`12.4 mm`）；若為 Bitfield 旗標，須在該列下方縮排展開各個 Bit 意義（如 `gnssFixOK = 1`, `carrSoln = 2 (Fixed)`）。
  * **底部 Raw Hex 檢視區**：提供最新收到封包的完整 Hex + ASCII 檢視，支援一鍵複製。

---

### FR-2：獨立雙級調速模組（Measurement Rate & Navigation Rate）

* **獨立模組**：將頻率控制從深層配置中獨立出來，設置在**頂部主工具列右側（或專屬常駐控制區 `Rate Control`）**，任何分頁皆可直接操作。
* **時間基準強制統一**：時間基準（Time Reference）統一鎖定為 **`UTC`（`timeRef = 0`）**，UI 顯示唯讀標籤 `Time Ref: UTC`。
* **雙向連動與等效頻率計算**：
  1. **Measurement Rate**：提供 `Period (ms)` 與 `Freq (Hz)` 雙向連動輸入框及常用預設（`1 Hz / 1000 ms`、`5 Hz / 200 ms`、`10 Hz / 100 ms`、`20 Hz / 50 ms`、`25 Hz / 40 ms`）。
  2. **Navigation Rate**：提供 `Nav Cycles` 整數輸入框（`1` ~ `127`，預設 `1`）。
  3. **等效輸出頻率顯示**：即時計算並醒目顯示：
     $$\text{Effective NAV Rate (Hz)} = \frac{1000}{\text{Meas Period (ms)} \times \text{Nav Cycles}}$$
     *範例*：`Meas = 100 ms (10 Hz)` + `Nav Cycles = 2` $\rightarrow$ 顯示 **`輸出頻率：5.0 Hz (每 200 ms 解算一次)`**。
* **跨世代指令映射**：
  * **Legacy (M8)**：發送 `UBX-CFG-RATE (0x06 0x08)`，`measRate = Period`, `navRate = Cycles`, `timeRef = 0 (UTC)`。
  * **Modern (M9/M10/F9/F11/F20)**：發送 `UBX-CFG-VALSET (0x06 0x8A)` 至 RAM 層，寫入 `CFG-RATE-MEAS (0x30210001)`、`CFG-RATE-NAV (0x30210002)` 與 `CFG-RATE-TIMEREF (0x20210003 = 0)`。
* **頻寬超載防呆**：當 `Effective NAV Rate >= 10 Hz` 且當前序列埠鮑率 `<= 38400 bps` 時，顯示**黃色頻寬警告提示**，提醒使用者切換至 `115200+ bps`。

---

### FR-3：頂部工具列 `NMEA` / `UBX` 協定雙向開關

將原本頂部的單次按鈕重構為兩個獨立的 **Toggle Switch**：`[ NMEA ]` 與 `[ UBX ]`，作用於當前 Active Port（USB 或 UART1）：

| 開關操作 | 執行動作 |
| :--- | :--- |
| **`UBX` 切為 `ON`** | 開啟 **UBX 基本套餐**（設為 `Rate = 1`）：<br>• Modern (M9/M10/F9/F11/F20)：`NAV-PVT`, `NAV-SAT`, `NAV-DOP`, `NAV-STATUS`, `MON-RF`, `MON-SPAN`<br>• Legacy (M8)：`NAV-PVT`, `NAV-SAT`, `NAV-DOP`, `NAV-STATUS`, `MON-HW`, `MON-SPAN` |
| **`UBX` 切為 `OFF`** | 將所有已知 UBX 週期性輸出訊息設為 `Rate = 0`。 |
| **`NMEA` 切為 `ON`** | 開啟 **NMEA 基本套餐**（設為 `Rate = 1`）：`GGA`, `RMC`, `GSA`, `GSV` |
| **`NMEA` 切為 `OFF`** | 將所有標準 NMEA 語句（`GGA`, `RMC`, `GSA`, `GSV`, `GST`, `VTG`, `ZDA`, `GLL`）設為 `Rate = 0`。 |

* **靜默模式提示**：當 `NMEA` 與 `UBX` 皆為 `OFF` 時，頂部狀態列顯示 **`🔇 靜默模式（週期輸出已手動關閉）`**，方便使用者前往 `Messages` 分頁進行單點 `Poll` 或單一封包開啟。
* **半開啟狀態偵測**：若在全關後於 `Messages` 分頁單獨開啟某一條 UBX/NMEA 封包，對應 Toggle 自動顯示為 **`Active (Custom)`** 半亮狀態。

---

### FR-4：設定持久化與出廠還原（`Save to Flash` & `Set Default`）

在頂部工具列重啟控制區（與 `Hot` / `Warm` / `Cold Start` 並列）新增兩個跨世代標準化按鈕：

1. **`Save to Flash`（設定固化）**：
   * **Legacy (M8)**：發送 `UBX-CFG-CFG (0x06 0x09)`，`clearMask = 0x00000000`, `saveMask = 0x00001F1F`, `loadMask = 0x00000000`, `deviceMask = 0x17`（BBR + Flash + EEPROM）。
   * **Modern (M9/M10/F9/F11/F20)**：將本次連線中修改過的 Key-Value 暫存表打包為 `UBX-CFG-VALSET (0x06 0x8A)`，設定 `layers = 0x07`（RAM + BBR + Flash）下發。
   * **確認反饋**：必須收到 `ACK-ACK` 才顯示成功提示；若收到 `ACK-NAK`（例如無外掛 Flash 的 ROM 版 M10 模組）則明確警告「寫入 Flash 失敗（模組可能僅支援 BBR）」。
2. **`Set Default`（恢復出廠預設）**：
   * 點擊後跳出**二次確認對話框**（警告通訊鮑率可能重設為出廠預設值 `9600` 或 `38400 bps`）。
   * **Legacy (M8)**：發送 `UBX-CFG-CFG`，`clearMask = 0x00001F1F`, `loadMask = 0x00001F1F`, `deviceMask = 0x17`。
   * **Modern (M9/M10/F9/F11/F20)**：發送 `UBX-CFG-VALDEL (0x06 0x8C)` 清除 `layers = 0x06`（BBR + Flash），並執行 `UBX-CFG-RST`（Controlled Software Reset）使 RAM 同步恢復出廠值。

---

### FR-5：量產級設定讀出匯出與載入燒錄驗證（Setting Read Out & Save / Load & Set）

為滿足研發基準檔保存與產線批量燒錄需求，在 `Configuration Center` 頂部與主工具列新增 **`Read & Save CFG`** 與 **`Load & Burn CFG`** 功能：

#### 5.1 讀取模組設定並存檔（`Setting Read Out & Save to File`）
* **來源層級選擇（Modern 平台）**：點擊 `Read & Save CFG` 時，可選擇讀取 **`RAM`（當前生效設定，預設）** 或 **`Flash / BBR`（已固化設定）**。
* **完整掃描機制**：
  * **Modern (M9/M10/F9/F11/F20)**：使用 `UBX-CFG-VALGET (0x06 0x8B)` 針對所有配置群組（`CFG-UART1`, `CFG-UART2`, `CFG-USB`, `CFG-I2C`, `CFG-SPI`, `CFG-RATE`, `CFG-NAVSPG`, `CFG-NAVHPG`, `CFG-PM`, `CFG-SIGNAL`, `CFG-MSGOUT`, `CFG-TP`, `CFG-ITFM`, `CFG-NMEA`, `CFG-INF`, `CFG-SBAS`, `CFG-QZSS`, `CFG-RTK`, `CFG-TMODE`）執行分頁輪詢（`position` 以 64 筆遞增直到回傳 `ACK-NAK` 結束該群組），完整擷取模組內所有非預設與活躍設定鍵值。
  * **Legacy (M8)**：依序輪詢並擷取 `CFG-PRT`（各 Port）、`CFG-RATE`、`CFG-NAV5`、`CFG-NAVX5`、`CFG-GNSS`、`CFG-TP5`、`CFG-PM2`、`CFG-SBAS`、`CFG-ITFM`、`CFG-NMEA` 以及所有已知訊息的 `CFG-MSG` 原始 Payload。
* **標準化 JSON 設定檔格式**：
  自動觸發瀏覽器下載 `{MOD}_{PROTVER}_{YYYYMMDD_HHMMSS}.json`，格式規範如下：
  ```json
  {
    "schemaVersion": "uReader-cfg-v1",
    "exportedAt": "2026-11-22T09:30:00Z",
    "deviceInfo": {
      "mod": "ZED-F9P",
      "fwVer": "HPG 1.51",
      "protVer": "27.50",
      "generation": "modern-kv"
    },
    "layer": "RAM",
    "kvEntries": [
      { "keyId": "0x30210001", "keyName": "CFG-RATE-MEAS", "type": "U2", "value": 100, "hex": "6400" },
      { "keyId": "0x20910007", "keyName": "CFG-MSGOUT-UBX_NAV_PVT_USB", "type": "U1", "value": 1, "hex": "01" }
    ],
    "legacyFrames": []
  }
  ```

#### 5.2 載入設定檔並燒錄驗證（`Load Setting from File & Set`）
* **前置相容性檢查（Pre-Flight Check）**：
  * 載入 `.json` 檔案後，自動比對檔案內的 `deviceInfo.generation` / `protVer` 與當前連線模組：
    * 若世代不相容（如將 Legacy M8 檔案載入至 F9P），**阻擋執行**並顯示明確錯誤原因。
    * 若同屬 Modern 世代但型號不同（如 F9P 檔案載入至 M10），顯示黃色提示「跨型號載入：不支援的專屬參數將自動跳過」。
* **燒錄選項面板**：
  * **寫入目標層（Target Layer）**：預設勾選 **`RAM + BBR + Flash`（量產固化模式）**，亦可切換為 `RAM Only`（僅測試）。
  * **寫入後自動回讀驗證（Auto Read-Back Verify）**：預設 **`ON`**。
  * **產線連續燒錄模式（Factory Batch Mode）**：開啟後，設定檔會常駐於記憶體並顯示懸浮大按鈕 **`⚡ Burn Next Device（燒錄下一顆）`**，作業員插拔換板後無需重新選檔，一鍵即可完成「寫入 $\rightarrow$ 固化 $\rightarrow$ 回讀驗證」。
* **分批下發與鮑率切換容錯**：
  * Modern 平台將 `kvEntries` 每 **32 筆**打包為一幀 `UBX-CFG-VALSET` 依序下發並等待 `ACK-ACK`，避免單幀過大導致序列埠緩衝區溢位。
  * 若設定檔內包含修改當前連線介面鮑率（如 `CFG-UART1-BAUDRATE` 或 `CFG-PRT`），系統會自動將該指令排在**最後一筆**下發，並在收到 `ACK-ACK` 後自動以新鮑率重連序列埠再執行回讀驗證。
* **量產驗證結果橫幅（Pass / Fail Banner）**：
  * 回讀驗證完成後，在畫面中央顯示高對比結果橫幅：
    * **`✅ PASS — 全部 86 項參數寫入並回讀驗證一致（已固化至 Flash）`**
    * **`❌ FAIL — 2 項參數驗證不符（點擊展開差異清單：Key / 預期值 / 實際回讀值）`**

---

## 3. 非功能性需求 (Non-Functional Requirements)

1. **高頻率 UI 渲染節流（Performance & Throttling）**：
   * 在 `10 Hz` ~ `25 Hz` 高更新率下，底層 Serial Parser 保持全速解析，但 `Messages` 分頁的 DOM 表格與頻率數字渲染須透過 `requestAnimationFrame` 節流（最高 `10 FPS` ~ `15 FPS`），確保連續運行 2 小時不掉幀或記憶體洩漏。
2. **單檔零依賴架構**：
   * 所有全系列指令字典、配置鍵定義與量產驗證邏輯皆須內嵌於單一 `index.html` 中，不可引入外部網路依賴。
3. **通訊埠智慧識別（Port-Aware）**：
   * 連線初期自動識別當前連接的是 `USB` 或 `UART1`，所有 `Toggle` 與 `Message Tree` 頻率調整預設僅作用於當前 Active Port。

---

## 附錄 A：u-blox 全系列（M8 / M9 / M10 / F9 / F11 / F20）官方指令說明書索引與架構對照

Coding Agent 在實作完整的 UBX 訊息字典（Message Schema）與配置鍵（Configuration Keys）時，請嚴格遵循以下官方文件定義與協定版本（`PROTVER`）分界規則：

### A.1 官方介面說明書（Interface Descriptions）清單

| 平台系列 | 代表模組型號 | 韌體與協定版本 (`PROTVER`) | 官方文件編號 | 官方文件名稱與下載連結 |
| :--- | :--- | :--- | :--- | :--- |
| **u-blox 8 / M8** | NEO-M8N, MAX-M8Q, NEO-M8P, ZOE-M8B | SPG 3.01 / HPG 1.43<br>**`PROTVER 15.00 – 23.01`** | **`UBX-13003221`** | *u-blox 8 / u-blox M8 Receiver description including protocol specification*<br>https://www.u-blox.com/docs/UBX-13003221 |
| **u-blox M9** | NEO-M9N, NEO-M9V | SPG 4.04 / MDR 2.16<br>**`PROTVER 32.01 / 35.16`** | **`UBX-21022436`**<br>**`UBX-22037308`** | *u-blox M9 SPG 4.04 Interface description*<br>https://www.u-blox.com/docs/UBX-21022436<br>*u-blox M9 MDR 2.16 Interface description*<br>https://www.u-blox.com/docs/UBX-22037308 |
| **u-blox M10** | MAX-M10S, NEO-M10S, MIA-M10Q | SPG 5.20 / SPG 5.30<br>**`PROTVER 34.20 / 34.30`** | **`UBXDOC-304424225-20395`**<br>**`UBXDOC-304424225-20128`** | *u-blox M10 SPG 5.30 Interface description*<br>https://www.u-blox.com/docs/UBXDOC-304424225-20395<br>*u-blox M10 SPG 5.20 Interface description (PDF)*<br>https://content.u-blox.com/sites/default/files/documents/u-blox-M10-SPG-5.20_InterfaceDescription_UBXDOC-304424225-20128.pdf |
| **u-blox F9** | ZED-F9P, ZED-F9R, ZED-F9H, ZED-F9T | HPG 1.51 / HPS 1.30<br>**`PROTVER 27.50 / 33.30`** | **`UBXDOC-963802114-13124`**<br>**`UBX-22010984`** | *u-blox F9 HPG 1.51 Interface description*<br>https://www.u-blox.com/docs/UBXDOC-963802114-13124<br>*u-blox F9 HPS 1.30 Interface description (PDF)*<br>https://content.u-blox.com/sites/default/files/documents/u-blox-F9-HPS-1.30_InterfaceDescription_UBX-22010984.pdf |
| **u-blox F11** | UBX-F11170 / F11 / M11 系列 | SPG 5.31<br>**`PROTVER 34.31+`** | **`UBXDOC-304424225-21576`** | *u-blox F11/M11 series SPG 5.31 Interface description (PDF)*<br>https://content.u-blox.com/sites/default/files/documents/UBX-F11-M11-series-SPG5.31_InterfaceDescription_UBXDOC-304424225-21576.pdf |
| **u-blox F20 / X20** | ZED-F20P, ZED-X20P | HPG 2.03 / HPG 2.12<br>**`PROTVER 50.10+`** | **`UBXDOC-304424225-21799`**<br>**`UBXDOC-304424225-20728`** | *u-blox X20 HPG 2.12 Interface description (ZED-X20P & ZED-F20P, PDF)*<br>https://content.u-blox.com/sites/default/files/documents/u-blox-X20-HPG-2.12_InterfaceDescription_UBXDOC-304424225-21799.pdf |

---

### A.2 跨世代協定架構分界與實作準則（給 Coding Agent 的核心規則）

1. **自動世代路由（依據 `UBX-MON-VER` 回傳之 `PROTVER`）**：
   * **`PROTVER < 27.0`（u-blox 8 / M8）$\rightarrow$ `Legacy CFG` 引擎**：
     * 全面使用傳統定長/變長配置封包：`CFG-PRT (0x06 0x00)`、`CFG-MSG (0x06 0x01)`、`CFG-RST (0x06 0x04)`、`CFG-RATE (0x06 0x08)`、`CFG-CFG (0x06 0x09)`、`CFG-NAV5 (0x06 0x24)`、`CFG-NAVX5 (0x06 0x23)`、`CFG-GNSS (0x06 0x3E)`、`CFG-TP5 (0x06 0x31)`、`CFG-ITFM (0x06 0x39)`。
     * **注意**：M8 平台**不支援** `CFG-VALSET / VALGET / VALDEL` 與 `NAV-SIG`（僅支援 `NAV-SAT`）及 `MON-RF`（需使用 `MON-HW` 讀取 AGC / Noise / Jamming 指標）。
   * **`PROTVER >= 27.0`（M9 / M10 / F9 / F10 / F11 / F20 / X20）$\rightarrow$ `Modern Key-Value` 引擎**：
     * 全面廢除舊式 `CFG-MSG / CFG-PRT / CFG-RATE`（部分早期 F9/M9 雖有向後相容，但 M10 / F11 / F20 已完全移除舊式 `CFG-*` 指令），統一使用：
       * `UBX-CFG-VALSET (0x06 0x8A)`（寫入 RAM `0x01` / BBR `0x02` / Flash `0x04`）
       * `UBX-CFG-VALGET (0x06 0x8B)`（讀取 RAM `0` / BBR `1` / Flash `2` / Default `7`）
       * `UBX-CFG-VALDEL (0x06 0x8C)`（清除 BBR `0x02` / Flash `0x04`）

2. **各平台專屬擴充指令差異處理**：
   * **M10 / F11 特性**：部分晶片型號（如 `MAX-M10S`）無外掛 Flash，執行 `CFG-VALSET` 帶入 `Flash (0x04)` 位元時會回傳 `ACK-NAK`；實作 `Save to Flash` 或量產燒錄時，若 `layers = 0x07` 收到 `ACK-NAK`，應提示使用者或提供自動降級寫入 `RAM + BBR (layers = 0x03)` 的選項。F11 額外支援雙頻 L1/L5 訊號配置鍵（`CFG-SIGNAL-GPS_L5_ENA`, `CFG-SIGNAL-GAL_E5A_ENA`, `CFG-SIGNAL-BDS_B2A_ENA`）與 LEAP 低功耗模式鍵。
   * **F9 / F20 高精度 RTK 特性**：務必在 `Messages` 樹狀字典中完整收錄高精度專屬封包：`UBX-NAV-HPPOSLLH (0x01 0x14)`、`UBX-NAV-HPPOSECEF (0x01 0x13)`、`UBX-NAV-RELPOSNED (0x01 0x3C)`、`UBX-NAV-SVIN (0x01 0x3B)`、`UBX-NAV-PL (0x01 0x62)`（Protection Level，F20/X20 支援）、`UBX-RXM-RAWX (0x02 0x15)`、`UBX-RXM-SFRBX (0x02 0x13)`、`UBX-RXM-RTCM (0x02 0x32)` 與 `UBX-RXM-COR (0x02 0x34)`。

3. **宣告式欄位解包器（Declarative Schema Decoder）資料型別規範**：
   * Coding Agent 撰寫全系列 UBX 欄位字典時，統一採用 u-blox 官方規格書資料型別標準（Little-Endian）：
     * `U1` / `I1`（1-byte 無號/有號整數）
     * `U2` / `I2`（2-byte 無號/有號整數）
     * `U4` / `I4`（4-byte 無號/有號整數）
     * `R4` / `R8`（IEEE 754 單精度/雙精度浮點數）
     * `X1` / `X2` / `X4`（1/2/4-byte Bitfield 旗標，需附帶 `bits: [{ mask, shift, name, enumMap }]` 定義以自動展開各 Bit 狀態）
     * `CH`（ASCII 字串，如 `MON-VER` 之 `swVersion` / `hwVersion` / `extension`）