---
name: gee-spatial-preprocessing
description: >
  Google Earth Engine (GEE) 空間前處理工作流程。當使用者需要對影像進行
  重投影（reproject）、重採樣（resample）、裁切（clip）、解析度統一、
  多波段堆疊或影像標準化時，必須載入此 Skill。
  觸發情境包含：提到 CRS 轉換、像素對齊、reproject、resample、focal_mean、
  reduceResolution、多影像空間對齊、影像前處理等操作。
  即使使用者說「幫我把兩個影像對齊到同一解析度」，也應觸發此 Skill。
---

# GEE 空間前處理 Skill

## 目的

對 GEE 影像執行必要的空間標準化操作，確保多個資料集在 CRS、解析度、
範圍三個維度上完全對齊，使後續分析不因空間不一致而產生錯誤。

---

## 領域知識

### GEE 的投影系統行為

GEE 預設以 **EPSG:4326（WGS84 經緯度）** 儲存和顯示影像，但：

- 計算**面積、距離、周長**時，必須轉換為公尺投影（例：UTM 或 Web Mercator EPSG:3857）
- `.reproject()` 會強制 GEE 在指定 CRS 下計算，**可能大幅增加運算量**，非必要不使用
- 偏好使用 `.resample()` + `.reproject()` 的組合，而非單獨使用 `.reproject()`

### 解析度處理的兩個方向

| 情境 | 操作 | GEE 方法 |
|------|------|---------|
| 高解析度 → 低解析度（降採樣） | 聚合像素（aggregation） | `.reduceResolution()` + `.reproject()` |
| 低解析度 → 高解析度（升採樣） | 插值（interpolation） | `.resample('bilinear')` 或 `.resample('bicubic')` |

> ⚠️ **升採樣不增加資訊量**，只影響像素大小，需在報告中說明。

### 重採樣內插方法選擇

| 資料類型 | 建議方法 | 原因 |
|---------|---------|------|
| 連續值（溫度、降雨、NDVI） | `bilinear` 或 `bicubic` | 平滑內插，保留數值連續性 |
| 分類資料（土地利用、土地覆蓋） | `nearestNeighbor` | 避免產生不存在的類別值 |

### 常見陷阱

1. **`.reproject()` 的代價**：強制指定投影後，GEE 會在全圖範圍執行，極耗資源。
   大區域分析請先 `.clip()` 再 `.reproject()`
2. **`.reduceResolution()` 需搭配 `.reproject()`**：單獨呼叫 `reduceResolution` 不會實際改變解析度
3. **影像對齊 ≠ 影像重疊**：即使 extent 相同，像素格點不對齊仍會導致分析錯誤
4. **`.select()` 順序影響堆疊**：`addBands` 後的波段順序決定後續索引，務必確認

---

## 輸入規格

```
必要輸入
├── 來源影像（ee.Image 物件或程式碼中的變數名稱）
├── 目標 CRS（例：'EPSG:32651'，UTM Zone 51N）
├── 目標解析度（像素大小，單位：公尺）
└── 研究區域 geometry（用於 clip，避免全球運算）

選填輸入
├── 重採樣方法（預設：連續值 'bilinear'、分類值 'nearestNeighbor'）
├── 是否需要堆疊多波段（是→提供所有影像變數名稱與目標波段名稱）
└── 是否需要 Z-score 標準化（適用機器學習前處理）
```

---

## 執行步驟

### Step 1：確認來源影像的投影與解析度

```javascript
// 先查詢，再決定操作
var sourceProj = sourceImage.projection();
print('Source CRS:', sourceProj.crs());
print('Source scale (m):', sourceProj.nominalScale());
```

### Step 2a：降採樣（高解析度 → 低解析度）

```javascript
// 例：Sentinel-2 (10m) 降採樣至 100m
var downsampled = sourceImage
  .reduceResolution({
    reducer: ee.Reducer.mean(),   // 連續值用 mean，分類值用 mode
    maxPixels: 1024               // 防止記憶體溢出
  })
  .reproject({
    crs: 'EPSG:32651',           // 目標 CRS
    scale: 100                   // 目標解析度（公尺）
  });
```

### Step 2b：升採樣（低解析度 → 高解析度）

```javascript
// 例：MODIS (500m) 升採樣至 100m
var upsampled = sourceImage
  .resample('bilinear')          // 連續值；分類資料改用 'nearestNeighbor'
  .reproject({
    crs: 'EPSG:32651',
    scale: 100
  });
```

### Step 3：裁切至研究區域

```javascript
// 必須在 reproject 後執行，節省運算資源
var clipped = processedImage.clip(studyArea);
```

### Step 4：多影像空間對齊堆疊

```javascript
// 確保所有影像使用相同 CRS 與 scale 後，再堆疊
var stackedImage = image1
  .addBands(image2.select('BAND_NAME').rename('LAYER_2'))
  .addBands(image3.select('BAND_NAME').rename('LAYER_3'))
  .clip(studyArea);

print('Stacked bands:', stackedImage.bandNames());
```

### Step 5（選用）：Z-score 標準化

```javascript
// 用於機器學習或多波段比較前的標準化
var bandNames = stackedImage.bandNames();
var meanDict = stackedImage.reduceRegion({
  reducer: ee.Reducer.mean(),
  geometry: studyArea,
  scale: 100,
  maxPixels: 1e9
});
var stdDict = stackedImage.reduceRegion({
  reducer: ee.Reducer.stdDev(),
  geometry: studyArea,
  scale: 100,
  maxPixels: 1e9
});

// 逐波段標準化
var normalized = stackedImage.subtract(ee.Image.constant(
  bandNames.map(function(b) { return ee.Number(meanDict.get(b)); })
)).divide(ee.Image.constant(
  bandNames.map(function(b) { return ee.Number(stdDict.get(b)); })
));
```

### Step 6：自我驗證

```javascript
// 確認輸出影像的 CRS 與 scale 符合預期
var outProj = clipped.projection();
print('Output CRS:', outProj.crs());
print('Output scale (m):', outProj.nominalScale());
print('Output bands:', clipped.bandNames());

// 抽樣確認數值範圍合理
var stats = clipped.reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: studyArea,
  scale: TARGET_SCALE,
  maxPixels: 1e8
});
print('Value range check:', stats);
```

---

## 輸出合約

產出的程式碼必須滿足以下條件：

- [ ] 明確指定目標 CRS 字串（如 `'EPSG:32651'`），不使用數字 EPSG code
- [ ] `.reduceResolution()` 與 `.reproject()` 成對出現，不單獨使用
- [ ] 分類資料使用 `nearestNeighbor`，連續資料使用 `bilinear` 或 `bicubic`
- [ ] 所有 `.reproject()` 操作在 `.clip()` 之前執行（先定義投影再裁切）
- [ ] 多影像堆疊前所有影像已統一 CRS 與 scale
- [ ] 包含 `print()` 驗證 CRS、scale 與數值範圍

---

## 參考資源

- GEE Projections 文件：https://developers.google.com/earth-engine/guides/projections
- Reducer 選項參考：`references/reducer-reference.md`（待補充）
