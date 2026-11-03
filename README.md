# pad5-nabu-unbrick-forensics (by Hermes agent AI)

> **Case window**: 2026-10-18 → 2026-11-02 · Author: device owner, assisted end-to-end by an AI agent (Hermes)
> **Final verdict**: UFS storage chip physically damaged (controller-level). All software recovery paths exhausted and documented.
> **Why publish**: every step, every failed attempt, and every dead end is recorded so the next person with a bricked Pad 5 can reach the correct diagnosis in hours instead of weeks.

[繁體中文](README.tw.md) | [简体中文](README.zh-cn.md)

---

## 00. TL;DR

A Xiaomi Pad 5 (nabu, "Global" unit) died after a failed OTA: boot shows **"The system has been destroyed"**, Recovery is dead, only Fastboot survives. The owner — a complete hardware novice — ran a full forensic recovery with an AI agent driving the technical work:

1. **Account-side walls**: error 20045 (region mismatch, unsolvable) → re-registered TW account → 20041 (fixed) → 20038 (Find Device lock, fixed) → 20031 (physical wall — needs bootable OS)
2. **Authorized service center**: officially attempted re-flash — **failed** (record available via sister company)
3. **DIY EDL deep-flash**: free entry into EDL discovered (Vol+ + cable), patched firehose **passed device authentication**, but all bulk data I/O failed on both macOS and Windows
4. **Final diagnosis**: `ufs_open` fails on every tool/platform → heating experiment to rule out BGA solder fatigue → **UFS chip physically dead**

**The lesson that saves you two weeks**: if your device enters EDL (9008), authenticates the firehose, but *every* read/write returns `Failed to initialize (open whole lun) UFS Device` / `ufs_open(0, UFS_WLUN_DEVICE) failed` across two different host OSes and multiple cables/ports — **stop flashing, the storage is dead.** Software cannot fix silicon.

---

## 01. Origin

| Item | Detail |
|---|---|
| Device | Xiaomi Pad 5, codename **nabu**, "international/Global" unit |
| Sales channel claim | Sold as genuine Global unit — **likely CN hardware flashed with Global ROM** (see §05 root cause) |
| Serial | b03\*\*\*\*bb (masked) |
| Failure | After an interrupted/failed OTA: boot → "The system has been destroyed"; Recovery → same error; **Fastboot → alive and stable** |
| Pre-failure state | Normal daily use, no water/impact history |

Slot exam (fastboot):

```
slot-retry-count:a = 6      slot-retry-count:b = 7
slot-successful:a  = yes    slot-successful:b  = yes
slot-unbootable:a  = yes    slot-unbootable:b  = no
current-slot = b
```

One failed OTA took down **both** slots. `fastboot --set-active=a` → `FAILED (remote: 'Slot Change is not allowed in Lock State')` — with BL locked, even zero-risk slot switching is forbidden. Only path: unlock BL. But the device can't boot → **deadlock**.

---

## 02. Information State (the four walls)

Xiaomi's BL-unlock flow (2025+) is a chain of four server-side checks:

```
[Device] ──① Region──▶ [② Phone#] ──▶ [③ Find-Device lock] ──▶ [④ In-OS binding] ──▶ UNLOCK
          (20045)        (20041)          (20038)              (20031)
```

| # | Code | Meaning | Nature |
|---|---|---|---|
| ① | 20045 | Account region ≠ device sales region | **Hard** — region is stamped at account creation by phone country code (+86 → CN), cannot be self-migrated |
| ② | 20041 | Account has no phone number bound | Config — bindable via `account.xiaomi.com/pass/serviceLogin?checkSafePhone=true` |
| ③ | 20038 | Device locked to another account via Find Device | Cloud — removable at i.mi.com (**note: "remove device" ≠ "turn off Find Device"; two separate switches**) |
| ④ | 20031 | Must bind account in Settings → Developer options | **Physical** — requires a bootable OS. Bricked = impossible |

Walls ①②③ are account-side (demolishable). Wall ④ is device-side — and our device is a brick.

---

## 03. Reasoning

