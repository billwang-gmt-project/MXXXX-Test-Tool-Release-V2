# M8123 Design Tool — Pseudocode vs JSON / Pptx 對齊報告

**日期：** 2026-04-14（§1–§2）；2026-04-14 同日新增 §3；2026-04-15 新增 Bug 9（calculator vs pseudocode review）
**Pseudocode 來源：** [M8123_design_tool_pseudocode.md](M8123_design_tool_pseudocode.md)
**JSON 權威：** [chips/M8123AA.json](../../../../chips/M8123AA.json)
**Pptx 權威：** [`20260414_M8123 Design Tool.pptx`](20260414_M8123%20Design%20Tool.pptx)
**處理原則：** Register layout 以 JSON 為準；公式數值以 pptx 為準。所有修正直接寫回 pseudocode，並在這份報告完整存證。

這份報告是 M8123 design tool 在實作前對齊 pseudocode 與權威 chip 定義（JSON）和原始投影片（pptx）的稽核紀錄。共三個 section：

1. **Pseudocode ↔ JSON register-layout 不一致（§A／§B／§C）** — 三處 pseudocode 與 [M8123AA.json](../../../../chips/M8123AA.json) 在 register 位置、欄位名稱或 bit 寬度上不一致。
2. **Pseudocode 內部 bug（Bug 1／2／3）** — 三處 pseudocode 自相矛盾或 copy/paste 錯誤，與 JSON 無關。
3. **Pptx 公式校正（Bug 4／5／6／7／8／9）** — 後續解析 `20260414_M8123 Design Tool.pptx` 並逐 slide 比對 calculator 後新增。又找出五個 pseudocode bug；2026-04-15 review calculator vs pseudocode 又補一個 ClassifyRail forbidden-band 的 off-by-one（Bug 9）。

每筆修正都已寫回 `M8123_design_tool_pseudocode.md`，並在受影響的行尾加上 `// fixed 2026-04-14` 註解，方便未來讀者循線回到本報告。

---

## Section 1 — Pseudocode ↔ JSON register-layout 不一致

### §A — Field 6 `soft_sw` → `ssw`

| | Pseudocode（原本） | JSON 權威 |
|---|---|---|
| 欄位名稱 | `soft_sw` | `ssw` |
| Register | `reg_03` | `REG03`（addressOffset 3）✓ |
| Bit range | `[7:6]`（2 bits） | `bitOffset: 6, bitWidth: 2` → `[7:6]` ✓ |
| 語意 | Pseudocode 視為布林：`1 if mode == 1 else 0` | JSON 定義 4-value enum：`square / 22.5 / 45 / sine` |

**Pseudocode 原文（line 80–81）：**
```
// ── Field 6: soft_sw[1:0]  (reg_03[7:6], 2 bits) ──
soft_sw ← 1  if mode == 1  else 0
```

**修正內容：**

- 把 pseudocode 與 calculator 中所有的 `soft_sw` 改名為 `ssw`。
- 公式 `mode == 1 → 1, else → 0` 保持不變 — pseudocode 的意圖是「mode 1 啟用 soft-start」，對應到 `ssw = 1`（enum 值 `22.5`）。chip 設計者命名 vs bit-field enum option 不一致是 chip 既有的奇怪事實，不在本工具修復範圍內；calculator 只忠實輸出公式結果。
- DataGrid 顯示的欄位名稱會跟 `M8123AA.json` 一致（`ssw`），方便 user 在 Register tab 對照。

**理由：** 純改名，不動公式。`ssw=1` 是不是真的對應到 mode 1 的所有情境，是 chip 設計者的責任，這份工具不負責判斷。

---

### §B — Field 12 `ss_time` bit 佈局

