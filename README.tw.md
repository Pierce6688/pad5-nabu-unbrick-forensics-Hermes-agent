# 小米平板 5（nabu）14 天變磚鑑識 — 從 20045 錯誤碼到 UFS 實體損壞定案

> **案件期間**：2026-10-18 → 2026-11-02 · 作者：設備持有人（硬體門外漢），全程由 AI 代理（Hermes）主導技術工作
> **最終定案**：UFS 儲存晶片實體損壞（控制器級）。所有軟體救援路徑已窮盡並全程記錄。
> **發布理由**：每一步、每次失敗、每條死路都如實記錄——下一個遇到 Pad 5 變磚的人，能在幾小時內走到正確診斷，而不是幾週。

[English](README.md) | [简体中文](README.zh-cn.md)

---

## 00. TL;DR

一台小米平板 5（nabu，「國際版」）在 OTA 失敗後死亡：開機顯示 **"The system has been destroyed"**，Recovery 同損，僅 Fastboot 存活。14 天裡，持有人（完全的硬體門外漢）在 AI 代理主導技術工作下完成全套鑑識級救援：

1. **帳號側三道牆**：20045（區域不符，無解）→ 重註台灣帳號 → 20041（破解）→ 20038（尋找裝置鎖，破解）→ 20031（實體牆——需要能開機的系統）
2. **官方授權維修中心**：正式送修嘗試刷機——**失敗**（維修記錄可由兄弟公司調閱）
3. **DIY EDL 深刷**：免費進入 EDL（音量上+插線）、patched firehose **通過設備認證**，但 macOS 與 Windows 雙平台的所有批次資料交換全部失敗
4. **最終診斷**：所有工具平台皆 `ufs_open` 失敗 → 熱敷實驗否證 BGA 虛焊假說 → **UFS 晶片實體死亡**

**替你省兩週的教訓**：如果設備能進 EDL（9008）、能過 firehose 認證，但**每次**讀寫都回 `Failed to initialize (open whole lun) UFS Device` / `ufs_open(0, UFS_WLUN_DEVICE) failed`，跨越兩個不同宿主系統與多條線材多個埠——**停止刷機，儲存晶片已死。軟體救不了矽。**

---

## 01. 起源

| 項目 | 內容 |
|---|---|
| 設備 | 小米平板 5，代號 **nabu**，「國際版」機 |
| 銷售宣稱 | 以正 Global 名義販售——**極可能是陸版硬體刷國際 ROM**（見 §05 根因） |
| 序號 | b03\*\*\*\*bb（已遮碼） |
| 故障 | OTA 中斷/失敗後：開機 → "The system has been destroyed"；Recovery → 同錯誤；**Fastboot → 存活且穩定** |
| 故障前 | 正常日常使用，無進水無摔機 |

槽位體檢（fastboot）：

```
slot-retry-count:a = 6      slot-retry-count:b = 7
slot-successful:a  = yes    slot-successful:b  = yes
slot-unbootable:a  = yes    slot-unbootable:b  = no
current-slot = b
```

一次失敗的 OTA 把**兩個槽位全部拖下水**。`fastboot --set-active=a` → `FAILED (remote: 'Slot Change is not allowed in Lock State')`——BL 鎖定下連零風險的切槽都被禁止。唯一出路：解鎖 BL。但設備無法開機 → **死鎖**。

---

## 02. 資訊狀況（四道牆）

小米 BL 解鎖流程（2025+）是四道伺服器端串聯校驗：

```
[設備] ──① 區域──▶ [② 手機號] ──▶ [③ 尋找裝置鎖] ──▶ [④ 系統內綁定] ──▶ 解鎖
          (20045)      (20041)        (20038)           (20031)
```

