# 新聞後台上稿 API（admin.news.tvbs-internal.com.tw）

TVBS 新聞後台上稿系統的 REST API 拓測結果。透過瀏覽器登入 session 直接呼叫，
不需模擬 UI 點擊。2026-08-05 用 Claude in Chrome 實際建立一篇測試草稿驗證過。

## 前置
- 起始網址：`https://admin.news.tvbs-internal.com.tw/`
- 需已在瀏覽器登入（Google SSO），API 皆用瀏覽器 cookie 認證，同源呼叫即可
- API base：`https://admin.news.tvbs-internal.com.tw/api/v1/`

## 讀取類 API（GET，安全）

| 用途 | Endpoint |
|---|---|
| 文章列表 | `GET /articles?limit=&page=&start_at=&end_at=`（Unix timestamp 區間） |
| 單篇文章詳細 | `GET /articles/{id}` |
| 分類 | `GET /categories`、`GET /categories/{id}/subcategories` |
| 來源 | `GET /source/article`、`GET /source/image` |
| 專欄作家 | `GET /columnists` |
| 參與人員 credits（人名+角色組合，回傳含 `credit_id`） | `GET /credits` |
| 警語標籤 | `GET /warnings` |
| 發佈通路 | `GET /disseminations?status=1` |
| 目前登入使用者 | `GET /auth/user` |

## 建立文章（POST，真實動作）

```
POST /api/v1/articles
```

用瀏覽器已登入分頁的 JS context 直接呼叫最省事（不需自己組 cookie）。
建立後狀態為「草稿」（`status: 0`），需要另外送審/排程/發布才會真正曝光。

### 必填欄位（UI 端會擋，繞過 UI 直接打 API 未驗證是否服務端也強制檢查）
- `title`（長標題，≤60字）
- `short_title`（短標題，≤40字）
- `content`（**Lexical 編輯器序列化 JSON 字串**，不是純文字或 HTML，見下方範例）
- `source_article_id`（來源，例如「記者自製」對應的 id，從 `GET /source/article` 查）
- `category_id`（分類 id，從 `GET /categories` 查；例如「生活」= 1）
- `hashtags`（標籤陣列，至少 1 個，例如 `["AI測試"]`）
- `credits`（參與人員陣列，每項需先知道 `credit_id`；`credit_id` 似乎是「角色+人名」的組合索引，
  無法只傳角色代碼+姓名字串，要先從既有文章或介面操作反查對應的 `credit_id`）

### content 欄位格式範例（Lexical JSON）

```json
{
  "root": {
    "children": [
      {
        "children": [
          {
            "detail": 0,
            "format": 0,
            "mode": "normal",
            "style": "",
            "text": "這是內文的一段文字。",
            "type": "text",
            "version": 1
          }
        ],
        "direction": "ltr",
        "format": "",
        "indent": 0,
        "type": "paragraph",
        "version": 1,
        "textFormat": 0,
        "textStyle": ""
      }
    ],
    "direction": "ltr",
    "format": "",
    "indent": 0,
    "type": "root",
    "version": 1
  }
}
```
`content` 存進資料庫時是這個結構 **序列化成字串**（見下方 GET 回傳範例）。

### 選填欄位
- `subcategory_id`（副分類）
- `columnist_id`（指定專欄）
- `featured_image` / `youtube_id` / `featured_type`（主圖或 YouTube 影片）
- `warnings`（警語陣列）
- `faq`、`citations`、`related_articles`
- `scheduled_at`（排程發布時間）
- `dissemination_id`（發佈通路）

### 實測：建立成功後 `GET /articles/{id}` 回傳範例

```json
{
  "title": "【AI測試】DenisAutoAPI拓測草稿",
  "short_title": "AI拓測草稿",
  "content": "{\"root\":{\"children\":[{\"children\":[{\"detail\":0,\"format\":0,\"mode\":\"normal\",\"style\":\"\",\"text\":\"...\",\"type\":\"text\",\"version\":1}],\"direction\":\"ltr\",\"format\":\"\",\"indent\":0,\"type\":\"paragraph\",\"version\":1,\"textFormat\":0,\"textStyle\":\"\"}],\"direction\":\"ltr\",\"format\":\"\",\"indent\":0,\"type\":\"root\",\"version\":1}}",
  "category_id": 1,
  "subcategory_id": null,
  "source_article_id": 3,
  "status": 0,
  "hashtags": ["AI測試"],
  "credits": [{"credit_id": 4, "name": "許岱軒"}]
}
```

## 使用者固定慣用值（每篇稿件預設）

