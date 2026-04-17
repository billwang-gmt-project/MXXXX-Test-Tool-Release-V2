# M8314AA V2 驗證檢查表

M8314AA 在 GmtTestToolV2 的重點驗證項目與觀察事項，給新進工程師做回歸測試用。

搭配範例 snapshot：`docs/chips/GuiTestSetting/M8314_*.json`

---

## 1. Dropdown 動態變化

| 下拉 | 什麼情況會動態改變 | UserMode 是否隱藏 |
| --- | --- | --- |
| 驅動頻率（Drive Freq 四欄，REG00） | 勾選 / 取消「超高頻驅動」時，四個下拉選項整批切換：一般模式=`32k 三角` / `64k 锯齿` / `32k 锯齿`；超高頻=`128k 三角` / `256k 锯齿` / `128k 锯齿`。兩組都 3 項且 index 對齊（0/2/3），當前選擇位置會保留 | 顯示 |
| SO Type（REG03） | 勾選 / 取消「超高頻驅動」時選項改變：一般模式 5 項 `4 / 6 / 8 / 10 / 12`（index 0–4）；超高頻 3 項 `2 / 4 / 8`（index 5–7）。兩組 index 不重疊，程式以**顯示值**（例如 `8`）在新列表回查，找到就改 index 保留選擇，找不到（如一般模式的 `10`/`12`、超高頻 `6`）則無法對應 | 顯示 |
| 啟動轉速（REG02 Field0203） | 隨「Cycle Count」下拉重新計算全部 8 個選項的 RPM 值：`RPM[i] = cycle × 6 × multiplier[i]`，cycle 取 `[8, 12, 16, 20, 24, 28, 32, 36]`，multiplier 固定 `[12, 16, 20, 24, 28, 32, 36, 40]`。選擇的 index 不變，但對應的 RPM 數字隨 Cycle Count 重算 | 顯示 |
| 開窗相位 | 切換操作模式：UserMode 只有 10° / 15° 兩項，BackDoor / Engineering 顯示四項；從 BackDoor 切回 UserMode 時，若原本選第 3/4 項會自動歸零 | 只顯示 10° / 15° |
| LA_sel 相角 | 切換操作模式：UserMode 只有 9.5° / 16° 兩項，BackDoor / Engineering 顯示四項；從 BackDoor 切回 UserMode 時，若原本選第 3/4 項會自動歸零 | 只顯示 9.5° / 16° |
| XP1 Shutdown Mode (%) | 隨 XP1 目前值即時重算四個選項（XP1、XP1 − 3.125、XP1 − 6.25、XP1 − 9.375，最小 0） | 顯示 |
| FG Ratio（REG09[5:0]） | 三個下拉共用同一組選項（由 SO Type 依 FG Ratio 公式動態產生、排序後的 RPM 值）。FG Ratio 欄位在 Online 模式由硬體策略即時 RPM 覆寫；Offline / PGM 依當前速度曲線模式取用 OOO→Open Loop 選項，其餘（COO / CCO / CCC）→Close Loop 選項（詳見第 4 節） | **隱藏**（XAML 以 `IsBackDoorOrEngineeringMode` 控制可見性，UserMode 整個 ListBoxItem 容器不顯示；內部屬性仍存在，snapshot 照常寫入 / 還原） |
| Open Loop RPM | 選項內容與 FG Ratio 共享；在 OOO 模式被 FG Ratio 取用，其他速度曲線模式下切換 COO / CCC 時會被 `AutoAssignRpmSelections` 強制跟隨 Close Loop 的 index | **隱藏**（同上條件） |
| Close Loop RPM | 選項內容與 FG Ratio 共享；COO / CCO / CCC 模式 FG Ratio 取自此下拉；index 變更時會推送到 `_mxxxxConfigService.CurrentConfig.CloseLoopRpm` 供 Online RPM 公式使用 | **隱藏**（同上條件，但 CCC 控制項本身在 BackDoor / Engineering 才顯示） |

---

