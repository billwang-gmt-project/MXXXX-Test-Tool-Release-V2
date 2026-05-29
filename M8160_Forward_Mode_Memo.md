# M8160AA Design Tool — Forward Mode 技術備忘錄

## Context

M8160AA 晶片有 4 支類比腳位（**Mode / XP1 / YP / SS**）讓使用者外接 resistor divider 設定行為。Design Tool 的 **Forward 模式** 是 *App → V → R + Reg* 方向：使用者填 9 項應用條件（loop type、極數、PWM mode……），工具反推 4 支腳位該打多少電壓 + 對應的 register field 值 + 哪些 register bit 會被工具強制鎖定。

---

## Part A. 規格詳解

### A.1 輸入 → Register Field 的整體對應表

> Forward 計算共產出 **4 row 腳位電壓 + 至多 5 row Forced + 6 row mode-derived + 1~3 row XP1/YP/SS derive ≈ 11~15 row register**。

#### 怎麼讀這張表

每個 register field 依「值從哪來」分成 3 類：

| 類別 | 行為 | 使用者 UI 上看得到嗎 |
| ---- | ---- | -------------------- |
| **Forced** | 工具強制鎖定成常數，**不管使用者怎麼選都覆蓋掉** | UI 上有對應 dropdown，但 Forced 規則生效時被無視 |
| **mode-derive** | 由 mode（0..7）決定的常數 | 沒對應 UI dropdown，使用者沒得選 |
| **XP1/YP/SS derive** | 由使用者**填的數值**反推 | 有 UI 輸入欄（duty %、MinRpm/MaxRpm、ss_time） |

#### 欄位定義

| 欄位 | 意思 |
| ---- | ---- |
| **類型** | Forced / mode-derive / XP1\|YP\|SS derive 三選一 |
| **Field** | register field 名稱（例如 `source_duty`） |
| **Register / Bits** | 寫到哪個 register 的哪幾個 bit；若是常數會直接寫 `= 0`/`= 1` |
| **公式 / 來源** | 為什麼鎖這個值；或值的計算公式 |
| **觸發條件** | 這個 row 何時被加入結果（不滿足條件時這 row 不存在） |
| **資料來源 input** | 這個 field 的值由哪個使用者輸入決定（Forced 列為常數，derive 列為某輸入欄） |
| **被覆蓋的 input** | **Forced row 專用**：使用者 UI 上有此欄位可填，但 Forced 規則會把它蓋掉 |

#### 讀法範例

第一列 `source_duty` 讀作：

> 「**Loop = Close 時**，工具會把 `reg_0d[6]` 強制寫 0（force Direct PWM）；這個值是常數不來自任何輸入；使用者在 UI 上選的 **Drive mode**（Direct/DC）在 Close-loop 模式下會被忽略。」

第二列 `cmd_inv` 同理：Close-loop 時 `reg_01[7]` 強制=0，使用者選的 **PWM polarity** 被覆蓋。

對應 pseudo-code：

```text
if LoopType == Close:
    addForced("source_duty", reg_0d[6]   = 0)   // force Direct PWM     → 覆蓋 Drive mode
    addForced("cmd_inv",     reg_01[7]   = 0)   // force PWM non-invert → 覆蓋 PWM polarity
    addForced("align_sw",    reg_0e[7]   = 1)   // force align_sw       → 覆蓋 Force startup
    addForced("soft_sw",     reg_03[7:6] = 0)   // force ssw=square     → 覆蓋 Soft switch
if HallType == NonFR:
    addForced("fr_pin_sw",   reg_0f[1]   = 0)   // force fr_pin_sw      → 覆蓋 Hall type 行為
```

#### 對應表

| 類型 | Field | Register / Bits | 公式 / 來源 | 觸發條件 | 資料來源 input | 被覆蓋的 input |
|------|-------|-----------------|-------------|----------|----------------|-----------------|
| **Forced** | `source_duty` | `reg_0d[6]` = 0 | force Direct PWM | Loop = Close | （常數 0） | Drive mode |
| **Forced** | `cmd_inv` | `reg_01[7]` = 0 | force PWM non-invert | Loop = Close | （常數 0） | PWM polarity |
| **Forced** | `align_sw` | `reg_0e[7]` = 1 | force align_sw | Loop = Close | （常數 1） | Force startup |
| **Forced** | `soft_sw` | `reg_03[7:6]` = 0 | force ssw=square | Loop = Close | （常數 0） | Soft switch |
| **Forced** | `fr_pin_sw` | `reg_0f[1]` = 0 | force fr_pin_sw | Hall = non-FR | （常數 0） | Hall type 行為 |
| **mode-derive** | `fan_curve` | `reg_01[4:3]` | open-loop→2、close-loop→1 | mode 群組 | mode（隱含 Loop type） | – |
| **mode-derive** | `rpm_ratio` | `reg_01[2:0]` | open-loop→7（常數）；close-loop→使用者 RpmRatioIndex | mode | rpm_ratio (close-loop) | – |
| **mode-derive** | `icmd_sw` | `reg_06[7]` = 0 | all modes → 0 | – | （常數 0） | – |
| **mode-derive** | `auto_soft_sw` | `reg_0a[7]` = 1 | all modes → 1 | – | （常數 1） | – |
| **mode-derive** | `duty_deb_sw` | `reg_0d[7]` | mode∈{4,5,6,7}→1；else→0 | mode | mode | – |
| **mode-derive** | `low_freq_rev` | `reg_0e[5]` = 1 | all modes → 1 | – | （常數 1） | – |
| **XP1 derive (open)** | `xp1` | `reg_02[4:0]` | `floor(Xp1Duty% / 1.5625)`，clamp[0,31] | mode∈{0,1,2,3,7} | Xp1 duty (%) | – |
| **XP1 derive (open, 4.64%)** | `xp1` | `reg_02[4:0]` = 3 | 4.64% 開路 fix | `\|duty − 4.64\| < 0.005` | Xp1 duty (%) | – |
| **XP1 derive (close)** | `rp1` | `reg_06[6:0]` | `round(MinRpm / rpm_ratio − 59)`，clamp[0,127] | mode∈{4,5,6} | MinRpm + rpm_ratio | – |
| **YP derive (open)** | `yp1` | `reg_04[6:0]` | `floor(Yp1Duty% / 0.390625)`，clamp[0,127] | mode∈{0,1,2,3,7} | Yp1 duty (%) | – |
| **YP derive (open, 4.64%)** | `yp1` | `reg_04[6:0]` = 12 | 4.64% 開路 fix | `\|duty − 4.64\| < 0.005` | Yp1 duty (%) | – |
| **YP derive (close)** | `rp2` | `reg_07[7:0]` | `round(MaxRpm / rpm_ratio / 4 − 59)`，clamp[0,255] | mode∈{4,5,6} | MaxRpm + rpm_ratio | – |
| **YP derive (close)** | `yp1` | `reg_04[6:0]` = 12 | 常數 ≈ 4.6%（mode 4/5/6 固定） | mode∈{4,5,6} | （常數 12） | – |
| **YP derive (close)** | `yp2` | `reg_05[6:0]` = 127 | 常數 ≈ 100% | mode∈{4,5,6} | （常數 127） | – |
| **SS derive (open)** | `ss_time` | `reg_08[7:5]` | 直接寫使用者選的索引，clamp[0,7] | mode∈{0,1,2,3,7} | ss_time | – |
| **SS derive (close)** | (隱含) | `reg_01[2:0]`（rpm_ratio） | 不另寫，由 rpm_ratio row 涵蓋 | mode∈{4,5,6} | rpm_ratio | – |

### A.2 Pin 電壓計算流程圖（合併 mode 分流）

```
判定 mode ─┬─→ mode ∈ {0,1,2,3,7}  (open-loop)
           │       │
           │       ├─ Mode V       = ModeVoltageTable[mode]
           │       │
           │       ├─ XP1 V        = Xp1HighBranch ? V_high : V_low
           │       │   field       = (Xp1Duty=4.64%) ? 3
           │       │                 : floor(Xp1Duty / 1.5625)   (5-bit)
           │       │   code        = field × 4 + 20             ←  ×4!
           │       │
           │       ├─ YP  V        = YpHighBranch  ? V_high : V_low
           │       │   field       = (Yp1Duty=4.64%) ? 12
           │       │                 : floor(Yp1Duty / 0.390625) (7-bit)
           │       │   code        = field × 1 + 20             ←  no ×4
           │       │
           │       └─ SS  V        = SsProfile(mode,pole)='A'?A:B
           │           ss_time idx = 使用者選的 0..7              (3-bit)
           │           V           = Profile[ss_time idx]
           │
           └─→ mode ∈ {4,5,6}      (close-loop)
                   │
                   ├─ Mode V       = ModeVoltageTable[mode]
                   │
                   ├─ XP1 V        = Xp1HighBranch ? V_high : V_low
                   │   rp1         = round(MinRpm / rpm_ratio − 59)  (7-bit, clamp[0,127])
                   │   code        = rp1 + 20                        ←  no ×4
                   │
                   ├─ YP  V        = rp2 × 5 / 256                   ←  Allen direct
                   │   rp2         = round(MaxRpm / rpm_ratio / 4 − 59) (8-bit, clamp[0,255])
                   │   yp1=12 (≈4.6%)  yp2=127 (≈100%)
                   │
                   └─ SS  V        = CloseLoopSsTable[RpmRatioIndex]
                       status      = (Min∈range AND Max∈range) ? Ok : OutOfRange
```