| | Pseudocode（原本） | JSON 權威 |
|---|---|---|
| 欄位名稱 | `ss_time` | `ss_time` ✓ |
| Register | `reg_08` | `REG08`（addressOffset 8）✓ |
| Bit range | `[7:5]`（3 bits，值 0–7） | `bitOffset: 5, bitWidth: 2` → `[6:5]`（2 bits，值 0–3） |
| Enum | （隱含 3-bit） | `0.5 s / 4 s / 8 s / 12 s`（4 個 option） |
| 衝突 | 主張 bit 7 屬於 `ss_time` | REG08 bit 7 是 `duty_deb_sw`（見 §C） |

**Pseudocode 原文（line 124）：**
```
// ── Field 12: ss_time[2:0]  (reg_08[7:5], 3 bits) ──
```

**Pseudocode 原文（line 134）：**
```
else:                                                    ss_time ← 30   // mode ∈ {4,5,6}
```

**Pseudocode 原文（line 131–133，linear 分支）：**
```
if ssV > 2.5:     ss_time ← round((adc3_low_inv-14)/2)
else:             ss_time ← round((adc3_low-14)/2)
```

**修正內容：**

- Pseudocode bit range 改為 `[6:5]`（2 bits，width 2）。
- **公式刻意不重新縮放。** mode∈{4,5,6} 的常數 `30` 和 linear 分支 `round((adc3_low±14)/2)` 都可能超過 3，本來就會超出 2-bit 欄位的合法範圍。
- Calculator 的 `M8123DesignFieldRow.IsOverflow = (value < 0 || value >= (1<<width))` 檢查（與 `M8160DesignFieldRow` 相同）會把這些 row 在 DataGrid 紅色標出，行為跟 M8160 Design Tool 標 `yp1=26`（mode 4/5/6）完全一致。「公式想寫 X，但 register 只裝得下 Y」是設計者該知道的訊息，silently clamp 反而會掩蓋問題。

**理由：** Bit 佈局以 JSON 為準（因為 register 硬體真的就是 2-bit）。公式維持原樣，讓 calculator 在 runtime 透過 overflow 標示揭露這個落差。如果 chip 設計者後續決定要 clamp 到 3 或除以 8 縮放，可以改 pseudocode 後重新移植 — 但這份報告不該替設計者偷做這個決定。

---

### §C — Field 15 `duty_deb_sw` register

| | Pseudocode（原本） | JSON 權威 |
|---|---|---|
| 欄位名稱 | `duty_deb_sw` | `duty_deb_sw` ✓ |
| Register | `reg_0d` | `REG08`（addressOffset 8） |
| Bit | `[7]` | `bitOffset: 7, bitWidth: 1` → `[7]` ✓ |

> 備註：M8123 general peripheral 沒有 REG0C 與 REG0D（暫存器只到 `0x00–0x0B`，下一段是 Backdoor 從 `0x7E` 開始）。

**Pseudocode 原文（line 148–149）：**
```
// ── Field 15: duty_deb_sw  (reg_0d[7], 1 bit) ──
duty_deb_sw ← 1  if mode in {0,1}  else 0
```

**修正內容：**

- Pseudocode register 參考 `reg_0d[7]` → `reg_08[7]`。
- 公式（`1 if mode in {0,1} else 0`）維持不變。

**理由：** 純粹 register 編號錯誤。Pseudocode 作者很可能從新版 datasheet 抄了 `[7]` 這個 bit 位置，但忘了更新 register number。REG08 bit 7 本來就空著（`ss_time` 只佔 `[6:5]`，見 §B），最終佈局完全自洽。

---

## Section 2 — Pseudocode 內部 bug（與 JSON 無關）

### Bug 1 — Field 21 `ki` 所有分支寫到 `kp`

**Pseudocode 原文（line 179–186）：**
```
// ── Field 21: ki  (reg_0b[2:0], 3 bit) ──
if mode in {4,5,6}:
    if rpm_ratio <= 1:      kp ← 5
    else if rpm_ratio <= 3: kp ← 4
    else if rpm_ratio <= 4: kp ← 3
    else if rpm_ratio <= 6: kp ← 2
    else if rpm_ratio <= 7: kp ← 1
else:                       kp ← 5
```