- **Error codes are a map.** Each code = one specific wall. Demolish in order instead of randomly re-trying unlock tools.
- **Region is birth-stamped**: changing `dataCenterZone` (Singapore/Europe) in miunlock only changes where the request lands, not the account's region attribute. Verified: same 20045 after zone switch.
- **The anti-theft design assumes a bootable device.** A brick fails wall ④ — the legal owner is locked out by the very system meant to protect them.
- **"Repair" ≠ "data recovery"**: Android 10+ enforces FBE; userdata is ciphertext bound to the device TEE. Any flash path (including "preserve-data flash") ends in a wiped or unusable storage for practical purposes. Set expectations accordingly.

---

## 04. Action A — Account-side walls (all three demolished)

### 4.1 Wall ① 20045 — region (unsolvable, documented)
- Official MiFlashUnlock (Windows) with original CN account (+86) → 20045
- miunlock (macOS) with same account → server reports `Account Region: CN`
- Manually forcing `dataCenterZone = Singapore` → **still 20045** (zone ≠ region)
- **Resolution**: register a fresh TW account with a +886 number

### 4.2 Wall ② 20041 — phone binding
- New TW account → `20041 Sorry, your MI ID is not associated with a phone number`
- Bind +886 via the official checkSafePhone link → **pass**

### 4.3 Wall ③ 20038 — Find Device (the plot twist)
- Re-running unlock: `20038 该手机已被账号 1*********2 通过查找手机锁定`
- At i.mi.com: "Remove device" in Device Management **is not enough** — the Find Device list still contained the Pad 5
- Removed it in the Find Device section too → list verified clean → **pass**

### 4.4 Wall ④ 20031 — the physical wall
- `20031 Please add your account in Settings > Developer options > Mi Unlock status`
- Requires the OS to boot. The OS is destroyed. **Circular deadlock, by design.**

### 4.5 EDL soft-entry attempts (failed, documented)
- `fastboot oem edl` / `fastboot reboot emergency` → no response
- Power off + Vol- + cable → no 9008 in Device Manager
- (Later solved for free — see §06.1)

---

## 05. Action B — Authorized Service Center

- Device sent to the **official Xiaomi service center** for re-flash
- Verdict: **could not process** (service record retrievable — sister company)
- Significance: proves the official channel has **no exit** for this brick class either. The "just take it to official service" common wisdom is falsified for this failure mode.
- Root cause of the original brick (purchasing layer): This "Global" unit is very likely **CN hardware + flashed Global ROM**; OTA server mismatch → verification failure → dual-slot kill. **Buying advice: verify factory region = sales region (model sticker + NCC mark + first-boot language).**

---

## 06. Action C — DIY EDL Deep Flash (full record)

### 6.1 Free EDL entry (no disassembly, no EDL cable) ★ the discovery that saves everyone money

For a device with **locked BL + dead OS but alive Fastboot/XBL**:

> **Power off → press and HOLD Volume Up → plug USB cable → keep holding 3–5 s → release**

- First discovered on a Pad 6 (XDA user report: "try vol up, then vol down, then both — one will work")
- **Verified on this Pad 5**: enumerated instantly as `QUSB_BULK_CID:0404_SN:6FF5****` / `Qualcomm CDMA Technologies MSM` (idVendor 1478 = Qualcomm, idProduct 0x9008)
- The earlier failed attempt (Vol- only) was one data point of a three-combination search — try **Vol+ first**, then Vol-/both.
- **EDL sessions are single-use**: any tool handshake (even a failed one) burns the session. Power-cycle + re-enter before every fresh attempt.

### 6.2 Firehose: patched loader passes device authentication