- 記者：許岱軒（`credit_id: 1`）
- 責任編輯：新聞中心（`credit_id: 5`）
- 分類：國際（`category_id: 4`）
- 來源：SOT（`source_article_id: 2`）

以上為 2026-08-05 實測稿號 4002520 存檔後、`GET /articles/4002520` 回傳確認的真實 id 對應，
之後產文預設就套這組（除非使用者另外指定）。

## 圖片：圖庫選圖（推薦，全走 API 不用碰本機檔案）

點「搜尋圖片」→ 關鍵字查詢圖庫 → 選一張 → 跳出「編輯圖片」彈窗（裁切/來源/浮水印）→ 完成。

1. `GET /api/v1/images/gallery?keyword=關鍵字&page=1&per_page=100` — 關鍵字圖庫搜尋，回傳圖片清單
2. 選圖後跳出「編輯圖片」彈窗：來源下拉、備註、浮水印（不使用／指定位置）、16:9 或 4:3 裁切
3. 點「完成」後套用到主圖，`圖片說明` 欄位會自動帶入 `（圖／{來源顯示文字}）`

## 圖片：本機檔案上傳

點「上傳檔案」會觸發瀏覽器原生 `<input type=file>` 選檔對話框——**這是作業系統層級的彈窗，
瀏覽器自動化工具完全碰不到**，必須靠瀏覽器擴充功能提供的「直接把檔案路徑塞進 file input」
機制才能繞過（Claude in Chrome 是 `file_upload` 工具，只能餵入使用者已分享給該 session 的檔案路徑）。

選檔後的完整流程（2026-08-05 實測拓出）：

1. 選檔後瀏覽器立刻用 **AWS S3 Presigned URL** 直接把檔案 PUT 上傳到暫存路徑
   （`https://t-news-news-images-tvbs-com-tw.s3.ap-northeast-1.amazonaws.com/temp/{uuid前綴}/{uuid}.{ext}?X-Amz-...`），
   presigned URL 是哪支 admin API 發的沒有捕捉到（發生在選檔瞬間，來不及攔截）
2. 跳出「上傳圖片」彈窗：
   - **來源**（必填下拉）：`TVBS`／`網新記者`／`AI 生成`／`中央社`／`香港01`／`網路溫度計`／
     `達志影像`／`美聯社`／`路透社`／`News1`／`其他`
   - **備註**（選填文字框）：攝影師、其他資訊等
   - **浮水印**：`不使用` 或 `指定位置`（指定位置有左上/右上/左下/右下 4 個角落按鈕）
   - 裁切比例 16:9 / 4:3
3. 點「完成」後呼叫：
   `GET /api/v1/images/process?uuid={uuid}&extension={ext}&width=&height=&crop_aspect_ratio_type=16x9&x=&y=&right=&bottom=&watermark_position=lb`
   （`watermark_position` 值對應角落，左下實測是 `lb`），這支把暫存圖裁切/加浮水印處理成可用版本
4. 圖片最終網址走 `news-images.tvbs.com.tw/api/v1/image/temp/{uuid}.{ext}?...`（temp 路徑，
   推測文章正式儲存後才會轉正式路徑，未驗證）

**限制**：圖片高度不得小於 600px（前端擋，未達標會直接跳錯誤「不得小於 600 高度」不會進到上傳流程）。

**已知落差**：2026-08-05 測試時最後一步「套用到主圖」沒有成功反映在畫面上（可能是點擊座標因彈窗版面
跳動而偏移，不是 API 本身失敗），S3 直傳 + `images/process` 這兩步的 API 呼叫本身已確認會正常執行成功。

**內文中插入圖片**：使用者指出內文編輯器工具列的「插入圖片」跳出的是同一個上傳/圖庫彈窗，
機制應該相同，但這次沒有實際點開驗證（點擊「插入」下拉選單那次操作沒有成功開啟選單），待補。

## 延伸閱讀

點「延伸閱讀」的「新增」→ 跳出文章選擇彈窗（跟文章列表同一套篩選 UI：區間/來源/關鍵字/稿號）→
勾選文章 → 確定。

- 開啟時會打 `GET /api/v1/more-news?limit=&page=&start_at=&end_at=&exclude_article_id={目前文章id}`
  （排除自己），用來列出可選文章
- 選定後存檔，`related_articles` 欄位會存成 `[{id, article_id, title, article_url, sort}]`

## AI 自動生成功能

兩個「自動」按鈕都是非同步 AI 生成，呼叫後要等幾秒（實測約 3~5 秒）才會有結果，呼叫時網路
請求會先顯示 `pending` 再變 `200`。