| # | 錯誤碼 | 含義 | 性質 |
|---|---|---|---|
| ① | 20045 | 帳號區域 ≠ 設備銷售區域 | **硬牆**——區域在帳號註冊瞬間由手號國碼（+86 → CN）寫死，不可自助遷移 |
| ② | 20041 | 帳號未綁手機號 | 配置問題——經 `account.xiaomi.com/pass/serviceLogin?checkSafePhone=true` 補綁 |
| ③ | 20038 | 設備被另一帳號以尋找裝置鎖定 | 雲端——i.mi.com 移除（**注意：「解除裝置綁定」≠「關閉尋找裝置」，兩個獨立開關**） |
| ④ | 20031 | 須在設定→開發者選項內綁定帳號 | **實體**——需要能開機的系統。磚機 = 不可能 |

①②③ 是帳號側（可拆）。④ 是設備側——而我們的設備是磚。

---

## 03. 研判

- **錯誤碼是地圖**。每個碼 = 一道具體的牆。按序拆牆，而不是拿解鎖工具亂撞。
- **區域出生即定**：miunlock 裡換 `dataCenterZone`（新加坡/歐洲）只改請求落點，不改帳號的區域屬性。實證：切區後仍 20045。
- **防盜設計隱含「設備能開機」假設**。磚機過不了第④關——合法持有人被保護自己的系統鎖在門外。
- **「修復」≠「救資料」**：Android 10+ 強制 FBE；userdata 是綁定原機 TEE 的密文。任何刷機路徑（含「保資料刷」）實際結果都是乾淨機或不可用儲存。預期要先擺正。

---

## 04. 行動 A——帳號側三道牆（全部拆除）

### 4.1 牆① 20045——區域（無解，記錄在案）
- Windows 官方 MiFlashUnlock + 原中國區帳號（+86）→ 20045
- macOS miunlock 同帳號 → 伺服器回 `Account Region: CN`
- 手動 `dataCenterZone = Singapore` → **仍 20045**（zone ≠ region）
- **解法**：以 +886 門號重註台灣帳號

### 4.2 牆② 20041——手機綁定
- 新台灣帳號 → `20041 Sorry, your MI ID is not associated with a phone number`
- 經官方 checkSafePhone 連結補綁 +886 → **過**

### 4.3 牆③ 20038——尋找裝置（最精彩的反轉）
- 重跑解鎖：`20038 該手機已被帳號 1*********2 透過查找手機鎖定`
- i.mi.com：「裝置管理」解除綁定**不夠**——尋找裝置清單裡平板還在
- 到「尋找裝置」區塊再移除一次 → 清單驗證乾淨 → **過**

### 4.4 牆④ 20031——實體牆
- `20031 Please add your account in Settings > Developer options > Mi Unlock status`
- 需要系統能開機。系統已毀。**原理性循環死鎖。**

### 4.5 EDL 軟進嘗試（失敗，記錄在案）
- `fastboot oem edl` / `fastboot reboot emergency` → 無回應
- 關機 + 音量下 + 插線 → 裝置管理員無 9008
-（後來免費解決——見 §06.1）

---

## 05. 行動 B——官方授權維修中心

- 設備送**小米官方授權維修中心**嘗試刷機
- 判定：**無法處理**（維修記錄可調閱——兄弟公司）
- 意義：證明官方體系對這類磚機**同樣沒有出口**。「送官方就對了」的常識被此故障模式證偽。
- 變磚根因（購買層）：此「Global 版」極可能是**陸版硬體 + 刷國際 ROM**；OTA 伺服器錯配 → 校驗失敗 → 雙槽全滅。**購機建議：確認原廠出廠區域 = 銷售區域（型號貼紙 + NCC 標記 + 首開語言）。**

---

## 06. 行動 C——DIY EDL 深刷（完全記錄）

### 6.1 免費 EDL 進入法（免拆機、免深刷線）★替所有人省錢的發現

對**BL 鎖定 + 系統死亡 + Fastboot/XBL 存活**的設備：

