# M8160 Design Tool — Old vs Allen Pseudocode Difference Table

本文件比較舊版計算邏輯（基於 pptx）與新版權威規格（`M8160_design_tool_pseudocode_allen.md`）之間的差異。
Codebase 和所有文件已於 2026-04-16 全面對齊 Allen 版。

---

## 1. Forbidden Bands（XP1 / YP / SS pin，不影響 Mode pin）

| 區間 | Old | Allen | 差異 |
|------|-----|-------|------|
| 中央 forbidden | step 120..136 | step 120..**135** | 上界少 1 step（136 變 linear） |
| 高段 forbidden | step **235**..245 | step **236**..245 | 下界多 1 step（235 變 linear） |
| 低段 forbidden | step 10..19 | step 10..19 | 無變化 |

連帶影響：Linear 高半段從 `137..234` 擴展為 `136..235`。

> Mode pin 的 forbidden gap（step 120..136）不變。

---

## 2. Register Field 公式差異

| # | Field | 項目 | Old | Allen |
|---|-------|------|-----|-------|
| 5 | `xp1` | LINEAR 分支 | `round((ADC1[6:0] - 20) / 4)` | `xp1V > 2.5`: `round((~ADC1[6:0] - 20) / 4)` ; else: `round((ADC1[6:0] - 20) / 4)` |
| 7 | `yp1` | LINEAR 分支 | `ADC2[6:0] - 20` | `ypV > 2.5`: `~ADC2[6:0] - 20` ; else: `ADC2[6:0] - 20` |
| 8 | `yp2` | 常數值 | `mode∈{4,5,6}` → 63 ; else → 127 | **all modes → 127** |
| 10 | `rp1` | non-fix 分支 | `ADC1[6:0] - 20` | `xp1V > 2.5`: `~ADC1[6:0] - 20` ; else: `ADC1[6:0] - 20` |
| 12 | `ss_time` | LINEAR 分支 | `round((ADC3[6:0] - 20) / 8)` | `ssV > 2.5`: `floor((~ADC3[6:0] - 20) / 8)` ; else: `floor((ADC3[6:0] - 20) / 8)` |

### ADC Inversion 邏輯（新增）

Allen 在 `xp1`、`yp1`、`rp1`、`ss_time` 的 LINEAR 分支新增了 **ADC code 反轉**：

```
adc_low     = adc & 0x7F          // ADC[6:0]（低 7 bit）
adc_low_inv = ~adc & 0x7F         // ~ADC[6:0]（反轉低 7 bit）

if pin_voltage > 2.5V:  use adc_low_inv
else:                   use adc_low
```

Old 版不區分，一律用 `adc_low`。

---

## 3. Register Mapping 差異

| # | Field | Old Register | Allen Register |
|---|-------|-------------|----------------|
| 12 | `ss_time` | `reg_08[7:5]`（3 bits） | `reg_08[6:5]`（**2 bits**） |

---

## 4. Rounding 差異

| Field | Old | Allen |
|-------|-----|-------|
| `xp1` | `round(AwayFromZero)` | `round(AwayFromZero)` — 無變化 |
| `ss_time` | `round(AwayFromZero)` | **`floor()`**（整數除法截斷） |

---

## 5. 無變化的 Fields

以下 field 在 Old 與 Allen 之間完全一致，沒有任何差異：

`cmd_inv`、`fan_curve`、`rpm_ratio`、`min_sd`、`soft_sw`、`icmd_sw`、`rp2`、`so_set`、`auto_soft_sw`、`duty_deb_sw`、`source_duty`、`align_sw`、`low_freq_rev`、`fr_pin_sw`