### A.3 Pseudo-code（端對端流程）

```
function Calculate(inputs) -> ResultRow[]:

    // ─ Step 1: mode classification ─
    mode = ClassifyMode(inputs)              // C1..O5 從上往下 first match
    if mode == null: return Invalid

    // ─ Step 2: Mode pin V (table lookup) ─
    addPin("Mode", ModeVoltageTable[mode])

    // ─ Step 2.5a: forced register ─
    if inputs.LoopType == Close:
        addReg("source_duty", reg_0d[6]   = 0)
        addReg("cmd_inv",     reg_01[7]   = 0)
        addReg("align_sw",    reg_0e[7]   = 1)
        addReg("soft_sw",     reg_03[7:6] = 0)
    if inputs.HallType == NonFR:
        addReg("fr_pin_sw",   reg_0f[1]   = 0)

    // ─ Step 2.5b: mode-derived 6 個常數 ─
    isOL = mode in {0,1,2,3,7}
    addReg("fan_curve",    reg_01[4:3] = isOL ? 2 : 1)
    addReg("rpm_ratio",    reg_01[2:0] = isOL ? 7 : inputs.RpmRatioIndex)
    addReg("icmd_sw",      reg_06[7]   = 0)
    addReg("auto_soft_sw", reg_0a[7]   = 1)
    addReg("duty_deb_sw",  reg_0d[7]   = (mode in {4,5,6,7}) ? 1 : 0)
    addReg("low_freq_rev", reg_0e[5]   = 1)

    // ─ Step 3: XP1 ─
    xp1High = SelectXp1HighBranch(inputs, mode)
    if mode in {0,1,2,3,7}:                          // open-loop
        if abs(inputs.Xp1DutyPercent - 4.64) < 0.005:
            addReg("xp1", reg_02[4:0] = 3)
            addPin("XP1", xp1High ? 5.0 : 0.0, status=OpenCircuit)
        else:
            field = clamp(floor(inputs.Xp1DutyPercent / 1.5625), 0, 31)
            (vH, vL) = VoltageFromField(field, times4=true)   // code=field*4+20
            addReg("xp1", reg_02[4:0] = field)
            addPin("XP1", xp1High ? vH : vL)
    else:                                            // close-loop {4,5,6}
        ratio = Rpm1RatioTable[inputs.RpmRatioIndex]
        field = clamp(round(inputs.MinRpm / ratio - 59), 0, 127)   // ← 含 −59 offset (2026-05-08 spec)
        (vH, vL) = VoltageFromField(field, times4=false)      // code=field+20
        addReg("rp1", reg_06[6:0] = field)
        addPin("XP1", xp1High ? vH : vL)

    // ─ Step 4: YP ─
    if mode in {4,5,6}:                              // close-loop（Allen direct）
        rp2 = clamp(round(inputs.MaxRpm / ratio / 4 - 59), 0, 255) // ← 含 −59 offset (2026-05-08 spec)
        addReg("rp2", reg_07[7:0] = rp2)
        addReg("yp1", reg_04[6:0] = 12)              // 常數 ≈ 4.6% (2026-05-08 從 26 改 12)
        addReg("yp2", reg_05[6:0] = 127)             // 常數 ≈ 100%
        addPin("YP", rp2 * 5 / 256.0)                // 無 +20 offset (Allen direct)
    else:                                            // open-loop
        ypHigh = SelectYpHighBranch(inputs, mode)
        if abs(inputs.Yp1DutyPercent - 4.64) < 0.005:
            addReg("yp1", reg_04[6:0] = 12)
            addPin("YP", ypHigh ? 5.0 : 0.0, status=OpenCircuit)
        else:
            yp1 = clamp(floor(inputs.Yp1DutyPercent / 0.390625), 0, 127)
            (vH, vL) = VoltageFromField(yp1, times4=false)
            addReg("yp1", reg_04[6:0] = yp1)
            addPin("YP", ypHigh ? vH : vL)

    // ─ Step 5: SS ─
    if mode in {4,5,6}:                              // close-loop
        ssV = CloseLoopSsTable[inputs.RpmRatioIndex].SsVoltage
        ratio = Rpm1RatioTable[inputs.RpmRatioIndex]
        rp1Upper = 167 * ratio                        // RP1 7-bit saturation 含 ADC −20 shift (MinRpm 上限)
        rp2Upper = 1079 * ratio                       // RP2 8-bit saturation (MaxRpm 上限)
        minLower = 59 * ratio
        minOk = inputs.MinRpm > minLower and inputs.MinRpm < rp1Upper
        maxOk = inputs.MaxRpm > minLower and inputs.MaxRpm < rp2Upper
        status = (minOk and maxOk) ? Ok : OutOfRange  // 2026-05-08 spec：RP1/RP2 雙 bound
        addPin("SS", ssV, status)                    // 不寫 ss_time
    else:                                            // open-loop
        profile = SelectSsProfile(mode, inputs.SoPole)   // 'A' or 'B'
        ss = clamp(inputs.SsTimeIndex, 0, 7)
        V = (profile == 'A') ? SsProfileA[ss] : SsProfileB[ss]
        addReg("ss_time", reg_08[7:5] = ss)
        addPin("SS", V)
```

### A.4 V_high vs V_low 分支選擇表（逐 mode 列出）

XP1 分支判定：

| mode | V_high (true) 條件 | V_low (false) 條件 |
|:----:|-------------------|-------------------|
| 0 | non-FR + SET | FR + VSP |
| 1 | non-FR | FR |
| 2 | Standby = Shutdown | Standby = Minimum |
| 3 | Standby = Shutdown | Standby = Minimum |
| 4 | pole = 6p | pole = 8p |
| 5 | Standby = Shutdown | Standby = Minimum |
| 6 | Standby = Shutdown | Standby = Minimum |
| 7 | non-FR + VSP | FR + SET |

YP 分支判定：

| mode | V_high (true) 條件 | V_low (false) 條件 |
|:----:|-------------------|-------------------|
| 0 | Direct + Invert | DC mode |
| 1 / 2 / 3 | ForceStartup = Non-force | ForceStartup = Force |
| 7 | Direct | DC mode |
| 4 / 5 / 6 | – | 永遠 V_low（Allen direct convention） |

### A.5 SS Profile 選擇表

| mode | pole | profile |
|:----:|:----:|:-------:|
| 0 / 2 / 7 | 4p | A |
| 0 / 2 / 7 | 8p | B |
| 1 | 4p | A |
| 1 | RD | B |
| 3 | 4p | A |
| 3 | 6p | B |
| 4 / 5 / 6 | – | 不適用，走 close-loop CloseLoopSsTable 查表 |

### A.6 SS 兩條 Profile 與 Close-loop 查表

```
ss_time idx  →    0      1      2      3      4      5      6      7
                0.5s    1s     2s     4s     6s     8s    10s    12s
Profile A    → 5.000  4.257  4.023  3.789  3.554  3.320  3.085  2.851  V
Profile B    → 0.000  0.722  0.957  1.191  1.425  1.660  1.894  2.128  V

CloseLoopSsTable (RpmRatioIndex 0..7):
  0  ratio=2.54   範圍 149.86..2740.66    SS=0.00 V
  1  ratio=3.72   範圍 219.48..4013.88    SS=0.92 V
  2  ratio=5.59   範圍 329.81..6031.61    SS=1.55 V
  3  ratio=8.3    範圍 489.70..8955.70    SS=2.18 V
  4  ratio=12.2   範圍 719.80..13163.80   SS=2.80 V
  5  ratio=17.79  範圍 1049.61..19195.41  SS=3.43 V
  6  ratio=25.93  範圍 1529.87..27978.47  SS=4.06 V
  7  ratio=37.96  範圍 2239.64..40958.84  SS=5.00 V
```

> 巧合（不是巧合）：Close-loop SS 電壓表 = `ModeVoltageTable[idx]`，因為兩者都是 8-step 0~5V 線性切割。

