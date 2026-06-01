我想建立一個台灣本島 2018–2024 年每日降雨量資料庫，用於後續水文與極端降雨分析。

請你到 Google Earth Engine 找一個合適且可公開使用的資料集，不要先假設 dataset ID。請完成：

1. 說明你選擇資料集的理由，包含時間解析度、空間解析度、單位、可用時間範圍、是否適合台灣。
2. 寫出 Python / Earth Engine API 的資料讀取流程。
3. 將資料整理成 xarray Dataset，維度至少包含 time, y, x。
4. 輸出為 Zarr，請設計合理 chunk，例如以月份或季度為 time chunk。
5. 請保留 metadata：dataset_id、variable_name、unit、spatial_resolution、temporal_resolution、CRS、AOI、processing_date。
6. 請加入基本資料驗證：時間是否連續、是否有缺日、降雨量是否出現不合理負值。
