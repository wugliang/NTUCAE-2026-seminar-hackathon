我想建立 2021–2025 年台灣西部平原的 Sentinel-2 月尺度雲遮罩後植生指標資料庫，用於農地變化分析。

請你不要直接假設資料集，先在 Google Earth Engine 判斷最適合的 Sentinel-2 資料來源與雲遮罩策略。需求如下：

1. 輸出每個月的 NDVI、EVI、cloud_fraction、valid_pixel_count。
2. 需要盡量避免雲、雲影與高反射地物造成的錯誤。
3. 最終輸出為 Zarr，維度為 time, y, x, variable。
4. 時間單位為 monthly，空間解析度可由你根據資料品質與計算成本決定，但要說明理由。
5. 請說明如果某月份有效像素太少，應該如何標記或補處理。
6. 請提供 Python GEE + xarray/Zarr pipeline，並列出 metadata 與品質檢查欄位。
7. 請特別說明你是否使用 QA60、SCL、S2 cloud probability 或其他 cloud mask 方法，以及為什麼。
