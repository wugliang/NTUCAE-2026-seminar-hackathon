---
name: gee-workflow
description: >
  Google Earth Engine (GEE) 端到端工作流程總調度（orchestrator / router）。
  當使用者的需求橫跨「擷取 → 座標對齊 → 空間前處理 → 匯出」其中兩段以上，
  或描述的是一段完整任務而非單一操作時，必須載入此 Skill。
  觸發情境包含：使用者一句話同時提到資料來源、區域、CRS／解析度、輸出格式
  （例：「我要台中 2023 年的 NDVI，對齊到 TWD97、100m，匯出成 GeoTIFF」），
  或說「幫我跑完整流程」、「從抓資料到匯出」、「end to end」、「整套做完」。
  本 Skill 只負責兩件事：判定任務該用哪幾個子 Skill、以及確保跨階段的共用
  參數一致；實際領域知識與正確性委派給四個子 Skill。
---

# GEE 工作流程總調度 Skill

## 目的

把使用者「一整段任務」對應到正確的子 Skill 組合，並確保這些子 Skill
之間的共用參數（研究區域、CRS、scale）全程一致。

本 Skill **不重複** 子 Skill 的領域知識，**不替使用者預設任務長怎樣**。
它只做兩件事：

1. **流程判定** — 這個任務需要哪幾個子 Skill？順序為何？
2. **參數一致性** — 跨階段共用的參數，是否從頭到尾用同一組值？

---

## 子 Skill 地圖

| 階段 | 子 Skill | 負責 | 主要產出物 |
|------|---------|------|-----------|
| 擷取 | `gee-data-retrieval` | 載入資料集、篩選、雲遮罩、reduce 成單一影像 | `ee.Image`（已 reduce） |
| 座標對齊 | `gee-coordinate-transform` | CRS 診斷、向量轉換、多資料集對齊 | 統一 CRS 的 geometry / Image |
| 空間前處理 | `gee-spatial-preprocessing` | reproject、resample、解析度統一、多波段堆疊 | 對齊好的 `ee.Image` |
| 匯出 | `gee-export-format` | Export 任務設定、格式、metadata、檔名 | Export Task |

> 載入子 Skill 後，**各段的輸入需求、做法、正確性都以該子 Skill 為準**，
> 本 Skill 不另立規格，也不重複驗收。

---

## 一、流程判定

依使用者實際需求，判斷需要哪幾段，不必每次跑滿四段。判定線索：

- **要結合第二個資料來源、或外部向量檔（shapefile/KML）嗎？** → 需要「座標對齊」
- **要改變或統一解析度、或堆疊多波段嗎？** → 需要「空間前處理」
- **最終要落地成檔案 / Asset / CSV 嗎？** → 需要「匯出」
- 其餘情況（只想抓圖看、印統計值），用到的段就更少。

判定後，先告知使用者本次會用到哪幾個子 Skill、順序為何，再依序載入執行。
每一段的輸入該問什麼、怎麼做，交給該子 Skill 自己處理。

---

## 二、參數一致性

跨階段共用、且必須一致的參數只有三個。在進入流程前確認一次，
之後每一段沿用同一組值，不要中途漂移：

| 共用參數 | 涉及階段 | 一致性要求 |
|---------|---------|-----------|
| 研究區域 `studyArea` | 全部 | 全程同一個 geometry，不中途更換 |
| 目標 CRS | 對齊 / 前處理 / 匯出 | 三處必須是同一個 EPSG（台灣建議 3826） |
| 目標 scale（解析度） | 前處理 / 匯出 | 兩處相同，且不小於來源資料原始解析度 |

> 其他參數（資料集、時間範圍、雲量閾值、輸出格式…）屬於單一階段，
> 由對應子 Skill 各自收集，本 Skill 不介入。

完成後，快速核對這三項是否從頭到尾一致即可——這是四個子 Skill
各自看不到、唯有外層能把關的部分。

---

## 參考資源

- 子 Skill：`gee-data-retrieval`、`gee-coordinate-transform`、`gee-spatial-preprocessing`、`gee-export-format`
- GEE 官方指南：https://developers.google.com/earth-engine/guides
