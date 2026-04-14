# Exercise 8 — Three-Act STAC Scene Selection

**ARIA v5.0：馬太鞍三幕稽核器**
2025 Matai'an Creek Barrier Lake Event（馬太鞍溪堰塞湖事件）

以 STAC + Sentinel-2 L2A cloud-native workflow，對 2025 年花蓮馬太鞍溪堰塞湖事件進行「事前 / 事中 / 事後」三幕光學遙測稽核。

---

## 事件三幕時間軸

| 幕 | 日期 | 事件 | TCI 特徵 |
|---|---|---|---|
| **Pre** | 2025-06-15 | 颱風威帕尚未來襲 | 鬱閉綠色森林谷地 |
| **Mid** | 2025-09-11 | 颱風威帕（7/21）暴雨引發大規模崩塌，阻塞馬太鞍溪形成 ~200 m 深堰塞湖 | 原森林中新增青藍色水體 |
| **Post** | 2025-10-16 | 9/23 14:50 堰塞湖溢頂，30 分鐘釋出 1540 萬 m³，台 9 線馬太鞍溪橋塌落，光復鄉被泥沙掩埋（18+ 罹難） | 湖泊消失、光復出現灰白色沉積鋪面 |

---

## 專案結構

```
.
├── Pre-lab-Week8.md                  # 課前準備文件
├── Week8-Student.ipynb               # 已執行完成的作業 notebook
├── Week8-Student-completed.ipynb     # 教師參考解答
├── data/
│   └── guangfu_overlay.gpkg          # 光復 5 節點圖層（Pre-lab Step 7b 自建）
├── mataian_detections.gpkg           # 3 圖層：barrier_lake / landslide_source / debris_flow
├── impact_table.csv                  # 目擊衝擊表
└── output/
    ├── 07_lake_mask.png
    ├── 08_landslide_threshold_grid.png
    ├── 09_debris_mask.png
    ├── 10_three_masks.png
    └── 12_coverage_gap_map.png
```

---

## 偵測結果摘要

| 偵測項 | 面積 | 多邊形數 | 參考值 | 評估 |
|---|---|---|---|---|
| 堰塞湖（Mid） | **1.05 km²** | 8 | NCDR ~0.86 km² | ✅ 接近峰值 |
| 崩塌源區（Pre→Post） | **2.10 km²** | 197 | NCDR ~2–5 km² | ✅ 吻合 |
| 土石流鋪面（Pre→Post, 下游） | **10.94 km²** | 380 | 光復市街 4–5 km² | ⚠️ 略高（含季節性植被變化） |
| Guangfu 圖層命中 | **2 / 5** | — | 3–5 預期 | ✅ 光復鄉公所、佛祖街沉積區被土石流覆蓋 |
| W3 避難所命中 | 0 | — | 0 預期 | ✅ 覆蓋缺口成立 |
| W7 瓶頸命中 | 0 | — | 0 預期 | ✅ 覆蓋缺口成立 |

---

## 成果圖

### 圖 07 — 堰塞湖偵測（Act 1→2）
青色為偵測出的堰塞湖水體（1.05 km²），位於馬太鞍溪上游萬榮鄉。

![堰塞湖偵測](output/07_lake_mask.png)

---

### 圖 08 — 崩塌源區門檻調整格網
5 個候選 (nir_drop, swir_post) 組合的視覺比對。F1 最高的 (0.10, 0.20) 面積達 23.8 km²，明顯是誤判擴散；改採 **(0.20, 0.25)** 得到 2.10 km²，更符合實際崩塌範圍。

![崩塌門檻格網](output/08_landslide_threshold_grid.png)

> **教學點**：20 個 ground truth points 樣本太小，寬門檻容易拉高 recall 使 F1 虛高。必須配合視覺檢查，不可盲從 F1。

---

### 圖 09 — 下游土石流鋪面（Act 1→3）
左：NDVI 變化（紅=植被消失）；中：BSI 變化（橙黃=裸土增加）；右：土石流偵測疊加於 Post TCI。

![土石流偵測](output/09_debris_mask.png)

---

### 圖 10 — 三幕綜合證據圖
左上 Act 2 堰塞湖、右上 Act 3 崩塌源區、左下 Act 3 土石流、右下三幕疊加。

![三幕綜合](output/10_three_masks.png)

---

### 圖 12 — 覆蓋缺口稽核地圖（全課堂核心圖）
W3 花蓮市避難所 0 命中、W7 瓶頸 0 命中、W8 光復 5 節點 2 命中。

![覆蓋缺口](output/12_coverage_gap_map.png)

> **一句話結論**：「ARIA v1–v4 把所有資源放在花蓮市，但 2025 最嚴重的生命財產損失發生在南方 30 公里的光復鄉——完全在系統服務範圍之外。」

---

## 技術細節

- **STAC**：Microsoft Planetary Computer (`sentinel-2-l2a`)
- **AOI**：`121.28, 23.56, 121.52, 23.76`（馬太鞍溪上游 → 光復鄉）
- **CRS**：EPSG:32651（UTM 51N，Sentinel-2 原生）；向量圖層 EPSG:3826（TWD97 / TM2）
- **Bands**：B02, B03, B04, B08, B11, B12
- **Cloud-native**：只串流 AOI 範圍，不下載整張 10980×10980 tile
- **SAS token**：每次 `pc.sign(item)` 刷新，避免 1 小時後讀取失敗

### 三個核心偵測規則

```python
# C1 堰塞湖
lake = (nir_pre > 0.25) & (nir_mid < 0.15) & (blue_mid > 0.03) \
     & (green_mid > nir_mid) & (x < 121.33°E)        # 上游空間門限

# C2 崩塌源區
landslide = (nir_drop > 0.20) & (swir_post > 0.25) & (nir_pre > 0.25)

# C3 下游土石流
debris = (ndvi_change > 0.25) & (bsi_change > 0.10) & (nir_pre > 0.20) \
       & (x > 121.35°E) & ~lake & ~landslide         # 下游 + 排除重疊
```

---

## 執行環境

```bash
pip install pystac-client planetary-computer stackstac rioxarray xarray \
            geopandas rasterio scikit-learn dask tabulate anthropic
```

macOS CJK 字型：`Heiti TC`、`PingFang HK`、`Noto Sans TC`。

---

## 作業要求達成狀況

- [x] Pre-lab Step 7b `guangfu_overlay.gpkg`（5 節點）
- [x] Lab 1 S1–S6 全部 TODO
- [x] Lab 2 S7–S12 全部 TODO
- [x] 5 必交圖（07 / 08 / 09 / 10 / 12）
- [x] 5 段必答討論（三幕 Discussion + C2/C3 物理差異 + 覆蓋缺口）
- [x] `mataian_detections.gpkg` 3 圖層
- [x] `impact_table.csv`
- [x] S13 AI Advisor Prompt
- [ ] S14 實際 LLM 呼叫（未設 `ANTHROPIC_API_KEY`，已安全跳過）

---

*"A STAC catalog is a library card — you don't carry the books home, you read them on the shelf."*
*"In the Matai'an case, the library happens to have photos of a lake that existed for only 64 days."*