> **關機 → 按住「音量上」不放 → 插 USB 線 → 保持 3–5 秒 → 鬆手**

- 最早見於 Pad 6（XDA 用戶報告：「音量上不行換音量下，再不行雙鍵同按，總有一個行」）
- **本 Pad 5 實證**：瞬間枚舉為 `QUSB_BULK_CID:0404_SN:6FF5****` / `Qualcomm CDMA Technologies MSM`（idVendor 1478 = Qualcomm，idProduct 0x9008）
- 之前音量下的失敗只是三組合搜尋的一個數據點——**先試音量上**，再試音量下/雙鍵。
- **EDL 會話一次性**：任何工具握手（哪怕失敗的）都會燒掉會話。每次新嘗試前必須斷電重進。

### 6.2 Firehose：patched loader 通過設備認證

| Loader | 大小 | SHA256（前綴） | 來源 |
|---|---|---|---|
| `prog_ufs_firehose_sm7150_ddr.elf`（patched） | 724,800 B | `03a44a6a…` | [Port-Windows-11-Xiaomi-Pad-5 releases](https://github.com/erdilS/Port-Windows-11-Xiaomi-Pad-5/releases) |
| 官方 ROM 內同名（簽名版） | 745,136 B | `60c0ca2c…` | `nabu_global_images_V14.0.8.0.TKXMIXM` 官方包 |

同名 = 同 SoC/儲存目標確認（SM7150 級 = 驍龍 730G…握手時設備回報 **SM8150 系 / 驍龍 860**——Pad 5 實際 SoC）。

Sahara 握手記錄（macOS，bkerler/edl）：

```
sahara - Protocol version: 2, Version supported: 1
CPU detected:      "SM8150"
PK_HASH:           0x1bebe386…
Serial:            0x6ff558a5
sahara - Loader successfully uploaded.
firehose - Xiaomi EDL Auth detected.
firehose - EDL Authenticated successfully.     ★ 認證牆對此 loader 不存在
```

**固件**：官方 Global 穩定版 `nabu_global_images_V14.0.8.0.TKXMIXM`（2024-01，最後一版 Global），3.83 GB，鏡像：`bigota.d.miui.com` / `bn.d.miui.com` / `hugeota.d.miui.com`（路徑規律 `/<MIUI 版本>/<檔名>`）。速度教訓：bigota 在本網路限速 15 KB/s；bn.d 以 aria2 16 連線跑出 27 MB/s（先用 `curl -r 0-8388607 -w "%{speed_download}"` 測速再選鏡像）。

### 6.3 macOS：握手 OK，批次 I/O 全滅——敗因分析

工具：bkerler/edl（V3.62）on macOS 26（Apple Silicon），libusb 1.0.30。

能成的：Sahara 握手、loader 上傳、小米 EDL 認證、小型 XML 交換、收尾的 `setbootablestoragedrive`。

永不成的：任何持續性批次傳輸——GPT 讀取、分區讀取、帶回執等待的寫入。症狀模式：

```
[寫入進度 100%] → DeviceClass - USBError(5, 'Input/Output Error')
               → USBError(19, 'No such device (it may have been disconnected)')
               → 設備重新枚舉 → 下一條指令…
```

一整輪 qfil「跑完」43 個 program 條目 + 86 個 patch，**ACK 確認數 = 0**——資料送出了，從未驗證落盤。（之後設備未開機。）
`--skipresponse` 繞不過握手階段的回執等待；進程零輸出掛起。
GPT 讀取嘗試死在 `cmd_read_buffer` → 讀取中設備掉線。

**結論**：macOS libusb + 此 loader/設備組合無法維持批次 EDL I/O。社群全部成功案例都在 Windows + QDLoader 驅動。

### 6.4 Windows：標準路，以及殺死它的東西

環境：Windows 筆電（**自帶 Qualcomm WWAN 模組——有關聯！**）、MiFlashPatched（2022-05-07 版）、QUD 驅動。

過程中解掉的問題（完整性記錄）：
- `IOException: 請求的資源正在使用中` → **COM 埠被別的軟體占用**。重開機也沒用（常駐服務）。解法：**裝置管理員 → 連接埠 → 9008 → 內容 → 連接埠設定 → 進階 → 改 COM 號碼**（COM4 → COM10），重插讓設備以新埠重新枚舉。
- 驅動：QUD.zip 的 inf 在解壓根層（`qcser.inf` 內含 `USB\VID_05C6&PID_9008`）。安裝：裝置管理員 → 更新驅動 → 瀏覽（⚠️ 勾「包含子資料夾」）→ 接受未簽章警告。或 `pnputil /add-driver C:\qud\qcser.inf /install`。

然後是真正的牆（日誌節錄）：

```
[COM10]:send command:<?xml…><setbootablestoragedrive value="1"/></data>
resdump:<log value="ERROR: Failed to initialize (open whole lun) UFS Device slot 0 partition 1" />
        <log value="ERROR: Failed to open the device:3 slot:0 partition:1 error:0" />
System.Exception: set boot partition failed
```

更早的嘗試在 20–50 秒報 `write time out, maybe devices was disconnected`，**進度條零移動**。同樣簽名出現在：每條線、每個埠（前/後、2.0/3.0）、接/不接電源、滿電狀態、雙系統。

**冒煙的槍**：MiFlash（Windows）與 edl.py（macOS）都到達 loader、都死在**第一個儲存操作**——`ufs_open` / `open whole lun`。快閃控制器從來沒開門。

### 6.5 熱敷實驗（BGA 虛焊假說）——否證

假說：UFS BGA 焊點微裂可致 ufs_open 失敗；受控加熱（約 65–70°C、約 4 分鐘、背蓋上半部主板區——攝影頭所在長邊向內延伸）或可讓焊點重接。

結果：**無變化**——EDL 進入 OK、loader 上傳 OK、刷寫依舊零進度 / 瞬間儲存錯誤。

判讀：以 DIY 可達精度，虛焊已排除。疊加雙系統雙工具死在同一操作 → **UFS 晶片內部故障（控制器級），非連接級**。

---

## 07. 結果——閉環的證據鏈

| 證據 | 排除了 |
|---|---|
| 雙系統握手 + 認證全過 | 線材、USB 棧、工具配置錯誤 |
| 雙系統死在同一操作（ufs_open / whole-lun init） | 宿主平台差異 |
| 官方授權中心也失敗 | 授權/帳號牆 |
| 熱敷實驗無變化 | BGA 虛焊（DIY 級） |
| OTA 事件前一切正常 | 先天缺陷 |
| 一次中斷的 OTA 先於變磚 | **OTA 事件實體損壞了 UFS（電壓/控制器級）** |

**定案**：UFS 儲存實體損壞。修復 = 換 UFS 晶片或換主板——對本機殘值（可用狀態約 NT$4–5 千）不具經濟性。處置：以零件/移植機身分售予刷機社群（fastboot 存活、無帳號鎖、附完整案件檔案）。

---

## 08. 教訓（給後來者）

1. **操作順序**：先查 BL 狀態（`fastboot oem device-info`）再做刷機夢；鎖定設備拒絕一切寫入。
2. **錯誤碼是地圖**——20045/20041/20038/20031 各指一道具體的牆；按序拆除。
3. **i.mi.com 的「解除裝置」≠「關閉尋找裝置」**——20038 要關的是後者。
4. **帳號區域由手號國碼出生即定**；切 zone 只是妝。
5. **免費 EDL 進入**：按住音量上 + 插線（適用鎖 BL、系統死、XBL 活的設備）。免深刷線、免拆機。會話一次性。
6. **Patched firehose 能過小米 EDL 認證**（此設備類）——人們怕的認證牆不是攔路虎。
7. **macOS + libusb + EDL 批次 I/O = 痛**。Windows + QDLoader 是社群實證路。
8. **先診斷再硬幹**：兩個系統、多條線多個埠全死在 `ufs_open`/whole-lun init 且零進度——**儲存已死**；停止燃燒會話。
9. **「修好」≠「救回資料」**：FBE 把 userdata 綁死在設備 TEE。現實的修復產出的是乾淨機，不是你的資料。
10. **購機建議**：跨區小米設備，驗「原廠出廠區域 = 銷售區域」（型號尾碼、NCC/官方標記、首開語言）。「陸版硬體刷 Global ROM」是一顆等著變磚的種子。

---

## 09. 附錄

### A. 錯誤碼速查（本案例實證 + 社群）

| 錯誤碼 | 訊息 | 牆 | 處理 |
|---|---|---|---|
| 20045 | 帳號區域 ≠ 銷售區域 | 區域 | 同區帳號（磚機死路） |
| 20041 | 未綁手機號 | 手機號 | checkSafePhone 連結補綁 |
| 20038 | 被其他帳號的尋找裝置鎖定 | 防盜 | i.mi.com → 尋找裝置 → 移除 |
| 20031/30002 | 開發者選項內綁定帳號 | 系統內 | 需可開機（磚機死路） |
| 10013 | 設備未啟用/未滿 7 天 | 啟用 | SIM 啟用 + 等待 |
| 20030 | 本月 token 額度 | 配額 | 等 1 號 |
| 10023 | 一帳號一年一機 | 配額 | 換帳號或等待 |

### B. 指令記錄

```bash
# 狀態
fastboot devices && fastboot oem device-info && fastboot getvar anti
fastboot getvar slot-unbootable:a ; fastboot --set-active=a   # → FAILED (Lock State)

# EDL 軟進（鎖定 nabu 無效）
fastboot oem edl ; fastboot reboot emergency

# macOS 工具
brew install android-platform-tools libusb aria2
uv tool install miunlock        # v1.7.4
git clone https://github.com/bkerler/edl && cd edl && python3 -m venv venv
venv/bin/pip install pyusb pyserial docopt colorama …（repo requirements，剔除錯誤的 requests>=2.34.2 釘選）

# 有效的 EDL 進入（音量上+插線），macOS 驗證：
ioreg -p IOUSB -l -w 0 | grep -iE '9008|QUSB'

# 固件直鏈（三鏡像同路徑）
https://bn.d.miui.com/V14.0.8.0.TKXMIXM/nabu_global_images_V14.0.8.0.TKXMIXM_20240112.0000.00_13.0_global_b27365260e.tgz
aria2c -x16 -s16 -k4M -o <file>.tgz <url> ; gzip -t <file>.tgz
```

### C. 測試點（若音量上進入法失效）

主板中央偏右外露窄條上的單顆鍍金圓點，正上方緊鄰絲印 **M514m/MAPEC** 的橙色 FPC，位於兩片銀色屏蔽罩之間——用迴紋針一端搭測試點、一端搭就近屏蔽罩（接地），插線瞬間短接。XDA 帖 4553627 有迴紋針實證。照片：[`nabu_EDL_testpoint_egsm.jpg`](nabu_EDL_testpoint_egsm.jpg)（圖源：維修社群圖庫站）。

### D. Windows 側快速卡

```
1. 解壓 MiFlashPatched → 以系統管理員執行 XiaoMiFlash.exe
2. 解壓 QUD → 裝置管理員 → 9008/QUSB_BULK_CID 帶⚠ → 更新驅動 → 瀏覽 QUD 資料夾（含子資料夾）→ 接受未簽章
3. 把 patched prog_ufs_firehose_sm7150_ddr.elf 覆蓋 ROM 的 images/prog_ufs_firehose_sm7150_ddr.elf
4. Select → ROM 資料夾（含 flash_all.bat 那層，不是 images/）
5. 右下單選："clean all"  ← 絕不選 "clean all and lock"
6. 遇「資源使用中」：連接埠設定→進階→改 COM 號碼，重插
7. Refresh → 設備列出現 → Flash → 期待 success 或本文記錄的 ufs_open 錯誤
```

### E. 全案原始文件（中文）

- [`reference/case_draft.md`](reference/case_draft.md)——v2 案件檔（日期、價格、維修中心記錄、逐牆細節）
- [`reference/edl_runbook.md`](reference/edl_runbook.md)——硬體定案前的 EDL runbook

---

## 10. AI 代理在本案的角色 ★

本案由**硬體門外漢**（設備持有人）與 **AI 代理（Hermes，CLI 形態的 AI 助理）**協作完成。分工方式才是可複製的資產：

| 任務 | 誰 | 怎麼做 |
|---|---|---|
| 工具發掘與選型（bkerler/edl、patched firehose、官方固件源、鏡像測速） | **代理** | 網路調研、源碼審讀、SHA256 驗證、aria2 調參（15 KB/s → 27 MB/s） |
| 把每個錯誤碼翻譯成「一道牆 + 拆牆計畫」 | **代理** | XDA/GitHub/論壇交叉比對 |
| 讀 edl-client 源碼預判行為（sparse 自動解包、set-bootable 邏輯、skipresponse 語義） | **代理** | 實打實的代碼審計，不靠猜 |
| 實體動作（按鍵組合、線材、拆機決策、送修） | **持有人** | 代理發步驟卡帶回報檢查點；持有人執行並回報 |
| 工具鏈運維（克隆、venv、下載、雜湊校驗、刷寫執行、日誌捕獲） | **代理** | 本機 shell、後台進程輪詢 |
| 即時日誌鑑識（發現零 ACK 寫入、USBError 5→19 模式、ufs_open 簽名） | **代理** | 每輪嘗試後對捕獲日誌做 grep/awk |
| 跨平台診斷（macOS → Windows 轉進、COM 埠占用修復、QUD 驅動 inf 追蹤） | **代理** | 失敗模式比對社群知識庫 |
| 判斷決策（何時停止刷機、熱敷實驗設計、最終定案、處置選項） | **代理提案 → 持有人拍板** | 每個分支決策點 |
| 成本/效益與殘值調研（維修 vs 零件 vs 賣社群） | **代理** | 即時市場快照 + 明確的「快照 ≠ 即時」標註 |

**代理做不到的**（誠實邊界）：實體按按鍵、跑維修中心、以觸感判斷加熱溫度、給設備斷電重啟。每個實體步驟都是一張帶回報檢查點的編號卡：*「做 X → 告訴我你看到什麼」*。持有人唯一需要的技能是照卡執行並描述螢幕上出現的東西——**零硬體先備知識**。

**為何公布 AI 角色**：本案可複製的資產不是「我們刷了一台 Pad 5」——而是**工作流本身**：AI 代理做調研、工具鏈、日誌鑑識、決策框架，門外漢以檢查點紀律執行實體步驟。這個模式適用於任何 Qualcomm 設備的 EDL-9008 級救援。

---

## 致謝與參考

- bkerler/edl——開源 EDL 工具鏈
- erdilS/Port-Windows-11-Xiaomi-Pad-5——patched firehose 與 EDL 指南
- XDA 帖：Mi Pad 5 EDL Test Point（#4553627）、Pad 6 音量上進 EDL（#4744305）、ufs_open 失敗案例（Mi 5 / Mi 8 / Mi 9T Pro / LG V30——同簽名）
- 維修社群圖庫站——測試點攝影
- 公開固件索引站——固件索引

*以上所有錯誤碼、價格、門店記錄、固件雜湊與實驗結果均為 2026-10-18 → 2026-11-02 一手實測資料。*