## 2. Speed Curve XP / YP 範圍

| 欄位 | 模式 | 範圍 | 步距 |
| --- | --- | --- | --- |
| XP1 | COO / CCO / OOO | 0% – 48.5%（共 32 檔） | 1.564% |
| XP1 | CCC | 7.82% – 48.5%（共 27 檔，去掉最低 5 檔） | 1.564% |
| XP2 | COO / CCO / OOO | 53.08% – 100%（共 16 檔） | 3.128% |
| XP2 | CCC | 53.08% – 96.87%（共 15 檔，去掉最高 1 檔） | 3.128% |
| YP1 | 全部 | 0% – 48.5%（共 32 檔） | 1.564% |
| YP2 | 全部 | 51.516% – 100%（共 32 檔） | 1.564% |
| RP1（最低轉速設置） | CCC | 0% – 49.266%（共 64 檔） | 0.782% |
| RP2（高轉保護點設置） | CCC | 50.734% – 99.218%（共 63 檔） | 0.782% |

觀察：

- OOO 模式的 XP1 / XP2 / XP3 順序若輸入顛倒，畫面會自動拉回遞增順序
- OOO 模式的 YP1 / YP2 / YP3 也會自動鉗制為遞增
- CCC 模式的 RP1 / RP2 / RP3 會自動鉗制
- 四個模式各自有獨立的警告訊息（順序異常或超範圍時顯示）

---

## 3. Snapshot Load / Save

| 檢查項目 | 儲存時 | 載入時 | 觀察 |
| --- | --- | --- | --- |
| 晶片名稱（ChipTag） | 寫入檔案 | 必須與當前晶片相符；UserMode 不符即拒絕載入，其他模式會詢問是否跨系列 | 會使用晶片別名比對 |
| FILE CRC、REG CRC、檔名 CRC | — | 三者必須一致 | 計算時暫存器不足 16 bytes 會補 0x00 |
| 超高頻驅動勾選狀態 | 寫入 | 還原後會觸發驅動頻率與 SO Type 下拉同步重填 | — |
| Open Loop / Close Loop RPM 選擇 | 以 `RpmSettingOpenLoopSelectedIndex` / `RpmSettingCloseLoopSelectedIndex` 兩個延伸屬性寫入 snapshot（不是存暫存器值，而是存當下選的下拉 index） | 讀回延伸屬性並寫回兩個 SelectedIndex，觸發 `OnOpenLoopRpmSelectionChanged` / `OnCloseLoopRpmSelectionChanged` 重算 RPM 並同步到 FG Ratio 欄位 | UserMode UI 完全隱藏但資料仍保留；Open/Close Loop 選單內容由 SO Type 與 FG Ratio 公式動態產生（RPM 值經排序），因此還原前必須先正確還原 SO Type，否則找不到對應 index |

---

## 4. FG Ratio 與 Speed Profile

FG Ratio 欄位在不同模式下的來源：

| Speed Profile | Offline / PGM 模式的 FG Ratio 來源 | Online 模式的 FG Ratio |
| --- | --- | --- |
| COO | 取自 Close Loop RPM 選項 | 由硬體策略即時 RPM 計算覆寫，同步流程不動作 |
| CCO | 取自 Close Loop RPM 選項 | 同上 |
| OOO | 取自 Open Loop RPM 選項 | 同上 |
| CCC | 取自 Close Loop RPM 選項 | 同上 |

儲存 snapshot 的流程：

- 先依當前速度曲線模式同步 FG Ratio 欄位值
- 確保存檔裡的 FG Ratio 與速度曲線模式一致，CRC 不會因為事後同步而對不上

觀察：

- Online 模式 FG Ratio 由硬體 RPM 即時接管，不會由儲存流程改值
- 載入 snapshot 後會重建所有動態下拉（超高頻驅動、速度曲線欄位、啟動轉速等）
- FG Ratio 同步的觸發時機：RPM 選項變更、速度曲線模式切換、載入 snapshot 之後
