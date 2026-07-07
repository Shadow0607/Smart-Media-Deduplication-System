# 智慧媒體去重與相簿整理系統架構紀錄

## 1. 專案定位

本專案是一套本機端桌面應用程式，用於大量照片、短影音與長影音的重複檢測、相似媒體分組與安全清理。

系統支援：

* 照片去重
* 短影音去重
* 長影音去重
* 人臉輔助判斷
* 縮圖快取
* SQLite 索引
* 背景掃描
* 安全刪除至系統回收桶

核心定位：

> 智慧媒體去重與相簿整理系統
> Smart Media Deduplication System

---

## 2. 核心技術選型

| 類別         | 技術                         | 用途                         |
| ---------- | -------------------------- | -------------------------- |
| Desktop UI | PySide6                    | 建立專業桌面介面                   |
| 圖片處理       | Pillow                     | 讀取圖片、產生縮圖                  |
| 圖片雜湊       | imagehash                  | pHash / dHash 相似圖片比對       |
| 影片處理       | FFmpeg / FFprobe           | 取得影片 metadata、抽取影片資訊       |
| 影片抽幀       | PyAV / OpenCV              | 從影片抽取 keyframes            |
| 人臉辨識       | InsightFace / ArcFace ONNX | 人臉 embedding 與人物相似判斷       |
| 推論後端       | ONNX Runtime               | 執行 ArcFace 模型              |
| 資料庫        | SQLite                     | 儲存媒體索引、hash、embedding、分組結果 |
| 背景任務       | QThread / QThreadPool      | 避免 UI 卡頓                   |
| 縮圖快取       | Local Cache Folder         | 避免大量原圖載入記憶體                |
| 安全刪除       | send2trash                 | 移至系統回收桶，避免永久刪除             |

---

## 3. 整體系統架構

```text
PySide6 Desktop App
│
├── UI Layer
│   ├── 掃描頁
│   ├── 照片重複群組頁
│   ├── 影片重複群組頁
│   ├── 人物分群頁
│   └── 確認清理頁
│
├── Scan Manager
│   ├── File Scanner
│   ├── Image Scanner
│   ├── Video Scanner
│   ├── Thumbnail Worker
│   ├── Face Analysis Worker
│   └── Progress Reporter
│
├── Index Database
│   ├── images
│   ├── videos
│   ├── video_frames
│   ├── faces
│   ├── duplicate_groups
│   └── face_clusters
│
├── Matching Engine
│   ├── Exact Duplicate Engine
│   ├── Similar Image Engine
│   ├── Short Video Matching Engine
│   ├── Long Video Matching Engine
│   ├── Face Similarity Engine
│   └── Final Scoring Engine
│
└── Cleanup Engine
    ├── Smart Auto Select
    ├── Delete Preview
    └── send2trash
```

---

## 4. 媒體類型分類

系統主要處理三種媒體：

```text
image
short_video
long_video
```

### 4.1 照片

判斷依據：

* 副檔名
* MIME type
* 圖片 metadata

常見格式：

```text
jpg, jpeg, png, heic, webp, bmp, tiff
```

### 4.2 短影音

建議定義：

```text
duration <= 60 秒
```

用途：

* LINE 影片
* IG Reels
* TikTok
* YouTube Shorts
* 手機相簿短片
* 社群轉存影片

### 4.3 長影音

建議定義：

```text
duration > 10 分鐘
```

用途：

* 活動錄影
* 會議錄影
* Vlog
* 教學影片
* 長時間手機錄影

### 4.4 中等影片

可選擇性保留一個中間類別：

```text
60 秒 < duration <= 10 分鐘
```

系統中可先歸類為 `short_video` 或獨立為 `medium_video`。

---

## 5. 去重邏輯總覽

不同媒體類型使用不同判斷方式。

```text
照片：
MD5 + pHash + EXIF + ArcFace

短影音：
MD5 + metadata + keyframe hash + audio fingerprint + ArcFace sampled frames

長影音：
MD5 + metadata + segment keyframe fingerprint + audio fingerprint
```

---

## 6. 照片去重流程

```text
1. 掃描圖片檔案
2. 取得檔案大小、修改時間、解析度
3. 計算 MD5
4. 計算 pHash / dHash
5. 產生縮圖
6. 寫入 SQLite
7. 使用 MD5 找完全重複
8. 使用 pHash 找相似照片
9. 使用 EXIF 時間與比例做分桶過濾
10. 對候選照片執行 ArcFace
11. 建立重複群組
12. 給出刪除建議
```

