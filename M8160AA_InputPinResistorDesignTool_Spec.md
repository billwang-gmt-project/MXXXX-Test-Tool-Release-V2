# M8160AA Input Pin Resistor Design Tool — 規格

## 這份文件在做什麼

M8160AA 晶片有 4 支類比腳位（Mode / XP1 / YP / SS）給使用者外接 resistor divider 設定行為。這份規格描述一個新的設計輔助工具：使用者選好應用條件（馬達極數、開/閉迴路、PWM 模式…），工具自動算出 4 支腳位該打多少電壓、建議 resistor pair、以及哪些 register bit 會被工具強制鎖定。

Refactor 自原始的 200 行 pseudo-code spec，整理成易於 review 的表格 + 流程。

> **與現行實作對齊狀態**（2026-05-07 更新）：§1 ~ §10 已對齊現行實作；§11 保留設計者（Allen）原始 pseudo-code 作為歷史源頭依據，**不再與實作完全一致**。和原稿的差異總覽見 §12 與 [`M8160_Forward_Mode_Memo.md`](M8160_Forward_Mode_Memo.md) §C。

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
| `reg_data_0d[6]` | 0 | 強制 Direct PWM（bypass #4） |
| `reg_data_01[7]` | 0 | PWM non-invert（bypass #5、#6） |
| `reg_data_0e[7]` | 1 | 強制 Force startup（bypass #8） |
| `reg_data_03[7:6]` | 0 | Soft switch = Square（bypass #9） |

當使用者選 **Hall type = non-FR**，再加一條：

| Register bit | 寫入值 | 對應原因 |
|--------------|:------:|----------|
| `reg_data_0f[1]` | 0 | bypass #7 在 non-FR 分支 |

> ⚠ 與 §11 設計原稿差異：原稿在開頭無條件寫 `reg_data_0f[1] = 0`（隱含「預設 non-FR」）；現行實作改為條件式 `if HallType == NonFR then write 0`。

### 1.2 Mode-derived 6 個常數（不在使用者控制下）

除了 §1.1 的 forced rows，工具一律 derive 下列 6 個常數 row（無對應 UI dropdown，使用者沒得選；commit `225a6893`）：

| Field | Register bit | 寫入規則 |
| ----- | ------------ | -------- |
| `fan_curve` | `reg_01[4:3]` | open-loop（mode∈{0,1,2,3,7}）→ 2、close-loop（mode∈{4,5,6}）→ 1 |
| `rpm_ratio` | `reg_01[2:0]` | open-loop → 7（常數）、close-loop → 使用者選的 `RpmRatioIndex` |
| `icmd_sw` | `reg_06[7]` | all modes → 0 |
| `auto_soft_sw` | `reg_0a[7]` | all modes → 1 |
| `duty_deb_sw` | `reg_0d[7]` | mode∈{4,5,6,7} → 1、其他 → 0 |
| `low_freq_rev` | `reg_0e[5]` | all modes → 1 |

來源：commit `225a6893`。§11 設計原稿完全未列。

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

**重點差異說明**（硬體工程師確認）：

- **O1/O2 群組**（mode 0 / 7，4p/8p、shutdown）— 差別在：
  1. YP 上半段 PWM 有無反相（O1a vs O2a）
  2. YP 下半段 DC mode 搭配 FR 差異（O1b vs O2b）
  3. 另外 mode 7 無 force startup 功能
- **O3/O4/O5 群組**（mode 1 / 2 / 3）— 差別在 Soft switch type：
  - O3：Mode 1 限定 soft switch = 22.5°
  - O4：Mode 2 走 FR
  - O5：Mode 3 走 non-FR

每一列的條件取 **完整 AND**（不是只列關鍵欄）以消除歧義。所有合法輸入組合在這張表恰好命中一列。

---

## 3. 電壓共用公式

XP1、YP 兩支腳位都用同一條公式（SS 不用，看 §6）。

```
V_high(code) = (256 − code) × 5 / 256        // 高半段（最低點 5 V，最高點接近 0 V）
V_low (code) =  code        × 5 / 256        // 低半段（最低點 0 V，最高點接近 5 V）
```

其中 `code = field_value + 20`，部分情境下 `field_value` 還要先 ×4（看 §4）。每 step 解析度 `5 / 256 ≈ 0.01953125 V`。

### 3.1 4.64% 開路（XP1 / YP1）

XP1 / YP1 dropdown 預設值即 **4.64 %**（dropdown 第 12 格，理論值 `38.28 × 12 / 99 = 4.6398`），使用者保留這個值代表「腳位不接外部 divider，腳位開路」（不是 clamp，而是 design-time 的「省略電阻」設定）。原始 chip-designer pseudocode（§11）寫 `xp1==4.6%`，本工具因 dropdown 改 0.01% 精度後選擇對齊到 dropdown 端點 4.64。

判定條件（容差 0.005，因為 dropdown round 到 0.01% 精度）：

```
abs(Xp1DutyPercent − 4.64) < 0.005   →  XP1 開路
abs(Yp1DutyPercent − 4.64) < 0.005   →  YP1 開路
```

開路時 calculator 不計算電阻：

| 欄位 | 開路時的值 |
| ---- | ---------- |
| `Status` | `OpenCircuit`（藍底淺藍字，UI 顯示 "Open circuit / no resistor"） |
| `TargetVoltage` | **依 §4 / §5 V_high/V_low branch**：V_high → `5.0 V`、V_low → `0.0 V`（與 spec §11 pseudo-code 對齊） |
| `RSimpleOhm` / `RTopOhm` / `RBotOhm` | `null`（DataGrid 顯示 "—"） |
| `ActualVoltage` | `null` |
| `xp1` register field（XP1 開路時） | **3**（V=5.0V → fix-H 或 V=0.0V → fix-L 都產生同 field，§9 #5 `xp1Fix → 3`） |
| `yp1` register field（YP1 開路時） | **12**（V=5.0V → fix-H 或 V=0.0V → fix-L 都產生同 field，§9 #7 `ypFix → 12`） |