**Bug：** 每個 assignment 都寫到 `kp`（也就是 Field 20），`ki` 從來沒有被指派過。這是經典 copy/paste bug — 整個 block 從 Field 20 複製過來但變數名忘了改。

**修正內容：** 把全部 6 個 assignment 從 `kp ←` 改成 `ki ←`。RHS 數值（5、4、3、2、1、5）和分支條件保持不變。

**理由：** Field 20（`kp`）和 Field 21（`ki`）是 PI 控制器的 P 和 I 係數，pseudocode 作者顯然是要分別計算的。兩個 block 結構完全對稱（都是 `mode ∈ {4,5,6}?` 加 `rpm_ratio` ladder），copy/paste 出處非常明顯。修正後 `ki` 才能拿到 chip 設計者真正想要的數值。

---

### Bug 2 — Field 11 `rp2` linear 分支沒有 `else`

**Pseudocode 原文（line 114–122）：**
```
// ── Field 11: rp2[7:0]  (reg_07[6:0], 7 bits) ──
if mode in {4,5,6}:
    if ypFix:               rp2 ← 251
    else:
        adc2_low ← adc2 and 0x7F
        adc2_low_inv ← ~adc2 and 0x7F
        if ypV > 2.5:                             rp2 ← adc2_low+20+128
        else if adc2>=20 and adc2=<107:           rp2 ← adc2_low+20+128
else:                                              rp2 ← 0
```

**兩個 bug：**

1. `=<` 不是合法的 pseudocode 比較運算子（應該是 `<=`）。
2. `else if adc2>=20 and adc2<=107` 分支沒有 fallback `else`，當 `ypV ≤ 2.5` 而且 `adc2` 落在 `[20, 107]` 之外時，`rp2` 是未初始化的。在 C#/MVVM 環境下這代表上一輪的數值會洩漏進來，是一個沉默的正確性 bug。

**修正內容：**
```
// ── Field 11: rp2[6:0]  (reg_07[6:0], 7 bits) ──      // fixed 2026-04-14: clarified branches
if mode in {4,5,6}:
    if ypFix:               rp2 ← 251
    else:
        adc2_low ← adc2 and 0x7F
        if ypV > 2.5:                             rp2 ← adc2_low+20+128
        else if adc2 >= 20 and adc2 <= 107:        rp2 ← adc2_low+20+128
        else:                                       rp2 ← 0
else:                                                rp2 ← 0
```

另外：原註解寫 `rp2[7:0]`（8 bits），但 JSON 明確說是 `bitWidth: 7`（`[6:0]`），註解一併改為 `[6:0]`。

**理由：** 兩個正向分支算的是同一條公式（`adc2_low+20+128`），所以「真正的 linear 範圍」是 `ypV > 2.5 OR 20≤adc2≤107`。落在外面的（例如 below-linear short：低 ADC 又低電壓）應該歸 `rp2 = 0`，跟其他欄位處理 below-linear 的方式一致（見 Field 5、7、10）。明確寫 `else → 0` 同時統一行為並避免未初始化變數的沉默 bug。

**`adc2_low_inv` 備註：** 原 pseudocode 算了 `adc2_low_inv` 但在 `rp2` block 裡完全沒用到，修正版直接拿掉。

---

### Bug 3 — Line 66 縮排混用 tab 和 space

**Pseudocode 原文（line 64–66）：**
```
else:                                                    // mode ∈ {2,3,5,6}
    if xp1V > 2.5: min_sd ← 0
    else:           min_sd ← 1
```
（line 65–66 開頭的縮排在原始檔案中混用 tab 和 space）

**修正內容：** 全部正規化為 space。語意完全不變。

