# 小米平板 5（nabu）14 天变砖鉴定 — 从 20045 错误码到 UFS 实体损坏定案

> **案件期间**：2026-10-18 → 2026-11-02 · 作者：设备持有人（硬件门外汉），全程由 AI 代理（Hermes）主导技术工作
> **最终定案**：UFS 储存芯片实体损坏（控制器级）。所有软件救援路径已穷尽并全程记录。
> **发布理由**：每一步、每次失败、每条死路都如实记录——下一个遇到 Pad 5 变砖的人，能在几小时内走到正确诊断，而不是几周。

[English](README.md) | [简体中文](README.zh-cn.md)

---

## 00. TL;DR

一台小米平板 5（nabu，「国际版」）在 OTA 失败后死亡：开机显示 **"The system has been destroyed"**，Recovery 同损，仅 Fastboot 存活。14 天里，持有人（完全的硬件门外汉）在 AI 代理主导技术工作下完成全套鉴识级救援：

1. **账号侧三道墙**：20045（区域不符，无解）→ 重注册台湾账号 → 20041（破解）→ 20038（查找设备锁，破解）→ 20031（实体墙——需要能开机的系统）
2. **官方授权维修中心**：正式送修尝试刷机——**失败**（维修记录可由兄弟公司调阅）
3. **DIY EDL 深刷**：免费进入 EDL（音量上+插线）、patched firehose **通过设备认证**，但 macOS 与 Windows 雙平台的所有批量资料交换全部失败
4. **最终诊断**：所有工具平台皆 `ufs_open` 失败 → 热敷实验否证 BGA 虚焊假说 → **UFS 芯片实体死亡**

**替你省两周的教训**：如果设备能进 EDL（9008）、能过 firehose 认证，但**每次**读写都回 `Failed to initialize (open whole lun) UFS Device` / `ufs_open(0, UFS_WLUN_DEVICE) failed`，跨越两个不同宿主系统与多条线材多个端口——**停止刷机，储存芯片已死。软件救不了硅。**

---

## 01. 起源

| 项目 | 内容 |
|---|---|
| 设备 | 小米平板 5，代号 **nabu**，「国际版」机 |
| 购入 ||
| 序列号 | b03\*\*\*\*bb（已脱敏） |
| 故障 | OTA 中断/失败后：开机 → "The system has been destroyed"；Recovery → 同错误；**Fastboot → 存活且稳定** |
| 故障前 | 正常日常使用，无进水无摔机 |

槽位体检（fastboot）：

```
slot-retry-count:a = 6      slot-retry-count:b = 7
slot-successful:a  = yes    slot-successful:b  = yes
slot-unbootable:a  = yes    slot-unbootable:b  = no
current-slot = b
```

一次失败的 OTA 把**两个槽位全部拖下水**。`fastboot --set-active=a` → `FAILED (remote: 'Slot Change is not allowed in Lock State')`——BL 锁定下连零风险的切槽都被禁止。唯一出路：解锁 BL。但设备无法开机 → **死锁**。

---

## 02. 信息状况（四道墙）

小米 BL 解锁流程（2025+）是四道服务器端串联校验：

```
[设备] ──① 区域──▶ [② 手机号] ──▶ [③ 查找设备锁] ──▶ [④ 系统内绑定] ──▶ 解锁
          (20045)      (20041)        (20038)           (20031)
```

| # | 错误码 | 含义 | 性质 |
|---|---|---|---|
| ① | 20045 | 账号区域 ≠ 设备销售区域 | **硬墙**——区域在账号注册瞬间由手号国码（+86 → CN）写死，不可自助迁移 |
| ② | 20041 | 账号未绑手机号 | 配置问题——经 `account.xiaomi.com/pass/serviceLogin?checkSafePhone=true` 补绑 |
| ③ | 20038 | 设备被另一账号以查找设备锁定 | 云端——i.mi.com 移除（**注意：「解除设备绑定」≠「关闭查找设备」，两个独立开关**） |
| ④ | 20031 | 须在设置→开发者选项内绑定账号 | **实体**——需要能开机的系统。砖机 = 不可能 |

①②③ 是账号侧（可拆）。④ 是设备侧——而我们的设备是砖。

---

## 03. 研判

