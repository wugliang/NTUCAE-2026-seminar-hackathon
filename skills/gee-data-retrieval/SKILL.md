---
name: gee-data-retrieval
description: >
  Google Earth Engine (GEE) 資料擷取工作流程。當使用者需要從 GEE 取得任何地理空間
  資料集時，必須載入此 Skill。觸發情境包含：植被分析（NDVI、EVI、Sentinel-2、MODIS、
  Landsat）、氣候水文（ERA5、GPM、CHIRPS、JRC 地表水）、土地利用（ESA WorldCover、
  Dynamic World、MCD12Q1）、地形（SRTM、NASADEM、LST 地表溫度）、災害環境
  （Sentinel-1 SAR、Sentinel-5P 空品、FIRMS 火災）。
  提到 ImageCollection、filterDate、filterBounds、雲遮罩、mosaic、median composite、
  clip、reduceRegion、scale 參數等操作時皆應觸發。
  即使使用者只說「幫我抓 NDVI」、「我要看淹水範圍」、「給我空污資料」，也應觸發此 Skill。
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

---

### 資料集索引（依應用面向）

#### 一、植被與農業

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 適用情境 |
|--------|-------------|-----------|-----------|---------|
| Sentinel-2 SR | `COPERNICUS/S2_SR_HARMONIZED` | 10m | 5天 | 精準農業、小區域植被、紅邊波段分析 |
| MODIS NDVI 16天 | `MODIS/061/MOD13A1` | 500m | 16天 | 長時序物候、大尺度季節趨勢 |
| Landsat 8 SR | `LANDSAT/LC08/C02/T1_L2` | 30m | 16天 | 2013年後中尺度農林分析 |
| Landsat 9 SR | `LANDSAT/LC09/C02/T1_L2` | 30m | 16天 | 2021年後，與 L8 波段相容 |

**關鍵波段對照：**
- Sentinel-2：`B4`=Red, `B8`=NIR, `B5/B6/B7`=Red Edge, `B11/B12`=SWIR
- Landsat 8/9 SR：`SR_B4`=Red, `SR_B5`=NIR，DN 需套用 scale factor（×0.0000275 - 0.2）

#### 二、氣候與水文

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 適用情境 |
|--------|-------------|-----------|-----------|---------|
| ERA5-Land 月 | `ECMWF/ERA5_LAND/MONTHLY_BY_HOUR` | ~9km | 每月 | 氣候變遷、長期趨勢 |
| ERA5-Land 日 | `ECMWF/ERA5_LAND/DAILY_AGGR` | ~9km | 每日 | 日尺度溫度、土壤水、蒸發 |
| GPM IMERG | `NASA/GPM_L3/IMERG_V07` | ~11km | 半小時/每日 | 暴雨事件、梅雨、水文逕流模擬 |
| JRC 地表水 | `JRC/GSW1_4/GlobalSurfaceWater` | 30m | 月/年統計 | 湖泊河流變遷、永久/季節性水體判別 |

**ERA5 版本選擇：** 依時間解析度需求選不同版本，`MONTHLY_BY_HOUR` 適合多年趨勢，`DAILY_AGGR` 適合事件分析。

#### 三、土地利用變遷

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 適用情境 |
|--------|-------------|-----------|-----------|---------|
| ESA WorldCover 2020 | `ESA/WorldCover/v100` | 10m | 靜態（2020） | 高精度現況分類，11類 |
| ESA WorldCover 2021 | `ESA/WorldCover/v200` | 10m | 靜態（2021） | 同上，更新版 |
| Dynamic World | `GOOGLE/DYNAMICWORLD/V1` | 10m | 近即時 | 即時土地變化偵測、機率值輸出 |
| MODIS 土地覆蓋年 | `MODIS/061/MCD12Q1` | 500m | 每年 | 2001年起逐年全球分類，長期都市化趨勢 |

**Dynamic World 注意：** 輸出為各類別機率值，需 `.select('label')` 取分類結果，或用 `.select(['trees','built','water'...])` 取機率層。

#### 四、地形與地表

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 適用情境 |
|--------|-------------|-----------|-----------|---------|
| SRTM DEM | `USGS/SRTMGL1_003` | 30m | 靜態 | 標準地形底圖，一行取坡度坡向 |
| NASADEM | `NASA/NASADEM_HGT/001` | 30m | 靜態 | SRTM 升級版，填補資料空洞（Voids）|
| MODIS LST 日 | `MODIS/061/MOD11A1` | 1km | 每日 | 都市熱島效應（UHI）、大範圍地溫監測 |

**SRTM 快速地形計算：**
```javascript
var srtm = ee.Image('USGS/SRTMGL1_003');
var terrain = ee.Terrain.products(srtm);  // 一次取得 elevation、slope、aspect
```

#### 五、災害與環境

| 資料集 | GEE Asset ID | 空間解析度 | 時間解析度 | 適用情境 |
|--------|-------------|-----------|-----------|---------|
| Sentinel-1 SAR GRD | `COPERNICUS/S1_GRD` | 10m | ~6–12天 | 颱風淹水範圍、崩塌偵測（穿透雲層）|
| Sentinel-5P NO₂ | `COPERNICUS/S5P/OFFL/L3_NO2` | ~3.5km | 每日 | 空氣污染、工業排放監測 |
| Sentinel-5P SO₂ | `COPERNICUS/S5P/OFFL/L3_SO2` | ~3.5km | 每日 | 火山噴發、工業污染 |
| FIRMS 火災 | `FIRMS` | 375m–1km | 近即時 | 林火偵測、燃燒範圍評估 |