> **與 Close-loop field clamp 區分**：close-loop（mode 4/5/6）的 `rp1` / `rp2` 經 `clamp(field, 0, 127)` / `clamp(field, 0, 255)` 截斷至 7/8-bit 上下限。此 clamp **靜默發生、不產生 status flag**（無 `Status = Clamped`），與 4.64% 開路無關。`Clamped` 為預留 enum 值，目前 Forward 計算路徑不會實際 set 此 status。

---

## 4. XP1 pin 電壓對照表

`xp1` 看 mode 跟子條件決定走哪一條分支。資料來源有兩種：

- **Open-loop modes 0/1/2/3/7**：使用者填 `xp1_duty(%)` → `field = floor(duty / 1.5625)` → `code = field × 4 + 20`（注意有 ×4）。
- **Close-loop modes 4/5/6**：`field = round(min_rpm / rpm_ratio − 59)`，clamp `[0, 127]` → `code = field + 20`（沒有 ×4）。

> ✓ 2026-05-08 起：close-loop XP1 公式回退至設計原稿（含 `−59` offset），與 §11 一致。歷史 commit `33fcc857`（2026-05-07）曾移除 `−59`，後因使用者新 spec 回退（詳見 [`M8160_Forward_Mode_Memo.md`](M8160_Forward_Mode_Memo.md) §C.4）。Forward 與 Reverse `rp1 = adc1[6:0] − 20` round-trip 不再對齊，Compare 視窗會出現固定 ±59 register field 差異（spec 容許）。

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

> ⚠ Close-loop **沒有** 4.64% 開路語意（design-time「腳位開路」只屬於 open-loop 的 XP1/YP1）。現行實作只做 `clamp(rp1, 0, 127)`；§11 設計原稿的 `xp1==60*rpm_ratio → 5V/0V` 規則 **不採用**。

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
| 4 / 5 / 6 | – | V_low（Allen direct） | `reg_data_07 = round(max_rpm / rpm_ratio / 4 − 59)`，clamp `[0, 255]`；pin V = `reg_data_07 × 5/256`（**無 +20 offset**） |

`reg_data_04` 那幾列都套 §3.1 的 4.64 % 開路規則（YP1 curve，open-loop）。

> ✓ 2026-05-08 起 close-loop YP 公式：
>
> 1. **公式來源**：原稿 `min_rpm`，現行實作 **改用 `max_rpm`**（YP 對應上限 RPM）。沿用自 commit `33fcc857`。
> 2. **−59 offset 保留**：2026-05-08 從無 offset 回退至有 offset，與 §11 設計原稿一致。
> 3. **+20 offset 不採用**：close-loop YP 沿 Allen direct convention，pin V = `field × 5/256`，與 Reverse `yp_pin[7:0] = reg_07[7:0]`（`rp2 = adc2`）對應。
> 4. **mode 4/5/6 額外固定**：`yp1 = 12`（`reg_04[6:0]` ≈ 4.6%）、`yp2 = 127`（`reg_05[6:0]` ≈ 100%）為硬寫常數。2026-05-08 yp1 從 26 改 12 對齊「YP1 固定 4.6%」spec。

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
| 3 | 4p | A |
| 3 | 6p | B |
| 4 / 5 / 6 | – | **看 §6.3** |

> 註：`mode == 2` 跟 `mode == 0/7` 共用第一組（4p → A、8p → B）；單獨列出的 4p/6p 規則屬於 `mode == 3`。

### 6.3 Close-loop SS 電壓表

Close-loop（mode 4/5/6）時，SS 電壓只看 `rpm_ratio`：

| rpm_ratio | min_rpm 合法範圍 (RP1, 7-bit) | max_rpm 合法範圍 (RP2, 8-bit) | SS 電壓 |
|----------:|------------------------------:|------------------------------:|--------:|
| 2.54  | 149.86 – 424.18   | 149.86 – 2740.66   | 0.00 V |
| 3.72  | 219.48 – 621.24   | 219.48 – 4013.88   | 0.92 V |
| 5.59  | 329.81 – 933.53   | 329.81 – 6031.61   | 1.55 V |
| 8.30  | 489.70 – 1386.10  | 489.70 – 8955.70   | 2.18 V |
| 12.20 | 719.80 – 2037.40  | 719.80 – 13163.80  | 2.80 V |
| 17.79 | 1049.61 – 2970.93 | 1049.61 – 19195.41 | 3.43 V |
| 25.93 | 1529.87 – 4330.31 | 1529.87 – 27978.47 | 4.06 V |
| 37.96 | 2239.64 – 6339.32 | 2239.64 – 40958.84 | 5.00 V |

> **2026-05-08 spec：min_rpm 與 max_rpm 上限不同**。代數推導：
>
> - `RP1` 公式 `round(min_rpm/ratio − 59)`，扣 ADC −20 階 shift → min_rpm 上限 `((128 − 20) + 59) × ratio = 167 × ratio`（2026-05-11 修訂；先前未扣 shift 為 187·ratio）
> - `RP2` 公式 `round(max_rpm/ratio/4 − 59)`，clamp `[0, 255]` → max_rpm 上限 `(255 × 4 + 59) × ratio = 1079 × ratio`
> - Min / Max 共用下限 `59 × ratio`（field=0 起算的 RPM）
>
> 工具的 `AutoPickRpmRatio` 與 SS pin OutOfRange 旗標都用此**雙 bound**檢查（`MinRpm < 167·ratio`（含 ADC −20 階 shift）且 `MaxRpm < 1079·ratio`）。重用 [`Helpers/M8160SpeedCurveCalculator.cs`](../../../Helpers/M8160SpeedCurveCalculator.cs) 的 `Rp1Factor = 167` / `Rp2Factor = 1079` 常數。