- **错误码是地图**。每个码 = 一道具体的墙。按序拆墙，而不是拿解锁工具乱撞。
- **区域出生即定**：miunlock 里换 `dataCenterZone`（新加坡/欧洲）只改请求落点，不改账号的区域属性。实证：切区后仍 20045。
- **防盗设计隐含「设备能开机」假设**。砖机过不了第④关——合法持有人被保护自己的系统锁在门外。
- **「修复」≠「救资料」**：Android 10+ 强制 FBE；userdata 是绑定原机 TEE 的密文。任何刷机路径（含「保资料刷」）实際结果都是干净机或不可用储存。预期要先摆正。

---

## 04. 行动 A——账号侧三道墙（全部拆除）

### 4.1 墙① 20045——区域（无解，记录在案）
- Windows 官方 MiFlashUnlock + 原中國区账号（+86）→ 20045
- macOS miunlock 同账号 → 服务器回 `Account Region: CN`
- 手动 `dataCenterZone = Singapore` → **仍 20045**（zone ≠ region）
- **解法**：以 +886 门号重注册台湾账号

### 4.2 墙② 20041——手机绑定
- 新台湾账号 → `20041 Sorry, your MI ID is not associated with a phone number`
- 经官方 checkSafePhone 链接补绑 +886 → **过**

### 4.3 墙③ 20038——查找设备（最精彩的反转）
- 重跑解锁：`20038 该手机已被账号 1*********2 透过查找手机锁定`
- i.mi.com：「设备管理」解除绑定**不够**——查找设备清单里平板还在
- 到「查找设备」区块再移除一次 → 清单验证乾淨 → **过**

### 4.4 墙④ 20031——实体墙
- `20031 Please add your account in Settings > Developer options > Mi Unlock status`
- 需要系统能开机。系统已毁。**原理性循环死锁。**

### 4.5 EDL 軟进尝试（失败，记录在案）
- `fastboot oem edl` / `fastboot reboot emergency` → 无回应
- 关机 + 音量下 + 插线 → 设备管理器无 9008
-（后来免费解决——見 §06.1）

---

## 05. 行动 B——官方授权维修中心

- 设备送**小米官方授权维修中心**尝试刷机
- 判定：**无法处理**（维修记录可调阅——兄弟公司）
- 意义：证明官方体系对这类砖机**同样没有出口**。「送官方就对了」的常识被此故障模式证伪。
- 变砖根因（购买层）：此「Global 版」极可能是**陆版硬件 + 刷国际 ROM**；OTA 服务器错配 → 校验失败 → 雙槽全灭。**购机建议：确认原厂出厂区域 = 销售区域（型号貼紙 + NCC 标记 + 首开语言）。**

---

## 06. 行动 C——DIY EDL 深刷（完全记录）

### 6.1 免费 EDL 进入法（免拆机、免深刷线）★替所有人省錢的发现

对**BL 锁定 + 系统死亡 + Fastboot/XBL 存活**的设备：

> **关机 → 按住「音量上」不放 → 插 USB 线 → 保持 3–5 秒 → 松手**

- 最早见于 Pad 6（XDA 用户报告：「音量上不行换音量下，再不行双键同按，总有一个行」）
- **本 Pad 5 实证**：瞬间枚举为 `QUSB_BULK_CID:0404_SN:6FF5****` / `Qualcomm CDMA Technologies MSM`（idVendor 1478 = Qualcomm，idProduct 0x9008）
- 之前音量下的失败只是三组合搜尋的一个数据点——**先試音量上**，再試音量下/双键。
- **EDL 会话一次性**：任何工具握手（哪怕失败的）都会烧掉会话。每次新尝试前必须断电重进。

### 6.2 Firehose：patched loader 通过设备认证