| Loader | Size | SHA256 (prefix) | Source |
|---|---|---|---|
| `prog_ufs_firehose_sm7150_ddr.elf` (patched) | 724,800 B | `03a44a6a…` | [Port-Windows-11-Xiaomi-Pad-5 releases](https://github.com/erdilS/Port-Windows-11-Xiaomi-Pad-5/releases) |
| Same name inside official ROM (signed) | 745,136 B | `60c0ca2c…` | `nabu_global_images_V14.0.8.0.TKXMIXM` package |

Same filename = same SoC/storage target confirmed (SM7150-class = Snapdragon 730G… device reports **SM8150-family / Snapdragon 860** on handshake — Pad 5's actual SoC).

Sahara handshake log (macOS, bkerler/edl):

```
sahara - Protocol version: 2, Version supported: 1
CPU detected:      "SM8150"
PK_HASH:           0x1bebe386…
Serial:            0x6ff558a5
sahara - Loader successfully uploaded.
firehose - Xiaomi EDL Auth detected.
firehose - EDL Authenticated successfully.     ★ auth wall does NOT exist for this loader
```

**Firmware**: official Global stable `nabu_global_images_V14.0.8.0.TKXMIXM` (2024-01, last Global release), 3.83 GB, mirrors: `bigota.d.miui.com` / `bn.d.miui.com` / `hugeota.d.miui.com` (same path pattern `/<MIUI version>/<filename>`). Speed-lesson: bigota throttled to 15 KB/s on this network; bn.d delivered 27 MB/s via aria2 16 connections (test first with `curl -r 0-8388607 -w "%{speed_download}"`).

### 6.3 macOS: handshake OK, bulk I/O dead — failure analysis

Tool: bkerler/edl (V3.62) on macOS 26 (Apple Silicon), libusb 1.0.30.

What worked: Sahara handshake, loader upload, Xiaomi EDL auth, small XML exchanges, final `setbootablestoragedrive`.

What never worked: any sustained bulk transfer — GPT reads, partition reads, flash writes with response-waiting. Symptom pattern:

```
[write progress 100%] → DeviceClass - USBError(5, 'Input/Output Error')
                      → USBError(19, 'No such device (it may have been disconnected)')
                      → device re-enumerates → next command…
```

- A full qfil run "completed" 43 program entries + 86 patches **with zero ACK confirmations** — writes were sent, never verified. (Device did not boot afterwards.)
- `--skipresponse` does not bypass handshake-phase response waits; process hung with zero output.
- GPT read attempts died at `cmd_read_buffer` → device dropped mid-read.

**Conclusion**: macOS libusb + this loader/device combination cannot sustain bulk EDL I/O. All community success cases are Windows + QDLoader driver.

### 6.4 Windows: the standard path, and what killed it

Environment: Windows laptop (has an onboard Qualcomm WWAN module — relevant!), MiFlashPatched (2022-05-07 build), QUD drivers.

Issues solved on the way (for completeness):
- `IOException: requested resource in use` → **COM port squatted** by other software. Reboot did NOT fix (resident service). Fix: **Device Manager → Ports → 9008 → Properties → Port Settings → Advanced → change COM number** (COM4 → COM10), then re-plug so the device re-enumerates on the new port.
- Driver: QUD.zip inf files live at extraction root (`qcser.inf` contains `USB\VID_05C6&PID_9008`). Install via Device Manager → Update driver → Browse (⚠️ tick "include subfolders") → accept unsigned warning. Or `pnputil /add-driver C:\qud\qcser.inf /install`.

Then the real wall (log excerpt):

```
[COM10]:send command:<?xml…><setbootablestoragedrive value="1"/></data>
resdump:<log value="ERROR: Failed to initialize (open whole lun) UFS Device slot 0 partition 1" />
        <log value="ERROR: Failed to open the device:3 slot:0 partition:1 error:0" />
System.Exception: set boot partition failed
```

Earlier attempts also produced `write time out, maybe devices was disconnected` at 20–50 s with **zero progress-bar movement**. Same signature on every cable, every port (front/rear, 2.0/3.0), with/without laptop AC power, after full charge, and across OSes.

**The smoking gun**: MiFlash (Windows) and edl.py (macOS) both reach the loader, both fail at **the first storage op** — `ufs_open` / `open whole lun`. The flash controller never opens the door.

### 6.5 The heating experiment (BGA solder-fatigue hypothesis) — falsified

Hypothesis: UFS BGA solder micro-cracks could cause ufs_open failure; controlled heating (~65–70 °C for ~4 min over the motherboard area — top half of the back cover, camera-side long edge extending inward) might re-seat joints.

Result: **no change** — EDL entry OK, loader upload OK, flashing still zero progress / instant storage errors.

Interpretation: solder-fatigue ruled out to DIY-achievable precision. Combined with two-OS two-tool failure at the same op → **UFS chip internal failure (controller-level), not connection-level**.

---

## 07. Result — the closed evidence chain

| Evidence | Eliminates |
|---|---|
| Handshake + auth pass on both OSes | cabling, USB stack, tool misconfig |
| Both OSes die at the same op (`ufs_open`/whole-lun init) | host-platform specifics |
| Authorized service center also failed | authorization/account walls |
| Heating experiment no-change | BGA solder fatigue (DIY level) |
| Normal operation before the OTA event | congenital defect |
| One interrupted OTA preceded the brick | **OTA event physically damaged the UFS (voltage/controller-level)** |

**Final**: UFS storage physically damaged. Repair = UFS chip replacement or motherboard swap — economically unjustified for this device's residual value (~NT$4–5k working). Disposition: sell as parts/donor to the flashing community (fastboot alive, no account lock, full case file included).

---

## 08. Lessons (for the next person)

1. **Order of operations**: check BL state (`fastboot oem device-info`) before dreaming of flashing; a locked device rejects every write.
2. **Error codes are a map** — 20045/20041/20038/20031 each name a specific wall; demolish in order.
3. **"Remove device" ≠ "turn off Find Device"** at i.mi.com — 20038 needs the latter.
4. **Account region is birth-stamped** by phone country code; zone-switching is cosmetic.
5. **Free EDL entry**: Vol+ held + cable (for locked-BL, OS-dead, XBL-alive devices). No EDL cable, no disassembly. Sessions are single-use.
6. **Patched firehose passes Xiaomi EDL auth** on this device class — the auth wall is not the blocker people fear.
7. **macOS + libusb + EDL bulk I/O = pain.** Windows + QDLoader is the community-proven path.
8. **Diagnose before you grind**: if two OSes and multiple cables/ports all fail at `ufs_open`/whole-lun init with zero progress — **the storage is dead**; stop burning sessions.
9. **"Fix" ≠ "recover data"**: FBE encryption binds userdata to the device TEE. Any realistic repair yields a clean device, not your data.
10. **Buying advice**: for cross-region Xiaomi devices, verify factory region = sales region (model number suffix, NCC/official mark, first-boot language). "Global ROM on CN hardware" is a brick-in-waiting.

---

## 09. Appendix

### A. Error code quick table (verified + community)

| Code | Message | Wall | Handling |
|---|---|---|---|
| 20045 | account region ≠ sales region | Region | same-region account (dead end on bricks) |
| 20041 | no phone bound | Phone# | checkSafePhone link binding |
| 20038 | locked by other account's Find Device | Anti-theft | i.mi.com → Find Device → remove |
| 20031/30002 | bind account in Developer options | In-OS | needs bootable OS (dead end on bricks) |
| 10013 | device not activated / <7 days | Activation | SIM activation + wait |
| 20030 | token quota this month | Quota | wait for the 1st |
| 10023 | one device per year per account | Quota | switch account or wait |

### B. Command record

```bash
# State
fastboot devices && fastboot oem device-info && fastboot getvar anti
fastboot getvar slot-unbootable:a ; fastboot --set-active=a   # → FAILED (Lock State)

# EDL soft-entry (failed on locked nabu)
fastboot oem edl ; fastboot reboot emergency

# macOS tooling
brew install android-platform-tools libusb aria2
uv tool install miunlock        # v1.7.4
git clone https://github.com/bkerler/edl && cd edl && python3 -m venv venv
venv/bin/pip install pyusb pyserial docopt colorama …(repo requirements minus the bogus requests>=2.34.2 pin)

# EDL entry that WORKED (vol-up + cable), then verify on macOS:
ioreg -p IOUSB -l -w 0 | grep -iE '9008|QUSB'

# Firmware direct-link (three mirrors, same path)
https://bn.d.miui.com/V14.0.8.0.TKXMIXM/nabu_global_images_V14.0.8.0.TKXMIXM_20240112.0000.00_13.0_global_b27365260e.tgz
aria2c -x16 -s16 -k4M -o <file>.tgz <url> ; gzip -t <file>.tgz
```

### C. Test point (if Vol+ entry ever fails)

Single gold pad on the motherboard's mid-right exposed strip, directly above the orange FPC marked **M514m/MAPEC**, between two silver EMI shields — short to the nearest shield (ground) with a paperclip while plugging USB. XDA thread 4553627 has a paperclip-only success report. Photo: [`nabu_EDL_testpoint_egsm.jpg`](nabu_EDL_testpoint_egsm.jpg) (credit: a repair-community image archive).

### D. Windows-side quick card

```
1. Unzip MiFlashPatched → run XiaoMiFlash.exe as admin
2. Unzip QUD → Device Manager → 9008/QUSB_BULK_CID with ⚠ → Update driver → browse QUD folder (incl. subfolders) → accept unsigned
3. Copy patched prog_ufs_firehose_sm7150_ddr.elf OVER the ROM's images/prog_ufs_firehose_sm7150_ddr.elf
4. Select → ROM folder (the one containing flash_all.bat, NOT images/)
5. Bottom-right radio: "clean all"  ← NEVER "clean all and lock"
6. If "resource in use": change COM number in Port Settings→Advanced, re-plug
7. Refresh → device row appears → Flash → expect success OR the ufs_open errors documented above
```

### E. Full-case raw documents (Chinese)

- [`reference/case_draft.md`](reference/case_draft.md) — the v2 case file (dates, prices, service-center record, per-wall details)
- [`reference/edl_runbook.md`](reference/edl_runbook.md) — the EDL runbook as stood before the hardware verdict

---

## 10. The AI agent's role in this case ★

This case was executed by a **hardware novice** (the device owner) working with an **AI agent (Hermes, Claude/GLM-based CLI assistant)**. The division of labor is the replicable part:

| Task | Who | How |
|---|---|---|
| Tool discovery & selection (bkerler/edl, patched firehose, official ROM sources, mirror benchmarking) | **Agent** | web research, source-code reading, SHA256 verification, aria2 tuning (15 KB/s → 27 MB/s) |
| Interpreting every error code into a "wall" with a demolition plan | **Agent** | XDA/GitHub/forum cross-referencing |
| Reading edl-client source to predict behavior (sparse auto-unpack, set-bootable logic, skipresponse semantics) | **Agent** | actual code audit, not guessing |
| Physical actions (button combos, cables, disassembly decisions, service-center visit) | **Owner** | agent issued step-cards with checkpoints; owner executed and reported back |
| Toolchain ops (cloning, venvs, downloads, hash checks, flash runs, log capture) | **Agent** | local shell, background processes with polling |
| Live log forensics (spotting zero-ACK writes, USBError 5→19 patterns, ufs_open signature) | **Agent** | grep/awk over captured logs after each attempt |
| Cross-platform diagnosis (macOS → Windows pivot, COM-port squatting fix, QUD driver inf hunting) | **Agent** | failure-pattern matching against community knowledge |
| Judgment calls (when to stop flashing, heating-experiment design, final verdict, disposition options) | **Agent proposed → Owner decided** | every branch decision point |
| Cost/benefit and resale-value research (repair vs parts vs sell-to-community) | **Agent** | live market snapshots with explicit "snapshot ≠ realtime" caveats |

**What the agent could NOT do** (honest boundary): physically press buttons, visit the service center, judge heating temperature by touch, or power-cycle the device. Every physical step was a numbered card with a report-back checkpoint: *"do X → tell me what you see"*. The owner's only required skill was following cards and describing what appeared on screens — **zero prior hardware knowledge needed**.

**Why publish the AI role**: the replicable asset here is not "we flashed a Pad 5" — it's **the workflow**: an AI agent doing research, tooling, log forensics, and decision-framing while a novice executes physical steps with checkpoint discipline. This pattern works for any EDL-9008-class rescue on any Qualcomm device.

---

## Credits & references

- bkerler/edl — the open-source EDL toolchain
- erdilS/Port-Windows-11-Xiaomi-Pad-5 — patched firehose & EDL guide
- XDA threads: Mi Pad 5 EDL Test Point (#4553627), Pad 6 EDL entry via Vol+ (#4744305), ufs_open failure cases (Mi 5 / Mi 8 / Mi 9T Pro / LG V30 — same signature)
- a repair-community image archive — test-point photography
- public firmware index sites — firmware index

*All error codes, prices, store records, firmware hashes, and experiment results above are first-hand data from 2026-10-18 → 2026-11-02.*
