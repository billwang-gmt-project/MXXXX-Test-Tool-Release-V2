# M8160AA Input Pin Resistor Design Tool — 規格 (待硬體工程師 review)

## 這份文件在做什麼

M8160AA 晶片有 4 支類比腳位（Mode / XP1 / YP / SS）給使用者外接 resistor divider 設定行為。這份規格描述一個新的設計輔助工具：使用者選好應用條件（馬達極數、開/閉迴路、PWM 模式…），工具自動算出 4 支腳位該打多少電壓、建議 resistor pair、以及哪些 register bit 會被工具強制鎖定。

Refactor 自原始的 200 行 pseudo-code spec，整理成易於 review 的表格 + 流程。

---

## 1. 使用者要填的 9 個輸入

每個 input 都用 dropdown 給選項，預設值標 `★`。

| # | 輸入 | 選項 | 備註 |
|---|------|------|------|
| 1 | **Loop type** | Open-loop ★ / Close-loop | 選 Close-loop 才會出現 #1.1 / #1.2 / #1.3 |
| 1.1 | **Min RPM** | 數字（close-loop 才出現） | 目標最低轉速 |
| 1.2 | **Max RPM** | 數字（close-loop 才出現） | 目標最高轉速 |
| 1.3 | **rpm_ratio** | 2.54 / 3.72 / 5.59 / 8.3 / 12.2 / 17.79 / 25.93 / 37.96 | 對應 REG01[2:0]。工具預設依 Min/Max RPM 自動挑最小可行 ratio，使用者可改 |
| 2 | **SO pole** | 4p ★ / 6p / 8p / RD | RD = reluctance |
| 3 | **Standby behaviour** | Shutdown ★ / Minimum | Minimum = 馬達在最低 duty 繼續轉 |
| 4 | **Drive mode** | Direct PWM ★ / DC mode | DC mode 還要選 VSP / SET（見 #6） |
| 5 | **PWM polarity** | Non-invert ★ / Invert | 只在 Drive mode = Direct PWM 時可選 |
| 6 | **DC source** | VSP ★ / SET | 只在 Drive mode = DC mode 時可選 |
| 7 | **Hall type** | non-FR (hall element) ★ / FR-enable (hall switch) | |
| 8 | **Force startup** | Force ★ / Non-force | Open-loop 才有；Close-loop 強制 Force |
| 9 | **Soft switch** | Square ★ / 22.5° / 45° / Sine | Open-loop 才有；Close-loop 強制 Square |

### 1.1 工具會自動鎖定的 register bit

當使用者選 **Close-loop**，下列 register bit 被工具強制寫死，使用者改不到：

| Register bit | 寫入值 | 對應原因 |
|--------------|:------:|----------|
| `reg_data_0d[3]` (`pwm_low_sw`) | 0 | 強制 Direct PWM（bypass #4） |
| `reg_data_01[7]` (`cmd_inv`) | 0 | PWM non-invert（bypass #5、#6） |
| `reg_data_0e[7]` (`align_sw`) | 1 | 強制 Force startup（bypass #8） |
| `reg_data_03[7:6]` (`ssw`) | 0 | Soft switch = Square（bypass #9） |

當使用者選 **Hall type = non-FR**，再加一條：

| Register bit | 寫入值 | 對應原因 |
|--------------|:------:|----------|
| `reg_data_0f[1]` (`fr_pin_sw`) | 0 | bypass #7 在 non-FR 分支 |

> ⚠ **待確認 1** — 原 spec 寫 "direct PWM" 對應 `reg_data_0d[6]`，但晶片定義檔裡 `pwm_low_sw` 在 `0x0D[3]`（`0x0D[6]` 是 `vsp_sw`）。請硬體工程師確認 "direct PWM" 該鎖哪一個 bit。

---

## 2. Mode pin 電壓表

`mode` 是 0~7 的離散索引，每一個對應 Mode pin 上一個目標電壓。

| mode | 目標電壓 | 適用條件（白話） |
|:----:|--------:|------------------|
| 0 | 0.00 V | open-loop、4p 或 8p、shutdown、PWM 反相 或 DC mode |
| 1 | 0.92 V | open-loop、4p 或 RD、shutdown、soft switch = 22.5° |
| 2 | 1.55 V | open-loop、4p 或 8p、Hall = FR-enable |
| 3 | 2.18 V | open-loop、4p 或 6p、Hall = non-FR |
| 4 | 2.80 V | close-loop、6p 或 8p |
| 5 | 3.43 V | close-loop、4p、Hall = non-FR |
| 6 | 4.06 V | close-loop、4p、Hall = FR-enable |
| 7 | 5.00 V | open-loop、4p 或 8p、shutdown、PWM 正相 或 DC mode |

