我想建立一個台灣 2020–2024 年 hourly 太陽能與氣象 forcing Zarr，供後續 PV potential / GHI 模型使用。

請你到 Google Earth Engine 找最適合的再分析資料集，並設計一個可擴充的 Zarr pipeline。需求如下：

1. 至少包含：
   - 2m air temperature
   - total precipitation
   - downward surface solar radiation 或可代表 GHI 的變數
   - 10m u/v wind
   - dew point 或可推相對濕度的變數
2. 請說明每個 band 的原始單位、是否為 instantaneous、accumulated、或 disaggregated hourly variable。
3. 如果輻射或降雨是累積量，請明確說明如何轉成 hourly W/m² 或 hourly precipitation depth。
4. 請設計 Zarr schema：
   - dimensions
   - variables
   - dtype
   - chunks
   - compressor
   - coordinate convention
   - metadata
5. 請說明直接用 GEE export GeoTIFF 再轉 Zarr、以及用 Xee 讀取 ImageCollection 再 to_zarr，兩種路線的優缺點。
6. 請加入品質檢查：
   - time index 是否完整
   - UTC / local time 是否清楚
   - radiation 夜間是否合理
   - precipitation 是否非負
   - temperature 是否仍為 K 或已轉 °C
7. 請不要只給概念，請提供可以實作的 Python pipeline skeleton。
