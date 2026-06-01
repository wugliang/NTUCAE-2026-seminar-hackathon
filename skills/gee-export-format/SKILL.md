---
name: gee-export-format
description: >
  Google Earth Engine (GEE) 匯出與格式化工作流程。當使用者需要將 GEE 分析結果
  匯出至 Google Drive、Cloud Storage 或 GEE Asset，或需要設定影像格式、
  metadata、檔名規則時，必須載入此 Skill。
  觸發情境包含：提到 Export.image、Export.table、toDrive、toAsset、toCloudStorage、
  下載影像、存成 GeoTIFF、CSV 匯出、任務提交（Task）等操作。
  即使使用者說「我要把結果存下來」或「怎麼下載 GEE 影像」，也應觸發此 Skill。
---

# GEE 匯出與格式化 Skill

## 目的

正確設定 GEE 的 Export 任務，確保輸出檔案的格式、解析度、CRS、
檔名與 metadata 符合後續分析需求，避免常見的匯出失敗或輸出錯誤。

---

## 領域知識

### GEE Export 的三種目的地

| 目的地 | 適用情境 | 限制 |
|--------|---------|------|
| `Export.image.toDrive()` | 個人研究、小規模資料 | 受 Google Drive 空間限制；需手動在 Task 頁面啟動 |
| `Export.image.toCloudStorage()` | 大型資料、自動化流程 | 需 GCS bucket 存取權限 |
| `Export.image.toAsset()` | 中間結果、需在 GEE 內部複用 | 受 GEE Asset 儲存空間限制（250 GB） |
| `Export.table.toDrive()` | FeatureCollection、統計結果 | 輸出格式：CSV、GeoJSON、KML、SHP |

### 影像匯出的必要參數

以下參數**必須明確指定**，不可依賴預設值：

| 參數 | 說明 | 常見錯誤 |
|------|------|---------|
| `scale` | 輸出像素大小（公尺） | 未指定→GEE 使用預設值（通常不適合） |
| `crs` | 輸出坐標系統 | 未指定→可能輸出為 EPSG:4326，坐標不直觀 |
| `region` | 輸出範圍（geometry） | 未指定→GEE 可能輸出全球範圍，極耗時 |
| `maxPixels` | 最大像素數量 | 預設 10^8，大區域需調高至 10^9 或更多 |
| `description` | Task 名稱 | 使用有意義的名稱方便管理 |
| `folder` | Drive 資料夾名稱 | 未指定→存至 Drive 根目錄 |

### GEE Task 的運作機制

- `Export.xxx` 只是**定義任務**，不會立即執行
- 需在 GEE Code Editor 右側的 **Tasks 頁面**點擊「Run」才會開始
- 任務在 GEE 伺服器背景執行，可關閉瀏覽器
- 大型匯出可能需要數分鐘至數小時
- 可用 Python Earth Engine API 自動提交任務（`task.start()`）

### GeoTIFF 最佳化設定

```
formatOptions
├── cloudOptimized: true   → 產生 Cloud-Optimized GeoTIFF（COG），支援部分讀取
└── noData: -9999          → 設定 NoData 值（缺失值標記）
```

---

## 輸入規格

```
必要輸入
├── 要匯出的 GEE 物件（ee.Image 或 ee.FeatureCollection 變數名稱）
├── 目的地類型（Drive / Cloud Storage / Asset）
├── 輸出解析度 scale（公尺）
├── 輸出 CRS
└── 研究區域 geometry（定義匯出範圍）

選填輸入
├── 檔案名稱規則（建議包含日期、資料集、區域、解析度）
├── Drive 資料夾名稱
├── Cloud Storage bucket 名稱（toCloudStorage 適用）
├── Asset 路徑（toAsset 適用）
├── 多波段輸出選項（合併或分開匯出）
├── NoData 值（預設 -9999）
└── 是否輸出為 Cloud-Optimized GeoTIFF（COG）
```

---

## 執行步驟

### Step 1：匯出前的影像最終確認

```javascript
// 確認影像狀態再匯出，避免匯出到錯誤的影像
print('Export target - bands:', imageToExport.bandNames());
print('Export target - CRS:', imageToExport.projection().crs());

// 用縮圖快速預覽
var vizParams = {min: 0, max: 1, bands: ['B4', 'B3', 'B2']};  // 依資料調整
Map.addLayer(imageToExport, vizParams, 'Export Preview');
Map.centerObject(studyArea, 10);
```

### Step 2：匯出至 Google Drive（最常用）

```javascript
Export.image.toDrive({
  image: imageToExport.toFloat(),    // 建議轉 Float32，避免整數截斷
  description: 'NDVI_Taiwan_2023_30m',  // Task 名稱，使用有意義的命名
  folder: 'GEE_Exports',             // Drive 資料夾
  fileNamePrefix: 'NDVI_Taiwan_2023_30m',  // 檔名前綴（副檔名自動加）
  region: studyArea,                 // 匯出範圍，必須指定
  scale: 30,                         // 輸出解析度（公尺）
  crs: 'EPSG:3826',                  // 輸出 CRS（台灣用 TWD97 TM2）
  maxPixels: 1e9,                    // 大區域需調高
  fileFormat: 'GeoTIFF',             // 推薦：GeoTIFF
  formatOptions: {
    cloudOptimized: true,            // 產生 COG，方便大檔案存取
    noData: -9999                    // NoData 標記
  }
});
```

