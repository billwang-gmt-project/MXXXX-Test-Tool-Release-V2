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

對應 pseudo-code（[M8160ForwardCalculator.cs:216-226](../../../../Helpers/M8160ForwardCalculator.cs#L216-L226)）：

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
| **XP1 derive (close)** | `rp1` | `reg_06[6:0]` | `round(MinRpm / rpm_ratio)`，clamp[0,127] | mode∈{4,5,6} | MinRpm + rpm_ratio | – |
| **YP derive (open)** | `yp1` | `reg_04[6:0]` | `floor(Yp1Duty% / 0.390625)`，clamp[0,127] | mode∈{0,1,2,3,7} | Yp1 duty (%) | – |
| **YP derive (open, 4.64%)** | `yp1` | `reg_04[6:0]` = 12 | 4.64% 開路 fix | `\|duty − 4.64\| < 0.005` | Yp1 duty (%) | – |
| **YP derive (close)** | `rp2` | `reg_07[7:0]` | `round(MaxRpm / rpm_ratio / 4)`，clamp[0,255] | mode∈{4,5,6} | MaxRpm + rpm_ratio | – |
| **YP derive (close)** | `yp1` | `reg_04[6:0]` = 26 | 常數（mode 4/5/6 預設） | mode∈{4,5,6} | （常數 26） | – |
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
                   │   rp1         = round(MinRpm / rpm_ratio)  (7-bit)
                   │   code        = rp1 + 20                   ←  no ×4
                   │
                   ├─ YP  V        = rp2 × 5 / 256              ←  Allen direct
                   │   rp2         = round(MaxRpm / rpm_ratio / 4) (8-bit)
                   │   yp1=26 (const)  yp2=127 (const)
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
        field = clamp(round(inputs.MinRpm / ratio), 0, 127)
        (vH, vL) = VoltageFromField(field, times4=false)      // code=field+20
        addReg("rp1", reg_06[6:0] = field)
        addPin("XP1", xp1High ? vH : vL)

    // ─ Step 4: YP ─
    if mode in {4,5,6}:                              // close-loop（Allen direct）
        rp2 = clamp(round(inputs.MaxRpm / ratio / 4), 0, 255)
        addReg("rp2", reg_07[7:0] = rp2)
        addReg("yp1", reg_04[6:0] = 26)              // 常數
        addReg("yp2", reg_05[6:0] = 127)             // 常數
        addPin("YP", rp2 * 5 / 256.0)                // 無 +20 offset
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
        (lo, hi, ssV) = CloseLoopSsTable[inputs.RpmRatioIndex]
        status = (inputs.MinRpm > lo and inputs.MaxRpm < hi) ? Ok : OutOfRange
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
| 1 / 2 / 3 | Force = Non-force | Force = Force |
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
  3  ratio=8.30   範圍 489.70..8955.70    SS=2.18 V
  4  ratio=12.20  範圍 719.80..13163.80   SS=2.80 V
  5  ratio=17.79  範圍 1049.61..19195.41  SS=3.43 V
  6  ratio=25.93  範圍 1529.87..27978.47  SS=4.06 V
  7  ratio=37.96  範圍 2239.64..40958.84  SS=5.00 V
```

> 巧合（不是巧合）：Close-loop SS 電壓表 = `ModeVoltageTable[idx]`，因為兩者都是 8-step 0~5V 線性切割。

### A.7 Auto rpm_ratio 演算法

當使用者沒鎖定 rpm_ratio（auto = on）：

```
AutoPickRpmRatio(MinRpm, MaxRpm):
    // pass 1：找完全包覆 (Min, Max) 的最小 ratio
    for idx in 0..7:
        (lo, hi, _) = CloseLoopSsTable[idx]
        if MinRpm > lo and MaxRpm < hi: return idx

    // pass 2：fallback，找能包 Max 的最小 ratio
    for idx in 0..7:
        if MaxRpm < CloseLoopSsTable[idx].MaxRpm: return idx

    // pass 3：都不行，回傳最大 ratio
    return 7
```

觸發點：使用者改 MinRpm 或 MaxRpm 時，若 auto = on 則重新挑。

### A.8 4.64% 開路語意

XP1 / YP1 dropdown 第 12 格的理論值是 `38.28 × 12 / 99 = 4.6398`，round 到 0.01% 精度後顯示為 **4.64%**，這個值代表「腳位開路、不接外部 divider」（design-time 的省略電阻設定，*不是* 運行 clamp）。

- 容差：`abs(duty − 4.64) < 0.005`
- XP1 開路 → `xp1 = 3`、TargetVoltage = `xp1High ? 5.0 : 0.0`
- YP1 開路 → `yp1 = 12`、TargetVoltage = `ypHigh ? 5.0 : 0.0`
- Status = `OpenCircuit`（藍底淺藍字，UI 顯示 "Open circuit / no resistor"）
- 兩個 branch 反推皆走 fix-H/fix-L → 同 register field，所以 Forward / Reverse Compare 在開路情境下仍 match。

> **與 close-loop clamp 不同**：close-loop 的 `MinRpm ≥ 60 × rpm_ratio` 走 `Status = Clamped`（黃底），對應端點 V_high → 5V 或 V_low → 0V，**與 4.64% 開路無關**。

### A.9 與 Reverse 的對齊點

| Forward 公式 | Reverse 對應 | 對齊方式 |
|--------------|-------------|----------|
| `rp1 = round(MinRpm / rpm_ratio)` | Reverse: `rp1 = adc1[6:0] − 20` | Forward 加 +20 offset 抵消 Reverse 的 −20 |
| `rp2 = round(MaxRpm / rpm_ratio / 4)` | Reverse: `rp2 = adc2`（直接） | Forward 用 Allen direct（無 +20），pin V = `rp2 × 5/256` |
| `yp1 = 26`（mode 4/5/6 常數） | Reverse: `yp1 = adc4 = 26 hardcode` | 兩端都 hardcode |
| `yp2 = 127`（mode 4/5/6 常數） | Reverse: `yp2 = adc5 = 127 hardcode` | 兩端都 hardcode |
| `xp1 = 3`（4.64% 開路 fix） | Reverse: `xp1Fix = xp1FixH \|\| xp1FixL` | 兩個 branch 反推同 field |
| `yp1 = 12`（4.64% 開路 fix） | Reverse: `ypFix = ypFixH \|\| ypFixL` | 同上 |

這 5 個 forced field 在 Compare 時被列入容忍清單（Forced），避免兩端硬體欄位不對等時誤報 mismatch。

### A.10 關鍵常數

| 常數 | 值 | 用途 |
|------|----|----|
| `Rpm1RatioTable[0..7]` | `[2.54, 3.72, 5.59, 8.3, 12.2, 17.79, 25.93, 37.96]` | rpm_ratio dropdown |
| `Xp1RegDutyPerBit` | `1.5625` (= 50 / 32) | open-loop XP1 每 bit duty% |
| `YpRegDutyPerBit` | `0.390625` (= 50 / 128) | open-loop YP1 每 bit duty% |
| `VPerStep` | `5.0 / 256` ≈ `0.01953125 V` | ADC 每 step 解析度 |

### A.11 結果狀態旗標

| 狀態 | 顏色 | 觸發條件 |
|------|------|----------|
| Ok | 綠 | 計算正常 |
| OpenCircuit | 藍底淺藍字 | XP1/YP1 duty = 4.64%（design-time 開路） |
| Clamped | 黃 | close-loop 端點 clamp |
| OutOfRange | 紅 | Min/Max RPM 超出 CloseLoopSsTable 該 ratio 的合法範圍 |
| Forbidden | 紅底白字 | Mode pin 落在禁帶 |
| Invalid | 紅 | 找不到對應 mode 列 |

### A.12 容易踩到的坑

1. **×4 只在 open-loop XP1**：`reg_02[4:0]` 是 5-bit field，要 ×4 才能對齊 8-bit ADC code。`reg_06[6:0]` 已是 7-bit，不要再 ×4。
2. **Close-loop YP 用 Allen direct**：`pin V = rp2 × 5/256`，**沒有 +20 offset**。和 XP1/open-loop YP 不同。
3. **Close-loop YP 公式用 MaxRpm**（不是 MinRpm；rp2 是上限）。
4. **Close-loop 不寫 ss_time**：rpm_ratio index 已隱含於 `reg_01[2:0]`，不另寫 `reg_08[7:5]`。
5. **rpm_ratio dropdown 顯示 ratio 值不是 index**（例如 `2.54`、`3.72` 而非 0、1）。
6. **fan_curve 在 open-loop = 2 (OOO)、close-loop = 1 (CCO)**：常數 derive，不是看使用者選的 4 個 speed curve UserControl。Forward 工具不直接決定 COO/COP。
7. **Compare 容忍清單**：5 個 forced field 在 Compare 時納入容忍，避免兩端硬體欄位不對等時誤報 mismatch。

---

## Part B. 端對端驗證範例

範例輸入：

```
Open-loop / 4p / Shutdown / Direct PWM / Non-invert / non-FR / Force / Square
                                                     ↑ ss_time = 2s
```

預期計算結果：

```
命中 mode classification rule O2a → mode = 7
Mode pin = 5.000 V
XP1 pin  = 4.766 V (V_high, mode=7 + non-FR + VSP=default)
YP pin   = 0.156 V (V_low, mode=7 + DC fallback)
SS pin   = 4.023 V (mode=7 + 4p → Profile A, ss_time=2s)
```

DataGrid 應顯示 row 數量：

- 4 row 腳位電壓（Mode / XP1 / YP / SS）
- 1 row Forced（non-FR：`fr_pin_sw = 0`）
- 6 row mode-derived 常數（fan_curve=2, rpm_ratio=7, icmd_sw=0, auto_soft_sw=1, duty_deb_sw=1, low_freq_rev=1）
- 3 row XP1/YP/SS derive（xp1, yp1, ss_time）

→ 共 14 row。

**Compare 驗證**：把計算出的 4 個 pin V 灌回 Reverse 模式輸入欄位，跑 Reverse → 應該得到相同的 9 個應用條件 + register field 值。Forced 清單中的 5 個 field 在 mismatch 時用淡色標示而非報錯。

---

## Part C. 設計原稿 vs 現行 codebase 對照（Refactored Pseudo-code）

設計者（Allen）原始 pseudo-code 出自 [`M8160AA_InputPinResistorDesignTool_Spec.md`](M8160AA_InputPinResistorDesignTool_Spec.md) §10。本節將其重寫為與 [M8160ForwardCalculator.cs](../../../../Helpers/M8160ForwardCalculator.cs) 一致的版本，並在每個分支標註 **「設計原稿」** 與 **「現行 codebase」** 的差異。

### C.1 重大差異總覽

| # | 主題 | 設計原稿 | 現行 codebase | 對應 commit / 程式碼 |
|---|------|----------|----------------|----------------------|
| 1 | Close-loop XP1 公式 | `reg_06[6:0] = min_rpm/rpm_ratio − 59` | `rp1 = round(min_rpm / rpm_ratio)`（無 −59 offset） | [M8160ForwardCalculator.cs:322](../../../../Helpers/M8160ForwardCalculator.cs#L322)；commit `33fcc857` |
| 2 | Close-loop YP 公式 | `reg_07 = (min_rpm/rpm_ratio − 59)/4`，pin V 用 `(reg_07 + 20) × 5/256` | `rp2 = round(max_rpm / rpm_ratio / 4)`（**改用 max_rpm**、無 −59 offset），pin V = `rp2 × 5/256`（Allen direct，**無 +20**） | [M8160ForwardCalculator.cs:413-416](../../../../Helpers/M8160ForwardCalculator.cs#L413-L416)；commit `33fcc857` |
| 3 | Close-loop YP `yp1` / `yp2` 常數 | 未列 | `yp1 = 26`、`yp2 = 127`（mode 4/5/6 hardcoded，與 Allen reverse calculator 對齊） | [M8160ForwardCalculator.cs:423-428](../../../../Helpers/M8160ForwardCalculator.cs#L423-L428) |
| 4 | XP1/YP1 4.6% 開路語意 | 文字寫 4.6%；只描述 pin V，未寫 register field | 容差 `abs(duty − 4.64) < 0.005`；同時寫 `xp1 = 3`（open-loop）/ `yp1 = 12`（open-loop）對齊 Reverse fix-H/fix-L | [M8160ForwardCalculator.cs:343-348, 446-451](../../../../Helpers/M8160ForwardCalculator.cs#L343-L348)；commit `a3da9541` |
| 5 | Close-loop XP1 開路 | `xp1==60*rpm_ratio` 視為開路 → 5V/0V | **不存在此語意**。Close-loop 只 clamp `rp1 ∈ [0,127]`，無 4.64% 開路概念 | [M8160ForwardCalculator.cs:323](../../../../Helpers/M8160ForwardCalculator.cs#L323) |
| 6 | Close-loop SS 範圍檢查 | `min_rpm < RPM < max_rpm` 單一條件 | `MinRpm > tableLo && MaxRpm < tableHi` 兩端皆檢查；越界 → `OutOfRange` | [M8160ForwardCalculator.cs:501-503](../../../../Helpers/M8160ForwardCalculator.cs#L501-L503) |
| 7 | `fr_pin_sw = 0` 觸發 | 開頭無條件寫（隱含預設 non-FR） | 條件式：`if HallType == NonFR` 才寫 | [M8160ForwardCalculator.cs:223-226](../../../../Helpers/M8160ForwardCalculator.cs#L223-L226) |
| 8 | Mode=7 分類條件 | `(pole==4p\|8p) & shutdown & (PWM_non_invert or DC_mode)` 單條 | 拆兩條：**O2a** Direct PWM + Non-invert、**O2b** DC mode + (non-FR+VSP \|\| FR+SET) | [M8160ForwardCalculator.cs:114-138](../../../../Helpers/M8160ForwardCalculator.cs#L114-L138) |
| 9 | Mode-derived 6 常數 | 完全未列 | `fan_curve` / `rpm_ratio` / `icmd_sw` / `auto_soft_sw` / `duty_deb_sw` / `low_freq_rev` 6 個 row | [M8160ForwardCalculator.cs:233-260](../../../../Helpers/M8160ForwardCalculator.cs#L233-L260)；commit `225a6893` |
| 10 | XP1 duty 精度 | 4.6%（兩位小數） | **4.64%**（容差 0.005，理論值 38.28×12/99=4.6398，顯示為 4.64%） | §A.8 / [M8160ForwardCalculator.cs:343](../../../../Helpers/M8160ForwardCalculator.cs#L343) |
| 11 | rpm_ratio dropdown 顯示 | 未提 | 顯示 ratio 數值（如 2.54、3.72），不是 index | commit `0b26bb2d` / `9501c91b` |

### C.2 Refactored Pseudo-code（與 codebase 對齊）

下面以設計原稿的結構為骨架，把每段改寫為現行 codebase 的等價邏輯。**標記 ⚠ 處表示與設計原稿語意不同**。

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

if LoopType == Close:
    if SoPole == P4 and HallType == NonFR:        mode = 5    // C1: 3.43V
    elif SoPole == P4 and HallType == FrEnable:   mode = 6    // C2: 4.06V
    elif SoPole in {P6, P8}:                      mode = 4    // C3: 2.80V
    else:                                         return Invalid  // Close + RD 無對應 row
else:  // Open-loop
    poleIs48  = SoPole in {P4, P8}
    poleIs46  = SoPole in {P4, P6}
    poleIs4Rd = SoPole in {P4, RD}

    if poleIs48 and Standby==Shutdown and DriveMode==DirectPwm and PwmPolarity==Invert:
        mode = 0   // O1a: 0V
    elif poleIs48 and Standby==Shutdown and DriveMode==DirectPwm and PwmPolarity==NonInvert:
        mode = 7   // O2a: 5V
    elif poleIs48 and Standby==Shutdown and DriveMode==DcMode:        // ⚠ 設計原稿把 DC mode 也塞進 O1a/O2a，現拆 O1b/O2b
        if (HallType==NonFR and DcSource==Set) or (HallType==FrEnable and DcSource==Vsp):
            mode = 0   // O1b
        elif (HallType==NonFR and DcSource==Vsp) or (HallType==FrEnable and DcSource==Set):
            mode = 7   // O2b
    elif poleIs4Rd and Standby==Shutdown and SoftSwitch==Deg22_5:
        mode = 1   // O3: 0.92V
    elif poleIs48 and HallType==FrEnable:
        mode = 2   // O4: 1.55V
    elif poleIs46 and HallType==NonFR:
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
    field = clamp(round(MinRpm / rpmRatio), 0, 127)                  // ⚠ 無 −59 offset（設計原稿有）
    code  = field + 20                                               // 7-bit，無 ×4
    vH = (256 - code) * 5/256
    vL = code * 5/256
    addDerive("rp1", reg_06[6:0] = field)
    addPin("XP1", xp1High ? vH : vL)
    // ⚠ 設計原稿的 `xp1==60*rpm_ratio → 5V/0V` 開路語意 codebase 不採用


// ═══════════════════════ Step 4: YP PIN (§5) ═══════════════════════

if mode in {4, 5, 6}:                          // Close-loop（Allen direct）
    rpmRatio = Rpm1RatioTable[RpmRatioIndex]
    rp2 = clamp(round(MaxRpm / rpmRatio / 4), 0, 255)                // ⚠ 改用 MaxRpm（設計原稿用 min_rpm）；無 −59
    addDerive("rp2", reg_07[7:0] = rp2)
    addDerive("yp1", reg_04[6:0] = 26)         // ⚠ Allen direct 常數（設計原稿未列）
    addDerive("yp2", reg_05[6:0] = 127)        // ⚠ Allen direct 常數（設計原稿未列）
    addPin("YP", rp2 * 5/256)                  // ⚠ 無 +20 offset（設計原稿有 +20）

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
    (lo, hi, ssV) = CloseLoopSsTable[RpmRatioIndex]
    status = (MinRpm > lo and MaxRpm < hi) ? Ok : OutOfRange  // ⚠ 兩端皆檢查（設計原稿 RPM 單值）
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
Rpm1RatioTable       = [2.54, 3.72, 5.59, 8.30, 12.20, 17.79, 25.93, 37.96]
SsProfileA           = [5.000, 4.257, 4.023, 3.789, 3.554, 3.320, 3.085, 2.851]   // ss_time 0..7
SsProfileB           = [0.000, 0.722, 0.957, 1.191, 1.425, 1.660, 1.894, 2.128]
CloseLoopSsTable[0..7] = (MinRpm, MaxRpm, SsVoltage):
    (  149.86,  2740.66, 0.00)   // ratio=2.54
    (  219.48,  4013.88, 0.92)   // ratio=3.72
    (  329.81,  6031.61, 1.55)   // ratio=5.59
    (  489.70,  8955.70, 2.18)   // ratio=8.30
    (  719.80, 13163.80, 2.80)   // ratio=12.20
    ( 1049.61, 19195.41, 3.43)   // ratio=17.79
    ( 1529.87, 27978.47, 4.06)   // ratio=25.93
    ( 2239.64, 40958.84, 5.00)   // ratio=37.96
Xp1RegDutyPerBit     = 1.5625    // = 50 / 32  (open-loop XP1 5-bit)
YpRegDutyPerBit      = 0.390625  // = 50 / 128 (open-loop YP1 7-bit)
VPerStep             = 5.0 / 256 ≈ 0.01953125 V
```

### C.3 為何 codebase 與設計原稿不同

1. **−59 offset 移除**：設計原稿的 `−59` 來自早期 firmware 假設「最低 RPM 對應 ADC code 59 → field 0」。實機驗證後改為純 `round(rpm/ratio)`，符合 Allen reverse calculator `rp1 = adc1[6:0] − 20` 的反推（commit `33fcc857`）。
2. **Close-loop YP 改用 MaxRpm**：設計原稿用 `min_rpm` 是筆誤；YP 對應上限 RPM，邏輯上必須用 max_rpm（commit `33fcc857`）。
3. **Allen direct YP 公式**：close-loop YP pin V 不再加 `+20` offset，與 Reverse `rp2 = adc2`（直接，無 −20）對齊。
4. **`yp1=26` / `yp2=127` 常數**：使用者口語要求「YP1 固定 4.6%」，hardcode 對齊 Reverse 的 default。
5. **Mode-derived 6 常數**：原設計工具只算 4 支腳位電壓 + 5 個 forced bit，但實際燒錄需要完整 register set；commit `225a6893` 補上。
6. **`xp1==60*rpm_ratio` 開路語意移除**：close-loop 沒有「腳位開路」這個 design-time 概念（4.64% 開路只屬於 open-loop XP1/YP1）。
7. **mode=7 拆 O2a/O2b**：設計原稿 `(PWM_non_invert or DC_mode)` 邏輯不嚴謹（DC mode 還要看 hall + DcSource 子分支）；Allen reverse calculator 已拆分，Forward 對齊。
