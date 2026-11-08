# 03｜Analysis Methodology — 分析方法與 Tableau 實作規格

> 版本 v1.0｜工具：Tableau + CSV｜2026-11-09  
> 對應文件：[專案章程](./01_Project_Charter.md)、[資料品質與 EDA](./02_Data_Profiling_EDA.md)。

[TOC]

## 1. 分析方法與基本原則

分析流程：**Business Question → Data Definition → Data Profiling → Descriptive Statistics → Visualization → Evidence → Recommendation → Validation**。

- 主分析期間：**2025-01-01～2025-12-31**，依 `Order Time`。
- 主銷售母體：`Order Status = 'delivered'`。
- 保留全量 40,500 筆，以狀態計算欄位保護不同 KPI 的母體。
- 不將相關性視為因果，不將訂單數／客戶數錯稱網站轉換率。
- 幣別未知；報表數字以「資料金額單位」呈現。
- 來源可能為模擬資料；結論以該份 CSV 為限。

## 2. 數據模型與前處理

資料粒度：1 列 = 1 個唯一 `Order Id`；每筆有一個商品名稱與數量。建議先建立標準日期欄位 `Order Date`，處理 `Order Time` 中的「上午／下午」，避免 Tableau 地區設定解析失敗。

建議匯入前保留原始 CSV 不變；由 Python／Tableau Prep 建立清理後 CSV，並記錄轉換步驟。`Created At` 不作為銷售趨勢日期。

設定固定 2025 範圍，不直接以資料最大日 2026-01-02 計算全年比較。

## 3. KPI Dictionary

| KPI | 定義 | 公式概念 | 限制 |
|---|---|---|---|
| Order Count | 所有訂單筆數 | COUNTD(Order Id) | 需固定日期範圍 |
| Gross Order Value | 所有訂單金額 | SUM(Total Amount) | 包含取消 |
| Non-cancelled Order Value | 未取消的訂單金額 | SUM(IF status != cancelled THEN amount) | 非結算營收 |
| Delivered Order Value | 已送達訂單金額 | SUM(IF status = delivered THEN amount) | 不等同會計收入 |
| Delivered Orders | 已送達訂單筆數 | COUNTD(IF delivered THEN Order Id) | 用於 AOV 分母 |
| Delivered AOV | 已送達金額／已送達筆數 | Delivered Value / Delivered Orders | 非每客消費額 |
| Cancellation Rate | 取消筆數／全部訂單筆數 | Cancelled Orders / Order Count | 狀態為快照 |
| Units Sold | 已送達訂單數量 | SUM(IF delivered THEN Quantity) | 不等同去重 SKU |
| Customer Count | 唯一客戶數 | COUNTD(Customer Id) | 應標示狀態母體 |
| Revenue Contribution | 類別已送達金額占比 | Category delivered / Total delivered | 需清楚處理篩選範圍 |

### 3.1 Tableau 計算欄位

```tableau
// Delivered Sales Amount
IF [Order Status] = "delivered" THEN [Total Amount] ELSE 0 END
```

```tableau
// Delivered Orders
COUNTD(IF [Order Status] = "delivered" THEN [Order Id] END)
```

```tableau
// Delivered AOV
IF [Delivered Orders] > 0
THEN SUM([Delivered Sales Amount]) / [Delivered Orders]
END
```

```tableau
// Cancelled Orders
COUNTD(IF [Order Status] = "cancelled" THEN [Order Id] END)
```

```tableau
// Cancellation Rate
IF COUNTD([Order Id]) > 0
THEN [Cancelled Orders] / COUNTD([Order Id])
END
```

```tableau
// Delivered Units
IF [Order Status] = "delivered" THEN [Quantity] ELSE 0 END
```

**重要：** Tableau 在欄位名和彙總層級上需要一致，計算欄位名稱以實際建立的名稱為準。2025 年全部訂單取消率應為 2,004／40,259 ≈ 4.98%。

### 3.2 金額及資料期間的篩選規則

- Overview 的年度主 KPI 用 2025 年；在標題明示「2025」。
- 狀態占比圖使用**全部狀態**；已送達 KPI 不應被儀表板「只顯示 delivered」篩選器意外改變取消率分母。
- 若要跨不同日期區間比較，須同時計算每區間分子與分母。
- MoM（Table Calculation）僅在月度排序正確時適用，第一期回傳 NULL；不可用 2026 年 1 月部分資料做完整月 YoY。

## 4. 描述性統計方法

| 方法 | 用途 | Tableau 建議呈現 |
|---|---|---|
| COUNT、COUNTD、SUM | 訂單、客戶及金額 | KPI Cards |
| AVG、MEDIAN | 平均與典型訂單金額 | KPI + Box Plot |
| STDEV、Q1／Q3、P90／P95 | 分散程度及高額訂單 | Box Plot／分布摘要 |
| Frequency Distribution | 客戶購買頻次 | Histogram |
| Percentage of Total | 品類貢獻 | Bar／Pareto |
| Month-over-Month | 月度變化率 | Line + Tooltip |
| Cross Tabulation | 地區 × 品類 | Heatmap |

2025 年已送達訂單金額：**平均 29,086.91、中位 10,928、樣本標準差 42,073.25**，顯示不宜只呈現平均數。

### 4.1 統計解釋規範