### 6.1 完全重複

完全重複主要依靠：

```text
MD5 相同
```

特性：

* 判斷快速
* 準確度高
* 適合找完全一樣的檔案

### 6.2 相似照片

相似照片主要依靠：

```text
pHash / dHash 漢明距離
```

可設定門檻：

```text
嚴格：Hamming distance <= 4
普通：Hamming distance <= 8
寬鬆：Hamming distance <= 12
```

### 6.3 人臉輔助

ArcFace 不取代 pHash，而是輔助判斷：

```text
pHash：判斷照片畫面是否相似
ArcFace：判斷照片中的人是否相同
```

用途：

* 避免誤刪不同人物但背景相似的照片
* 協助連拍照片選出最佳版本
* 支援人物分群
* 增加智慧相簿整理能力

---

## 7. 短影音去重流程

短影音通常長度短、畫面變化快，可以使用較密集的抽樣。

```text
1. 掃描影片
2. 使用 ffprobe 取得 metadata
3. 判斷 duration 是否屬於 short_video
4. 產生影片縮圖
5. 每 1 秒抽 1 幀
6. 每個抽樣 frame 計算 pHash / dHash
7. 抽取 audio fingerprint
8. 若有候選重複組，再執行 ArcFace
9. 計算影片相似度
10. 建立短影音重複群組
```

短影音相似度可設計為：

```text
video_similarity =
    frame_hash_similarity * 0.60
  + audio_similarity      * 0.25
  + metadata_similarity   * 0.15
```

若加入人臉：

```text
video_similarity =
    frame_hash_similarity * 0.45
  + audio_similarity      * 0.20
  + face_similarity       * 0.25
  + metadata_similarity   * 0.10
```

---

## 8. 長影音去重流程

長影音不能逐秒抽幀，否則成本過高，應採用分段抽樣。

```text
1. 掃描影片
2. 使用 ffprobe 取得 duration、fps、resolution、codec、bitrate
3. 判斷為 long_video
4. 建立影片縮圖
5. 依照時間比例抽樣
6. 每 30 秒或 60 秒抽 1 幀
7. 計算每個抽樣 frame 的 pHash
8. 抽取 audio fingerprint
9. 比對 duration、解析度、音訊與 keyframe fingerprint
10. 判斷是否為完整重複、相似版本或片段重複
```

建議抽樣策略：

```text
- 影片開頭 5%
- 影片中段 50%
- 影片結尾 95%
- 每 30 秒或 60 秒抽一幀
- 若有 scene detection，可抽場景切換點
```

---

## 9. ArcFace 模組定位

ArcFace 的定位是：

```text
人臉相似輔助模組
```

不建議作為主要去重依據。

### 9.1 適合使用 ArcFace 的情境

* 人像照片去重
* 連拍照片篩選
* 人物分群
* 婚禮、聚會、家庭相簿整理
* 判斷兩張相似照片中的人物是否相同
* 判斷短影音中的主角是否相同

### 9.2 不適合單獨使用 ArcFace 的情境

* 風景照
* 食物照
* 文件截圖
* 商品照片
* 無人臉影片
* 純畫面重複判斷

### 9.3 ArcFace 執行時機

建議只在候選組上執行：

```text
先用 MD5 / pHash / metadata 找候選重複
再對候選照片或影片 frame 執行 ArcFace
```

避免對全部媒體暴力跑人臉模型。

---

## 10. 效能優化策略

### 10.1 Blocking / Bucketing

避免所有媒體兩兩比對。

可使用條件：

```text
- 檔案大小接近
- 解析度相同或接近
- EXIF 時間接近
- 長寬比相同
- duration 接近
- codec 相同
- bitrate 接近
```

### 10.2 Thumbnail Caching

所有圖片與影片只在 UI 中顯示縮圖，不直接載入原始檔。

建議縮圖大小：

```text
128x128
256x256
```

快取位置：

```text
cache/thumbnails/
```

### 10.3 Background Worker

所有重工作業都必須放到背景執行緒：

```text
- 檔案掃描
- MD5 計算
- pHash 計算
- 影片 metadata 讀取
- 影片抽幀
- ArcFace 推論
- SQLite 寫入
```