### 2.1 Mode 判定表（每一列互斥不重疊）

判定一個輸入組合屬於哪個 mode，依下列表格從上往下找。每一列的條件都是 **完整 AND 條件**（所有欄都要滿足才算命中）。

| # | Loop | Pole | Standby | Drive mode | PWM polarity | Hall | → mode |
|---|------|------|---------|------------|--------------|------|:------:|
| C1 | Close | 4p | – | – | – | non-FR | **5** |
| C2 | Close | 4p | – | – | – | FR | **6** |
| C3 | Close | 6p / 8p | – | – | – | – | **4** |
| O1a | Open | 4p / 8p | Shutdown | Direct-PWM | Invert | – | **0** |
| O1b | Open | 4p / 8p | Shutdown | DC mode | – | non-FR + SET<br>FR + VSP | **0** |
| O2a | Open | 4p / 8p | Shutdown | Direct-PWM | Non-invert | – | **7** |
| O2b | Open | 4p / 8p | Shutdown | DC mode | – | non-FR + VSP<br>FR + SET | **7** |
| O3 | Open | 4p / RD | Shutdown | – | – | – （需 soft switch = 22.5°） | **1** |
| O4 | Open | 4p / 8p | – | – | – | FR | **2** |
| O5 | Open | 4p / 6p | – | – | – | non-FR | **3** |

> ⚠ **待確認 2** — 原 spec O1/O2 兩列都列了 `4p & shutdown` 重疊，O4/O5 也跟前面幾列重疊。這份 refactor 要求每一列的條件取 **完整 AND**（不是只列關鍵欄）以消除歧義。請硬體工程師驗證：所有合法的輸入組合在這張表都恰好命中一列、不可漏。

---

## 3. 電壓共用公式

XP1、YP 兩支腳位都用同一條公式（SS 不用，看 §6）。

```
V_high(code) = (256 − code) × 5 / 256        // 高半段（最低點 5 V，最高點接近 0 V）
V_low (code) =  code        × 5 / 256        // 低半段（最低點 0 V，最高點接近 5 V）
```

其中 `code = field_value + 20`，部分情境下 `field_value` 還要先 ×4（看 §4）。

> ⚠ **待確認 3** — 原 spec 寫 `0.01953125 mV/step`，但 `5 / 256 = 0.01953125`，正確單位應該是 **V**（不是 mV）。本 refactor 已改成 V。請硬體工程師確認。

### 3.1 4.6% 越界保護

當使用者填的 duty 低於 **4.6%**（最小可表示值），工具不算公式，直接把該腳位電壓 clamp 到 0 V 或 5 V 端點（看分支），UI 顯示 "out of range, clamped" 警告。

---

## 4. XP1 pin 電壓對照表

`xp1` 看 mode 跟子條件決定走哪一條分支。資料來源有兩種：

- **Open-loop modes 0/1/2/3/7**：使用者填 `xp1_duty(%)` → `field = floor(duty / 1.5625)` → `code = field × 4 + 20`（注意有 ×4）。
- **Close-loop modes 4/5/6**：`field = min_rpm / rpm_ratio − 59` → `code = field + 20`（沒有 ×4）。

