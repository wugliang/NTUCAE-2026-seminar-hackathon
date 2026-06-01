---
name: gee-coordinate-transform
description: >
  Google Earth Engine (GEE) 座標轉換與多資料集空間對齊工作流程。
  當使用者需要轉換坐標系統、對齊不同來源的地理資料、處理 shapefile/KML/GeoJSON 的
  投影問題、或確保多個資料集在空間上完全對齊時，必須載入此 Skill。
  觸發情境包含：提到 CRS、EPSG、TWD97、WGS84、UTM、坐標轉換、geometry 對齊、
  向量與柵格資料對齊、座標系統不一致等問題。
  即使使用者說「我的 shapefile 和 GEE 影像位置對不上」，也應觸發此 Skill。
---

# GEE 座標轉換 Skill

## 目的

確保所有匯入 GEE 的向量與柵格資料使用統一的坐標參考系統（CRS），
解決多資料集空間對齊問題，避免因坐標系統不一致導致的分析錯誤。

---

## 領域知識

### GEE 坐標系統規則

- GEE 的 **地圖顯示** 使用 EPSG:3857（Web Mercator）
- GEE 的 **資料儲存與運算** 預設 EPSG:4326（WGS84 地理坐標）
- **上傳至 GEE 的 Asset** 會自動轉換為 EPSG:4326，但原始精度可能損失
- **運算輸出的 CRS** 由輸入影像的 `.projection()` 決定，除非明確覆蓋

### 台灣研究常用坐標系統

| 名稱 | EPSG | 用途 | 注意事項 |
|------|------|------|---------|
| WGS84 地理坐標 | 4326 | GEE 預設，全球通用 | 單位為度，不可直接計算距離 |
| TWD97 TM2 中央經線 121° | 3826 | 台灣官方投影，全島 | 台灣政府資料標準格式 |
| TWD97 TM2 中央經線 119° | 3825 | 澎湖、離島 | 離島資料用此系統 |
| WGS84 UTM Zone 51N | 32651 | 台灣北中部（121°E 附近） | 國際研究常用 |
| WGS84 UTM Zone 50N | 32650 | 台灣西南部 | 較少使用 |
| Web Mercator | 3857 | GEE 地圖顯示 | 不適合面積計算 |

> 台灣研究建議優先使用 EPSG:3826（TWD97 TM2），與政府開放資料相容性最佳。

### GEE 中的向量坐標轉換限制

GEE **沒有內建的直接向量 CRS 轉換功能**（不像 QGIS/Proj4）。
替代方案：
1. 上傳前在本地端（QGIS、Python pyproj）先轉換好
2. 在 GEE 中透過 `.transform()` 對 geometry 物件進行變換

### Geometry 物件與坐標

```javascript
// GEE 中建立帶有明確 CRS 的 geometry
var point_WGS84 = ee.Geometry.Point([121.5654, 25.0330]);  // 預設 EPSG:4326
var point_UTM = ee.Geometry.Point([303395, 2771786], 'EPSG:32651');

// 轉換 geometry 的 CRS
var point_reprojected = point_WGS84.transform('EPSG:32651', 0.001);
//   第二個參數為最大誤差（度），越小越精確但越慢
```

---

## 輸入規格

```
必要輸入
├── 來源資料的 CRS（如不確定，說明如何查詢）
├── 目標 CRS（依研究需求決定）
└── 資料類型（向量 geometry / 柵格 Image / Feature）

選填輸入
├── 轉換精度要求（最大允許誤差，單位：度，預設 0.001）
├── 是否需要驗證轉換結果（建議：是）
└── 本地端坐標轉換腳本需求（Python pyproj 或 QGIS）
```

---

## 執行步驟

### Step 1：診斷坐標系統問題

```javascript
// 查詢影像的投影資訊
var imgProj = image.projection();
print('Image CRS:', imgProj.crs());
print('Image transform:', imgProj.transform());

// 查詢 Feature 的坐標
var feature = ee.Feature(featureCollection.first());
print('Feature geometry type:', feature.geometry().type());
// 注意：GEE 中的 FeatureCollection 幾何通常已為 EPSG:4326
```

### Step 2a：GEE 內的 Geometry 坐標轉換

