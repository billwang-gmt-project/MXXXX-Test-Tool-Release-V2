# M8160AA Input Pin Resistor Design Tool — Refactored Spec (待 review)

## Context

要在 M8160AA chip family 下加一個新 tool。跟現有的 **M8160 Design Tool**（reverse 方向：ADC voltage → register value）不同，這個工具是 **forward 方向**：使用者用 dropdown 挑 application 設定，工具算出 4 支 external pin（Mode / XP1 / YP / SS）的 target ADC 電壓、外部建議 resistor pair、以及對應 register-field 的 bypass / force 值。

當前任務只是 **refactor 原始 spec 給你 review**，還不寫 code。Approve 之後再走 writing-plans skill 出實作計畫。

## Reading map

- 原始 spec 文字（保留逐字）— 對話記錄裡 (`§9` 不複製)
- 重整後 deduplicated rules — `Inputs`、`Mode classifier`、`XP1 voltage`、`YP voltage`、`SS voltage`
- Open questions — `Review checklist`

---

## 1. Inputs (UI dropdowns)

視窗放 9 個 dropdown，default 標 `★`。

| # | Field | Options | Notes |
|---|-------|---------|-------|
| 1 | **Loop type** | Open-loop ★ / Close-loop | Close-loop 多要 #1.1 / #1.2 / #1.3 |
| 1.1 | **Min RPM** | numeric (close-loop only) | 使用者填目標 min |
| 1.2 | **Max RPM** | numeric (close-loop only) | 使用者填目標 max |
| 1.3 | **rpm_ratio** | 2.54 / 3.72 / 5.59 / 8.3 / 12.2 / 17.79 / 25.93 / 37.96 | Bound 到 `REG01[2:0]` (`rpm_ratio` field, 看 [M8160AA.json:63-77](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/chips/M8160AA.json#L63-L77))。Auto-suggest：挑最小 ratio 同時 `(60·r ≤ MaxRPM)` 且 `(min_rpm_table ≤ MinRPM)`。使用者可手動 override。 |
| 2 | **SO pole** | 4p ★ / 6p / 8p / RD | RD = reluctance |
| 3 | **Standby behaviour** | Shutdown ★ / Minimum | Minimum = 馬達在 min duty 繼續轉 |
| 4 | **Drive mode** | Direct PWM ★ / DC mode | DC mode 還要選 VSP / SET (#6) |
| 5 | **PWM polarity** | Non-invert ★ / Invert | 只在 Drive mode = Direct PWM 時 active |
| 6 | **DC source** | VSP ★ / SET | 只在 Drive mode = DC mode 時 active |
| 7 | **Hall type** | non-FR (hall element) ★ / FR-enable (hall switch) | |
| 8 | **Force startup** | Force ★ / Non-force | Open-loop only；close-loop forced ★ |
| 9 | **Soft switch** | Square ★ / 22.5° / 45° / Sine | Open-loop only；close-loop forced Square |

### Bypass rules (close-loop)

當 **Loop type = Close-loop** 時，下面這幾個 input 變 read-only，register 也 force 死：

```
reg_data_0d[3] (pwm_low_sw)  ← 0   bypass #4 (force Direct PWM)
reg_data_01[7] (cmd_inv)     ← 0   bypass #5 (PWM non-invert) 和 #6
reg_data_0e[7] (align_sw)    ← 1   bypass #8 (Force startup)
reg_data_03[7:6] (ssw)       ← 0   bypass #9 (Soft switch = square)
```

另外當 **Hall type = non-FR** 時，`reg_data_0f[1]` (`fr_pin_sw`) force 0，bypass #7 在那個 branch。

> ⚠ **Spec discrepancy 1** — 原文寫 `reg_data_0d[6]` 為 "direct PWM"；M8160AA.json 裡 `pwm_low_sw` 在 `0x0D[3]`（`vsp_sw` 才在 `0x0D[6]`）。Refactor 用 JSON-correct field name `pwm_low_sw` at `[3]`。請 hardware engineer 確認 "direct PWM" 應該是 bypass `pwm_low_sw` 還是 `vsp_sw`。

---

## 2. Mode classifier (決定 Mode-pin voltage)

`mode` 是 0–7 discrete index，對應 **Mode ADC pin** 上一個 target 電壓。8 個電壓配對到既有的 [M8160AdcCalculator.cs:60+](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Helpers/M8160AdcCalculator.cs#L60) `ModeBands` 表。

| mode | V_target | Use |
|------|---------:|-----|
| 0 | 0.00 V | open-loop, 4p/8p, shutdown, (PWM_invert OR DC_mode) |
| 1 | 0.92 V | open-loop, 4p/RD, shutdown, soft_switch=22.5° |
| 2 | 1.55 V | open-loop, 4p/8p, FR-enable |
| 3 | 2.18 V | open-loop, 4p/6p, non-FR |
| 4 | 2.80 V | close-loop, 6p OR 8p |
| 5 | 3.43 V | close-loop, 4p, non-FR |
| 6 | 4.06 V | close-loop, 4p, FR-enable |
| 7 | 5.00 V | open-loop, 4p/8p, shutdown, (PWM_non_invert OR DC_mode) |

### 2.1 Decision table (disambiguate-by-sub-condition)

每個 row 當 **single predicate** 看，row 之間互斥不重疊。

| # | Loop | Pole | Standby | DriveMode | PWM | Hall | → mode |
|---|------|------|---------|-----------|-----|------|--------|
| C1 | Close | 4p | – | – | – | non-FR | **5** |
| C2 | Close | 4p | – | – | – | FR | **6** |
| C3 | Close | 6p / 8p | – | – | – | – | **4** |
| O1a | Open | 4p / 8p | Shutdown | Direct-PWM | Invert | – | **0** |
| O1b | Open | 4p / 8p | Shutdown | DC mode | – | non-FR + SET<br>FR + VSP | **0** |
| O2a | Open | 4p / 8p | Shutdown | Direct-PWM | Non-invert | – | **7** |
| O2b | Open | 4p / 8p | Shutdown | DC mode | – | non-FR + VSP<br>FR + SET | **7** |
| O3 | Open | 4p / RD | Shutdown | – | – | – (soft_switch=22.5°) | **1** |
| O4 | Open | 4p / 8p | – | – | – | FR | **2** |
| O5 | Open | 4p / 6p | – | – | – | non-FR | **3** |

> ⚠ **Spec discrepancy 2** — 原文 O1/O2 重疊（兩邊都列 `4p & shutdown`），O4/O5 也跟 O1–O3 重疊。Refactor 依使用者選擇 [Q4 answer]：每 row 的 predicate 要 **完整 conjunction**（所有 sub-condition）。建議 unit test 把所有 4 × 4 × 2 × 2 × 2 × 2 × 2 = 512 input combo 全跑一次，每個 legal combo 必須恰好 match 一 row。

---

## 3. Voltage formula primitive

3 支 pin（XP1 / YP / SS）共用一條 ADC formula。原 spec 寫 `0.01953125 mV` 是 **單位錯誤** — 實際 scale 是 `5 V / 256 = 0.01953125 V/step`。Refactor 全部 normalize 成 **V**。

```
V_high(code) = (256 − code) × 5/256        // negative-slope branch, low end at 5 V
V_low (code) =  code        × 5/256        // positive-slope branch, low end at 0 V
```

`code = field_value + 20`。`+20` offset 跟 `×4` step（如有）來自 M8160AA ADC band layout（看 [M8160AdcCalculator.cs:264](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Helpers/M8160AdcCalculator.cs#L264)）。

### 3.1 4.6 % out-of-range guard

原 spec 用 `xp1 == 4.6%` / `yp1 == 4.6%` 當 sentinel。依 [Q1 answer]，**4.6 % 是最小可表示的 duty**；任何 duty input 低於這個值 → **clamp 到 endpoint voltage**（0 V 或 5 V，看 branch）。UI 要顯示 "out-of-range, clamped to 0 V/5 V" badge。

---

## 4. XP1 pin voltage

`xp1` formula 看 `mode` 跟 Hall / Drive-mode sub-condition。Register source 有兩種：

- **Open-loop modes 0/1/2/3/7**：使用者填 `xp1_duty(%)` → `field = floor(duty / 1.5625)` → `code = field × 4 + 20`（注意 `×4` step）。
- **Close-loop modes 4/5/6**：算 `field = min_rpm / rpm_ratio − 59` → `code = field + 20`。

| mode | sub-condition | branch | source field |
|------|---------------|--------|--------------|
| 0 (shutdown) | non-FR + SET | V_high | `reg_data_02[4:0]` × 4 |
| 0 (shutdown) | FR + VSP | V_low | `reg_data_02[4:0]` × 4 |
| 1 (shutdown) | non-FR | V_high | `reg_data_02[4:0]` × 4 |
| 1 (shutdown) | FR | V_low | `reg_data_02[4:0]` × 4 |
| 7 (shutdown) | non-FR + VSP | V_high | `reg_data_02[4:0]` × 4 |
| 7 (shutdown) | FR + SET | V_low | `reg_data_02[4:0]` × 4 |
| 2 + FR | shutdown | V_high | `reg_data_02[4:0]` × 4 |
| 2 + FR | minimum | V_low | `reg_data_02[4:0]` × 4 |
| 3 + non-FR | shutdown | V_high | `reg_data_02[4:0]` × 4 |
| 3 + non-FR | minimum | V_low | `reg_data_02[4:0]` × 4 |
| 4 + FR + 6p | shutdown | V_high | `reg_data_06[6:0]` (no ×4) |
| 4 + FR + 8p | shutdown | V_low | `reg_data_06[6:0]` (no ×4) |
| 5 + non-FR | shutdown | V_high | `reg_data_06[6:0]` (no ×4) |
| 5 + non-FR | minimum | V_low | `reg_data_06[6:0]` (no ×4) |
| 6 + FR | shutdown | V_high | `reg_data_06[6:0]` (no ×4) |
| 6 + FR | minimum | V_low | `reg_data_06[6:0]` (no ×4) |

`reg_data_06[6:0]` 對應 `rp1`（[M8160AA.json:265](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/chips/M8160AA.json#L265)）。Close-loop 的 sentinel `xp1 == 60·rpm_ratio` 對應 max RPM，依 §3.1 clamp 到 endpoint。

---

## 5. YP pin voltage

`yp1` source field = `reg_data_04[6:0]`（[M8160AA.json:209](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/chips/M8160AA.json#L209)）。Close-loop 用 `reg_data_07`（[M8160AA.json:297](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/chips/M8160AA.json#L297)）。

| mode | sub-condition | branch | source field |
|------|---------------|--------|--------------|
| 0 | Direct-PWM + Invert | V_high | `reg_data_04[6:0]` |
| 0 | DC mode | V_low | `reg_data_04[6:0]` |
| 1 / 2 / 3 | Non-force startup | V_high | `reg_data_04[6:0]` |
| 1 / 2 / 3 | Force startup | V_low | `reg_data_04[6:0]` |
| 7 | Direct-PWM | V_high | `reg_data_04[6:0]` |
| 7 | DC mode | V_low | `reg_data_04[6:0]` |
| 4 / 5 / 6 | – | V_low | `reg_data_07 = (min_rpm/rpm_ratio − 59) / 4` |

`reg_data_04` 那幾 row 都套 §3.1 的 4.6 % guard。

---

## 6. SS pin voltage

SS pin **不用 §3 formula**，是一個 fixed lookup table on `(mode, pole, ss_time)`。`ss_time` 是 `REG08[7:5]`（[M8160AA.json:348](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/chips/M8160AA.json#L348)）。使用者從 `0.5s / 1s / 2s / 4s / 6s / 8s / 10s / 12s` 8 個 option 挑（對應 `REG08[7:5]` index 0..7）。

### 6.1 Lookup table

兩條 voltage profile 共用同一個 `ss_time` axis：

| ss_time | V_profileA (5→2.85) | V_profileB (0→2.13) |
|---------|--------------------:|--------------------:|
| 0.5 s   | 5.000 V | 0.000 V |
| 1 s     | 4.257 V | 0.722 V |
| 2 s     | 4.023 V | 0.957 V |
| 4 s     | 3.789 V | 1.191 V |
| 6 s     | 3.554 V | 1.425 V |
| 8 s     | 3.320 V | 1.660 V |
| 10 s    | 3.085 V | 1.894 V |
| 12 s    | 2.851 V | 2.128 V |

### 6.2 Profile selector

| mode | pole | profile |
|------|------|---------|
| 0 / 2 / 7 | 4p | A |
| 0 / 2 / 7 | 8p | B |
| 1 | 4p | A |
| 1 | RD | B |
| 2 | 4p | A |
| 2 | 6p | B |
| 4 / 5 / 6 | – | **看 §6.3** |

> ⚠ **Spec discrepancy 3** — 原文 `mode==2` 出現兩次（一次在 `mode==0/2/7`，一次單獨）。Refactor 採單獨那個 block 為主（4p/6p split）。請 hardware engineer 確認。

### 6.3 Close-loop SS voltage (mode 4 / 5 / 6)

Close-loop 時 SS voltage 直接由 `rpm_ratio` 決定（對應 `REG01[2:0]` 8-option table）：

| rpm_ratio | RPM range (`min_rpm < RPM < max_rpm`) | SS voltage |
|----------:|---------------------------------------|-----------:|
| 2.54 | 149.86 – 2740.66 | 0.00 V |
| 3.72 | 219.48 – 4013.88 | 0.92 V |
| 5.59 | 329.81 – 6031.61 | 1.55 V |
| 8.30 | 489.70 – 8955.70 | 2.18 V |
| 12.20 | 719.80 – 13163.80 | 2.80 V |
| 17.79 | 1049.61 – 19195.41 | 3.43 V |
| 25.93 | 1529.87 – 27978.47 | 4.06 V |
| 37.96 | 2239.64 – 40958.84 | 5.00 V |

工具要驗證使用者填的 `min_rpm` / `max_rpm` (input #1.1 / #1.2) 落在 row 的 RPM range 內，否則該 row 標紅。

---

## 7. Output presentation (DataGrid)

每個受影響的 pin / register field 一個 row。Column 設計：

| Column | Source | Example |
|--------|--------|---------|
| **Pin / Field** | Mode / XP1 / YP / SS, 或 `reg_data_XX[B:b]` | `Mode pin` / `reg_data_0d[3]` |
| **Target voltage (V)** | from §2 / §4 / §5 / §6 | `2.80 V` |
| **Suggested R_top / R_bot (Ω)** | reuse [M8160AdcCalculator.cs](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Helpers/M8160AdcCalculator.cs) reverse path (50 µA pull-high model) | `47k / 33k` |
| **Register value (dec / hex / bin)** | bypass / force field 用 | `0 / 0x00 / 0b0` |
| **Source rule** | 哪一條 spec rule 觸發 | `§2 row C3 (close-loop, 6p)` |
| **Status** | OK / Clamped / Out-of-range / Forbidden-band | colour-coded badge |

沒被影響的 field 不顯示（依 CLAUDE.md "surgical changes" 原則）。

### 7.1 UI mockup — Input panel

ASCII wireframe（左側 input dropdown stack，每個 group 用 GroupBox 包）：

```
┌─ M8160AA Input Pin Resistor Design Tool ──────────────────────────────────┐
│                                                                            │
│  ┌─ Loop type ────────────────────────┐   ┌─ Hall / Startup ─────────────┐ │
│  │  Loop type     [ Open-loop      ▼] │   │  Hall type   [ non-FR     ▼] │ │
│  │  ─ close-loop only ─                │   │  Force start [ Force      ▼] │ │
│  │  Min RPM       [ 200            ]   │   │              ─ open-loop ─    │ │
│  │  Max RPM       [ 2500           ]   │   └──────────────────────────────┘ │
│  │  rpm_ratio     [ 2.54  (auto)  ▼] │                                    │
│  └────────────────────────────────────┘   ┌─ Soft switch ────────────────┐ │
│                                            │  Soft switch [ Square     ▼] │ │
│  ┌─ Motor / Drive ────────────────────┐   │              ─ open-loop ─    │ │
│  │  SO pole       [ 4p             ▼] │   └──────────────────────────────┘ │
│  │  Standby       [ Shutdown       ▼] │                                    │
│  │  Drive mode    [ Direct PWM     ▼] │   ┌─ Soft start time ────────────┐ │
│  │  PWM polarity  [ Non-invert     ▼] │   │  ss_time     [ 2 s        ▼] │ │
│  │              ─ Direct PWM only ─    │   └──────────────────────────────┘ │
│  │  DC source     [ VSP            ▼] │                                    │
│  │              ─ DC mode only ─       │   [   Calculate   ]  [  Reset  ]  │
│  └────────────────────────────────────┘                                    │
│                                                                            │
│  ───────────────── Result DataGrid (see §7.2) ──────────────────────────── │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

Greyed-out (`─ … ─`) lines 表示該 dropdown 在當前 loop / drive mode 下 **bypass**（不可改）。Bypass 時 ComboBox `IsEnabled=false`，背景套 `ReadOnlyHighlightBackground` (theme-aware)。

### 7.2 UI mockup — Result DataGrid

Example：使用者填 `Open-loop / 4p / Shutdown / Direct PWM / Non-invert / non-FR / Force / Square` → mode = 7：

| Pin / Field        | Voltage | R_top   | R_bot   | RegValue (dec/hex/bin) | Source rule                  | Status |
|--------------------|--------:|--------:|--------:|------------------------|------------------------------|--------|
| **Mode pin**       | 5.000 V | open    | 0 Ω     | —                      | §2 row O2a                   | OK     |
| **XP1 pin**        | 4.766 V | 4.7 kΩ  | 100 kΩ  | reg_data_02[4:0] = 3   | §4 mode=7, V_high            | OK     |
| **YP pin**         | 0.156 V | 95.3 kΩ | 3.16 kΩ | reg_data_04[6:0] = 5   | §5 mode=7 + DC, V_low        | Clamped|
| **SS pin**         | 4.023 V | 19.6 kΩ | 80.6 kΩ | REG08[7:5] = 2         | §6.2 mode=7, 4p, profile A   | OK     |
| reg_data_0e[7]     |   —     |  —      |  —      | 1 / 0x01 / 0b1         | §1 force align_sw            | OK     |
| reg_data_03[7:6]   |   —     |  —      |  —      | 0 / 0x00 / 0b00        | §1 force ssw=square          | OK     |
| reg_data_0f[1]     |   —     |  —      |  —      | 0 / 0x00 / 0b0         | §1 force fr_pin_sw (non-FR)  | OK     |

Status colour code：

- **OK** — 綠色 / `SuccessForeground`
- **Clamped** — 黃色 / `WarningForeground`（4.6 % 以下落在 endpoint）
- **Out-of-range** — 紅色 / `ErrorForeground`（min/max RPM 超出 §6.3 range）
- **Forbidden-band** — 紅底白字（落在 §2 ModeBands 的 forbidden gap）

DataGrid 使用既有 [BankComparisonDialog.xaml](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Views/Dialogs/BankComparisonDialog.xaml) 的 row-template + theme-aware brush 慣例。

---

## 7.5 Calculation pseudo-code

整個 calculation flow 一次跑完，input → 完整 output rows。

### 7.5.1 Top-level entry

```
function Calculate(inputs) -> List<ResultRow>:
    rows = []

    // Step 1: classify mode (§2.1)
    mode = ClassifyMode(inputs)
    if mode == null:
        return [Row(Pin="Mode", Status=Invalid, Rule="no §2.1 row matched")]

    // Step 2: emit Mode-pin voltage (§2)
    rows.add(Row(Pin="Mode pin",
                 Voltage = ModeBands[mode].V,
                 Rule = "§2 row " + matchedRow))

    // Step 3: emit close-loop bypass / force registers (§1 bypass rules)
    if inputs.LoopType == Close:
        rows.add(RegRow("reg_data_0d[3]", 0, "§1 force Direct PWM"))
        rows.add(RegRow("reg_data_01[7]", 0, "§1 force PWM non-invert"))
        rows.add(RegRow("reg_data_0e[7]", 1, "§1 force align_sw"))
        rows.add(RegRow("reg_data_03[7:6]", 0, "§1 force ssw=square"))
    if inputs.HallType == NonFR:
        rows.add(RegRow("reg_data_0f[1]", 0, "§1 force fr_pin_sw"))

    // Step 4: XP1 pin (§4)
    rows.add(CalcXp1(inputs, mode))

    // Step 5: YP pin (§5)
    rows.add(CalcYp(inputs, mode))

    // Step 6: SS pin (§6)
    rows.add(CalcSs(inputs, mode))

    // Step 7: convert each voltage row to suggested R_top / R_bot
    for r in rows where r.Voltage != null:
        (r.Rtop, r.Rbot) = SuggestResistorPair(r.Voltage)   // reuse M8160AdcCalculator

    return rows
```

### 7.5.2 Mode classifier

```
function ClassifyMode(in) -> int? :
    if in.LoopType == Close:
        if in.Pole == 4p and in.HallType == NonFR:    return 5  // C1
        if in.Pole == 4p and in.HallType == FR:       return 6  // C2
        if in.Pole == 6p or in.Pole == 8p:            return 4  // C3
        return null

    // Open-loop
    if in.Standby == Shutdown and (in.Pole == 4p or in.Pole == 8p):
        if in.DriveMode == DirectPWM and in.PwmInvert:        return 0  // O1a
        if in.DriveMode == DirectPWM and not in.PwmInvert:    return 7  // O2a
        if in.DriveMode == DC:
            // O1b: gives mode 0
            if (in.HallType == NonFR and in.DcSource == SET) or
               (in.HallType == FR    and in.DcSource == VSP):
                return 0
            // O2b: gives mode 7
            if (in.HallType == NonFR and in.DcSource == VSP) or
               (in.HallType == FR    and in.DcSource == SET):
                return 7

    if in.Standby == Shutdown and (in.Pole == 4p or in.Pole == RD)
       and in.SoftSwitch == Deg22_5:                           return 1  // O3

    if (in.Pole == 4p or in.Pole == 8p) and in.HallType == FR:  return 2  // O4
    if (in.Pole == 4p or in.Pole == 6p) and in.HallType == NonFR: return 3 // O5

    return null
```

### 7.5.3 Voltage primitive (shared)

```
function VoltageFromField(fieldValue, useTimes4 = false) -> (V_high, V_low):
    code = (useTimes4 ? fieldValue * 4 : fieldValue) + 20
    V_high = (256 - code) * 5.0 / 256.0     // negative-slope branch
    V_low  =  code        * 5.0 / 256.0     // positive-slope branch
    return (V_high, V_low)
```

### 7.5.4 XP1 calculator

```
function CalcXp1(in, mode) -> ResultRow:
    // Open-loop branch — uses reg_data_02[4:0] with ×4 step
    if mode in {0,1,2,3,7}:
        if in.Xp1DutyPct < 4.6:
            return Row(Pin="XP1", Voltage=ChooseEndpoint(branch),
                       Status=Clamped, Rule="§3.1 below 4.6%")
        field02 = floor(in.Xp1DutyPct / 1.5625)            // 0..31
        (V_high, V_low) = VoltageFromField(field02, useTimes4=true)
        branch = SelectXp1Branch(in, mode)                  // V_high or V_low (table §4)
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_02[4:0]",
                   RegValue=field02, Rule="§4 mode=" + mode)

    // Close-loop branch — uses reg_data_06[6:0]
    if mode in {4,5,6}:
        if in.MinRpm >= 60 * in.RpmRatio:                  // sentinel: Min == max-RPM
            return Row(Pin="XP1", Voltage=ChooseEndpoint(branch),
                       Status=Clamped, Rule="§3.1 RPM at endpoint")
        field06 = round(in.MinRpm / in.RpmRatio - 59)      // §8 Q6: round vs floor
        (V_high, V_low) = VoltageFromField(field06, useTimes4=false)
        branch = SelectXp1Branch(in, mode)
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_06[6:0]",
                   RegValue=field06, Rule="§4 close-loop mode=" + mode)
```

`SelectXp1Branch(in, mode)` 直接查 §4 table 第 3 欄（V_high / V_low）。

### 7.5.5 YP calculator

```
function CalcYp(in, mode) -> ResultRow:
    if mode in {4,5,6}:
        // Close-loop: uses reg_data_07 (no ×4)
        if in.MinRpm < /* some lower bound */:
            return Row(Pin="YP", Status=OutOfRange, Rule="§5 RPM too low")
        field07 = round((in.MinRpm / in.RpmRatio - 59) / 4)
        (_, V_low) = VoltageFromField(field07, useTimes4=false)
        return Row(Pin="YP", Voltage=V_low, RegField="reg_data_07",
                   RegValue=field07, Rule="§5 close-loop mode=" + mode)

    // mode in {0,1,2,3,7} — uses reg_data_04[6:0]
    if in.Yp1DutyPct < 4.6:
        return Row(Pin="YP", Voltage=ChooseEndpoint(branch),
                   Status=Clamped, Rule="§3.1 below 4.6%")
    field04 = floor(in.Yp1DutyPct / 0.39)                  // 0..127
    (V_high, V_low) = VoltageFromField(field04, useTimes4=false)
    branch = SelectYpBranch(in, mode)                       // table §5 col 3
    return Row(Pin="YP", Voltage=branch, RegField="reg_data_04[6:0]",
               RegValue=field04, Rule="§5 mode=" + mode)
```

### 7.5.6 SS calculator

```
function CalcSs(in, mode) -> ResultRow:
    // Close-loop modes 4/5/6 — direct rpm_ratio lookup (§6.3)
    if mode in {4,5,6}:
        v = SS_TABLE_63[in.RpmRatio]                       // 8 entries
        // Validate min/max RPM fall inside row's window
        (lo, hi) = SS_TABLE_63_RANGE[in.RpmRatio]
        status = (in.MinRpm > lo and in.MaxRpm < hi) ? OK : OutOfRange
        return Row(Pin="SS", Voltage=v, Status=status, Rule="§6.3 ratio=" + in.RpmRatio)

    // Open-loop — pick profile A or B per §6.2
    profile = SelectSsProfile(mode, in.Pole)                // A / B / null
    if profile == null:
        return Row(Pin="SS", Status=Invalid, Rule="§6.2 no profile")
    v = (profile == A ? SS_TABLE_A[in.SsTime] : SS_TABLE_B[in.SsTime])
    return Row(Pin="SS", Voltage=v,
               RegField="REG08[7:5]", RegValue=in.SsTimeIndex,
               Rule="§6.2 mode=" + mode + " pole=" + in.Pole + " profile=" + profile)
```

`SS_TABLE_A` / `SS_TABLE_B` / `SS_TABLE_63` 都是 §6.1 / §6.3 寫死的 8-entry array。

### 7.5.7 Resistor suggestion

```
function SuggestResistorPair(targetV) -> (Rtop, Rbot):
    // Reuse existing M8160AdcCalculator reverse path (50 µA pull-high source).
    // Returns standard E96 resistor values bracketing the divider ratio.
    return M8160AdcCalculator.VoltageToResistorPair(targetV)
```

---

## 8. Review checklist

實作 plan 開出去前，請確認：

1. **Spec discrepancy 1** — `reg_data_0d[3]` vs `[6]` for "Direct PWM" force。
2. **Spec discrepancy 2** — Mode classifier matrix 完整（沒漏 input combo）且 row 互斥。
3. **Spec discrepancy 3** — `mode==2` SS profile 用 4p / 6p（不是 `mode==0/2/7` 繼承的 4p / 8p）。
4. **§3** — 確認單位是 V 不是 mV (256 × 0.01953125 = 5 ⇒ V)。
5. **§4** — 確認 `reg_data_02 × 4` step factor 只 apply 在 open-loop XP1（close-loop 沒有）。
6. **§5 mode 4/5/6** — `reg_data_07 = (min_rpm/rpm_ratio − 59) / 4` 要 integer，要 `floor` 還是 `Math.Round`？
7. **§7** — register row：只列 **bypass / force** 的 field，還是把使用者隱含挑的 `xp1`/`yp1`/`rp1`/`rp2`/`ss_time` 也列出來？

---

## 9. Original spec (kept for reference)

> 原始 200-line spec 故意 **不複製** 到這份 design doc 避免 drift。文字留在對話 transcript。Approve 之後 implementation plan 只引用 refactored 的 §1–§7。

---

## 10. Documentation deliverables (post-approval, 不在 plan-mode 內做)

Approve 之後：

1. 複製一份到 `docs/spec_walkthrough/M8160AA_InputPinResistorDesignTool_Spec.md`（依 Q3 answer）。
2. Implementation plan（走 `writing-plans` skill）會把 §1–§7 翻成：
   - 新 `M8160InputPinResistorViewModel : ObservableObject`
   - 新 `M8160InputPinResistorWindow.xaml`（model 仿 [M8160DesignToolWindow.xaml](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Views/Mxxxx/M8160aa/M8160DesignToolWindow.xaml)）
   - 新 static helper class for mode classifier matrix (table §2.1)
   - Sidebar button hook 在 [MxxxxViewModel.cs](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/ViewModels/MxxxxViewModel.cs)（仿既有 `OpenM8160DesignToolCommand`）
   - Reuse [M8160AdcCalculator.cs](d:/github/Motor_Test_APP/GmtTestToolWrokspace/GmtTestToolV2/Helpers/M8160AdcCalculator.cs) 做 V → R 換算

Plan-mode 不寫 code。