| 功能 | Endpoint | 效果 |
|---|---|---|
| 標籤自動生成（標籤區塊的「自動」） | `POST /api/v1/ai/hashtags:generate` | 依內文分析結果**取代**現有標籤（不是附加） |
| 社群&SEO 自動生成（社群&SEO 區塊的「自動」） | `POST /api/v1/ai/og-description:generate` | 一次填入：社群標題（=長標題）、社群摘述、社群圖片（=主圖）、SEO 標題（=長標題）、SEO 摘述；內文很短時摘述會直接回傳原內文，沒有真的做摘要壓縮 |

呼叫這兩支時不需自己帶內文參數，應該是後端直接讀該篇文章目前存的內容，未逐一驗證 request body。

## 更新文章（PUT，2026-08-05 實測成功）— 幾乎所有欄位都能純 API 寫入

```
PUT /api/v1/articles/{id}
```

**關鍵發現**：後台「儲存」按鈕背後就只打這一支 PUT。也就是說原本以為「只能用 UI 操作」的
議題包、延伸閱讀、圖說、社群&SEO，**全部都是包在這支 PUT 的 body 裡送出的，不是各自獨立的 API**。
只要組好完整 body 就能純 API 完成整篇稿的所有欄位。

### 完整 body 欄位（實測 200 成功）

```js
{
  title, short_title, content,          // content 是 Lexical JSON 字串
  source_article_id, category_id,
  featured_type: "image",               // 必填
  is_explicit: false,                   // 必填
  display_rss: true,                    // 必填
  hashtags: ["標籤1","標籤2"],
  credits: [{credit_id: 1, name: "許岱軒"}, {credit_id: 5, name: "新聞中心"}],
  featured_image: {url, caption, alt},  // caption/alt 就是「圖片說明」，可直接改
  further_topic_id: 636,                // 議題包！636 = 全球必讀
  related_articles: [{related_article_id: 4002405, sort: 1}, ...],  // 延伸閱讀
  metadata: {og_title, og_description, og_image, meta_title, meta_description},  // 社群&SEO
  dissemination_id: 1
}
```

### 踩雷提醒
- `related_articles` 寫入時欄位名是 **`related_article_id`**（不是 `article_id`）；
  但 `GET` 回傳時欄位名是 `article_id`，兩邊名稱不一致，直接把 GET 結果丟回 PUT 會噴
  422 `The related_articles.0.related_article_id field is required`。
- 建立（POST）時 `featured_type`/`is_explicit`/`display_rss` 三個欄位都是必填，漏了會依序噴 422。
- 更新時建議先 `GET /articles/{id}` 抓現況，改想改的欄位後整包 PUT 回去（等同覆蓋式更新）。

### 議題包清單 API

```
GET /api/v1/topics?limit=&page=       // 議題包列表（注意不是 further-topics，那支是 404）
```
文章上存的是 `further_topic_id`；`GET /articles/{id}` 回傳的 `further_topic` 物件含 `{id, title, cover_image}`。
「全球必讀」的 id 是 **636**。

### 內文插圖的 Lexical 節點格式

內文圖片在 `content` 的 Lexical JSON 裡是一個 `type: "image"` 的節點，和 `paragraph` 節點並列在
`root.children`，欄位為：

```json
{
  "type": "image", "version": 1,
  "src": "圖片URL", "altText": "...", "figcaption": "圖說文字",
  "source": "來源", "width": 0, "height": 0, "maxWidth": 0, "showAltEditor": false
}
```
所以**內文插圖與其圖說也能純 API 寫入**，只要把這個節點插進 `content` 的 children 陣列
（固定放在第 1 個和第 2 個 paragraph 之間）即可，不需要操作 Lexical 編輯器 UI。
唯一還是得靠瀏覽器的只有「把本機檔案變成圖片 URL」這一步（S3 上傳）。

## 發稿檢查清單（踩過的雷，2026-08-05 補充）

實際發一篇稿（稿號 4002543）時漏掉/做錯的地方，之後每次發稿都要對照這份清單：

- [ ] **議題包**：不能漏選。找不到精確對應主題的議題包時（例如查了關鍵字「SpaceX」「馬斯克」
  「美股」「財報」都沒有現成議題包），**預設 fallback 選「全球必讀」**，不要留空。