### Step 3：匯出至 Cloud Storage（大規模/自動化）

```javascript
Export.image.toCloudStorage({
  image: imageToExport.toFloat(),
  description: 'NDVI_Taiwan_2023_30m_GCS',
  bucket: 'your-gcs-bucket-name',          // GCS bucket 名稱
  fileNamePrefix: 'outputs/NDVI_Taiwan_2023_30m',  // 路徑 + 檔名
  region: studyArea,
  scale: 30,
  crs: 'EPSG:3826',
  maxPixels: 1e9,
  fileFormat: 'GeoTIFF',
  formatOptions: { cloudOptimized: true, noData: -9999 }
});
```

### Step 4：匯出至 GEE Asset（供 GEE 內部複用）

```javascript
Export.image.toAsset({
  image: imageToExport,
  description: 'Store_NDVI_Composite_2023',
  assetId: 'users/YOUR_USERNAME/NDVI_Taiwan_2023',  // 完整 Asset 路徑
  region: studyArea,
  scale: 30,
  crs: 'EPSG:3826',
  maxPixels: 1e9
});
```

### Step 5：匯出統計結果（FeatureCollection → CSV）

```javascript
// 計算區域統計，結果存為 FeatureCollection
var statsFC = image.reduceRegions({
  collection: studyAreas,
  reducer: ee.Reducer.mean().combine({
    reducer2: ee.Reducer.stdDev(),
    sharedInputs: true
  }),
  scale: 30,
  crs: 'EPSG:3826'
});

// 匯出為 CSV
Export.table.toDrive({
  collection: statsFC,
  description: 'NDVI_Stats_by_Region_2023',
  folder: 'GEE_Exports',
  fileNamePrefix: 'NDVI_Stats_2023',
  fileFormat: 'CSV'   // 或 'GeoJSON', 'KML', 'SHP'
});
```

### Step 6：Python API 自動提交任務

```python
# 使用 Python Earth Engine API 自動執行（適合批次處理）
import ee
ee.Initialize()

task = ee.batch.Export.image.toDrive(
    image=image_to_export,
    description='NDVI_Taiwan_2023_30m',
    folder='GEE_Exports',
    fileNamePrefix='NDVI_Taiwan_2023_30m',
    region=study_area,
    scale=30,
    crs='EPSG:3826',
    maxPixels=1e9
)
task.start()

# 監控任務狀態
import time
while task.active():
    status = task.status()
    print(f"Status: {status['state']}")
    time.sleep(30)

print("Final status:", task.status()['state'])
```

### Step 7：自我驗證

提交任務後，確認以下事項：

```javascript
// 在 GEE Code Editor 中印出匯出設定摘要
print('=== Export Configuration Summary ===');
print('Scale (m):', 30);
print('CRS:', 'EPSG:3826');
print('Max pixels:', 1e9);
print('NoData value:', -9999);

// 估算輸出大小（粗略）
var area = studyArea.area(1);  // 單位：平方公尺
var estimatedPixels = area.divide(30 * 30);  // scale^2
print('Estimated pixel count:', estimatedPixels);
// 若超過 1e9，需調高 maxPixels 或縮小 region
```

---

## 檔名命名規範（建議）

```
格式：{資料集}_{研究區域}_{時間範圍}_{解析度}
範例：
  NDVI_Taiwan_2023_30m.tif
  CHIRPS_Rain_Taiwan_2020-2023_annual_5km.tif
  LandUse_Taichung_2022_10m.tif
  SampleStats_AllSites_2023.csv
```

---

## 常見匯出錯誤排查

| 錯誤訊息 | 原因 | 解決方式 |
|---------|------|---------|
| `Total request size ... exceeds limit` | maxPixels 不夠大 | 調高 `maxPixels: 1e10` |
| `Computed value is too large` | region 太大或 scale 太小 | 增大 scale 或縮小 region |
| `No bands in the image` | 匯出空影像 | 執行 Step 1 確認影像有波段 |
| Task 長時間 READY | GEE 伺服器排隊 | 等待，或分拆成多個小任務 |
| Drive 檔案分割（多個 .tif） | 超過單檔 4GB 限制 | GEE 自動分割，正常現象 |

---

## 輸出合約

產出的程式碼必須滿足以下條件：

- [ ] `scale`、`crs`、`region`、`maxPixels` 四個參數全部明確指定
- [ ] `description` 使用有意義的命名，包含資料集與時間資訊
- [ ] 影像在匯出前呼叫 `.toFloat()` 確保數值精度
- [ ] `formatOptions` 包含 `noData: -9999`
- [ ] 提供匯出前的 Step 1 預覽驗證程式碼
- [ ] 提供像素數量估算，確認 `maxPixels` 設定足夠

---

## 參考資源

- GEE Export 文件：https://developers.google.com/earth-engine/guides/exporting_images
- Cloud-Optimized GeoTIFF 說明：https://cogeo.org/
- Python Earth Engine API：https://developers.google.com/earth-engine/guides/python_install