**理由：** 純美觀問題 — 原本的 tab 在某些 Markdown viewer 裡 render 不可預期。Pseudocode 是參考文件，視覺一致比保留原始空白更重要。

---

## Section 3 — Pptx cross-reference（2026-04-14 新增）

完成最初的 §1/§2 對齊後，把 [`20260414_M8123 Design Tool.pptx`](20260414_M8123%20Design%20Tool.pptx) 解析出文字並逐 field 與 calculator 比對。又找出 5 個公式 bug — 全部都在 pseudocode 端，跟 JSON 無關 — 並同步修進 `M8123_design_tool_pseudocode.md` 和 `Helpers/M8123AdcCalculator.cs`。

### Bug 4 — Field 5 `xp1` 漏掉 `/4` 除法

**Pptx slide 10 worked example：**
```
Ex : XP1 = 20%
  Register: reg_data_02[4:0] x 1.5625% = 13 x 1.5625% = 20.3125%
  ADC: 20% ⇒ 1.39V / 19.53125mV = 71.168 step
  xp1 (0.39~4.59) : ( ADC1[6:0] – 20 ) / 4 = ( 71.168 – 20 ) / 4 = 12.79
```

**Pseudocode 原本公式（line 76–77）：**
```
if xp1V > 2.5:    xp1 ← adc1_low_inv - 20
else:             xp1 ← adc1_low - 20
```

**修正內容：** 把分子用 `/ 4` 包起來：
```
if xp1V > 2.5:    xp1 ← round((adc1_low_inv - 20) / 4)
else:             xp1 ← round((adc1_low - 20) / 4)
```

**理由：** ADC 0.39 V – 4.59 V 的範圍跨了大約 215 個 step，但 5-bit 欄位只有 32 個 step — `/4` 正好把 1.5625%/step 的解析度對應到 4 個 ADC step 的粒度。沒有這個除法的話，每個輸入電壓都會 overflow 5-bit 欄位。

### Bug 5 — Field 8 `yp2` 把 physical 值 255 直接寫進 7-bit register

**Pptx slide 5：**
```
yp2[7:0] = 128 + reg_data_05[6:0]
  All mode : 255
```

**Pseudocode 原本：** `yp2 ← 255`
**修正內容：** `yp2 ← 127`（因為 `255 = 128 + 127`）

**理由：** `yp2[7:0]` 是 motor driver 真正消費的 8-bit *physical* 值，是把一個 hard-coded 的 `1` bit 接在 7-bit register 前面組合出來的，所以 register 永遠只裝下面 7 bit。physical 常數 255 對應到 register 值 127（7-bit 最大值），完美塞得下不會 overflow。直接寫 255 會 overflow 7-bit 欄位 — pseudocode 作者把 `128 +` 這個 offset label 誤讀成可以一起寫進 register 的 literal。

### Bug 6 — Field 10 `rp1` fix value 7 → 1

**Pptx slide 6：**
```
rp1[6:0] reg_data_06[6:0]
  ( mode[4] | mode[5] | mode[6] ) ( xp1_fix_h | xp1_fix_l ) : 1
```

**Pseudocode 原本：** `rp1 ← 7`（fix 和 mode∉{4,5,6} 兩條分支都是 7）
**修正內容：** fix value → 1（pptx 確認）；mode∉{4,5,6} 維持 0（reset value，pptx 沒指定）。

**理由：** Pseudocode 的編輯失誤。Pptx 寫得很清楚。

### Bug 7 — Field 11 `rp2` physical 與 register 混淆

**Pptx slide 6：**
```
rp2[7:0] = 128 + reg_data_07[6:0]
  ( mode[4] | mode[5] | mode[6] ) :
    ( yp_fix_h | yp_fix_l ) : 251
    0.39 < yp < 2.49 : ADC2[6:0]+20
    2.51 < yp < 4.59 : ADC2[6:0]+20
```