- [ ] **延伸閱讀**：不能只選 1～2 篇，**至少要選 10 篇**。用相關關鍵字（人名/主題）搜尋既有文章，
  同一頁通常有 10～12 篇可選，直接整頁勾選即可湊到門檻。
  - 注意：重新打開「延伸閱讀」新增彈窗時，已選清單會顯示「已選擇 0 篇」（看起來像重置），
    但實際點確定後會跟原本已存的延伸閱讀**合併去重**，不會覆蓋掉舊的，可以放心追加。
  - 彈窗內選取的筆數如果超過系統上限（實測上限似乎是 10），超過的項目存檔後不會有編號、
    不會真的生效，所以抓 10～11 篇送出最保險。
- [ ] **圖片說明（圖說）**：如果圖片來源檔名本身已經寫好完整圖說（例如
  `SpaceX上市後首度公布營收，優於預期，星鏈與人工智慧事業成長強勁。（圖／達志影像美聯社）.jpg`），
  **必須整段照抄檔名內容**，包含來源後綴的完整寫法（例如「達志影像美聯社」，不能自己簡化成
  「達志影像」）。系統選完來源下拉選單後會自動帶入 `（圖／{來源顯示文字}）` 這種簡短版本，
  這只是預設值，**要手動覆蓋成檔名裡的完整版本**，主圖和內文插圖兩處的圖說都要改，不能只改一處。
- [ ] 內文中插入圖片時，圖片下方系統一樣會自動帶一行簡短圖說，也要比照上一條手動改成完整版本。
- [ ] **內文插圖固定位置**：圖片固定放在**第一段與第二段中間**（不是文章開頭或結尾），
  寫內文時可以先用一段純文字佔位（例如 `__IMAGE_PLACEHOLDER__`）標記這個位置，之後再替換成圖片。

## API 可行性總表（2026-08-05 完整探索後結論）

| 項目 | 純 API 可行？ | 方式 |
|---|---|---|
| 建立文章（全欄位） | ✅ | `POST /api/v1/articles` |
| 更新文章（全欄位） | ✅ | `PUT /api/v1/articles/{id}` |
| 標題／短標題／內文文字 | ✅ | POST/PUT 的 `title`/`short_title`/`content` |
| 來源／分類／參與人員 | ✅ | `source_article_id`/`category_id`/`credits` |
| 標籤 | ✅ | `hashtags`；或 `POST /ai/hashtags:generate` 讓 AI 生 |
| 社群&SEO | ✅ | `metadata`；或 `POST /ai/og-description:generate` 讓 AI 生 |
| **圖片說明（圖說）** | ✅ | `featured_image.caption` / `.alt`（原本誤判為 UI-only） |
| **議題包** | ✅ | `further_topic_id`；清單查 `GET /topics`（原本誤判為 UI-only） |
| **延伸閱讀** | ✅ | `related_articles: [{related_article_id, sort}]`（原本誤判為 UI-only） |
| **內文插圖（含圖說）** | ✅ | 在 `content` 的 Lexical JSON 插入 `type:"image"` 節點（原本誤判為 UI-only） |
| **存檔** | ✅ | 就是 PUT 本身，不需要點 UI 按鈕（原本誤判為 UI-only） |
| 圖庫選圖 | ✅ | `GET /images/gallery?keyword=` 查到圖片 URL 後直接填進欄位 |
| **本機檔案上傳** | ❌ | 唯一真正需要瀏覽器的一步：檔案要經由 `<input type=file>` 才能上到 S3。<br>需靠 claude-in-chrome 的 `file_upload` 工具塞路徑繞過原生對話框，<br>接著才是 S3 presigned PUT + `GET /images/process` |
| 送審／排程／發布 | ❓ | 仍未拓測，狀態機 API 未知 |

**結論**：整篇稿件除了「把本機圖檔變成圖片 URL」這一步之外，其餘全部都能純 API 完成。
如果圖片改用圖庫既有圖（`GET /images/gallery`），則可以做到 100% 純 API 發稿。

## 尚未拓測（下一步待補）

- 送審（草稿→審核中）：狀態變更 API，未拓測
- 排程/發布：`scheduled_at` 欄位與/或另一支狀態切換 API，未拓測
- 主圖上傳的 presigned URL 是哪支 API 發的（發生在選檔瞬間，來不及攔截）
- `credit_id` 完整對照表（角色 × 人名 → id）未列全，需要時查 `GET /credits`
- 常見問題（`faq`）、引用（`citations`）、警語（`warnings`）的實際存檔格式未拓測

## 注意事項

- 建立文章、送出上傳圖片、AI自動生成後存檔都是**真實動作**，會在正式後台留下真實紀錄
- 2026-08-05 測試建立的稿號 **4002520**（【AI測試】DenisAutoAPI拓測草稿）目前留著沒清除，
  後台介面未找到刪除按鈕，使用者已知情並選擇先留著