---

## 7. 計算流程（白話版）

工具拿到使用者填的 9 個輸入後，依序做：

1. **判定 mode** — 套 §2.1 表，找出唯一命中的列（C1~O5），得到 mode 0~7。
2. **填 Mode pin 電壓** — 直接從 §2 對照表查 mode 對應電壓。
3. **加入強制 register row** — 依 §1.1 的鎖定規則（close-loop 4 條 + non-FR 1 條），把被工具寫死的 bit 加到結果列表。
4. **加入 mode-derived 6 常數 row** — 依 §1.2 寫 `fan_curve` / `rpm_ratio` / `icmd_sw` / `auto_soft_sw` / `duty_deb_sw` / `low_freq_rev`。
5. **算 XP1 pin 電壓** — 套 §4 表選分支，套 §3 公式（或 §3.1 4.64% 開路）。
6. **算 YP pin 電壓** — 套 §5 表選分支，套 §3 公式（或 §3.1 4.64% 開路）；close-loop 額外加 `yp1=12 (≈4.6%)` / `yp2=127 (≈100%)` 常數 row。
7. **算 SS pin 電壓** — close-loop 走 §6.3 查表（不寫 ss_time row）。SS 電壓由 ratio 索引單純查表；OutOfRange 旗標由 RP1/RP2 雙 bound 決定（`MinRpm < 167·ratio`（含 ADC −20 shift）且 `MaxRpm < 1079·ratio`，均 > `59·ratio`）。Open-loop 走 §6.2 + §6.1 兩段查表。
8. **算建議 resistor 對** — 每一支腳位的目標電壓 → 推算外接 R_top / R_bot 一對標準值（資料源自既有的 ADC 對照模型）。
9. **整合輸出** — 全部塞進 result table（看 §8）。

任何一步若找不到對應列、或數值越界，該 row 在 result table 標紅（status = Out-of-range / Invalid）。

---

## 8. 結果呈現（DataGrid）

每一支受影響的腳位 / register bit 一個 row，欄位設計如下：

| 欄位 | 內容說明 | 範例 |
|------|----------|------|
| **Pin / Field** | Mode / XP1 / YP / SS，或 `reg_data_XX[B:b]` | `Mode pin` / `reg_data_0d[6]` |
| **Voltage** | 目標電壓 | `2.80 V` |
| **R_top / R_bot** | 建議外接 resistor 對（Ω） | `47k / 33k` |
| **Register value** | bit 值（dec / hex / bin） | `0 / 0x00 / 0b0` |
| **Source rule** | 哪一條 spec 規則觸發 | `§2 row C3（close-loop, 6p）` |
| **Status** | OK / Clamped / Out-of-range / Forbidden-band | 顏色標籤 |

Result table 列出 **四類** row：

1. **強制鎖定**（§1.1）— close-loop 強制（4 條）+ non-FR 強制（1 條），最多 5 個 bit
2. **mode-derived 常數**（§1.2）— `fan_curve` / `rpm_ratio` / `icmd_sw` / `auto_soft_sw` / `duty_deb_sw` / `low_freq_rev` 共 6 個 row（與 mode 是否 OL/CL 有關）
3. **工具 derive**（§4 / §5 / §6）— 使用者隱含選到的：
   - open-loop：`reg_data_02[4:0]` (xp1)、`reg_data_04[6:0]` (yp1)、`REG08[7:5]` (ss_time)
   - close-loop：`reg_data_06[6:0]` (rp1)、`reg_data_07[7:0]` (rp2)、`reg_data_04[6:0] = 12` (yp1 const ≈ 4.6%)、`reg_data_05[6:0] = 127` (yp2 const ≈ 100%)；**不寫 ss_time**（rpm_ratio 索引已隱含於 §1.2 row）
4. **腳位電壓** — Mode / XP1 / YP / SS 四支腳位

Forward 計算共產出 **4 row 腳位電壓 + 至多 5 row Forced + 6 row mode-derived + 1~3 row XP1/YP/SS derive ≈ 11~15 row register**。

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

範例輸入：`Open-loop / 4p / Shutdown / Direct PWM / Non-invert / non-FR / Force / Square` + `Xp1Duty = 20.31 %` / `Yp1Duty = 10.16 %` / `SsTime = 2 s` → 命中 §2.1 row O2a → mode = 7：

| Pin / Field        | Voltage | RegValue (dec/hex/bin) | Source rule                  | Status |
|--------------------|--------:|------------------------|------------------------------|--------|
| **Mode pin**       | 5.000 V | —                      | §2 row O2a                   | OK     |
| **XP1 pin**        | 3.672 V | —                      | §4 mode=7, V_high            | OK     |
| **YP pin**         | 4.102 V | —                      | §5 mode=7 + DirectPWM, V_high| OK     |
| **SS pin**         | 4.023 V | —                      | §6.2 mode=7, 4p, profile A   | OK     |
| reg_data_02[4:0]   |   —     | 12 / 0x0C / 0b01100    | §4 derive (xp1 = 18.75 %)    | OK     |
| reg_data_04[6:0]   |   —     | 26 / 0x1A / 0b0011010  | §5 derive (yp1 = 10.16 %)    | OK     |
| reg_08[7:5]        |   —     | 2 / 0x02 / 0b010       | §6 derive (ss_time = 2 s)    | OK     |
| reg_0f[1]          |   —     | 0 / 0x00 / 0b0         | §1.1 force fr_pin_sw (non-FR)| OK     |
| reg_01[4:3]        |   —     | 2 / 0x02 / 0b10        | §1.2 fan_curve (open-loop)   | OK     |
| reg_01[2:0]        |   —     | 7 / 0x07 / 0b111       | §1.2 rpm_ratio (open-loop=7) | OK     |
| reg_06[7]          |   —     | 0 / 0x00 / 0b0         | §1.2 icmd_sw                 | OK     |
| reg_0a[7]          |   —     | 1 / 0x01 / 0b1         | §1.2 auto_soft_sw            | OK     |
| reg_0d[7]          |   —     | 1 / 0x01 / 0b1         | §1.2 duty_deb_sw (mode=7)    | OK     |
| reg_0e[5]          |   —     | 1 / 0x01 / 0b1         | §1.2 low_freq_rev            | OK     |