避免 PySide6 UI 顯示「沒有回應」。

---

## 11. SQLite 資料庫設計

### 11.1 images

```sql
CREATE TABLE images (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT UNIQUE NOT NULL,
    file_size INTEGER,
    width INTEGER,
    height INTEGER,
    aspect_ratio REAL,
    created_time TEXT,
    modified_time TEXT,
    exif_datetime TEXT,
    md5 TEXT,
    phash TEXT,
    dhash TEXT,
    thumbnail_path TEXT,
    scanned_at TEXT
);
```

### 11.2 videos

```sql
CREATE TABLE videos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT UNIQUE NOT NULL,
    file_size INTEGER,
    md5 TEXT,
    duration REAL,
    width INTEGER,
    height INTEGER,
    fps REAL,
    codec TEXT,
    bitrate INTEGER,
    created_time TEXT,
    modified_time TEXT,
    media_type TEXT,
    thumbnail_path TEXT,
    scanned_at TEXT
);
```

### 11.3 video_frames

```sql
CREATE TABLE video_frames (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    video_id INTEGER NOT NULL,
    timestamp REAL,
    frame_index INTEGER,
    phash TEXT,
    dhash TEXT,
    thumbnail_path TEXT,
    FOREIGN KEY(video_id) REFERENCES videos(id)
);
```

### 11.4 faces

```sql
CREATE TABLE faces (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    media_id INTEGER NOT NULL,
    media_type TEXT NOT NULL,
    timestamp REAL,
    face_index INTEGER,
    bbox_x INTEGER,
    bbox_y INTEGER,
    bbox_w INTEGER,
    bbox_h INTEGER,
    confidence REAL,
    embedding BLOB,
    cluster_id INTEGER
);
```

### 11.5 duplicate_groups

```sql
CREATE TABLE duplicate_groups (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_type TEXT,
    media_type TEXT,
    confidence REAL,
    reason TEXT,
    created_at TEXT
);
```

### 11.6 duplicate_group_items

```sql
CREATE TABLE duplicate_group_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id INTEGER NOT NULL,
    media_id INTEGER NOT NULL,
    media_type TEXT NOT NULL,
    similarity_score REAL,
    recommended_action TEXT,
    reason TEXT,
    FOREIGN KEY(group_id) REFERENCES duplicate_groups(id)
);
```

### 11.7 face_clusters

```sql
CREATE TABLE face_clusters (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    representative_face_id INTEGER,
    face_count INTEGER,
    created_at TEXT
);
```

---

## 12. UI/UX Layout

系統介面建議分為五個主要頁面。

---

### 12.1 掃描頁

功能：

* 選擇相簿路徑
* 選擇是否包含子資料夾
* 選擇掃描媒體類型
* 設定去重嚴格度
* 啟用或停用 ArcFace
* 顯示掃描進度

建議選項：

```text
[ ] 掃描照片
[ ] 掃描短影音
[ ] 掃描長影音
[ ] 啟用 ArcFace 人臉輔助
[ ] 僅對候選重複組執行 ArcFace
[ ] 建立人物分群
```

---

### 12.2 照片重複群組頁

顯示資訊：

```text
- 縮圖
- 檔案大小
- 解析度
- 修改日期
- pHash 距離
- 人臉相似度
- 建議保留 / 建議刪除
```

功能：

```text
- 群組切換
- 圖片放大預覽
- 智慧自動勾選
- 手動勾選刪除
```

---

### 12.3 影片重複群組頁

顯示資訊：

```text
- 影片縮圖
- 影片長度
- 解析度
- FPS
- Codec
- Bitrate
- 畫面相似度
- 音訊相似度
- 人臉相似度
- 建議保留 / 建議刪除
```

影片分類：

```text
- 完全重複
- 高度相似
- 片段重複
- 畫面相似但音訊不同
- 音訊相同但畫面不同
```

---

### 12.4 人物分群頁

功能：

```text
- 顯示人物群組
- 顯示每個人物出現的照片與影片
- 支援重新命名人物
- 支援合併人物群組
- 支援排除錯誤人臉
```

---

### 12.5 確認清理頁

功能：

```text
- 顯示所有準備刪除的媒體
- 顯示刪除原因
- 顯示保留版本
- 最後確認
- 使用 send2trash 移至回收桶
```