| mode | 子條件 | 分支 | 來源欄位 |
|:----:|--------|:----:|----------|
| 0（shutdown） | non-FR + SET | V_high | `reg_data_02[4:0]` × 4 |
| 0（shutdown） | FR + VSP | V_low | `reg_data_02[4:0]` × 4 |
| 1（shutdown） | non-FR | V_high | `reg_data_02[4:0]` × 4 |
| 1（shutdown） | FR | V_low | `reg_data_02[4:0]` × 4 |
| 7（shutdown） | non-FR + VSP | V_high | `reg_data_02[4:0]` × 4 |
| 7（shutdown） | FR + SET | V_low | `reg_data_02[4:0]` × 4 |
| 2 + FR | shutdown | V_high | `reg_data_02[4:0]` × 4 |
| 2 + FR | minimum | V_low | `reg_data_02[4:0]` × 4 |
| 3 + non-FR | shutdown | V_high | `reg_data_02[4:0]` × 4 |
| 3 + non-FR | minimum | V_low | `reg_data_02[4:0]` × 4 |
| 4 + FR + 6p | shutdown | V_high | `reg_data_06[6:0]`（無 ×4） |
| 4 + FR + 8p | shutdown | V_low | `reg_data_06[6:0]`（無 ×4） |
| 5 + non-FR | shutdown | V_high | `reg_data_06[6:0]`（無 ×4） |
| 5 + non-FR | minimum | V_low | `reg_data_06[6:0]`（無 ×4） |
| 6 + FR | shutdown | V_high | `reg_data_06[6:0]`（無 ×4） |
| 6 + FR | minimum | V_low | `reg_data_06[6:0]`（無 ×4） |

Close-loop 的 sentinel：當 `min_rpm == 60 × rpm_ratio`（剛好打到最高轉速），依 §3.1 clamp 到端點。

---

## 5. YP pin 電壓對照表

`yp1` 來源欄位是 `reg_data_04[6:0]`；close-loop 改用 `reg_data_07`。

| mode | 子條件 | 分支 | 來源欄位 |
|:----:|--------|:----:|----------|
| 0 | Direct-PWM + Invert | V_high | `reg_data_04[6:0]` |
| 0 | DC mode | V_low | `reg_data_04[6:0]` |
| 1 / 2 / 3 | Non-force startup | V_high | `reg_data_04[6:0]` |
| 1 / 2 / 3 | Force startup | V_low | `reg_data_04[6:0]` |
| 7 | Direct-PWM | V_high | `reg_data_04[6:0]` |
| 7 | DC mode | V_low | `reg_data_04[6:0]` |
| 4 / 5 / 6 | – | V_low | `reg_data_07 = (min_rpm / rpm_ratio − 59) / 4` |

`reg_data_04` 那幾列都套 §3.1 的 4.6 % 保護。

---

## 6. SS pin 電壓查表

SS pin **不用 §3 公式**，直接由查表決定。

### 6.1 ss_time 對應電壓表

使用者從 8 個 ss_time 選一個（對應 REG08[7:5] 索引 0~7）：

| ss_time | Profile A（5 V → 2.85 V） | Profile B（0 V → 2.13 V） |
|:-------:|--------------------------:|--------------------------:|
| 0.5 s   | 5.000 V | 0.000 V |
| 1 s     | 4.257 V | 0.722 V |
| 2 s     | 4.023 V | 0.957 V |
| 4 s     | 3.789 V | 1.191 V |
| 6 s     | 3.554 V | 1.425 V |
| 8 s     | 3.320 V | 1.660 V |
| 10 s    | 3.085 V | 1.894 V |
| 12 s    | 2.851 V | 2.128 V |

### 6.2 Open-loop profile 選擇

| mode | pole | profile |
|:----:|:----:|:-------:|
| 0 / 2 / 7 | 4p | A |
| 0 / 2 / 7 | 8p | B |
| 1 | 4p | A |
| 1 | RD | B |
| 2 | 4p | A |
| 2 | 6p | B |
| 4 / 5 / 6 | – | **看 §6.3** |

> ⚠ **待確認 4** — 原 spec 中 `mode == 2` 出現兩次（一次列在 `mode == 0/2/7`，一次單獨列出 4p/6p）。本 refactor 採單獨列出的 4p/6p 為主。請硬體工程師確認哪一個是對的。

### 6.3 Close-loop SS 電壓表

Close-loop（mode 4/5/6）時，SS 電壓只看 `rpm_ratio`：

| rpm_ratio | RPM 合法範圍（min_rpm < RPM < max_rpm） | SS 電壓 |
|----------:|----------------------------------------|--------:|
| 2.54 | 149.86 – 2740.66 | 0.00 V |
| 3.72 | 219.48 – 4013.88 | 0.92 V |
| 5.59 | 329.81 – 6031.61 | 1.55 V |
| 8.30 | 489.70 – 8955.70 | 2.18 V |
| 12.20 | 719.80 – 13163.80 | 2.80 V |
| 17.79 | 1049.61 – 19195.41 | 3.43 V |
| 25.93 | 1529.87 – 27978.47 | 4.06 V |
| 37.96 | 2239.64 – 40958.84 | 5.00 V |