> 範例輸入是 open-loop，§1.1 close-loop 4 條 forced row 不出現（只剩 non-FR 1 條），總計 14 row（4 pin + 1 forced + 6 mode-derived + 3 derive）。Close-loop 範例會多 4 條 forced row、少 1 條 ss_time row、多 yp1=12 / yp2=127 兩條 close-loop 常數 row。
>
> R_top / R_bot 欄位省略；建議 resistor 對由 §10.7 的 `SuggestResistorPair(targetV)` 從目標電壓推算（E96 標準值），不在範例 spec 中固定列出。

Status 顏色（Forward 結果旗標）：

- **Ok** — 綠（預設，計算正常）
- **OpenCircuit** — 藍底淺藍字（XP1 / YP1 duty = 4.64 %，腳位開路、無需電阻）
- **OutOfRange** — 紅底白字（close-loop SS 範圍檢查 RP1/RP2 雙 bound 不通過：`MinRpm ∈ (59·ratio, 167·ratio)`（含 ADC −20 shift）且 `MaxRpm ∈ (59·ratio, 1079·ratio)`，2026-05-11 spec）
- **Invalid** — 紅底白字（SS open-loop profile 找不到，mode + pole 組合無對應）
- **Clamped** — 黃底淺黃字（**預留**，目前 Forward 計算路徑不會 set 此值；close-loop `clamp(field, 0, 127)` / `clamp(field, 0, 255)` 為靜默 clamp，不產生 status flag）
- **Forbidden** — 紅底白字（**預留**，Forward 模式 forbidden 由 ViewModel 個別 `IsXxxForbidden` 屬性處理，不走此旗標）

---

## 9. 硬體工程師 review 結果

下列 7 項已由硬體工程師確認，本份 spec 已套用：

| # | 項目 | 結論 |
|:-:|------|------|
| 1 | "Direct PWM" 強制鎖定的 bit | `reg_data_0d[6]` |
| 2 | §2.1 Mode 判定表完整性 | 9 列已涵蓋全部合法輸入組合、互斥不漏；O1/O2 / O3/O4/O5 群組差異另見 §2.1 重點差異說明 |
| 3 | §3 電壓公式單位 | **V**（5/256 ≈ 0.01953125 V/step） |
| 4 | §6.2 SS profile 的 mode 對應 | `mode == 2` 套用 4p→A / 8p→B；4p/6p 拆分屬於 `mode == 3` |
| 5 | §4 ×4 step 適用範圍 | 只在 **open-loop XP1**；close-loop 不適用 |
| 6 | §5 close-loop 整數捨入 | **四捨五入**（`Math.Round`） |
| 7 | §8 Result 列表範圍 | **全部都列**：強制鎖定（§1.1）+ mode-derived 6 常數（§1.2，commit `225a6893`）+ 工具 derive 的 `xp1` / `yp1` / `rp1` / `rp2` / `ss_time` / close-loop `yp1=12 (≈4.6%)` / `yp2=127 (≈100%)` |
| 8 | Close-loop XP1 / YP 公式 | 含 `−59` offset（2026-05-08 與 §11 設計原稿對齊；歷史 commit `33fcc857` 曾移除）；YP 改用 `max_rpm` 並走 Allen direct convention（無 `+20`） |

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
        rows.add(RegRow("reg_data_0d[6]", 0, "§1.1 force Direct PWM"))
        rows.add(RegRow("reg_data_01[7]", 0, "§1.1 force PWM non-invert"))
        rows.add(RegRow("reg_data_0e[7]", 1, "§1.1 force align_sw"))
        rows.add(RegRow("reg_data_03[7:6]", 0, "§1.1 force ssw=square"))
    if inputs.HallType == NonFR:
        rows.add(RegRow("reg_data_0f[1]", 0, "§1.1 force fr_pin_sw"))

    // Step 3.5: mode-derived 6 個常數 (§1.2)
    isOL = mode in {0,1,2,3,7}
    rows.add(RegRow("reg_01[4:3]", isOL ? 2 : 1,             "§1.2 fan_curve"))
    rows.add(RegRow("reg_01[2:0]", isOL ? 7 : RpmRatioIndex, "§1.2 rpm_ratio"))
    rows.add(RegRow("reg_06[7]",   0,                        "§1.2 icmd_sw"))
    rows.add(RegRow("reg_0a[7]",   1,                        "§1.2 auto_soft_sw"))
    rows.add(RegRow("reg_0d[7]",   (mode in {4,5,6,7}) ? 1 : 0, "§1.2 duty_deb_sw"))
    rows.add(RegRow("reg_0e[5]",   1,                        "§1.2 low_freq_rev"))

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
        if abs(in.Xp1DutyPct - 4.64) < 0.005:
            // 4.64% = 腳位開路（不接外部 divider）→ 不算電阻
            // §11 pseudo-code：依 V_high/V_low branch 輸出 5V 或 0V；reverse calculator
            // 對兩者都走 fix-H/fix-L → 同 field = 3（`xp1Fix = xp1FixH || xp1FixL`）。
            branch = SelectXp1Branch(in, mode)        // V_high or V_low (§4)
            openV = 5.0 if branch == V_high else 0.0
            return Row(Pin="XP1", Voltage=openV,
                       RegValue=3, Status=OpenCircuit,
                       Rule=f"§3.1 4.64% open circuit ({branch} → fix → 3)")
        field02 = floor(in.Xp1DutyPct / 1.5625)            // 0..31
        (V_high, V_low) = VoltageFromField(field02, useTimes4=true)
        branch = SelectXp1Branch(in, mode)                  // §4 表第 3 欄
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_02[4:0]",
                   RegValue=field02, Rule="§4 mode=" + mode)

    // Close-loop：reg_data_06[6:0]，無 ×4
    if mode in {4,5,6}:
        # 2026-05-08 修訂：RP1 公式含 −59，回退至 §11 設計原稿
        # pin V 沿用 §11 +20 convention（VoltageFromField 內部 code = field + 20），對應
        # Reverse `xp1_pin[6:0] − 20 = reg_06[6:0]`（M8160AdcCalculator.cs:586，未改）
        # 注意：Forward 加 −59 後與 Reverse round-trip 不再對齊，Compare 視窗會出現
        # 固定 ±59 register field 差異（spec 容許）
        field06 = clamp(round(in.MinRpm / in.RpmRatio - 59), 0, 127)   // 含 −59 offset
        (V_high, V_low) = VoltageFromField(field06, useTimes4=False)
        branch = SelectXp1Branch(in, mode)                  // §4 表 mode 4/5/6 列
        return Row(Pin="XP1", Voltage=branch, RegField="reg_data_06[6:0]",
                   RegValue=field06, Rule="§4 close-loop mode=" + mode)