| Loader | 大小 | SHA256（前缀） | 来源 |
|---|---|---|---|
| `prog_ufs_firehose_sm7150_ddr.elf`（patched） | 724,800 B | `03a44a6a…` | [Port-Windows-11-Xiaomi-Pad-5 releases](https://github.com/erdilS/Port-Windows-11-Xiaomi-Pad-5/releases) |
| 官方 ROM 内同名（签名版） | 745,136 B | `60c0ca2c…` | `nabu_global_images_V14.0.8.0.TKXMIXM` 官方包 |

同名 = 同 SoC/储存目标确认（SM7150 级 = 驍龍 730G…握手时设备回报 **SM8150 系 / 驍龍 860**——Pad 5 实際 SoC）。

Sahara 握手记录（macOS，bkerler/edl）：

```
sahara - Protocol version: 2, Version supported: 1
CPU detected:      "SM8150"
PK_HASH:           0x1bebe386…
Serial:            0x6ff558a5
sahara - Loader successfully uploaded.
firehose - Xiaomi EDL Auth detected.
firehose - EDL Authenticated successfully.     ★ 认证墙对此 loader 不存在
```

**固件**：官方 Global 稳定版 `nabu_global_images_V14.0.8.0.TKXMIXM`（2024-01，最后一版 Global），3.83 GB，镜像：`bigota.d.miui.com` / `bn.d.miui.com` / `hugeota.d.miui.com`（路径規律 `/<MIUI 版本>/<文件名>`）。速度教训：bigota 在本网络限速 15 KB/s；bn.d 以 aria2 16 连线跑出 27 MB/s（先用 `curl -r 0-8388607 -w "%{speed_download}"` 测速再选镜像）。

### 6.3 macOS：握手 OK，批量 I/O 全灭——败因分析

工具：bkerler/edl（V3.62）on macOS 26（Apple Silicon），libusb 1.0.30。

能成的：Sahara 握手、loader 上传、小米 EDL 认证、小型 XML 交换、收尾的 `setbootablestoragedrive`。

永不成的：任何持续性批量传输——GPT 读取、分区读取、带回执等待的写入。症状模式：

```
[写入进度 100%] → DeviceClass - USBError(5, 'Input/Output Error')
               → USBError(19, 'No such device (it may have been disconnected)')
               → 设备重新枚举 → 下一条指令…
```

一整轮 qfil「跑完」43 个 program 条目 + 86 个 patch，**ACK 确认数 = 0**——资料送出了，从未验证落盘。（之后设备未开机。）
`--skipresponse` 绕不过握手阶段的回执等待；进程零输出挂起。
GPT 读取尝试死在 `cmd_read_buffer` → 读取中设备掉线。

**结论**：macOS libusb + 此 loader/设备组合无法維持批量 EDL I/O。社区全部成功案例都在 Windows + QDLoader 驱动。

### 6.4 Windows：标准路，以及杀死它的东西

环境：Windows 笔记本（**自带 Qualcomm WWAN 模块——有关联！**）、MiFlashPatched（2022-05-07 版）、QUD 驱动。

过程中解掉的问题（完整性记录）：
- `IOException: 请求的资源正在使用中` → **COM 端口被别的软件占用**。重启也没用（常驻服务）。解法：**设备管理器 → 端口 → 9008 → 属性 → 端口设置 → 高级 → 改 COM 号**（COM4 → COM10），重插让设备以新端口重新枚举。
- 驱动：QUD.zip 的 inf 在解压根层（`qcser.inf` 内含 `USB\VID_05C6&PID_9008`）。安装：设备管理器 → 更新驱动 → 浏览（⚠️ 勾「包含子文件夹」）→ 接受未签名警告。或 `pnputil /add-driver C:\qud\qcser.inf /install`。

然后是真正的墙（日誌节录）：

```
[COM10]:send command:<?xml…><setbootablestoragedrive value="1"/></data>
resdump:<log value="ERROR: Failed to initialize (open whole lun) UFS Device slot 0 partition 1" />
        <log value="ERROR: Failed to open the device:3 slot:0 partition:1 error:0" />
System.Exception: set boot partition failed
```

更早的尝试在 20–50 秒报 `write time out, maybe devices was disconnected`，**进度条零移动**。同样签名出现在：每条线、每个端口（前/后、2.0/3.0）、接/不接电源、满电状态、双系统。

**冒烟的枪**：MiFlash（Windows）与 edl.py（macOS）都到达 loader、都死在**第一个储存操作**——`ufs_open` / `open whole lun`。闪存控制器从来没开门。

### 6.5 热敷实验（BGA 虚焊假说）——否证

假说：UFS BGA 焊点微裂可致 ufs_open 失败；受控加热（约 65–70°C、约 4 分钟、背盖上半部主板区——摄像头所在长边向内延伸）或可让焊点重接。

结果：**无变化**——EDL 进入 OK、loader 上传 OK、刷写依旧零进度 / 瞬间储存错误。

判读：以 DIY 可达精度，虚焊已排除。叠加双系统双工具死在同一操作 → **UFS 芯片内部故障（控制器级），非连接级**。

---

## 07. 结果——闭环的证据链

| 证据 | 排除了 |
|---|---|
| 双系统握手 + 认证全过 | 线材、USB 栈、工具配置错误 |
| 双系统死在同一操作（ufs_open / whole-lun init） | 宿主平台差异 |
| 官方授权中心也失败 | 授权/账号墙 |
| 热敷实验无变化 | BGA 虚焊（DIY 级） |
| OTA 事件前一切正常 | 先天缺陷 |
| 一次中断的 OTA 先于变砖 | **OTA 事件实体损坏了 UFS（電压/控制器级）** |

**定案**：UFS 储存实体损坏。修复 = 换 UFS 芯片或换主板——对本机残值（可用状态约 NT$4–5 千）不具经济性。处置：以零件/移植机身份售予刷机社区（fastboot 存活、无账号锁、附完整案件档案）。

---

## 08. 教训（给后来者）

1. **操作顺序**：先查 BL 状态（`fastboot oem device-info`）再做刷机梦；锁定设备拒绝一切写入。
2. **错误码是地圖**——20045/20041/20038/20031 各指一道具体的墙；按序拆除。
3. **i.mi.com 的「解除设备」≠「关闭查找设备」**——20038 要关的是后者。
4. **账号区域由手号国码出生即定**；切 zone 只是妆。
5. **免费 EDL 进入**：按住音量上 + 插线（适用锁 BL、系统死、XBL 活的设备）。免深刷线、免拆机。会话一次性。
6. **Patched firehose 能过小米 EDL 认证**（此设备类）——人们怕的认证墙不是拦路虎。
7. **macOS + libusb + EDL 批量 I/O = 痛**。Windows + QDLoader 是社区实证路。
8. **先诊断再硬干**：两个系统、多条线多个端口全死在 `ufs_open`/whole-lun init 且零进度——**储存已死**；停止燃烧会话。
9. **「修好」≠「救回资料」**：FBE 把 userdata 绑死在设备 TEE。現实的修复產出的是干净机，不是你的资料。
10. **购机建议**：跨区小米设备，验「原厂出厂区域 = 销售区域」（型号尾码、NCC/官方标记、首开语言）。「陆版硬件刷 Global ROM」是一颗等着变砖的种子。

---

## 09. 附录

### A. 错误码速查（本案例实证 + 社区）

| 错误码 | 消息 | 墙 | 处理 |
|---|---|---|---|
| 20045 | 账号区域 ≠ 销售区域 | 区域 | 同区账号（砖机死路） |
| 20041 | 未绑手机号 | 手机号 | checkSafePhone 链接补绑 |
| 20038 | 被其他账号的查找设备锁定 | 防盗 | i.mi.com → 查找设备 → 移除 |
| 20031/30002 | 开发者选项内绑定账号 | 系统内 | 需可开机（砖机死路） |
| 10013 | 设备未启用/未满 7 天 | 启用 | SIM 启用 + 等待 |
| 20030 | 本月 token 额度 | 配額 | 等 1 号 |
| 10023 | 一账号一年一机 | 配額 | 换账号或等待 |

### B. 指令记录

```bash
# 状态
fastboot devices && fastboot oem device-info && fastboot getvar anti
fastboot getvar slot-unbootable:a ; fastboot --set-active=a   # → FAILED (Lock State)

# EDL 軟进（锁定 nabu 无效）
fastboot oem edl ; fastboot reboot emergency

# macOS 工具
brew install android-platform-tools libusb aria2
uv tool install miunlock        # v1.7.4
git clone https://github.com/bkerler/edl && cd edl && python3 -m venv venv
venv/bin/pip install pyusb pyserial docopt colorama …（repo requirements，剔除错误的 requests>=2.34.2 钉选）

# 有效的 EDL 进入（音量上+插线），macOS 验证：
ioreg -p IOUSB -l -w 0 | grep -iE '9008|QUSB'

# 固件直链（三镜像同路径）
https://bn.d.miui.com/V14.0.8.0.TKXMIXM/nabu_global_images_V14.0.8.0.TKXMIXM_20240112.0000.00_13.0_global_b27365260e.tgz
aria2c -x16 -s16 -k4M -o <file>.tgz <url> ; gzip -t <file>.tgz
```

### C. 測試点（若音量上进入法失效）

主板中央偏右外露窄条上的单颗镀金圆点，正上方紧邻丝印 **M514m/MAPEC** 的橙色 FPC，位于两片银色屏蔽罩之间——用回形针一端搭測試点、一端搭就近屏蔽罩（接地），插线瞬间短接。XDA 帖 4553627 有回形针实证。照片：[`nabu_EDL_testpoint_egsm.jpg`](nabu_EDL_testpoint_egsm.jpg)（圖源：维修社区图库站）。

### D. Windows 侧快速卡

```
1. 解压 MiFlashPatched → 以系统管理員执行 XiaoMiFlash.exe
2. 解压 QUD → 设备管理器 → 9008/QUSB_BULK_CID 带⚠ → 更新驱动 → 浏览 QUD 文件夹（含子文件夹）→ 接受未签名
3. 把 patched prog_ufs_firehose_sm7150_ddr.elf 覆蓋 ROM 的 images/prog_ufs_firehose_sm7150_ddr.elf
4. Select → ROM 文件夹（含 flash_all.bat 那层，不是 images/）
5. 右下单选："clean all"  ← 絕不选 "clean all and lock"
6. 遇「资源使用中」：端口设置→高级→改 COM 号，重插
7. Refresh → 设备列出现 → Flash → 期待 success 或本文记录的 ufs_open 错误
```

### E. 全案原始文件（中文）

- [`reference/case_draft.md`](reference/case_draft.md)——v2 案件档（日期、价格、维修中心记录、逐墙细节）
- [`reference/edl_runbook.md`](reference/edl_runbook.md)——硬件定案前的 EDL runbook

---

## 10. AI 代理在本案的角色 ★

本案由**硬件门外汉**（设备持有人）与 **AI 代理（Hermes，CLI 形態的 AI 助理）**协作完成。分工方式才是可复制的资产：

| 任務 | 誰 | 怎么做 |
|---|---|---|
| 工具发掘与选型（bkerler/edl、patched firehose、官方固件源、镜像测速） | **代理** | 網路调研、源码审读、SHA256 验证、aria2 调参（15 KB/s → 27 MB/s） |
| 把每个错误码翻译成「一道墙 + 拆墙计划」 | **代理** | XDA/GitHub/論壇交叉比对 |
| 读 edl-client 源码预判行为（sparse 自动解包、set-bootable 逻辑、skipresponse 语义） | **代理** | 实打实的代码审计，不靠猜 |
| 实体动作（按键组合、线材、拆机决策、送修） | **持有人** | 代理发步骤卡带回报检查点；持有人执行并回报 |
| 工具链运维（克隆、venv、下载、哈希校验、刷写执行、日誌捕获） | **代理** | 本机 shell、后台进程輪詢 |
| 实时日誌鉴识（发现零 ACK 写入、USBError 5→19 模式、ufs_open 签名） | **代理** | 每輪尝试后对捕获日誌做 grep/awk |
| 跨平台诊断（macOS → Windows 转进、COM 端口占用修复、QUD 驱动 inf 追踪） | **代理** | 失败模式比对社区知识库 |
| 判断决策（何时停止刷机、热敷实验设计、最终定案、处置选项） | **代理提案 → 持有人拍板** | 每个分支决策点 |
| 成本/效益与残值调研（维修 vs 零件 vs 卖社区） | **代理** | 实时市场快照 + 明确的「快照 ≠ 实时」标注 |

**代理做不到的**（诚实边界）：实体按按键、跑维修中心、以触感判断加热温度、给设备断电重启。每个实体步骤都是一张带回报检查点的编号卡：*「做 X → 告诉我你看到什么」*。持有人唯一需要的技能是照卡执行并描述屏幕上出现的东西——**零硬件先备知识**。

**为何公布 AI 角色**：本案可复制的资产不是「我们刷了一台 Pad 5」——而是**工作流本身**：AI 代理做调研、工具链、日誌鉴识、决策框架，门外汉以检查点纪律执行实体步骤。这个模式适用于任何 Qualcomm 设备的 EDL-9008 级救援。

---

## 致谢与参考

- bkerler/edl——开源 EDL 工具链
- erdilS/Port-Windows-11-Xiaomi-Pad-5——patched firehose 与 EDL 指南
- XDA 帖：Mi Pad 5 EDL Test Point（#4553627）、Pad 6 音量上进 EDL（#4744305）、ufs_open 失败案例（Mi 5 / Mi 8 / Mi 9T Pro / LG V30——同签名）
- 维修社区图库站——測試点摄影
- 公开固件索引站——固件索引

*以上所有错误码、价格、门店记录、固件哈希与实验结果均为 2026-10-18 → 2026-11-02 一手实測资料。*