工具會檢查使用者填的 Min/Max RPM 是否落在該列的 RPM 範圍內，超出則該列標紅。

---

## 7. 計算流程（白話版）

工具拿到使用者填的 9 個輸入後，依序做：

1. **判定 mode** — 套 §2.1 表，找出唯一命中的列（C1~O5），得到 mode 0~7。
2. **填 Mode pin 電壓** — 直接從 §2 對照表查 mode 對應電壓。
3. **加入強制 register row** — 依 §1.1 的鎖定規則，把被工具寫死的 bit 加到結果列表。
4. **算 XP1 pin 電壓** — 套 §4 表選分支，套 §3 公式（或 §3.1 clamp）。
5. **算 YP pin 電壓** — 套 §5 表選分支，套 §3 公式（或 §3.1 clamp）。
6. **算 SS pin 電壓** — close-loop 走 §6.3 查表，open-loop 走 §6.2 + §6.1 兩段查表。
7. **算建議 resistor 對** — 每一支腳位的目標電壓 → 推算外接 R_top / R_bot 一對標準值（資料源自既有的 ADC 對照模型）。
8. **整合輸出** — 全部塞進 result table（看 §8）。

任何一步若找不到對應列、或數值越界，該 row 在 result table 標紅（status = Out-of-range / Invalid）。

---

## 8. 結果呈現（DataGrid）

每一支受影響的腳位 / register bit 一個 row，欄位設計如下：

| 欄位 | 內容說明 | 範例 |
|------|----------|------|
| **Pin / Field** | Mode / XP1 / YP / SS，或 `reg_data_XX[B:b]` | `Mode pin` / `reg_data_0d[3]` |
| **Voltage** | 目標電壓 | `2.80 V` |
| **R_top / R_bot** | 建議外接 resistor 對（Ω） | `47k / 33k` |
| **Register value** | 強制鎖定的 bit 值（dec / hex / bin） | `0 / 0x00 / 0b0` |
| **Source rule** | 哪一條 spec 規則觸發 | `§2 row C3（close-loop, 6p）` |
| **Status** | OK / Clamped / Out-of-range / Forbidden-band | 顏色標籤 |

沒被影響的欄位不顯示，避免雜訊。

### 8.1 Input panel 草圖

左側分組擺 dropdown，灰掉的列在當前狀態下不可改：

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
│  ───────────────── Result DataGrid (見 §8.2) ───────────────────────────── │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

灰掉（`─ … ─`）的列表示在當前 loop / drive mode 下不可改。

### 8.2 Result table 範例

範例輸入：`Open-loop / 4p / Shutdown / Direct PWM / Non-invert / non-FR / Force / Square` → 命中 §2.1 row O2a → mode = 7：

| Pin / Field        | Voltage | R_top   | R_bot   | RegValue (dec/hex/bin) | Source rule                  | Status |
|--------------------|--------:|--------:|--------:|------------------------|------------------------------|--------|
| **Mode pin**       | 5.000 V | open    | 0 Ω     | —                      | §2 row O2a                   | OK     |
| **XP1 pin**        | 4.766 V | 4.7 kΩ  | 100 kΩ  | reg_data_02[4:0] = 3   | §4 mode=7, V_high            | OK     |
| **YP pin**         | 0.156 V | 95.3 kΩ | 3.16 kΩ | reg_data_04[6:0] = 5   | §5 mode=7 + DC, V_low        | Clamped|
| **SS pin**         | 4.023 V | 19.6 kΩ | 80.6 kΩ | REG08[7:5] = 2         | §6.2 mode=7, 4p, profile A   | OK     |
| reg_data_0e[7]     |   —     |  —      |  —      | 1 / 0x01 / 0b1         | §1.1 force align_sw          | OK     |
| reg_data_03[7:6]   |   —     |  —      |  —      | 0 / 0x00 / 0b00        | §1.1 force ssw=square        | OK     |
| reg_data_0f[1]     |   —     |  —      |  —      | 0 / 0x00 / 0b0         | §1.1 force fr_pin_sw         | OK     |

Status 顏色：

- **OK** — 綠
- **Clamped** — 黃（落在 §3.1 的 4.6 % 端點）
- **Out-of-range** — 紅（min/max RPM 超出 §6.3 範圍）
- **Forbidden-band** — 紅底白字（落在 Mode pin 禁帶）