```

### 10.5 YP 計算

```
function CalcYp(in, mode) -> ResultRow:
    // Close-loop：reg_data_07，無 ×4
    if mode in {4,5,6}:
        # 2026-05-08 修訂：RP2 公式含 −59，使用 max_rpm（不是 min_rpm），Allen direct convention
        # pin V = field × 5/256（無 +20 offset），對應 Reverse `yp_pin[7:0] = reg_07[7:0]`
        # （M8160AdcCalculator.cs:601，未改）
        field07 = clamp(round(in.MaxRpm / in.RpmRatio / 4 - 59), 0, 255) // 含 −59 offset
        V_low = field07 * 5.0 / 256.0                        // Allen direct: 無 +20
        # YP1 / YP2 為 mode 4/5/6 常數
        yield RegisterRow(reg="reg_data_04[6:0]", value=12,  rule="yp1 ≈ 4.6% (spec 2026-05-08)")
        yield RegisterRow(reg="reg_data_05[6:0]", value=127, rule="yp2 ≈ 100% (constant)")
        return Row(Pin="YP", Voltage=V_low, RegField="reg_data_07",
                   RegValue=field07, Rule="§5 close-loop mode=" + mode + " (rp2, Allen direct)")

    // Open-loop：reg_data_04[6:0]（YP1 curve，mode ∈ {0,1,2,3,7}）
    if abs(in.Yp1DutyPct - 4.64) < 0.005:
        // 4.64% = 腳位開路（不接外部 divider）→ 不算電阻
        // YP2 curve（close-loop, mode ∈ {4,5,6}）不適用此規則
        // §11 pseudo-code：依 V_high/V_low branch 輸出 5V 或 0V；reverse calculator
        // 對兩者都走 fix-H/fix-L → 同 field = 12（`ypFix = ypFixH || ypFixL`）。
        branch = SelectYpBranch(in, mode)         // V_high or V_low (§5)
        openV = 5.0 if branch == V_high else 0.0
        return Row(Pin="YP", Voltage=openV,
                   RegValue=12, Status=OpenCircuit,
                   Rule=f"§3.1 4.64% open circuit ({branch} → fix → 12)")
    field04 = floor(in.Yp1DutyPct / 0.390625)              // 0..127, 50/128 per bit
    (V_high, V_low) = VoltageFromField(field04, useTimes4=false)
    branch = SelectYpBranch(in, mode)                       // §5 表第 3 欄
    return Row(Pin="YP", Voltage=branch, RegField="reg_data_04[6:0]",
               RegValue=field04, Rule="§5 mode=" + mode)
```

### 10.6 SS 計算

```
function CalcSs(in, mode) -> ResultRow:
    // Close-loop：直接 rpm_ratio 查表 (§6.3)
    // 2026-05-08 spec：OutOfRange 改用 RP1/RP2 雙 bound（之前單 bound 漏 RP1 max）
    if mode in {4,5,6}:
        v = SS_TABLE_63_VOLTAGE[in.RpmRatio]               // 8 entries (SS 電壓)
        ratio = ratios[in.RpmRatio]
        rp1Upper = 167 * ratio                              // RP1 7-bit saturation 含 ADC −20 shift (MinRpm 上限)
        rp2Upper = 1079 * ratio                             // RP2 8-bit saturation (MaxRpm 上限)
        minLower = 59 * ratio                               // 共用下限 (field=0)
        minOk = in.MinRpm > minLower and in.MinRpm < rp1Upper
        maxOk = in.MaxRpm > minLower and in.MaxRpm < rp2Upper
        status = (minOk and maxOk) ? OK : OutOfRange
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

## 11. 原始 pseudo-code（硬體工程師提供版）

下列為硬體工程師交付的最終 pseudo-code 全文，作為 §1 ~ §10 refactor 的源頭依據。實作以 §1 ~ §10 為主，本節作為交叉對照。