**和 Bug 5 同類錯誤：** `128 +` 是 physical offset，不是 register 內容。原 pseudocode 寫的是：
- `rp2 ← 251`（應該是 123，因為 `251 = 128 + 123`）
- `rp2 ← adc2_low + 20 + 128`（應該是 `adc2_low + 20`；那個 `+128` 是 driver 硬體後段才加上去的）

**修正內容：**
- `ypFix → 123`
- linear 分支 → `adc2_low + 20`
- 明確 `else → 0`（同 Bug 2 的清理）

### Bug 8 — Field 12 `ss_time` fix 與 mode{4,5,6} 常數

**Pptx slide 6：**
```
ss_time[4:0] reg_data_08[7:5]   ← pptx header 自相矛盾（5-bit 欄位塞 3-bit slot）
  ( mode[0] | ... ) : ( ss_fix_h | ss_fix_l ) : 0
  ss < 0.39 : 0
  0.39 < ss < 2.49 : ( ADC3[6:0] - 14 ) / 2
  2.51 < ss < 4.59 : ( ~ADC3[6:0] - 14 ) / 2
  ss > 4.59 : 0
  ( mode[4] | mode[5] | mode[6] ) : 7
```

**Pseudocode 原本：** fix→3, {4,5,6}→30
**修正內容：** fix→0, {4,5,6}→7（依 pptx）

**理由：** 純粹的轉錄錯誤。Pptx 才是正確值。2-bit JSON 欄位仍然會被 linear 分支 `round((ADC-14)/2)` overflow — 這是刻意的，並透過 `IsOverflow` UI 標示揭露。

### Bug 9 — `ClassifyRail` forbidden ranges 上下界 off-by-one

**Pptx slide 9 ADC step 表：**

```
step 10..19   ≈ 0.195 V .. 0.371 V   forbidden
step 120..136 ≈ 2.344 V .. 2.676 V   forbidden
step 235..245 ≈ 4.590 V .. 4.785 V   forbidden
```

**Pseudocode 原本（line 12）：**

```
if code(v) in {10..19, 120..135, 236..245}:
    return FORBIDDEN
```

**修正內容：** 中段 `120..135` → `120..136`、上段 `236..245` → `235..245`。

**理由：** Pseudocode 兩個邊界 step 各被漏抓 — 原本的範圍會把 step 136 和 step 235 誤分類為 LINEAR，但這兩個 step 在 pptx slide 9 屬於 forbidden 區段。修正後與 pptx 完全對齊。

### Pptx 自身的問題故意不傳到 calculator

Pptx 本身有幾處不一致，**不會**複製到 calculator。這些被視為投影片本身的編寫缺陷，依 JSON 修正：

| Pptx 寫的 | JSON 權威 | 處理 |
|---|---|---|
| Field 6 「soft_sw」 | `ssw` 在 reg_03[7:6] | 用 JSON 名稱 `ssw`（同 §1 §A） |
| Field 12「ss_time[4:0]」（5-bit）在「reg_data_08[7:5]」（3-bit slot） | 2-bit 在 reg_08[6:5] | 用 JSON 2-bit（同 §1 §B） |
| Field 15「reg_data_0d[7]」for duty_deb_sw | reg_08[7] | 用 JSON reg_08（同 §1 §C） |
| Field 21 header「ki reg_data_0b[5:4]」（與 kp 同 slot） | ki 在 reg_0b[2:0] | 用 JSON `[2:0]` |

Pptx 和原本的 pseudocode 在 Fields 6/12/15 都犯了同樣的三個過時參考錯誤，這暗示 pseudocode 是從更早一版還在用錯誤 register 編號的投影片轉錄過來的。JSON 是最年輕也最準的事實來源。

---

## 完整修正一覽表（12 筆）