---

## 9. 待硬體工程師 review 的 7 件事

工具動工前，請逐項確認：

1. **bit 位置** — "Direct PWM" 強制鎖定的是 `reg_data_0d[3]` (`pwm_low_sw`) 還是 `reg_data_0d[6]` (`vsp_sw`)？（§1.1 待確認 1）
2. **Mode 判定表完整性** — §2.1 表 9 列是否涵蓋所有合法輸入組合、且互斥不漏？建議用所有合法組合掃過驗證。（§2.1 待確認 2）
3. **電壓單位** — §3 公式單位是 V 不是 mV，請確認。（§3 待確認 3）
4. **mode==2 的 SS profile** — §6.2 採 4p/6p 還是繼承 4p/8p？（§6.2 待確認 4）
5. **×4 step 的範圍** — §4 的 `× 4` 只適用 open-loop XP1，close-loop 不適用，對嗎？
6. **整數捨入** — §5 close-loop 的 `(min_rpm / rpm_ratio − 59) / 4` 要無條件捨去（floor）還是四捨五入（round）？
7. **Result 列表範圍** — §8 DataGrid 是只列 **被工具強制鎖定** 的 register，還是把使用者隱含選到的（`xp1`、`yp1`、`rp1`、`rp2`、`ss_time`）也都列出？

---

## 10. Pseudo-code（給想 trace 細節的人）

§7 是白話流程，這節是把同一套邏輯寫成 pseudo-code，給想驗證細節的工程師對照。

### 10.1 Top-level flow

```
function Calculate(inputs) -> List<ResultRow>:
    rows = []

    // Step 1: 判定 mode (§2.1)
    mode = ClassifyMode(inputs)
    if mode == null:
        return [Row(Pin="Mode", Status=Invalid, Rule="no §2.1 row matched")]

    // Step 2: Mode pin 電壓 (§2)
    rows.add(Row(Pin="Mode pin",
                 Voltage = ModeVoltageTable[mode],
                 Rule = "§2 row " + matchedRow))

    // Step 3: close-loop 強制鎖定的 register (§1.1)
    if inputs.LoopType == Close:
        rows.add(RegRow("reg_data_0d[3]", 0, "§1.1 force Direct PWM"))
        rows.add(RegRow("reg_data_01[7]", 0, "§1.1 force PWM non-invert"))
        rows.add(RegRow("reg_data_0e[7]", 1, "§1.1 force align_sw"))
        rows.add(RegRow("reg_data_03[7:6]", 0, "§1.1 force ssw=square"))
    if inputs.HallType == NonFR:
        rows.add(RegRow("reg_data_0f[1]", 0, "§1.1 force fr_pin_sw"))

    // Step 4~6: 三支腳位
    rows.add(CalcXp1(inputs, mode))
    rows.add(CalcYp(inputs, mode))
    rows.add(CalcSs(inputs, mode))

    // Step 7: 每個有電壓的 row 推算建議 R 對
    for r in rows where r.Voltage != null:
        (r.Rtop, r.Rbot) = SuggestResistorPair(r.Voltage)

    return rows
```

### 10.2 Mode 判定

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
            if (in.HallType == NonFR and in.DcSource == SET) or
               (in.HallType == FR    and in.DcSource == VSP):
                return 0                                              // O1b
            if (in.HallType == NonFR and in.DcSource == VSP) or
               (in.HallType == FR    and in.DcSource == SET):
                return 7                                              // O2b

    if in.Standby == Shutdown and (in.Pole == 4p or in.Pole == RD)
       and in.SoftSwitch == Deg22_5:                                  return 1  // O3

    if (in.Pole == 4p or in.Pole == 8p) and in.HallType == FR:        return 2  // O4
    if (in.Pole == 4p or in.Pole == 6p) and in.HallType == NonFR:     return 3  // O5

    return null
```

### 10.3 共用電壓公式

```
function VoltageFromField(fieldValue, useTimes4 = false) -> (V_high, V_low):
    code = (useTimes4 ? fieldValue * 4 : fieldValue) + 20
    V_high = (256 - code) * 5.0 / 256.0     // 高半段
    V_low  =  code        * 5.0 / 256.0     // 低半段
    return (V_high, V_low)