> **§3.1 開路 vs 原始 pseudo-code 差異**：
>
> 1. **觸發值**：原始 pseudo-code 寫 `if(xp1==4.6%)` / `if(yp1==4.6%)`；實作端 dropdown 改用 0.01% 精度後改為 `if(abs(duty - 4.64) < 0.005)`（dropdown 第 12 格 `38.28 × 12 / 99 = 4.6398` round 到 2 位小數 = 4.64，是最接近原 4.6 的 grid 值）。語意等價，差異只在數字呈現。
> 2. **輸出電壓**：實作完整對齊原始 pseudo-code，依 §4 / §5 V_high/V_low branch 分別輸出 `5.0 V` 或 `0.0 V`。Status 統一為 `OpenCircuit`（藍底，UI 顯示 "Open circuit / no resistor"），並把 `RSimpleOhm` / `RTopOhm` / `RBotOhm` 設為 `null`（DataGrid 顯示 "—"）以表達「不需要電阻」。Reverse 路徑對 5.0V / 0.0V 都會走 fix-H / fix-L 分支算出相同的 register field（`xp1Fix = xp1FixH || xp1FixL` → `xp1=3`；`ypFix` 同理 → `yp1=12`），所以 Forward / Reverse Compare 在開路情境下 match。

```
// ─────────── inputs ───────────
//1.open loop or close loop
//1.1.close loop(keyin max/min rpm, rpm_ratio=lookup table(ss pin)
//2.SO pole(4p/6p/8p/rd)
//3.shutdown or min
//4.direcr PWM or DC mode(VSP/SET)
//5.PWM invert(under direcr PWM)
//6.VSP or SET(under DC mode)
//7.FR enable(hall SW) or non-FR(hall element)
//8.force startup(反正 or 不反正)
//9.soft switch degree(22.5/45/sine)

	reg_data_0f[1]=0;//non-FR, bypass step7

//Mode PIN
if(close_loop)
	reg_data_0d[6] ← 0;//direct PWM, bypass step4
	reg_data_01[7] ← 0;//PWM non-invert, bypass step5/6
	reg_data_0e[7] ← 1;//force startup, bypass step8
	reg_data_03[7:6] ← 0;//soft switch, bypass step9
	if(pole==4p)
		if(FR_enable)
			mode ← 6;//4.06V
		else
			mode ← 5;//3.43V
	else if(pole==6p or pole==8p)
		mode ← 4;//2.8V
else if(open_loop)
	if((pole==4p or 8p) & shutdown & (PWM_invert or DC_mode))
		mode ← 0;//0V
	else if((pole==4p or 8p) & shutdown & (PWM_non_invert or DC_mode))
		mode ← 7;//5V
	else if((pole==4p or rd) & shutdown & soft_switch==22.5)
		mode ← 1;//0.92V
	else if((pole==4p or 8p) & FR_enable)
		mode ← 2;//1.55V
	else if((pole==4p or 6p) & non_FR)
		mode ← 3;//2.18V

//XP1 PIN
if(mode==0 & shutdown)
	if(non_FR & SET)
		if(xp1==4.6%)
			xp1_pin ← 5v;
		else
			xp1_pin ← (256 - ((reg_data_02[4:0] * 4) + 20)) * 0.01953125V
	else(FR_enable & VSP)
		if(xp1==4.6%)
			xp1_pin ← 0v;
		else
			xp1_pin ← ((reg_data_02[4:0] * 4) + 20) * 0.01953125V
if(mode==1 & shutdown)
	if(non_FR)
		if(xp1==4.6%)
			xp1_pin ← 5v;
		else
			xp1_pin ← (256 - ((reg_data_02[4:0] * 4) + 20)) * 0.01953125V
	else(FR_enable)
		if(xp1==4.6%)
			xp1_pin ← 0v;
		else
			xp1_pin ← ((reg_data_02[4:0] * 4) + 20) * 0.01953125V
if(mode==7 & shutdown)
	if(non_FR & VSP)
		if(xp1==4.6%)
			xp1_pin ← 5v;
		else
			xp1_pin ← (256 - ((reg_data_02[4:0] * 4) + 20)) * 0.01953125V
	else(FR_enable & SET)
		if(xp1==4.6%)
			xp1_pin ← 0v;
		else
			xp1_pin ← ((reg_data_02[4:0] * 4) + 20) * 0.01953125V
else if((mode==2) & FR_enable)
	if(shutdown)
		if(xp1==4.6%)
			xp1_pin ← 5v;
		else
			xp1_pin ← (256 - ((reg_data_02[4:0] * 4) + 20)) * 0.01953125V
	else(minimum)
		if(xp1==4.6%)
			xp1_pin ← 0v;
		else
			xp1_pin ← ((reg_data_02[4:0] * 4) + 20) * 0.01953125V
else if((mode==3) & non_FR)
	if(shutdown)
		if(xp1==4.6%)
			xp1_pin ← 5v;
		else
			xp1_pin ← (256 - ((reg_data_02[4:0] * 4) + 20)) * 0.01953125V
	else(minimum)
		if(xp1==4.6%)
			xp1_pin ← 0v;
		else
			xp1_pin ← ((reg_data_02[4:0] * 4) + 20) * 0.01953125V
else if((mode==4) & FR_enable & shutdown)
	if(pole==6p)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 5v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (256 - (reg_data_06[6:0] + 20)) * 0.01953125V
	else(pole==8p)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 0v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (reg_data_06[6:0] + 20) * 0.01953125V
else if((mode==5) & non_FR)
	if(shutdown)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 5v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (256 - (reg_data_06[6:0] + 20)) * 0.01953125V
	else(minimum)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 0v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (reg_data_06[6:0] + 20) * 0.01953125V
else if((mode==6) & FR_enable)
	if(shutdown)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 5v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (256 - (reg_data_06[6:0] + 20)) * 0.01953125V
	else(minimum)
		if(xp1==60*rpm_ratio)
			xp1_pin ← 0v;
		else
			reg_data_06[6:0] ← min_rpm/rpm_ratio - 59;
			xp1_pin ← (reg_data_06[6:0] + 20) * 0.01953125V

//YP PIN
if(mode==0)
	if(direcr_PWM & PWM_invert)
		if(yp1==4.6%)
			yp1_pin ← 5v;
		else
			yp1_pin ← (256 - (reg_data_04[6:0] + 20)) * 0.01953125V
	else(DC_mode)
		if(yp1==4.6%)
			yp1_pin ← 0v;
		else
			yp1_pin ← (reg_data_04[6:0] + 20) * 0.01953125V
else if(mode==1 or mode==2 or mode==3)
	if(non_force_startup)
		if(yp1==4.6%)
			yp1_pin ← 5v;
		else
			yp1_pin ← (256 - (reg_data_04[6:0] + 20)) * 0.01953125V
	else(force startup)
		if(yp1==4.6%)
			yp1_pin ← 0v;
		else
			yp1_pin ← (reg_data_04[6:0] + 20) * 0.01953125V
if(mode==7)
	if(direcr_PWM)
		if(yp1==4.6%)
			yp1_pin ← 5v;
		else
			yp1_pin ← (256 - (reg_data_04[6:0] + 20)) * 0.01953125V
	else(DC_mode)
		if(yp1==4.6%)
			yp1_pin ← 0v;
		else
			yp1_pin ← (reg_data_04[6:0] + 20) * 0.01953125V
else if(mode==4 or mode==5 or mode==6)
			reg_data_07 ← (min_rpm/rpm_ratio - 59)/4;
			yp1_pin ← (reg_data_07 + 20) * 0.01953125V

//SS PIN
if(mode==0 or mode==2 or mode==7)
	if(pole==4p)
		case(ss)
		0.5s	:	ss_pin ← 5v;
		1s	:	ss_pin ← 4.257v;
		2s	:	ss_pin ← 4.023v;
		4s	:	ss_pin ← 3.789v;
		6s	:	ss_pin ← 3.554v;
		8s	:	ss_pin ← 3.320v;
		10s	:	ss_pin ← 3.085v;
		12s	:	ss_pin ← 2.851v;
	else(pole==8p)
		case(ss)
		0.5s	:	ss_pin ← 0v;
		1s	:	ss_pin ← 0.722v;
		2s	:	ss_pin ← 0.957v;
		4s	:	ss_pin ← 1.191v;
		6s	:	ss_pin ← 1.425v;
		8s	:	ss_pin ← 1.660v;
		10s	:	ss_pin ← 1.894v;
		12s	:	ss_pin ← 2.128v;
else if(mode==1)
	if(pole==4p)
		case(ss)
		0.5s	:	ss_pin ← 5v;
		1s	:	ss_pin ← 4.257v;
		2s	:	ss_pin ← 4.023v;
		4s	:	ss_pin ← 3.789v;
		6s	:	ss_pin ← 3.554v;
		8s	:	ss_pin ← 3.320v;
		10s	:	ss_pin ← 3.085v;
		12s	:	ss_pin ← 2.851v;
	else(pole==rd)
		case(ss)
		0.5s	:	ss_pin ← 0v;
		1s	:	ss_pin ← 0.722v;
		2s	:	ss_pin ← 0.957v;
		4s	:	ss_pin ← 1.191v;
		6s	:	ss_pin ← 1.425v;
		8s	:	ss_pin ← 1.660v;
		10s	:	ss_pin ← 1.894v;
		12s	:	ss_pin ← 2.128v;
else if(mode==3)
	if(pole==4p)
		case(ss)
		0.5s	:	ss_pin ← 5v;
		1s	:	ss_pin ← 4.257v;
		2s	:	ss_pin ← 4.023v;
		4s	:	ss_pin ← 3.789v;
		6s	:	ss_pin ← 3.554v;
		8s	:	ss_pin ← 3.320v;
		10s	:	ss_pin ← 3.085v;
		12s	:	ss_pin ← 2.851v;
	else(pole==6p)
		case(ss)
		0.5s	:	ss_pin ← 0v;
		1s	:	ss_pin ← 0.722v;
		2s	:	ss_pin ← 0.957v;
		4s	:	ss_pin ← 1.191v;
		6s	:	ss_pin ← 1.425v;
		8s	:	ss_pin ← 1.660v;
		10s	:	ss_pin ← 1.894v;
		12s	:	ss_pin ← 2.128v;
else if(mode==4 or mode==5 or mode==6)
		case(ss)
		//min_rpm < RPM < max_rpm
		149.86  < RPM < 2740.66 	:	ss_pin ← 0v;	//rpm_ratio=2.54
		219.48  < RPM < 4013.88 	:	ss_pin ← 0.92v;	//rpm_ratio=3.72
		329.81  < RPM < 6031.61 	:	ss_pin ← 1.55v;	//rpm_ratio=5.59
		489.7   < RPM < 8955.7  	:	ss_pin ← 2.18v;	//rpm_ratio=8.3
		719.8   < RPM < 13163.8 	:	ss_pin ← 2.80v;	//rpm_ratio=12.2
		1049.61 < RPM < 19195.41	:	ss_pin ← 3.43v;	//rpm_ratio=17.79
		1529.87 < RPM < 27978.47	:	ss_pin ← 4.06v;	//rpm_ratio=25.93
		2239.64 < RPM < 40958.84	:	ss_pin ← 5v;	//rpm_ratio=37.96
```

