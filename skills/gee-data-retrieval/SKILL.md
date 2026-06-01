---
name: gee-data-retrieval
description: >
  Google Earth Engine (GEE) 資料擷取工作流程。當使用者需要從 GEE 取得氣候、
  降雨、土地利用、植被指數或任何地理空間資料集時，必須載入此 Skill。
  觸發情境包含：提到 ImageCollection、篩選日期範圍、選取波段、計算統計值、
  mosaic、median composite、clip to region 等操作。
  即使使用者只說「幫我抓 NDVI 資料」或「我要 CHIRPS 降雨量」，也應觸發此 Skill。
---

# GEE 資料擷取 Skill

## 目的

從 Google Earth Engine 的資料目錄正確擷取影像集（ImageCollection）或影像（Image），
套用合法的篩選條件與 scale 參數，確保輸出資料可用於後續分析。

---

## 領域知識

### GEE 資料模型基礎

- **Image**：單一時間點的多波段影像，屬性（properties）儲存於 `.toDictionary()`
- **ImageCollection**：多個 Image 的集合，不可直接匯出，需先 reduce（取 median、mean 等）
- **FeatureCollection**：向量資料，與 Image 分開處理
- **ee.Date**：GEE 內部時間格式，字串須用 `ee.Date('YYYY-MM-DD')` 包覆

### 常用資料集識別碼與注意事項

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 關鍵波段 |
|--------|-------------|-----------|-----------|---------|
| Landsat 8 SR | `LANDSAT/LC08/C02/T1_L2` | 30m | 16天 | SR_B2–B7, QA_PIXEL |
| Sentinel-2 SR | `COPERNICUS/S2_SR_HARMONIZED` | 10–60m | 5天 | B2–B12, QA60 |
| MODIS NDVI | `MODIS/061/MOD13A2` | 500m | 16天 | NDVI, EVI |
| CHIRPS 降雨 | `UCSB-CHG/CHIRPS/DAILY` | ~5km | 每日 | precipitation |
| ERA5 氣候 | `ECMWF/ERA5_LAND/DAILY_AGGR` | ~9km | 每日 | 依需求 |
| ESA 土地利用 | `ESA/WorldCover/v200` | 10m | 年度 | Map |
| SRTM 地形 | `USGS/SRTMGL1_003` | 30m | 靜態 | elevation |

### 常見陷阱

1. **Scale 參數**：呼叫 `.reduceRegion()` 或 `.sampleRegions()` 時**必須**明確指定 `scale`，
   否則 GEE 會用預設值（通常是不合適的 1 度）導致錯誤或不合理結果
2. **Cloud masking 順序**：先 mask 再 reduce，不是 reduce 再 mask
3. **日期過濾**：`.filterDate()` 的結束日期為**排除**（exclusive），需 +1 天
4. **ImageCollection ≠ Image**：永遠先 `.first()` 或 `.median()` 後再操作像素值
5. **投影系統**：預設為 EPSG:4326，計算面積/距離前須轉換為公尺投影

---

## 輸入規格

執行前確認使用者提供以下資訊（如未提供，主動詢問）：

```
必要輸入
├── 資料集名稱或類型（例：Landsat 8、CHIRPS 降雨）
├── 時間範圍（起始日期、結束日期，格式 YYYY-MM-DD）
├── 研究區域（geometry 物件 或 asset path 或 描述）
└── 目標波段或變數名稱

選填輸入
├── 雲覆蓋率閾值（Sentinel/Landsat 適用，預設 20%）
├── 時間聚合方式（median、mean、sum、max，預設 median）
└── 輸出 CRS（預設 EPSG:4326）
```

---

## 執行步驟

### Step 1：載入資料集，建立 ImageCollection

```javascript
// 使用完整 Asset ID，避免縮寫
var collection = ee.ImageCollection('ASSET_ID_HERE')
  .filterDate('YYYY-MM-DD', 'YYYY-MM-DD')  // 注意結束日期 exclusive
  .filterBounds(studyArea);                 // studyArea 為 geometry 物件
```

**驗證點**：印出 `print('Collection size:', collection.size())` 確認集合非空。

### Step 2：雲遮罩（Landsat / Sentinel 適用）

```javascript
// Landsat 8 C02 SR 雲遮罩
function maskL8sr(image) {
  var qaMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
  var satMask = image.select('QA_RADSAT').eq(0);
  return image.updateMask(qaMask).updateMask(satMask)
    .multiply(0.0000275).add(-0.2)  // 套用 scale factor（C02 必要）
    .copyProperties(image, ['system:time_start']);
}

// Sentinel-2 雲遮罩（使用 QA60 波段）
function maskS2clouds(image) {
  var qa = image.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
    .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask)
    .divide(10000)  // Sentinel-2 DN → reflectance
    .copyProperties(image, ['system:time_start']);
}
```

### Step 3：波段計算（如 NDVI、EVI）

```javascript
// NDVI 計算
var addNDVI = function(image) {
  var ndvi = image.normalizedDifference(['NIR_BAND', 'RED_BAND'])
    .rename('NDVI');
  return image.addBands(ndvi);
};

collection = collection.map(addNDVI);
```

> 波段名稱依資料集而異，參見上方「常用資料集」表格。

### Step 4：時間聚合，取得單一 Image

```javascript
// 依需求選擇聚合方式
var compositeImage = collection.median();  // 推薦：對離群值不敏感
// 或
var compositeImage = collection.mean();
var compositeImage = collection.sum();     // 適用降雨量累計
```

### Step 5：裁切至研究區域

```javascript
var clipped = compositeImage.clip(studyArea);
```

### Step 6：自我驗證

執行後列印以下資訊確認結果合理：

```javascript
print('Image bands:', clipped.bandNames());
print('Image CRS:', clipped.projection().crs());
print('Sample pixel value:', clipped.sample({
  region: studyArea,
  scale: 30,   // 依資料集填入
  numPixels: 5
}));
```

---

## 輸出合約

產出的程式碼必須滿足以下條件：

- [ ] `ee.ImageCollection` 已正確 reduce 為單一 `ee.Image`
- [ ] 使用完整且正確的 Asset ID（可在 GEE Catalog 驗證）
- [ ] 所有 `.reduceRegion()` / `.sampleRegions()` 皆明確指定 `scale`
- [ ] 雲遮罩函式（如適用）在 reduce 之前套用
- [ ] 輸出影像已使用 `.clip()` 限定範圍
- [ ] 程式碼附有 `print()` 驗證步驟

---

## 參考資源

- GEE 資料目錄：https://developers.google.com/earth-engine/datasets
- Scale 選擇原則：`references/scale-guide.md`（待補充）
- 各資料集波段對照：`references/band-reference.md`（待補充）