```

### 10.4 XP1 計算

```
function CalcXp1(in, mode) -> ResultRow:
    // Open-loop：reg_data_02[4:0]，含 ×4
    if mode in {0,1,2,3,7}:
        if in.Xp1DutyPct < 4.6:
            return Row(Pin="XP1", Voltage=ChooseEndpoint(branch),
                       Status=Clamped, Rule="§3.1 below 4.6%")
        field02 = floor(in.Xp1DutyPct / 1.5625)            // 0..31
        (V_high, V_low) = VoltageFromField(field02, useTimes4=true)
        branch = SelectXp1Branch(in, mode)                  // §4 表第 3 欄
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_02[4:0]",
                   RegValue=field02, Rule="§4 mode=" + mode)

    // Close-loop：reg_data_06[6:0]，無 ×4
    if mode in {4,5,6}:
        if in.MinRpm >= 60 * in.RpmRatio:
            return Row(Pin="XP1", Voltage=ChooseEndpoint(branch),
                       Status=Clamped, Rule="§3.1 RPM at endpoint")
        field06 = round(in.MinRpm / in.RpmRatio - 59)      // §9 待確認 6
        (V_high, V_low) = VoltageFromField(field06, useTimes4=false)
        branch = SelectXp1Branch(in, mode)
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_06[6:0]",
                   RegValue=field06, Rule="§4 close-loop mode=" + mode)
```

### 10.5 YP 計算

```
function CalcYp(in, mode) -> ResultRow:
    // Close-loop：reg_data_07，無 ×4
    if mode in {4,5,6}:
        field07 = round((in.MinRpm / in.RpmRatio - 59) / 4)
        (_, V_low) = VoltageFromField(field07, useTimes4=false)
        return Row(Pin="YP", Voltage=V_low, RegField="reg_data_07",
                   RegValue=field07, Rule="§5 close-loop mode=" + mode)

    // Open-loop：reg_data_04[6:0]
    if in.Yp1DutyPct < 4.6:
        return Row(Pin="YP", Voltage=ChooseEndpoint(branch),
                   Status=Clamped, Rule="§3.1 below 4.6%")
    field04 = floor(in.Yp1DutyPct / 0.39)                  // 0..127
    (V_high, V_low) = VoltageFromField(field04, useTimes4=false)
    branch = SelectYpBranch(in, mode)                       // §5 表第 3 欄
    return Row(Pin="YP", Voltage=branch, RegField="reg_data_04[6:0]",
               RegValue=field04, Rule="§5 mode=" + mode)
```

### 10.6 SS 計算

```
function CalcSs(in, mode) -> ResultRow:
    // Close-loop：直接 rpm_ratio 查表 (§6.3)
    if mode in {4,5,6}:
        v = SS_TABLE_63[in.RpmRatio]                       // 8 entries
        (lo, hi) = SS_TABLE_63_RANGE[in.RpmRatio]
        status = (in.MinRpm > lo and in.MaxRpm < hi) ? OK : OutOfRange
        return Row(Pin="SS", Voltage=v, Status=status,
                   Rule="§6.3 ratio=" + in.RpmRatio)

    // Open-loop：依 mode + pole 選 profile A/B (§6.2)，再用 ss_time 查 §6.1 表
    profile = SelectSsProfile(mode, in.Pole)                // A / B / null
    if profile == null:
        return Row(Pin="SS", Status=Invalid, Rule="§6.2 no profile")
    v = (profile == A ? SS_TABLE_A[in.SsTime] : SS_TABLE_B[in.SsTime])
    return Row(Pin="SS", Voltage=v,
               RegField="REG08[7:5]", RegValue=in.SsTimeIndex,
               Rule="§6.2 mode=" + mode + " pole=" + in.Pole + " profile=" + profile)
```

`SS_TABLE_A` / `SS_TABLE_B` 來自 §6.1，`SS_TABLE_63` / `SS_TABLE_63_RANGE` 來自 §6.3。

### 10.7 建議 resistor 對

```
function SuggestResistorPair(targetV) -> (Rtop, Rbot):
    // 依 50 µA pull-high 模型反推 divider，找最接近的 E96 標準值對
    return ReverseLookup(targetV)
```

---

## 11. 原始 pseudo-code spec

原始 200 行 pseudo-code 留在開發團隊的對話紀錄中，本份規格不重複貼，避免兩邊飄。本 refactor 只用 §1 ~ §10 的整理結果作為實作依據。