---

## 12. §11 設計原稿 vs 現行實作差異總覽

§11 是 Allen 交付的最終 pseudo-code，作為 §1 ~ §10 refactor 的源頭依據。實作演進後 §1 ~ §10 已對齊現行實作。下表整理兩者差異，**閱讀 §11 時請對照本表**。完整逐段對照見 [`M8160_Forward_Mode_Memo.md`](M8160_Forward_Mode_Memo.md) §C。

| # | 主題 | §11 設計原稿 | 現行實作 | 對應 commit |
| - | ---- | -------------- | -------------- | ---- |
| 1 | Close-loop XP1 公式 | `reg_06[6:0] = min_rpm/rpm_ratio − 59` | `rp1 = round(min_rpm / rpm_ratio − 59)`（**含 `−59`**，2026-05-08 回退；歷史 commit `33fcc857` 曾移除） | 2026-05-08 |
| 2 | Close-loop YP 公式 | `reg_07 = (min_rpm/rpm_ratio − 59) / 4`，pin V = `(reg_07 + 20) × 5/256` | `rp2 = round(max_rpm / rpm_ratio / 4 − 59)`（**改用 max_rpm**、含 `−59`），pin V = `rp2 × 5/256`（Allen direct，無 `+20`） | 2026-05-08 |
| 3 | Close-loop YP 常數 | 未列 | `yp1 = 12 (≈4.6%)` (`reg_04[6:0]`)、`yp2 = 127 (≈100%)` (`reg_05[6:0]`) hardcoded（mode 4/5/6，2026-05-08 yp1 從 26 改 12） | 2026-05-08 |
| 4 | XP1 / YP1 開路語意 | 文字寫 `xp1==4.6%` / `yp1==4.6%`，只描述 pin V | 容差 `abs(duty − 4.64) < 0.005`；同時寫 `xp1 = 3` / `yp1 = 12`（open-loop 對齊 Reverse fix-H/fix-L） | `a3da9541` |
| 5 | Close-loop XP1 開路 | `xp1==60*rpm_ratio` 視為開路 → 5V/0V | **不採用**。close-loop 只 clamp `rp1 ∈ [0,127]`，無 4.64% 開路概念 | – |
| 6 | Close-loop SS 範圍檢查 | `min_rpm < RPM < max_rpm` 單一條件 | RP1/RP2 雙 bound：`MinRpm ∈ (59·ratio, 167·ratio)`（含 ADC −20 shift）且 `MaxRpm ∈ (59·ratio, 1079·ratio)`；越界 → `OutOfRange`（2026-05-11 spec：RP1 Max 從 187·ratio 改為 167·ratio；2026-05-08 雙 bound 引入） | 2026-05-11 |
| 7 | `fr_pin_sw = 0` 觸發 | 開頭無條件寫（隱含預設 non-FR） | 條件式 `if HallType == NonFR` 才寫 | – |
| 8 | mode = 7 分類條件 | `(pole==4p\|8p) & shutdown & (PWM_non_invert or DC_mode)` 單條 | 拆兩條：**O2a** Direct PWM + Non-invert、**O2b** DC mode + (non-FR+VSP \|\| FR+SET) | – |
| 9 | Mode-derived 6 常數 | 完全未列 | `fan_curve` / `rpm_ratio` / `icmd_sw` / `auto_soft_sw` / `duty_deb_sw` / `low_freq_rev` 六個 row | `225a6893` |
| 10 | XP1 / YP1 duty 精度 | 4.6%（兩位小數） | **4.64%**（dropdown 第 12 格 `38.28 × 12 / 99 = 4.6398` round 到 0.01% 精度），容差 `0.005` | §1.1 |
| 11 | rpm_ratio dropdown 顯示 | 未提 | 顯示 ratio 數值（例如 `2.54`、`3.72`），不是 index | `0b26bb2d` / `9501c91b` |

