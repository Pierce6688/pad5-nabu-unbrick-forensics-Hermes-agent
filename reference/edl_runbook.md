# Pad 5 (nabu) EDL 深刷 DIY Runbook
> 2026-10-25 定稿 v2 · 全环节已桌面验证（仅差实机 9008 接入）· 前置：i.mi.com 已移除设备（无账号锁威胁）

## 0. 现状与铁律
- BL **locked**、双槽俱损、Recovery 坏、软进 EDL（`fastboot oem edl` / `reboot emergency`）已证实无效
- → **进 EDL 只有硬件路**：拆机短接测试点（屏面拆解）或 EDL 短接线（Hydra V2 类）
- 全部操作 Mac 即可完成，不依赖 Windows MiFlash

## 1. 已备妥的工具（全部本机就位，逐一验证过）
| 项 | 路径 | 验证结果 |
|---|---|---|
| bkerler/edl 工具链 | `~/workspace/edl/`（venv） | ✅ `edl.py --help` 跑通；qfil 源码审读完毕 |
| patched firehose（免签） | `~/workspace/edl/loaders/prog_ufs_firehose_sm7150_ddr.elf` | ✅ 724,800B，SHA256 `03a44a6a…f97` |
| ROM 内置签名 firehose（对照） | `…/images/prog_ufs_firehose_sm7150_ddr.elf` | ✅ 745,136B，SHA256 `60c0ca2c…719f`——**同名确证 patched 版型号正确** |
| 官方 Global 固件 | `~/workspace/edl/firmware/nabu_global_images_V14.0.8.0.TKXMIXM_13.0/` | ✅ 3.83GB gzip 完整（SHA256 `0862edbb…7683`），77 分区镜像+6 XML 齐 |
| libusb 1.0.30 / fastboot | brew | ✅ |

固件要点（已核）：SECTOR_SIZE=4096；rawprogram0-5 + patch0-5（共 132 program 条目，xbl_a 在 rawprogram1、xbl_b 在 rawprogram2 → qfil 收尾自动 set bootable）；userdata.img=6MB sparse（展开 12GB 空 ext4，保留分区但格式化=出厂态）；无 anti_version.txt→默认 1，本机 anti=1 兼容。

## 2. 实机操作流程

### 2.1 进 EDL + Mac 识别（★测试点位置 2026-10-25 已确认，见同目录 nabu_EDL_testpoint_egsm.jpg）
1. 关机。**屏面拆解**（拆机教程站/instructions/how-to-disassemble-xiaomi-pad-5/ 有全步骤图文）：
   - 加热屏幕约 70°C（吹风机高热绕边缘 2-3 分钟）→ 塑料片沿边缝滑开（浅，防排线）
   - 卸 1 颗 Phillips #000 螺丝 → 掀开金属支架 → **先断电池排线 BN61**
   - 断屏幕排线，移开屏幕
2. 测试点：**主板中央偏右，两块银色屏蔽罩之间外露窄条上缘、紧邻橙色 FPC 排线（丝印 M514m/MAPEC）的正上方**——单颗镀金圆点（图库站 图黄圈处）。
   XDA 实证（帖子 4553627）：有人只用**回形针**短接成功，无需 EDL 线
3. 短接手法：镊子/回形针一端搭测试点、一端搭**就近屏蔽罩（接地）**——保持压住 → 插 USB 线 → 听到 Mac 识别声/约 5 秒后松开
4. 检查：`ioreg -p IOUSB -l -w 0 | grep -iE '9008|QUSB'`（macOS 无需 QDLoader 驱动，libusb 直抓）

### 2.2 先读后写（任何卡住加 `--debugmode`）
```bash
cd ~/workspace/edl
F=loaders/prog_ufs_firehose_sm7150_ddr.elf
venv/bin/python edl.py printgpt --loader=$F     # 分区表能读=firehose 接管成功
```

### 2.3 全包线刷（= MiFlash clean all，等效官方 flash_all）
```bash
IMG=firmware/nabu_global_images_V14.0.8.0.TKXMIXM_13.0/images
RP=$IMG/rawprogram0.xml,$IMG/rawprogram1.xml,$IMG/rawprogram2.xml,$IMG/rawprogram3.xml,$IMG/rawprogram4.xml,$IMG/rawprogram5.xml
PT=$IMG/patch0.xml,$IMG/patch1.xml,$IMG/patch2.xml,$IMG/patch3.xml,$IMG/patch4.xml,$IMG/patch5.xml
venv/bin/python edl.py qfil "$RP" "$PT" "$IMG" --loader=$F
venv/bin/python edl.py reset
```
- qfil 会依序写 132 条目（sparse 自动展开）、打 patch、最后自动 set bootable 分区
- 单分区失败：重进 EDL 重跑整条 qfil（幂等，进度不丢）

### 2.4 验证
- 重启出现 Mi logo → 进 fastboot 查 `fastboot getvar slot-unbootable:a` 应为 no
- 若仍异常：重跑 2.3（EDL 下反复刷无损耗）

## 3. 风险与止损
| 风险 | 缓解 |
|---|---|
| 拆机断排线 | 屏面缓慢开，电池排线先断 |
| patched firehose 被拒 | 回退 ROM 自带签名版（若设备认签）；再不行走 技术博客/bkerler Loaders 找替代 |
| 误刷大陆包 | 只用 `…TKXMIXM…global…` 目录，文件名含 global 才准进 |
| 数据全丢 | qfil 含 userdata 空镜像 = 清数据；保数据须刷前先 `edl.py r userdata out.bin`（12GB+，自行取舍） |

## 4. 退路
- DIY 失败 → 维修店 EDL（维修店A ***)***-2849 / 维修店B ***)***-**** / 维修店C ***)***-6818，行情 NT$800–1,500）
- 维修店包：`~/workspace/edl/` 整目录 + 本 runbook（firehose+固件+命令全在）