> ⚠️ **2026-05-08 spec：`MinRpm`/`MaxRpm` 欄位 = RP2 寬鬆範圍（59·ratio .. 1079·ratio），不是 SS 物理 range。** 硬體 spec 表（[RPM ratio bound 表](M8160AA_InputPinResistorDesignTool_Spec.md#63-close-loop-ss-對照表)）揭露 close-loop 實際有兩種上限：
>
> - **RP1 (7-bit, MinRpm)**：`(59·ratio, 167·ratio)` — 含 ADC −20 階 shift；例如 ratio=2.54 → (149.86, 424.18)
> - **RP2 (8-bit, MaxRpm)**：`(59·ratio, 1079·ratio)` — 例如 ratio=2.54 → (149.86, 2740.66)
>
> 表中 `MaxRpm` 欄沿用舊單 bound 寫法（= RP2 max），`AutoPickRpmRatio`（§A.7）與 `CalcSs` 的 `OutOfRange` 旗標已改用雙 bound（不再讀此欄）。`SsVoltage` 欄仍以 ratio idx 作為查表索引使用。

### A.7 Auto rpm_ratio 演算法

當使用者沒鎖定 rpm_ratio（auto = on）：

```
AutoPickRpmRatio(MinRpm, MaxRpm):
    ratios = M8160SpeedCurveCalculator.Ratios          // [2.54, 3.72, 5.59, ..., 37.96]
    rp1Factor = 167    // RP1 7-bit upper-bound coefficient（含 ADC −20 階 shift；(128 − 20) + 59 = 167）
    rp2Factor = 1079   // RP2 8-bit upper-bound coefficient
    minFactor = 59     // RP1/RP2 lower-bound coefficient

    // pass 1：雙 bound — MinRpm fit 在 RP1 範圍、MaxRpm fit 在 RP2 範圍
    for idx in 0..7:
        ratio = ratios[idx]
        minOk = MinRpm > minFactor*ratio and MinRpm < rp1Factor*ratio
        maxOk = MaxRpm > minFactor*ratio and MaxRpm < rp2Factor*ratio
        if minOk and maxOk: return idx

    // pass 2：fallback，找能包 Max（RP2 上限）的最小 ratio
    for idx in 0..7:
        if MaxRpm < rp2Factor * ratios[idx]: return idx

    // pass 3：都不行，回傳最大 ratio
    return 7
```

觸發點：使用者改 MinRpm 或 MaxRpm 時，若 auto = on 則重新挑。

**為何不再用 `CloseLoopSsTable` 的 `(lo, hi)`：** 該表的 `hi` 欄 = RP2 max（1079·ratio），漏掉 RP1 max（167·ratio < RP2 max）。舊版邏輯讓 (MinRpm, MaxRpm) 在 RP1 已 saturate 時仍被當成「fit」，硬體實際 MinRpm 操作點被 clamp 到 RP1 field 上限。雙 bound 對齊既存 [`M8160SpeedCurveCalculator.SelectRpmRatio`](../../../Helpers/M8160SpeedCurveCalculator.cs)（已正確使用 `Rp1Factor` / `Rp2Factor`）。

**雙 bound 預算表（pass 1 接受區間）：** RP1 區間 `MinRpm ∈ (59·ratio, 167·ratio)`（含 ADC −20 階 shift）；RP2 區間 `MaxRpm ∈ (59·ratio, 1079·ratio)`。

| idx | rpm_ratio | RP1 區間（MinRpm ∈） | RP2 區間（MaxRpm ∈） |
| :-: | :-: | :-: | :-: |
| 0 | 2.54 | (149.86, 424.18) | (149.86, 2740.66) |
| 1 | 3.72 | (219.48, 621.24) | (219.48, 4013.88) |
| 2 | 5.59 | (329.81, 933.53) | (329.81, 6031.61) |
| 3 | 8.30 | (489.70, 1386.10) | (489.70, 8955.70) |
| 4 | 12.20 | (719.80, 2037.40) | (719.80, 13163.80) |
| 5 | 17.79 | (1049.61, 2970.93) | (1049.61, 19195.41) |
| 6 | 25.93 | (1529.87, 4330.31) | (1529.87, 27978.47) |
| 7 | 37.96 | (2239.64, 6339.32) | (2239.64, 40958.84) |

> Pass 1 從 idx=0 開始往上掃，第一個同時滿足 `MinRpm ∈ RP1` 且 `MaxRpm ∈ RP2` 的 idx 即為結果。若無 idx 同時通過 → 進入 pass 2（找 `MaxRpm < 1079·ratio` 的最小 idx）；pass 2 仍不命中 → pass 3 回傳 idx=7。

**演算法步驟對照（含三個 pass）：**

| Pass | 目的 | 條件 | 命中行為 |
| --- | --- | --- | --- |
| 1 | 嚴格雙 bound | `MinRpm ∈ RP1[idx]` **且** `MaxRpm ∈ RP2[idx]` | 回傳第一個命中的 idx |
| 2 | Fallback：只保 MaxRpm | `MaxRpm < 1079·ratios[idx]` | 回傳第一個命中的 idx（MinRpm 可能在 RP1 saturate） |
| 3 | 全失敗 | — | 回傳 idx=7（最大 ratio） |

**驗證範例：**

| (MinRpm, MaxRpm) | 嘗試過程 | 結果 |
| --- | --- | --- |
| (500, 4000)  | idx=0：500 > 424.18（RP1 上界 fail）→ idx=1：500 ∈ (219.48, 621.24) ✓ 且 4000 ∈ (219.48, 4013.88) ✓ | **idx=1** |
| (1000, 4000) | idx=0：1000 > 424.18 fail → idx=1：1000 > 621.24 fail → idx=2：1000 ∈ (329.81, 933.53)... 1000 > 933.53 fail → idx=3：1000 ∈ (489.70, 1386.10) ✓ 且 4000 ∈ (489.70, 8955.70) ✓ | **idx=3** |
| (600, 2500)（觸發舊 bug） | idx=0：600 > 424.18 fail（舊邏輯誤判通過）→ idx=1：600 ∈ (219.48, 621.24) ✓ 且 2500 ∈ (219.48, 4013.88) ✓ | **idx=1** |

### A.8 4.64% 開路語意

XP1 / YP1 dropdown 第 12 格的理論值是 `38.28 × 12 / 99 = 4.6398`，round 到 0.01% 精度後顯示為 **4.64%**，這個值代表「腳位開路、不接外部 divider」（design-time 的省略電阻設定，*不是* 運行 clamp）。

- 容差：`abs(duty − 4.64) < 0.005`
- XP1 開路 → `xp1 = 3`、TargetVoltage = `xp1High ? 5.0 : 0.0`
- YP1 開路 → `yp1 = 12`、TargetVoltage = `ypHigh ? 5.0 : 0.0`
- Status = `OpenCircuit`（藍底淺藍字，UI 顯示 "Open circuit / no resistor"）
- 兩個 branch 反推皆走 fix-H/fix-L → 同 register field，所以 Forward / Reverse Compare 在開路情境下仍 match。

> **與 close-loop field clamp 區分**：close-loop（mode 4/5/6）的 `rp1` / `rp2` 經 `clamp(field, 0, 127)` / `clamp(field, 0, 255)` 截斷至 7/8-bit 上下限。此 clamp **靜默發生、不產生 status flag**（無 `Status = Clamped`），與 4.64% 開路無關。`Clamped` 為預留 enum 值（DataGrid 樣式已 wire），目前 Forward 計算路徑不會實際 set 此 status。

### A.9 與 Reverse 的對齊點

#### A.9.1 Derive 公式對齊（6 組 derive field）

下表為 Forward 與 Reverse 兩端 derive field 的計算規則對照。**「兩端公式互為逆運算」僅適用於 ADC 推導路徑**（rp1/rp2/yp1 close-loop 正常分支、xp1/yp1 open-loop 正常分支）；fix / 端點 / mode 不命中等分支則屬於 hardcode 對齊（兩端值相同但不互為運算逆元）。

| # | Field | Forward 行為 | Reverse 行為 | 對齊方式 |
| --- | --- | --- | --- | --- |
| 1 | **rp1** `reg_06[6:0]` | mode∈{4,5,6}：`field = clamp(round(MinRpm/rpm_ratio − 59), 0, 127)`（**含 −59 offset**，2026-05-08 spec）；其他 mode 不寫此 row | mode∈{4,5,6}：xp1Fix → 1；否則 `field = adc1[6:0] − 20` 或 `(256−adc1)[6:0] − 20`（依 xp1V 大於/小於 2.5V 切支線）。**mode∉{4,5,6} → reset value 64** | close-loop **不對稱**：Forward 含 `−59`、Reverse 含 `−20`，兩端不再互為逆運算。Forward 寫的 pin V = `(field+20) × 5/256`（field 已減 59），Reverse 反推 `field = adc − 20`，因此 Compare 視窗 rp1 會出現固定 ±59 register field 差異（spec 容許）；xp1Fix 分支見 #5；open-loop Forward 缺 row → Compare 由 Reverse `64` expected 補齊 |
| 2 | **rp2** `reg_07[7:0]` | mode∈{4,5,6}：`field = clamp(round(MaxRpm/rpm_ratio/4 − 59), 0, 255)`（**含 −59 offset**，2026-05-08 spec）；其他 mode 不寫此 row | mode∈{4,5,6}：ypFix → **226**；否則 `field = adc2[7:0]`（直接 8-bit、無 −20）。**mode∉{4,5,6} → 0** | close-loop **不對稱**：Forward 含 `−59`、Reverse `rp2 = adc2`（直接相等，無 offset），兩端不再互為逆運算。Forward pin V = `rp2 × 5/256`（Allen direct，**無 +20**），對應 Reverse `yp_pin[7:0] = reg_07[7:0]`。Compare 視窗 rp2 會出現固定 ±59 register field 差異（spec 容許）；ypFix 分支見 #6；open-loop Forward 缺 row → Compare 由 Reverse `0` expected 補齊。**邊界注意**：當 Forward field ≥ 246（pin V ≥ 4.805V，命中 fix-H 門檻）或 field ≤ 9（pin V ≤ 0.176V，命中 fix-L 門檻），Reverse 會改走 #6 的 ypFix 分支 → 226，導致 Compare 在 ±59 之外再疊加 fix 邊界 mismatch（詳見「邊界 mismatch 區間」）。**RPM range 預先檢查**：`AutoPickRpmRatio` / `CalcSs` OutOfRange 已用 RP1/RP2 雙 bound 過濾掉 RP1 saturation 場景（`MinRpm < 167·ratio`，含 ADC −20 shift，2026-05-11 從 187 改為 167），但 RP2 fix-H/fix-L 仍可能觸發（與 RP1 飽和無關，是 ADC 端電壓邊界） |
| 3 | **yp1** `reg_04[6:0]`（close-loop 常數） | mode∈{4,5,6}：固定寫 `12` (≈4.6%，2026-05-08 從 26 改 12) | mode∈{4,5,6}：固定 `12` (2026-05-11 同步從 26 改 12) | ✓ 對齊：兩端 hardcode 同為 12 (≈4.6%)，Compare round-trip 對齊 |
| 4 | **yp2** `reg_05[6:0]` | **僅** mode∈{4,5,6} 寫 `127`；其他 mode 不寫此 row | **all modes** 寫 `127` | close-loop 兩端皆 hardcode 127；open-loop Forward 缺 row → Compare 由 Reverse `127` expected 補齊（duty 99.61% ≈ 100%） |
| 5 | **xp1 / rp1 fix**（4.64% open circuit） | mode∈{0,1,2,3,7} + `\|Xp1Duty − 4.64\| < 0.005`：寫 `xp1 = 3`，pin V = V_high → 5V / V_low → 0V | xp1V 命中 fix 端（IsFixH/IsFixL）：mode∈{0,1,2,3,7} → `xp1 = 3`、mode∈{4,5,6} → `rp1 = 1` | open-loop 兩端 hardcode 同 field=3；close-loop Forward 不觸發開路語意（無 4.64% 概念），Compare 對 `rp1 = 1` 由 Reverse 單側提供 |
| 6 | **yp1 / rp2 fix**（fix-H / fix-L 端點） | mode∈{0,1,2,3,7} + `\|Yp1Duty − 4.64\| < 0.005`：寫 `yp1 = 12`，pin V = 5V / 0V | ypV 命中 fix 端（IsFixH：code ∈ 246..255；IsFixL：code ∈ 0..9，HTML canonical code-based）：mode∈{0,1,2,3,7} → `yp1 = 12`、mode∈{4,5,6} → `rp2 = 226` | open-loop 兩端 hardcode 同 field=12；close-loop **不對稱**：Forward 不偵測 fix 端（永遠走 row 2 的 `round(MaxRpm/rpm_ratio/4 − 59)` 公式），Reverse 看到 Forward 算出的 pin V code 落在 0..9 或 246..255 會單側 hardcode 226，造成 Compare mismatch |

##### Forward / Reverse row 數不對稱現象

| Mode 範圍 | Forward 寫 rp1 | Forward 寫 rp2 | Forward 寫 yp2 | Reverse rp1 default | Reverse rp2 default |
| --- | :---: | :---: | :---: | :---: | :---: |
| Open-loop {0,1,2,3,7} | ✗ | ✗ | ✗ | 64 | 0 |
| Close-loop {4,5,6} | ✓ | ✓ | ✓ | ADC 推導 / fix=1 | ADC 推導 / fix=226 |

→ Compare 在 open-loop 對 rp1/rp2/yp2 使用 Reverse expected 值（64/0/127）作為單側 baseline。

##### 邊界 mismatch 區間（close-loop fix-H / fix-L 端點）

Close-loop 正常分支 Forward / Reverse 互為逆運算的前提是「Forward 算出的 pin V 落在 Reverse 的 linear 視窗內」。當 Forward 推出的電壓踩到 fix 端門檻時，Reverse 會切到 fix 分支 hardcode，導致 Compare mismatch。**這不是 calculator bug，是 Reverse fix 偵測機制的副作用**：

| Field | Forward 公式 | Forward fix-H 觸發條件 | Forward fix-L 觸發條件 |
| --- | --- | --- | --- |
| **rp1** `reg_06[6:0]` | `field = round(MinRpm/rpm_ratio − 59)`，pin V = `(field+20) × 5/256` | `field ≥ 226` → code ≥ 246 → Reverse `xp1Fix → 1` | `field ≤ 0`（clamp 下限）+ `(0+20) × 5/256 = 0.391V` → 無法觸發（fix-L 為 code ∈ 0..9，code 20 在 linear band） |
| **rp2** `reg_07[7:0]` | `field = round(MaxRpm/rpm_ratio/4 − 59)`，pin V = `field × 5/256`（Allen direct，無 +20） | `field ≥ 246` → code ≥ 246 → Reverse `ypFix → 226` | `field ≤ 9` → code ≤ 9 → Reverse `ypFix → 226` |

**rp2 mismatch 的 MaxRpm 觸發範圍**（含 −59 offset，按 rpm_ratio 排序）：

> 觸發條件：`round(MaxRpm/rpm_ratio/4 − 59) ≥ 246` → `MaxRpm ≥ (246 + 59) × 4 × ratio = 1220 × ratio`

| RpmRatio Index | rpm_ratio | MaxRpm 觸發 fix-H 門檻（≥ 1220 × ratio） |
| ---: | ---: | ---: |
| 0 | 2.54 | ≥ 3,099 |
| 1 | 3.72 | ≥ 4,539 |
| 2 | 5.59 | ≥ 6,820 |
| 3 | 8.3 | ≥ 10,126 |
| 4 | 12.2 | ≥ 14,884 |
| 5 | 17.79 | ≥ 21,704 |
| 6 | 25.93 | ≥ 31,635 |
| 7 | 37.96 | ≥ 46,311 |

> 例：mode=5（close-loop 4p NonFR）+ ratio=2.54 + MaxRpm=3100 → `field = round(3100/2.54/4 − 59) = round(304.13 − 59) = round(245.13) = 245`（剛好在邊界內）；MaxRpm=3110 → `field = round(306.10 − 59) = 247 ≥ 246` → pin V = 4.824V → Reverse 走 ypFix → rp2=226；Forward 247 vs Reverse 226 mismatch。Compare 標 ⚠ mismatch 屬於**已知邊界限制**，與 −59 offset 造成的 ±59 系統性差異**疊加**出現。

避開方法：把 MaxRpm 設低於上表門檻（例如 ratio=2.54 時 MaxRpm < 3099），或選下一個 ratio 讓 field 落在 linear 區間。

#### A.9.2 Forced fields（5 個，Compare 容忍清單）

§A.9.1 為 derive 公式對齊；本節為 **Forced**（工具強制覆蓋使用者 UI 選擇）。Forced field 在 Compare 時被列入容忍清單（不視為 mismatch），原因是這些 register bit 不反映使用者意圖，由 Forward 工具固定寫入常數。

| # | Forced Field | Register / Bits | 觸發條件 | 被覆蓋的 input |
| --- | --- | --- | --- | --- |
| 1 | `source_duty` | `reg_0d[6]` = 0 | Loop = Close（force Direct PWM） | Drive mode |
| 2 | `cmd_inv` | `reg_01[7]` = 0 | Loop = Close（force PWM non-invert） | PWM polarity |
| 3 | `align_sw` | `reg_0e[7]` = 1 | Loop = Close（force align_sw） | Force startup |
| 4 | `soft_sw` | `reg_03[7:6]` = 0 | Loop = Close（force ssw=square） | Soft switch |
| 5 | `fr_pin_sw` | `reg_0f[1]` = 0 | Hall = non-FR | Hall type 行為 |

對應 pseudo-code 見 §A.1 的 Forced 區塊。

### A.10 關鍵常數

| 常數 | 值 | 用途 |
|------|----|----|
| `Rpm1RatioTable[0..7]` | `[2.54, 3.72, 5.59, 8.3, 12.2, 17.79, 25.93, 37.96]` | rpm_ratio dropdown |
| `Xp1RegDutyPerBit` | `1.5625` (= 50 / 32) | open-loop XP1 每 bit duty% |
| `YpRegDutyPerBit` | `0.390625` (= 50 / 128) | open-loop YP1 每 bit duty% |
| `VPerStep` | `5.0 / 256` ≈ `0.01953125 V` | ADC 每 step 解析度 |

### A.10.1 `VoltageFromField` — register field → pin V 換算

`M8160ForwardCalculator.VoltageFromField` 是 Forward 計算的核心電壓換算函式：給定一個 register field 值，回傳該腳位的 **(V_high, V_low)** 兩種候選電壓，再由各 pin 的 branch 判定（§A.4）挑其中一個當 target voltage。

```csharp
public static (double VHigh, double VLow) VoltageFromField(int field, bool times4)
{
    int code = (times4 ? field * 4 : field) + 20;   // ← 共用 +20 offset；times4 只用在 open-loop XP1
    return (VoltageHigh(code), VoltageLow(code));
}

// 兩半段電壓公式（§3）：
//   VoltageHigh(code) = (256 − code) × 5 / 256
//   VoltageLow(code)  =  code        × 5 / 256
```

#### 三個關鍵設計點

| 點 | 說明 |
| --- | --- |
| **`+20` offset** | code 永遠先加 20 才換算電壓。對應 spec §11 的 `+20` convention，也對齊 Reverse `xp1_pin[*] − 20 = reg[*]`（`M8160AdcCalculator.cs:586`）。**例外**：close-loop YP（`rp2`）走 Allen direct，**不經過 `VoltageFromField`**，直接 `rp2 × 5/256`（無 +20，見 §A.12 #2）。 |
| **`times4` 參數** | 只有 **open-loop XP1**（`reg_02[4:0]`，5-bit）傳 `true`。因為 5-bit field 要 ×4 才能對齊 8-bit ADC code 的滿刻度。其餘呼叫（open-loop YP `reg_04[6:0]` 7-bit、close-loop XP1 `rp1` `reg_06[6:0]` 7-bit）皆 `times4=false`。 |
| **回傳兩個值** | V_high / V_low 是同一 code 在 ADC 高半段 / 低半段的兩種接法電壓。實際採用哪一個由 `SelectXp1HighBranch` / `SelectYpHighBranch`（§A.4）依 mode + 子條件決定，`VoltageFromField` 本身不做 branch 判定。 |

#### 三個呼叫點（對應 §A.3 pseudo-code）

| 呼叫點 | mode | field | `times4` | code 公式 | 程式位置 |
| --- | --- | --- | :---: | --- | --- |
| open-loop XP1 | {0,1,2,3,7} | `xp1` `reg_02[4:0]` | `true` | `field × 4 + 20` | `M8160ForwardCalculator.cs:365` |
| open-loop YP | {0,1,2,3,7} | `yp1` `reg_04[6:0]` | `false` | `field + 20` | `M8160ForwardCalculator.cs:480` |
| close-loop XP1 | {4,5,6} | `rp1` `reg_06[6:0]` | `false` | `field + 20` | `M8160ForwardCalculator.cs:324` |

> close-loop YP（`rp2`）刻意**不在**此表——它走 Allen direct（`rp2 × 5/256`），是 `VoltageFromField` 的唯一例外路徑。

#### 數值範例（取自 Part B 端對端範例）

```text
open-loop XP1：field = 12, times4 = true
  code   = 12 × 4 + 20 = 68
  V_high = (256 − 68) × 5 / 256 = 3.672 V    ← mode=7 走 V_high branch
  V_low  =  68         × 5 / 256 = 1.328 V

open-loop YP：field = 26, times4 = false
  code   = 26 + 20 = 46
  V_high = (256 − 46) × 5 / 256 = 4.102 V    ← mode=7 走 V_high branch
  V_low  =  46        × 5 / 256 = 0.898 V
```

### A.11 結果狀態旗標

Forward 結果共定義 6 種狀態旗標，**2026-05-11 起 Forbidden 旗標已啟用**（之前為預留，現實際產生於 XP1/YP pin V 落在 ADC step 116..139 forbidden band 時，即 HTML canonical 中央 gap）。Clamped 仍為預留值。

| 狀態 | 顏色 | 觸發條件 | 實際使用 |
| ---- | ---- | -------- | :---: |
| Ok | 綠（預設） | 計算正常 | ✓ |
| OpenCircuit | 藍底淺藍字 (`#1B3A5A` / `#90CAF9`) | XP1/YP1 duty = 4.64%（design-time 開路） | ✓ |
| OutOfRange | 紅底白字 (`#7A1B1B` / White) | close-loop SS：RP1/RP2 雙 bound 任一不滿足（`MinRpm ∈ (59·ratio, 167·ratio)`（含 ADC −20 shift）且 `MaxRpm ∈ (59·ratio, 1079·ratio)`，2026-05-11 spec） | ✓ |
| Invalid | 紅底白字 (`#7A1B1B` / White) | SS open-loop profile 找不到（mode + pole 組合無對應） | ✓ |
| **Forbidden** | 紅底白字 (`#7A1B1B` / White) | **XP1/YP pin V 落在 ADC 中央 forbidden gap**（HTML canonical step 116..139 ≈ 2.27..2.72V）。具體觸發 field 區間：rp1（close-loop, code = field+20） ∈ [96, 119]、rp2（close-loop, Allen direct, code = field） ∈ [116, 139]、xp1（open-loop, ×4, code = field×4+20） ∈ [24, 29]、yp1（open-loop, code = field+20） ∈ [96, 119]。Pin Summary 對應 row 標 Forbidden（紅）；整體 `IsValid = false`。SS pin 不會落在 forbidden（profile 表 + close-loop 表都已避開）。2026-05-11 起啟用此旗標；2026-05-25 起邊界對齊 HTML canonical SEG[]（自 120..136 改為 116..139） | ✓ |
| Clamped | 黃底淺黃字 (`#5A4A1B` / `#FFE082`) | （預留）目前無觸發路徑 | ✗ dead |

> **註**：Reverse 模式的「Forbidden band」概念走 `RailStatus.Forbidden` + ViewModel 的 `IsModeForbidden` / `IsXp1Forbidden` 等屬性，與 Forward 的 `M8160ForwardStatus.Forbidden` **同源（共用 `M8160AdcCalculator.RailForbiddenBands` 常數）但走不同 UI 通道**——Forward 是 Pin Summary row Status 欄、Reverse 是 ViewModel 旗標屬性。
>
> **AutoPickRpmRatio 設計決策（2026-05-11）**：auto 選擇 rpm_ratio **不**偏好 forbidden-band-free 的 ratio——使用者直覺優先，回傳最小雙 bound 通過的 ratio 即可。若該 ratio 算出的 rp1/rp2 field 落在 forbidden 區，由 Pin Summary 的 Forbidden 旗標（紅）警示使用者，由使用者自行決定是否手動調整 MinRpm/MaxRpm 或關掉 auto 換 ratio。設計理念：auto 不要偷偷換掉使用者輸入的 RPM 意圖；風險呈現比風險規避優先。

### A.12 容易踩到的坑

1. **×4 只在 open-loop XP1**：`reg_02[4:0]` 是 5-bit field，要 ×4 才能對齊 8-bit ADC code。`reg_06[6:0]` 已是 7-bit，不要再 ×4。
2. **Close-loop YP 用 Allen direct**：`pin V = rp2 × 5/256`，**沒有 +20 offset**。和 XP1/open-loop YP 不同。
3. **Close-loop YP 公式用 MaxRpm**（不是 MinRpm；rp2 是上限）。設計原稿（[Allen Spec §10](M8160AA_InputPinResistorDesignTool_Spec.md)）寫 `min_rpm` 是 typo；現行實作已修正為 `max_rpm`（commit `33fcc857`，詳見 §C.1 #2 / §C.3 #2）。
4. **Close-loop 不寫 ss_time**：rpm_ratio index 已隱含於 `reg_01[2:0]`，不另寫 `reg_08[7:5]`。
5. **rpm_ratio dropdown 顯示 ratio 值不是 index**（例如 `2.54`、`3.72` 而非 0、1）。
6. **fan_curve 在 open-loop = 2 (OOO)、close-loop = 1 (CCO)**：常數 derive，不是看使用者選的 4 個 speed curve UserControl。Forward 工具不直接決定 COO/COP。
7. **Compare 容忍清單**：5 個 forced field 在 Compare 時納入容忍，避免兩端硬體欄位不對等時誤報 mismatch。

---

## Part B. 端對端驗證範例

完整範例輸入（11 個應用條件 + 3 個數值欄位）：

```text
LoopType        = Open
SoPole          = P4
Standby         = Shutdown
DriveMode       = DirectPwm
PwmPolarity     = NonInvert
HallType        = NonFR
ForceStartup    = Force         (mode=7 不看此欄位)
SoftSwitch      = Square        (mode=7 不看此欄位)
Xp1DutyPercent  = 20.31         (open-loop only)
Yp1DutyPercent  = 10.16         (open-loop only)
SsTime          = 2 (idx)       (對應 2s)
```

預期計算結果：

```text
Mode 分類：命中 §2.1 row O2a (poleIs48 + Shutdown + DirectPwm + NonInvert) → mode = 7

Mode pin = 5.000 V                  (ModeVoltageTable[7])

XP1 pin: SelectXp1HighBranch(mode=7) = !(FR && Vsp) = !(false && false) = true → V_high
  field = floor(20.31 / 1.5625) = 12               (clamp [0,31])
  code  = 12 × 4 + 20 = 68                         (×4 for 5-bit)
  V_high = (256 - 68) × 5 / 256 = 188 × 5 / 256
        = 3.672 V                                  ← XP1 pin

YP pin: SelectYpHighBranch(mode=7) = (DriveMode == DirectPwm) = true → V_high
  field = floor(10.16 / 0.390625) = 26             (clamp [0,127])
  code  = 26 + 20 = 46                             (no ×4 for 7-bit)
  V_high = (256 - 46) × 5 / 256 = 210 × 5 / 256
        = 4.102 V                                  ← YP pin

SS pin: SelectSsProfile(mode=7, P4) = 'A'
  V = SsProfileA[2] = 4.023 V                      ← SS pin
```

DataGrid 應顯示 row 數量：

- 4 row 腳位電壓（Mode / XP1 / YP / SS）
- 1 row Forced（non-FR → `fr_pin_sw = 0`；open-loop 不觸發 4 個 close-loop forced）
- 6 row mode-derived 常數（fan_curve=2, rpm_ratio=7, icmd_sw=0, auto_soft_sw=1, duty_deb_sw=1, low_freq_rev=1）
- 3 row XP1/YP/SS derive（xp1=12, yp1=26, ss_time=2）

→ 共 14 row。

**Compare 驗證**：把計算出的 4 個 pin V 灌回 Reverse 模式輸入欄位，跑 Reverse → 應該得到相同的應用條件 + register field 值。Forced 清單中的 5 個 field（§A.9.2）在 mismatch 時用淡色標示而非報錯。

---

## Part C. 設計原稿 vs 現行實作對照（Refactored Pseudo-code）

設計者（Allen）原始 pseudo-code 出自 [`M8160AA_InputPinResistorDesignTool_Spec.md`](M8160AA_InputPinResistorDesignTool_Spec.md) §10。本節將其重寫為與現行實作一致的版本，並在每個分支標註 **「設計原稿」** 與 **「現行實作」** 的差異。

### C.1 重大差異總覽

> **讀法**：commit 欄填具體 hash 者，表示現行實作與設計原稿仍有 spec-required 差異；填 `6c9b9b89` 者，表示 2026-05-08 spec 變動造成（可能是「重新對齊」或「再度偏離」設計原稿，視 #1 / #2 / #3 / #6 各自情境而定）。

| # | 主題 | 設計原稿 | 現行實作 | 對應 commit |
| --- | --- | --- | --- | --- |
| 1 | Close-loop XP1 公式 | `reg_06[6:0] = min_rpm/rpm_ratio − 59` | `rp1 = round(min_rpm / rpm_ratio − 59)`（**含 −59 offset**，**對齊設計原稿**；歷史 commit `33fcc857` 曾為了與 Reverse round-trip 對齊而移除，2026-05-08 重新加回） | `6c9b9b89` |
| 2 | Close-loop YP 公式 | `reg_07 = (min_rpm/rpm_ratio − 59)/4`，pin V 用 `(reg_07 + 20) × 5/256` | `rp2 = round(max_rpm / rpm_ratio / 4 − 59)`（**改用 max_rpm**、含 −59 offset），pin V = `rp2 × 5/256`（Allen direct，**無 +20**） | `6c9b9b89` |
| 3 | Close-loop YP `yp1` / `yp2` 常數 | 未列 | `yp1 = 12 (≈4.6%)`、`yp2 = 127 (≈100%)`（mode 4/5/6 hardcoded，2026-05-08 yp1 從 26 改 12 對齊「4.6%」spec） | `6c9b9b89` |
| 4 | XP1/YP1 4.6% 開路語意 | 文字寫 4.6%；只描述 pin V，未寫 register field | 容差 `abs(duty − 4.64) < 0.005`；同時寫 `xp1 = 3`（open-loop）/ `yp1 = 12`（open-loop）對齊 Reverse fix-H/fix-L | `a3da9541` |
| 5 | Close-loop XP1 開路 | `xp1==60*rpm_ratio` 視為開路 → 5V/0V | **不存在此語意**。Close-loop 只 clamp `rp1 ∈ [0,127]`，無 4.64% 開路概念 | – |
| 6 | Close-loop SS 範圍檢查 | `min_rpm < RPM < max_rpm` 單一條件 | RP1/RP2 雙 bound：`MinRpm ∈ (59·ratio, 167·ratio)`（含 ADC −20 shift）且 `MaxRpm ∈ (59·ratio, 1079·ratio)`；越界 → `OutOfRange`（2026-05-11 spec：RP1 Max 從 187·ratio 改為 167·ratio；2026-05-08 雙 bound 引入） | `6c9b9b89` |
| 7 | `fr_pin_sw = 0` 觸發 | 開頭無條件寫（隱含預設 non-FR） | 條件式：`if HallType == NonFR` 才寫 | – |
| 8 | Mode=7 分類條件 | `(pole==4p\|8p) & shutdown & (PWM_non_invert or DC_mode)` 單條 | 拆兩條：**O2a** Direct PWM + Non-invert、**O2b** DC mode + (non-FR+VSP \|\| FR+SET) — **2026-05-28 update**：O2b 已移除（DC mode 不再支援），Forward 規則表新增 Drive mode 欄統一 Direct-PWM，見 `docs/superpowers/specs/2026-05-28-m8160-mode-classification-refactor-design.md`。 | – |
| 9 | Mode-derived 6 常數 | 完全未列 | `fan_curve` / `rpm_ratio` / `icmd_sw` / `auto_soft_sw` / `duty_deb_sw` / `low_freq_rev` 6 個 row | `225a6893` |
| 10 | XP1 duty 精度 | 4.6%（兩位小數） | **4.64%**（容差 0.005，理論值 38.28×12/99=4.6398，顯示為 4.64%） | §A.8 |
| 11 | rpm_ratio dropdown 顯示 | 未提 | 顯示 ratio 數值（如 2.54、3.72），不是 index | `0b26bb2d` / `9501c91b` |

### C.2 Refactored Pseudo-code（與現行實作對齊）

下面以設計原稿的結構為骨架，把每段改寫為現行實作的等價邏輯。**標記 ⚠ 處表示與設計原稿語意不同**。

```text
// ═══════════════════════ Inputs ═══════════════════════
// 9 項應用條件：
//   1. LoopType        ∈ {Open, Close}
//   2. SoPole          ∈ {P4, P6, P8, RD}
//   3. Standby         ∈ {Shutdown, Minimum}
//   4. DriveMode       ∈ {DirectPwm, DcMode}
//   5. PwmPolarity     ∈ {NonInvert, Invert}        (DirectPwm 才用)
//   6. DcSource        ∈ {Vsp, Set}                 (DcMode 才用)
//   7. HallType        ∈ {NonFR, FrEnable}
//   8. ForceStartup    ∈ {Force, NonForce}          (Open-loop 才用)
//   9. SoftSwitch      ∈ {Square, Deg22_5, Deg45, Sine}  (Open-loop 才用)
// Close-loop only:  MinRpm, MaxRpm, RpmRatioIndex (0..7), IsRpmRatioAuto
// Open-loop only:   Xp1DutyPercent, Yp1DutyPercent, SsTime (idx 0..7)


// ═══════════════════════ Step 1: Mode 分類 (§2.1 row C1..O5) ═══════════════════════
// 從上往下 first-match。無命中 → return Invalid。
// ⚠ 2026-05-28 update：規則表加入 Drive=DirectPwm 嚴格欄位，並加嚴 PwmPolarity/SoftSwitch
//   條件；DC mode 任意組合 fall through → null。原 O1b/O2b 已移除。

if LoopType == Close:
    if SoPole==P4 and Drive==DirectPwm and Polarity==NonInvert
       and Hall==NonFR and SoftSwitch==Square:
        mode = 5    // C1: 3.43V
    elif SoPole==P4 and Drive==DirectPwm and Polarity==NonInvert
         and Hall==FrEnable and SoftSwitch==Square:
        mode = 6    // C2: 4.06V
    elif SoPole in {P6, P8} and Standby==Shutdown and Drive==DirectPwm
         and Polarity==NonInvert and Hall==FrEnable and SoftSwitch==Square:
        mode = 4    // C3: 2.80V
    else:
        return Invalid
else:  // Open-loop
    poleIs48  = SoPole in {P4, P8}
    poleIs46  = SoPole in {P4, P6}
    poleIs4Rd = SoPole in {P4, RD}

    if poleIs48 and Standby==Shutdown and Drive==DirectPwm
       and Polarity==Invert and SoftSwitch==Square:
        mode = 0   // O1a: 0V
    elif poleIs48 and Standby==Shutdown and Drive==DirectPwm
         and Polarity==NonInvert and SoftSwitch==Square:
        mode = 7   // O2a: 5V
    elif poleIs4Rd and Standby==Shutdown and Drive==DirectPwm
         and Polarity==NonInvert and SoftSwitch==Deg22_5:
        mode = 1   // O3: 0.92V
    elif poleIs48 and Drive==DirectPwm and Polarity==NonInvert
         and Hall==FrEnable and SoftSwitch==Square:
        mode = 2   // O4: 1.55V
    elif poleIs46 and Drive==DirectPwm and Polarity==NonInvert
         and Hall==NonFR and SoftSwitch==Square:
        mode = 3   // O5: 2.18V
    else:
        return Invalid

addPin("Mode", ModeVoltageTable[mode])    // §2 [0, 0.92, 1.55, 2.18, 2.80, 3.43, 4.06, 5.00]


// ═══════════════════════ Step 2: Forced rows (§1.1) ═══════════════════════
// ⚠ 設計原稿把 fr_pin_sw=0 寫在開頭（隱含一律 non-FR），現改為條件觸發。

if LoopType == Close:
    addForced("source_duty", reg_0d[6]   = 0)    // bypass step 4: Drive mode
    addForced("cmd_inv",     reg_01[7]   = 0)    // bypass step 5: PWM polarity
    addForced("align_sw",    reg_0e[7]   = 1)    // bypass step 8: Force startup
    addForced("soft_sw",     reg_03[7:6] = 0)    // bypass step 9: Soft switch
if HallType == NonFR:
    addForced("fr_pin_sw",   reg_0f[1]   = 0)    // bypass step 7


// ═══════════════════════ Step 2.5: Mode-derived 6 常數 (§A.1) ═══════════════════════
// ⚠ 設計原稿完全未列。為與 Allen reverse calculator 對齊新增。

isOL = mode in {0, 1, 2, 3, 7}

addDerive("fan_curve",    reg_01[4:3] = isOL ? 2 : 1)
addDerive("rpm_ratio",    reg_01[2:0] = isOL ? 7 : RpmRatioIndex)
addDerive("icmd_sw",      reg_06[7]   = 0)                          // all modes
addDerive("auto_soft_sw", reg_0a[7]   = 1)                          // all modes
addDerive("duty_deb_sw",  reg_0d[7]   = (mode in {4,5,6,7}) ? 1 : 0)
addDerive("low_freq_rev", reg_0e[5]   = 1)                          // all modes


// ═══════════════════════ Step 3: XP1 PIN (§4) ═══════════════════════

xp1High = SelectXp1HighBranch(mode, inputs)   // §A.4 表

if mode in {0, 1, 2, 3, 7}:                   // Open-loop
    if abs(Xp1DutyPercent - 4.64) < 0.005:    // ⚠ 4.64% 開路（設計原稿寫 4.6%）
        addDerive("xp1", reg_02[4:0] = 3)     // ⚠ 設計原稿未寫 register field 值
        addPin("XP1", xp1High ? 5.0 : 0.0, status=OpenCircuit)
    else:
        field = clamp(floor(Xp1DutyPercent / 1.5625), 0, 31)         // 5-bit field
        code  = field * 4 + 20                                       // ×4 對齊 8-bit ADC
        vH = (256 - code) * 5/256
        vL = code * 5/256
        addDerive("xp1", reg_02[4:0] = field)
        addPin("XP1", xp1High ? vH : vL)

else:                                          // Close-loop {4, 5, 6}
    rpmRatio = Rpm1RatioTable[RpmRatioIndex]
    field = clamp(round(MinRpm / rpmRatio − 59), 0, 127)             // ✓ 對齊設計原稿（含 −59 offset，2026-05-08 重新加回；歷史 commit 33fcc857 曾移除）
    code  = field + 20                                               // 7-bit，無 ×4（pin V 換算不變）
    vH = (256 - code) * 5/256
    vL = code * 5/256
    addDerive("rp1", reg_06[6:0] = field)
    addPin("XP1", xp1High ? vH : vL)
    // ⚠ 設計原稿的 `xp1==60*rpm_ratio → 5V/0V` 開路語意現行實作不採用
    //   （`60*rpm_ratio` ≈ field=0 端點 RPM；與 `−59` offset 是同一條物理線：
    //    `field=0 → code=20 → RPM=59·ratio ≈ 60·ratio`，
    //    設計原稿用 `60*rpm_ratio` 描述「腳位開路」門檻，
    //    現行實作不偵測此端點，只 clamp `rp1 ∈ [0,127]`）


// ═══════════════════════ Step 4: YP PIN (§5) ═══════════════════════

if mode in {4, 5, 6}:                          // Close-loop（Allen direct）
    rpmRatio = Rpm1RatioTable[RpmRatioIndex]
    rp2 = clamp(round(MaxRpm / rpmRatio / 4 − 59), 0, 255)           // ⚠ 含 −59 offset（2026-05-08 重新加回，對齊設計原稿）；改用 MaxRpm（設計原稿筆誤寫 min_rpm）
    addDerive("rp2", reg_07[7:0] = rp2)
    addDerive("yp1", reg_04[6:0] = 12)         // ≈ 4.6%（2026-05-08 從 26 改 12 對齊「YP1 固定 4.6%」spec）
    addDerive("yp2", reg_05[6:0] = 127)        // ≈ 100%（Allen direct 常數）
    addPin("YP", rp2 * 5/256)                  // ⚠ 無 +20 offset（Allen direct，與 Reverse `yp_pin = reg_07` 對應）

else:                                          // Open-loop {0, 1, 2, 3, 7}
    ypHigh = SelectYpHighBranch(mode, inputs)  // §A.4 表
    if abs(Yp1DutyPercent - 4.64) < 0.005:
        addDerive("yp1", reg_04[6:0] = 12)     // ⚠ 設計原稿未寫 register field 值
        addPin("YP", ypHigh ? 5.0 : 0.0, status=OpenCircuit)
    else:
        field = clamp(floor(Yp1DutyPercent / 0.390625), 0, 127)      // 7-bit，50/128 per bit
        code  = field + 20                                           // 無 ×4
        vH = (256 - code) * 5/256
        vL = code * 5/256
        addDerive("yp1", reg_04[6:0] = field)
        addPin("YP", ypHigh ? vH : vL)


// ═══════════════════════ Step 5: SS PIN (§6) ═══════════════════════

if mode in {4, 5, 6}:                          // Close-loop：§6.3 對照表
    ssV     = CloseLoopSsTable[RpmRatioIndex].SsVoltage
    ratio   = Rpm1RatioTable[RpmRatioIndex]
    rp1Up   = 167 * ratio                       // RP1 7-bit saturation 含 ADC −20 shift (MinRpm 上限)
    rp2Up   = 1079 * ratio                      // RP2 8-bit saturation (MaxRpm 上限)
    minLow  = 59 * ratio                        // 共用下限 (field=0)
    minOk   = MinRpm > minLow and MinRpm < rp1Up
    maxOk   = MaxRpm > minLow and MaxRpm < rp2Up
    status  = (minOk and maxOk) ? Ok : OutOfRange  // ⚠ RP1/RP2 雙 bound（2026-05-08 spec；設計原稿 RPM 單值）
    addPin("SS", ssV, status)
    // 不寫 reg_08[7:5]：rpm_ratio 索引已隱含於 reg_01[2:0]

else:                                          // Open-loop：§6.2 profile 選擇
    profile = SelectSsProfile(mode, SoPole)    // §A.5 表，'A' 或 'B'
    ssIdx = clamp(SsTime, 0, 7)
    V = (profile == 'A') ? SsProfileA[ssIdx] : SsProfileB[ssIdx]
    addDerive("ss_time", reg_08[7:5] = ssIdx)
    addPin("SS", V)


// ═══════════════════════ Helpers ═══════════════════════

SelectXp1HighBranch(mode, i):
    if mode == 0:  return not (i.HallType==FrEnable and i.DcSource==Vsp)  // non-FR+SET → high; FR+VSP → low
    if mode == 1:  return i.HallType == NonFR
    if mode == 7:  return not (i.HallType==FrEnable and i.DcSource==Set)  // non-FR+VSP → high; FR+SET → low
    if mode == 2:  return i.Standby == Shutdown
    if mode == 3:  return i.Standby == Shutdown
    if mode == 4:  return i.SoPole == P6                                  // 6p → high; 8p → low
    if mode == 5:  return i.Standby == Shutdown
    if mode == 6:  return i.Standby == Shutdown
    return true

SelectYpHighBranch(mode, i):
    if mode == 0:           return i.DriveMode == DirectPwm and i.PwmPolarity == Invert
    if mode in {1, 2, 3}:   return i.ForceStartup == NonForce
    if mode == 7:           return i.DriveMode == DirectPwm
    return true   // mode 4/5/6 不走此 helper（直接 V_low）

SelectSsProfile(mode, pole):
    if mode in {0, 2, 7}:
        if pole == P4: return 'A'
        if pole == P8: return 'B'
    if mode == 1:
        if pole == P4: return 'A'
        if pole == RD: return 'B'
    if mode == 3:
        if pole == P4: return 'A'
        if pole == P6: return 'B'
    return null

// ═══════════════════════ 常數表 ═══════════════════════

ModeVoltageTable     = [0.00, 0.92, 1.55, 2.18, 2.80, 3.43, 4.06, 5.00]
Rpm1RatioTable       = [2.54, 3.72, 5.59, 8.3, 12.2, 17.79, 25.93, 37.96]
SsProfileA           = [5.000, 4.257, 4.023, 3.789, 3.554, 3.320, 3.085, 2.851]   // ss_time 0..7
SsProfileB           = [0.000, 0.722, 0.957, 1.191, 1.425, 1.660, 1.894, 2.128]
CloseLoopSsTable[0..7] = (MinRpm, MaxRpm, SsVoltage):
    (  149.86,  2740.66, 0.00)   // ratio=2.54
    (  219.48,  4013.88, 0.92)   // ratio=3.72
    (  329.81,  6031.61, 1.55)   // ratio=5.59
    (  489.70,  8955.70, 2.18)   // ratio=8.3
    (  719.80, 13163.80, 2.80)   // ratio=12.2
    ( 1049.61, 19195.41, 3.43)   // ratio=17.79
    ( 1529.87, 27978.47, 4.06)   // ratio=25.93
    ( 2239.64, 40958.84, 5.00)   // ratio=37.96
Xp1RegDutyPerBit     = 1.5625    // = 50 / 32  (open-loop XP1 5-bit)
YpRegDutyPerBit      = 0.390625  // = 50 / 128 (open-loop YP1 7-bit)
VPerStep             = 5.0 / 256 ≈ 0.01953125 V
```

### C.3 為何現行實作與設計原稿不同

§C.1 列出 *what*；本節補充 *why*（按差異性質分組）。

- **#1 / #2 / #3（close-loop 公式與 hardcode）**：使用者 2026-05-08 spec 要求；YP 改 max_rpm 是修正設計原稿筆誤（YP 對應上限 RPM）。Forward 含 −59、Reverse 維持 `−20` / 直接相等，兩端不再 round-trip → Compare 容許 ±59 差異（詳 §C.4）。
- **#4（4.64% 開路語意）**：dropdown 改 0.01% 精度後，端點 4.6398 顯示為 4.64；同時補寫 register field 對齊 Reverse fix-H/fix-L。
- **#5（close-loop 無開路）**：「腳位開路」是 design-time 概念，僅 open-loop XP1/YP1 有；close-loop 只 clamp。
- **#7（fr_pin_sw 條件式）/ #8（mode=7 拆 O2a/O2b）**：設計原稿邏輯不嚴謹，Allen reverse calculator 已拆分，Forward 對齊。**2026-05-28 update**：O2b 已移除，DC mode 一律 Error。
- **#9（mode-derived 6 常數）**：原工具只算 pin V + forced bit，實際燒錄需要完整 register set。
- **#10 / #11**：UI 顯示精度調整（dropdown 改 0.01% / ratio 數值），不涉及計算邏輯。

### C.4 變更歷史（Close-loop XP1/YP 公式反覆紀錄）

| 日期 | 變更 | 理由 | Commit |
| --- | --- | --- | --- |
| 2026-05-07 | RP1/RP2 移除 −59 offset；yp1 hardcode 設 26 | 對齊 Allen Reverse round-trip（`rp1 = adc1 − 20`、`rp2 = adc2`） | `33fcc857` |
| 2026-05-08 | RP1/RP2 重新加回 −59 offset；yp1 從 26 改 12 (≈4.6%) | 使用者新 spec：`reg_06 = (min_rpm/rpm_ratio) − 59`、`reg_07 = (max_rpm/rpm_ratio/4) − 59`、yp1 固定 4.6% | `6c9b9b89` |
| 2026-05-08 (續) | `AutoPickRpmRatio` + `CalcSs` close-loop OutOfRange 改用 RP1/RP2 雙 bound | 硬體 spec 表揭露 RP1 max=187·ratio < RP2 max=1079·ratio；舊單 bound（`CloseLoopSsTable.MaxRpm` = RP2 寬鬆 bound）讓 MinRpm > RP1 max 仍被誤判為 fit，造成 RP1 silent saturation。重用 `M8160SpeedCurveCalculator.Rp1Factor` / `Rp2Factor` 常數。 | `6c9b9b89` |
| 2026-05-11 | RP1 Max coefficient 從 187 改為 **167** | 使用者更新硬體 Design AA 表格：RP1 Max 須扣除 ADC −20 階 shift，即 `(128 − 20) + 59 = 167`。`M8160SpeedCurveCalculator.Rp1Factor = 167`，§A.7 預算表 RP1 區間上界全部重算（ratio=2.54 → 424.18、… ratio=37.96 → 6339.32）。RP1 Min（59·ratio）、RP2 Min/Max 不變。 | (待 commit) |
| 2026-05-11 (續) | Forward 啟用 `Status = Forbidden` 旗標 | 硬體 spec 揭露 **XP1/YP/SS 三支 pin 在 ADC step 120..136（≈ 2.34..2.66V）為 forbidden band**（之前 spec 只描述 Mode pin 的 forbidden gap）。Forward 計算若 rp1/rp2/xp1/yp1 field 落在對應 forbidden code 區間 → Pin Summary row 標 Forbidden（紅）、`IsValid = false`。原本 Pin V = 2.500V 邊界（如 MinRpm=1382 + ratio=8.3 → rp1=108 → code=128）會讓 Reverse 反推 `rp1 = adc & 0x7F = 0 − 20 = −20`、`rpm_ratio` 因 xp1 為 Forbidden 退到 reset value 0；現由 Forward 端先攔截。**`AutoPickRpmRatio` 不偏好 forbidden-free ratio**——使用者直覺優先，最小可行 ratio 即回傳，由 Pin Summary Forbidden 警示使用者自行決定。 | (待 commit) |
| 2026-05-25 | ADC 對照表抽出 `M8160AdcTable`，邊界對齊 HTML canonical | 把 SEG 模型從 `M8160AdcCalculator` 抽出為獨立的 `M8160AdcTable`（直接 port 自 `M8160_ADC_tool.html`）。中央 forbidden gap `120..136` → `116..139`；上 forbidden `235..245` → `236..245`；linear endpoint `119/137/234` → `115/140/235`；FIX 判定改為 code-based（`code ∈ 246..255 / 0..9`，不再用 V≥4.8 / V≤0.195 門檻）。`VoltageToXp1/Yp1/Yp2Ratio` 改委派 canonical curve + ClampForbidden 統一處理 forbidden gap。`M8160ForwardCalculator` 三處 forbidden band 偵測（rp1/xp1/rp2/yp1）的觸發 field 區間隨之更新；其餘 Forward 邏輯（−59 offset / RP1/RP2 雙 bound / yp1=12 / yp2=127 / AutoPickRpmRatio）不變。新增 `GmtTestTool.Tests/UnitTests/M8160AdcTableTests.cs` golden test 鎖定 0..255 每個 code 與 HTML 直譯版一致。 | (待 commit) |
| 2026-05-26 | Help 檔精簡 + `M8160_ADC_tool.html` 移入 Help/ | 移除 `Help/M8160_Design_Tool_brief.md` 與 `Help/M8160_Design_Tool_brief_beginner.md`（內容已被 Forward/Reverse spec help 涵蓋）。把 `docs/chips/MXXXX/M8160/M8160_ADC_tool.html` 連同 SEG[] canonical 互動工具 git-mv 到 `Help/M8160_ADC_tool.html`，由 Design Tool 視窗的 Help 選單 → ADC ↔ V Conversion Tool 透過 WebView2 開啟（外部 Google Fonts link 已移除，改用 CSS 系統字型 fallback）。所有 code/doc references 自 `docs/chips/MXXXX/M8160/M8160_ADC_tool.html` 更新到 `Help/M8160_ADC_tool.html`。 | (待 commit) |

**Round-trip 提醒**：Forward 加 −59 後，Forward → Reverse Compare 視窗會出現固定 ±59 的 register field 差異（rp1/rp2）。Reverse calculator (`M8160AdcCalculator.cs`) 維持 `rp1 = adc1[6:0] − 20`、`rp2 = adc2` 不變（使用者明確要求 Forward / Reverse 行為不對稱）。Pin V 換算公式（XP1 用 `code = field + 20`、YP 用 Allen direct 無 +20）也維持不變，因此 Forward pin V 結果會因 field 平移而自然向下偏 ≈ 1.15 V（= 59 × 5/256）。