### 12.1 為何現行實作與設計原稿不同

1. **`−59` offset 保留**（2026-05-08）：原稿假設「最低 RPM 對應 ADC code 59 → field 0」。歷史 commit `33fcc857`（2026-05-07）曾移除以對齊 Reverse round-trip，但 2026-05-08 使用者新 spec 要求回退至 §11 設計原稿。Forward 與 Reverse `rp1 = adc1[6:0] − 20`、`rp2 = adc2` 不再 round-trip 對齊，Compare 視窗會出現固定 ±59 register field 差異（spec 容許）。詳見 [`M8160_Forward_Mode_Memo.md`](M8160_Forward_Mode_Memo.md) §C.4。
2. **Close-loop YP 改用 `MaxRpm`**：原稿用 `min_rpm` 是筆誤；YP 對應上限 RPM，邏輯上必須用 max_rpm。沿用自 commit `33fcc857`。
3. **Allen direct YP 公式**：close-loop YP pin V 不加 `+20` offset，對應 Reverse `yp_pin[7:0] = reg_07[7:0]`（直接相等）。
4. **`yp1=12 (≈4.6%)` / `yp2=127 (≈100%)` 常數**：使用者新 spec「YP1 固定 4.6%、YP2 固定 100%」。`0.390625 × 12 = 4.6875% ≈ 4.6%`；2026-05-08 yp1 從 26 改 12，舊值 26 = 10.16% 與 4.6% 口語不符。
5. **Mode-derived 6 常數**：原設計工具只算 4 支腳位電壓 + 5 個 forced bit，但實際燒錄需要完整 register set；commit `225a6893` 補上。
6. **`xp1==60*rpm_ratio` 開路語意移除**：close-loop 沒有「腳位開路」這個 design-time 概念（4.64% 開路只屬於 open-loop XP1/YP1）。
7. **mode = 7 拆 O2a / O2b**：原稿 `(PWM_non_invert or DC_mode)` 邏輯不嚴謹（DC mode 還要看 hall + DcSource 子分支）；Allen reverse calculator 已拆分，Forward 對齊。