```javascript
// 情境：將一個 geometry 從 TWD97 (EPSG:3826) 轉換為 WGS84 (EPSG:4326)
var geom_twd97 = ee.Geometry.Point([303395, 2771786], 'EPSG:3826');
var geom_wgs84 = geom_twd97.transform('EPSG:4326', 0.001);
print('WGS84 coordinates:', geom_wgs84.coordinates());
```

```javascript
// 情境：FeatureCollection 整體轉換
var fc_reprojected = featureCollection.map(function(feature) {
  return feature.setGeometry(
    feature.geometry().transform('EPSG:32651', 0.001)
  );
});
```

### Step 2b：本地端轉換（上傳前，Python）

當 GEE 的原生轉換不足以處理複雜情況時，建議在本地端處理：

```python
# 使用 pyproj 進行坐標轉換（Python 本地端執行，非 GEE）
from pyproj import Transformer

transformer = Transformer.from_crs("EPSG:3826", "EPSG:4326", always_xy=True)

# 單點轉換
lon, lat = transformer.transform(303395, 2771786)
print(f"WGS84: {lon:.6f}, {lat:.6f}")

# GeoDataFrame 批次轉換（需 geopandas）
import geopandas as gpd
gdf = gpd.read_file('your_shapefile.shp')
gdf_wgs84 = gdf.to_crs('EPSG:4326')
gdf_wgs84.to_file('output_wgs84.shp')
```

### Step 3：柵格影像多資料集對齊

當兩個影像來自不同資料集，需確保完全對齊：

```javascript
// 以參考影像的投影為基準，對齊目標影像
var referenceProj = referenceImage.projection();

var alignedImage = targetImage
  .resample('bilinear')      // 連續值；分類資料改用 'nearestNeighbor'
  .reproject(referenceProj); // 完全對齊至參考影像的格網

// 驗證對齊結果
print('Reference projection:', referenceProj);
print('Aligned projection:', alignedImage.projection());
```

### Step 4：向量與柵格對齊確認

```javascript
// 確保 FeatureCollection 與 Image 在同一坐標系下操作
var samplePoints = ee.FeatureCollection([
  ee.Feature(ee.Geometry.Point([121.5654, 25.0330])),
  ee.Feature(ee.Geometry.Point([120.9738, 23.9739]))
]);

// 從影像中取樣（GEE 自動處理坐標系統差異，但 scale 必須明確）
var samples = image.sampleRegions({
  collection: samplePoints,
  scale: 30,              // 依影像解析度調整
  projection: 'EPSG:4326' // 明確指定
});
print('Sample results:', samples);
```

### Step 5：自我驗證

```javascript
// 驗證轉換後坐標在預期範圍內（以台灣為例）
var bounds = transformedGeom.bounds();
var coords = bounds.coordinates().get(0);
print('Bounding box:', coords);

// 台灣預期範圍（WGS84）：
// 經度 119.3° ~ 122.1°，緯度 21.9° ~ 25.4°
// 若轉換後超出此範圍，坐標系統選擇可能有誤
```

---

## 常見問題排查

| 症狀 | 可能原因 | 解決方式 |
|------|---------|---------|
| 影像與向量顯示在不同位置 | CRS 不一致 | 檢查兩者 `.projection().crs()` |
| 計算面積結果是 0 或極大值 | 使用了地理坐標（度）計算 | 轉換為公尺投影後再計算 |
| `sampleRegions` 返回空結果 | 向量不在影像範圍內（可能因 CRS 錯誤） | 在地圖上 `Map.addLayer()` 目視確認位置 |
| `.transform()` 執行極慢 | maxError 設太小或 geometry 過於複雜 | 增大 maxError 至 1 或先 simplify |

---

## 輸出合約

產出的程式碼必須滿足以下條件：

- [ ] 所有 geometry 建立時明確標注 CRS（不依賴預設值）
- [ ] `.transform()` 呼叫包含明確的 `maxError` 參數
- [ ] 向量與柵格操作前確認雙方 CRS 一致
- [ ] 包含目視驗證步驟（`Map.addLayer()`）和數值驗證（`print()`）
- [ ] 如需本地端轉換，提供 Python pyproj 或 geopandas 程式碼

---

## 參考資源

- GEE Geometry 文件：https://developers.google.com/earth-engine/guides/geometries
- EPSG 查詢：https://epsg.io/
- 台灣坐標系統說明：`references/taiwan-crs.md`（待補充）