重要原則：

```text
不直接永久刪除檔案
```

---

## 13. 智慧自動勾選規則

### 13.1 照片保留優先順序

```text
1. 解析度較高
2. 檔案品質較高
3. 檔案大小較合理
4. 修改時間較新
5. 有 EXIF 原始資訊
6. 人臉較清楚
7. 非截圖或壓縮轉傳版本
```

### 13.2 影片保留優先順序

```text
1. duration 較完整
2. 解析度較高
3. bitrate 較高
4. fps 較高或較穩定
5. codec 較通用
6. 檔案較新
7. 無浮水印或裁切較少
8. 人臉較清楚
```

### 13.3 不應自動刪除的情況

```text
- duration 差很多
- 畫面相似但音訊不同
- 音訊相同但畫面不同
- pHash 相似但 ArcFace 判斷人物不同
- 解析度差異大但內容不完全一致
- 長影音疑似只包含短片段重複
```

這些情況應標記為：

```text
需要人工確認
```

---

## 14. 重複群組類型

建議群組類型：

```text
exact_duplicate
similar_image
short_video_duplicate
long_video_duplicate
partial_video_duplicate
same_person_group
manual_review_required
```

說明：

| 群組類型                    | 說明                 |
| ----------------------- | ------------------ |
| exact_duplicate         | MD5 完全相同           |
| similar_image           | pHash / dHash 高度相似 |
| short_video_duplicate   | 短影音內容高度相似          |
| long_video_duplicate    | 長影音整體高度相似          |
| partial_video_duplicate | 長影音或短影音片段重複        |
| same_person_group       | 同一人物媒體群組           |
| manual_review_required  | 需要人工確認             |

---

## 15. 建議開發順序

### 第一版：照片基礎去重

```text
- PySide6 基礎 UI
- 選擇資料夾
- 掃描照片
- MD5
- pHash
- SQLite
- 縮圖快取
- 照片重複群組
- send2trash
```

### 第二版：影片 metadata 與縮圖

```text
- ffprobe 讀取影片資訊
- duration 判斷短影音 / 長影音
- 影片縮圖
- videos 資料表
- 影片列表 UI
```

### 第三版：短影音去重

```text
- 短影音抽幀
- frame pHash
- 短影音相似度計算
- 短影音重複群組
```

### 第四版：長影音去重

```text
- 長影音分段抽樣
- 長影音 fingerprint
- audio fingerprint
- 片段重複判斷
```

### 第五版：ArcFace 人臉輔助

```text
- InsightFace / ArcFace ONNX
- 人臉偵測
- face embedding
- faces 資料表
- 人臉相似度
- 人物分群
```

### 第六版：智慧清理與完整 UI

```text
- 智慧自動勾選
- 刪除風險提示
- 人工確認頁
- 完整錯誤處理
- 掃描暫停 / 繼續
- 匯出報告
```

---

## 16. 核心設計原則

### 16.1 不暴力兩兩比對

避免：

```text
N x N 暴力比對
```

應採用：

```text
Blocking / Bucketing / Hash Index / Candidate Filtering
```

---

### 16.2 不直接載入原圖

UI 只讀：

```text
cache thumbnails
```

需要放大或刪除時才讀取原始檔。

---

### 16.3 不阻塞 UI

所有重工作業必須在背景執行緒。

---

### 16.4 不直接永久刪除

所有刪除都透過：

```text
send2trash
```

移至系統回收桶。

---

### 16.5 ArcFace 只作為輔助

ArcFace 不應取代 pHash / video fingerprint，而是用於降低誤刪風險。

---

## 17. 後期可擴充功能

```text
- 支援 HEIC
- 支援 RAW
- 支援 NAS 掃描
- 支援多資料夾索引
- 支援重複檔案報告匯出
- 支援人物命名
- 支援相簿整理建議
- 支援相似影片片段定位
- 支援 GPU 加速
- 支援任務暫停與恢復
- 支援索引增量更新
```

---

## 18. 一句話總結

本系統以 SQLite 作為媒體索引核心，結合 MD5、pHash、影片 keyframe fingerprint、audio fingerprint 與 ArcFace 人臉 embedding，在 PySide6 桌面介面中提供照片、短影音與長影音的智慧去重、人物輔助判斷與安全清理功能。