- 「差異很小」是描述性說法，不等於「統計上不顯著」。
- 如需比較各地取消率，可先計算比例與信賴區間，再視目的採卡方檢定與效果量 Cramér's V；需檢查假設及多重比較問題。
- 即使統計上顯著，也須考慮效果量與決策成本。
- 如果資料是模擬生成，檢定結果不得自動推論真實市場。

## 5. 四大分析維度

### 5.1 Time｜時間

**問題：** 2025 年各月已送達訂單金額、筆數與客單價如何變動？

- 維度：月（`DATETRUNC('month', [Order Date])`）。
- 指標：Delivered Order Value、Delivered Orders、AOV、MoM。
- 圖表：月度折線圖、類別 × 月份熱力圖。
- 限制：2025 年只有一個完整年份，不能確定多年季節性。

### 5.2 Product｜商品

**問題：** 銷售額是否集中於少數類別？是件數多還是價格高？

- 維度：Product Category、Product Name。
- 指標：Delivered Sales、Delivered Units、加權平均售價 = Delivered Sales / Delivered Units、品類占比。
- 圖表：品類排序長條圖、Top 10 商品、Pareto。
- 數學拆解：`類別銷售額 = 類別銷售件數 × 類別加權平均單價`。
- 限制：缺 COGS、利潤及唯一 SKU，無法做獲利排名。

### 5.3 Customer｜客戶

**問題：** 客戶購買頻次與已送達消費金額如何分布？

- Frequency：2025 年每位客戶已送達訂單數。
- Monetary：2025 年每位客戶已送達訂單金額合計。
- Recency：以固定 `2025-12-31` 作為基準日，計算距最近已送達訂單日期的天數；若有 2025-12-31 晚間資料仍可同日計 0 天。
- RFM 可以採分位數 1–5 分群，但須註明分位數重複值及分群邏輯。
- 限制：觀察窗左截斷，不能得出完整客戶終身價值（CLV）或真正的新舊客完整生命週期。

### 5.4 Geography｜區域

**問題：** 16 個配送地區的銷售規模及品類組成有何差異？

- 維度：Shipping Region。
- 指標：已送達金額、訂單數、AOV、取消率。
- 圖表：區域排行、熱力圖；若 Tableau 可正確地理編碼則呈現地圖。
- 限制：不是台灣所有縣市，避免推論「全台市場占有率」；差異小時以表格簡報即可。

### 5.5 Operations｜跨維度補充

- Order Status × Product Category / Payment Method / Shipping Region。
- 只分析**狀態分布與取消率**；缺狀態時間歷史，不計算配送時效或延遲原因。

## 6. Dashboard UI／UX 規格

建議桌面優先 **1440 × 900 px**；採 Tableau Tiled Containers，優先視覺順序為 KPI → 趨勢 → 分類 → 細節／註解。

| 頁面 | 主要問題 | 必須有的元件 |
|---|---|---|
| 01 Executive Overview | 業績及完成狀態如何？ | KPI Cards、月度趨勢、品類結構、狀態占比 |
| 02 Product & Revenue | 哪些商品創造金額？ | 類別水平長條、Top 10、件數×價格拆解 |
| 03 Time & Customer | 何時銷售較高？哪些客戶較活躍？ | 月度趨勢、消費分布、購買頻次 |
| 補充 Geography | 區域與品類是否不同？ | 地區排名、熱力圖或地圖 |
| 補充 Operations | 訂單狀態分布如何？ | 100% 堆疊圖、取消率交叉表 |

**視覺規範：**
- 使用少量固定語意色：主要 KPI 同一藍綠色；pending 橘色、cancelled 紅色。
- 同一類指標統一數字格式，清楚標示幣別未知。
- Tooltip 至少顯示母體、日期、金額、筆數、占比。
- 避免 3D、裝飾圖及無法比較精確大小的圖形；明確顯示空值與無資料。
- 支援 Filter Actions；特別測試篩選之後分母是否仍正確。

## 7. Tableau Story 規劃

| Story Point | 核心問題 | 證據類型 | 預期結論層級 |
|---|---|---|---|
| 1 | 2025 年整體表現？ | Delivered KPI + 月趨勢 | 已知數值 |
| 2 | 營收集中在哪？ | 3C／家居結構 | 已知集中度 |
| 3 | 件數或單價主導？ | 銷售件數與加權均價 | 待圖表驗證 |
| 4 | 顧客／區域有何補充線索？ | Frequency／Region 交叉表 | 待完成分析 |
| 5 | 管理者下一步做什麼？ | 建議 + 限制 + 補充資料 | 後續行動 |

每點使用「**問題 → 證據 → 解讀 → 限制 → 建議**」。不臆測促銷或廣告效果；缺資料時列出需要補的證據。

## 8. Tableau QA 驗收

- [ ] 主 KPI：2025 已送達筆數 **28,095**；金額 **817,196,808**。
- [ ] 2025 全訂單 **40,259**；取消 **2,004**。
- [ ] 全期間 40,500 筆；已送達 28,257 筆；全訂單金額 1,166,503,556。
- [ ] 3C + 家居金額占比與 EDA 相符（注意全期間／年度母體不同）。
- [ ] 日期和狀態切換時，KPI 的分子與分母同步。
- [ ] Tooltip、地區、月份排序與 NULL 情形通過檢查。
- [ ] 已保存 Tableau Workbook（`.twbx`）或可重連資料的 `.twb`。
- [ ] 明確顯示來源、更新日期、幣別及資料限制。

---
**下一份文件：** [05_Executive_Report.md](./05_Executive_Report.md)。