---

### 常見陷阱

1. **Scale 參數**：呼叫 `.reduceRegion()` 或 `.sampleRegions()` 時**必須**明確指定 `scale`，
   否則 GEE 會用預設值導致錯誤或不合理結果

2. **日期過濾 exclusive**：`.filterDate('2023-01-01', '2023-12-31')` **不包含** 12/31，
   正確寫法為 `.filterDate('2023-01-01', '2024-01-01')`

3. **ImageCollection ≠ Image**：永遠先 `.median()` / `.first()` 後再操作像素值，
   不能對 ImageCollection 直接 `.clip()` 或 `.select()`

4. **Landsat C02 Scale Factor**：SR 波段原始 DN 值必須轉換：`×0.0000275 + (-0.2)`，
   未轉換直接計算 NDVI 會得到錯誤結果

5. **Sentinel-2 DN → 反射率**：`divide(10000)` 後才是 0–1 的反射率值

6. **Sentinel-1 無雲遮罩**：SAR 資料不需要也不能套用光學影像的雲遮罩函式，
   但需用 `.filter(ee.Filter.eq('instrumentMode', 'IW'))` 篩選正確模式

7. **ERA5 時間版本混用**：`MONTHLY_BY_HOUR` 與 `DAILY_AGGR` 的波段名稱不完全相同，
   切換版本時需重新確認波段名稱

8. **Dynamic World 是機率圖**：`label` 波段才是分類結果，直接取全部波段會拿到 9 個機率層

---

## 輸入規格

執行前確認使用者提供以下資訊（如未提供，主動詢問）：

```
必要輸入
├── 資料集名稱或類型（對照上方索引表選擇正確 Asset ID）
├── 時間範圍（起始日期、結束日期，格式 YYYY-MM-DD）
├── 研究區域（geometry 物件 或 asset path 或 描述）
└── 目標波段或變數名稱

選填輸入
├── 雲覆蓋率閾值（Sentinel-2/Landsat 適用，預設 20%）
├── 時間聚合方式（median、mean、sum、max，預設 median）
└── 輸出 CRS（預設 EPSG:4326）
```

---

## 執行步驟

### Step 1：載入資料集，建立 ImageCollection

```javascript
var collection = ee.ImageCollection('ASSET_ID_HERE')
  .filterDate('YYYY-MM-DD', 'YYYY-MM-DD')  // 結束日期 exclusive，需 +1 天
  .filterBounds(studyArea);

print('Collection size:', collection.size());  // 確認非空
```

### Step 2：雲遮罩（光學影像適用）

```javascript
// Landsat 8/9 C02 SR 雲遮罩 + Scale Factor 轉換
function maskL8sr(image) {
  var qaMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
  var satMask = image.select('QA_RADSAT').eq(0);
  return image.updateMask(qaMask).updateMask(satMask)
    .multiply(0.0000275).add(-0.2)
    .copyProperties(image, ['system:time_start']);
}

// Sentinel-2 SR 雲遮罩 + DN → 反射率
function maskS2clouds(image) {
  var qa = image.select('QA60');
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
    .and(qa.bitwiseAnd(cirrusBitMask).eq(0));
  return image.updateMask(mask)
    .divide(10000)
    .copyProperties(image, ['system:time_start']);
}

// Sentinel-1 SAR：不套用雲遮罩，改篩選模式
var s1 = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filterDate('YYYY-MM-DD', 'YYYY-MM-DD')
  .filterBounds(studyArea);
```

### Step 3：指數計算（如 NDVI）

```javascript
// 波段名稱依資料集對照上方索引表
var addNDVI = function(image) {
  return image.addBands(
    image.normalizedDifference(['NIR_BAND', 'RED_BAND']).rename('NDVI')
  );
};
collection = collection.map(addNDVI);
```

### Step 4：時間聚合

```javascript
var compositeImage = collection.median();  // 光學影像推薦，對雲殘影不敏感
// var compositeImage = collection.sum();  // 降雨量累計用 sum
// var compositeImage = collection.mean(); // 溫度等連續量用 mean
```

### Step 5：裁切

```javascript
var clipped = compositeImage.clip(studyArea);
```

### Step 6：自我驗證

```javascript
print('波段清單:', clipped.bandNames());
print('CRS:', clipped.projection().crs());
var stats = clipped.reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: studyArea,
  scale: 30,  // 依資料集解析度填入
  maxPixels: 1e8
});
print('數值範圍檢查:', stats);
// 光學反射率應在 0–1；NDVI 應在 -1–1；LST 應在合理溫度範圍
```

---

## 輸出合約

產出的程式碼必須滿足以下條件：

- [ ] 使用上方索引表中的完整 Asset ID，不自行縮寫或猜測
- [ ] `ee.ImageCollection` 已正確 reduce 為單一 `ee.Image`
- [ ] 所有 `.reduceRegion()` / `.sampleRegions()` 明確指定 `scale`
- [ ] Landsat/Sentinel-2 套用對應的 scale factor 轉換
- [ ] 光學影像的雲遮罩在 reduce 之前套用；SAR 影像改用模式篩選
- [ ] `.filterDate()` 結束日期已 +1 天（確保 inclusive）
- [ ] 輸出影像已 `.clip()` 限定範圍
- [ ] 包含 `print()` 驗證數值範圍合理性

---

## 參考資源

- GEE 資料目錄：https://developers.google.com/earth-engine/datasets
- 各資料集詳細波段說明：`references/band-reference.md`（待補充）