| # | 類型 | Field／位置 | 原本 | 修正後 | 來源 |
|---|---|---|---|---|---|
| **§A** | JSON layout | Field 6 `soft_sw`（line 80） | `soft_sw @ reg_03[7:6]` | `ssw @ reg_03[7:6]` | `M8123AA.json`（4-enum） |
| **§B** | JSON layout | Field 12 `ss_time`（line 124） | `ss_time[2:0] @ reg_08[7:5]`（3-bit） | `ss_time @ reg_08[6:5]`（2-bit；overflow 透過 `IsOverflow` 揭露） | `M8123AA.json` |
| **§C** | JSON layout | Field 15 `duty_deb_sw`（line 148） | `duty_deb_sw @ reg_0d[7]`（REG0C/0D 不存在） | `duty_deb_sw @ reg_08[7]` | `M8123AA.json` |
| **Bug 1** | Internal | Field 21 `ki`（line 181–186） | 5 個分支全部寫 `kp ← …`（從 Field 20 copy/paste） | `ki ← …` | self-evident copy/paste |
| **Bug 2** | Internal | Field 11 `rp2`（line 114–122） | `=<` typo + 缺 `else`（未初始化）+ 註解 bit 寬度錯 | `<=` + 明確 `else → 0` + `[6:0]` | self-evident |
| **Bug 3** | Internal | Field 4 `min_sd`（line 66） | tab 和 space 混用 | 統一為 space | 美觀 |
| **Bug 4** | Pptx | Field 5 `xp1`（line 76–77） | `adc1_low − 20` | `round((adc1_low − 20) / 4)` | pptx slide 10 worked example |
| **Bug 5** | Pptx | Field 8 `yp2`（line 98） | `yp2 ← 255`（overflow 7-bit） | `yp2 ← 127`（physical 255 = 128 + 127） | pptx slide 5（`yp2[7:0] = 128 + reg_data_05[6:0]`） |
| **Bug 6** | Pptx | Field 10 `rp1`（line 106） | fix → 7, mode∉{4,5,6} → 7 | fix → 1, mode∉{4,5,6} → 0 | pptx slide 6 |
| **Bug 7** | Pptx | Field 11 `rp2` linear branch | `adc2 + 20 + 128`，fix `251`（physical） | `adc2 + 20`，fix `123`（register） | pptx slide 6（`rp2[7:0] = 128 + reg_data_07[6:0]`） |
| **Bug 8** | Pptx | Field 12 `ss_time` 常數 | fix → 3, mode{4,5,6} → 30 | fix → 0, mode{4,5,6} → 7 | pptx slide 6 |
| **Bug 9** | Calc-vs-doc | `ClassifyRail`（line 12） | `{10..19, 120..135, 236..245}` | `{10..19, 120..136, 235..245}` | calculator + pptx slide 9 ADC step 表 |

---

## 本報告不涵蓋的範圍

- **公式本身的語意正確性** — 例如 `fan_curve = 2` 在 mode 0 是不是真的對、`rp2 = adc2 + 20`（register）是不是正確的 duty 等。這些是 chip 設計者的責任。本報告只負責 pseudocode 在語法和欄位映射層面對齊 JSON 與 pptx。
- **硬體驗證** — 沒人在 silicon 上實測過「輸入這組電壓 → 算出這組 register 值」是否真的能讓 M8123 跑起來。這超出 design tool 移植的範圍。
- **`rp1` 在 mode∉{4,5,6} 的值** — pptx 沒指定，目前用 register reset value（`0`）。chip 設計者請確認。
- **`xp1` / `yp1` / `ss_time` linear 分支的 rounding 模式** — 為了與 M8160 calculator 一致，選用 banker's rounding（`MidpointRounding.AwayFromZero`）。Slide 10 worked example 算出 `12.79`（從 `(71.168 – 20) / 4`），用 `AwayFromZero` 會四捨五入到 13，符合預期。如果 chip 設計者想改成 truncation 或 `Math.Floor` 也可以。

當 pseudocode、JSON 或 pptx 出新版時，請更新這份報告，不要靜默改寫歷史。
