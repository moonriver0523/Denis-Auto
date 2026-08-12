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

**2026-08-12 更正**：整個編輯頁面**只有標籤區塊有一顆「自動」按鈕**，社群&SEO 區塊沒有對應的
UI 按鈕（用 `read_page` 對整頁 DOM 做過完整掃描確認，全站唯一一個 `button "自動"` 就是標籤那顆）。
之前這份文件寫「社群&SEO 自動生成（社群&SEO 區塊的「自動」）」是**沒有實測過的錯誤記載**，
`POST /api/v1/ai/og-description:generate` 至今仍未拓通、也未確認正確 request body（試過
`{article_id}`、`{id}`、`{content, title}` 皆回 422 `文章內容為必填項目`）。

標籤自動生成（已用 XHR 攔截確認）：

| 功能 | Endpoint | 實際 request body | 效果 |
|---|---|---|---|
| 標籤自動生成（標籤區塊的「自動」） | `POST /api/v1/ai/hashtags:generate` | `{"text": "<content欄位的Lexical JSON字串>"}`（欄位名是 `text`，不是 `content`） | 依內文分析結果**取代**現有標籤（不是附加） |

og-description:generate 待下次找到真正的觸發入口（或請教有更完整編輯權限/角色的帳號畫面
是否有這顆按鈕）後再補。

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
- [ ] **段落之間要有一行空行**：兩個文字段之間要插入一個「空的 paragraph 節點」，
  不能讓文字段直接相鄰（2026-08-05 使用者手動修正 4002680 後補的規則）。
  - **圖片節點前後不用加空行**，圖片直接接文字段即可（image 節點本身已有間距）。
  - 實作：`children` 陣列組成 `[段1, 空段, 段2, 空段, 段3, 圖, 段4, 空段, 段5, …]`
  - 空段結構就是 text 為空字串的一般 paragraph：
    ```js
    const EMPTY = {children:[{detail:0,format:0,mode:"normal",style:"",text:"",type:"text",version:1}],
      direction:"ltr",format:"",indent:0,type:"paragraph",version:1,textFormat:0,textStyle:""};
    ```
  - 組陣列的寫法（文字段之間插空段、圖片不插）：
    ```js
    const out=[];
    blocks.forEach((b,i)=>{
      if(i>0 && b.type!=='image' && out[out.length-1].type!=='image') out.push(EMPTY);
      out.push(b);
    });
    ```

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
| **本機檔案上傳** | ✅ | 純 API 可行：`upload-url` → S3 PUT → `images/process` →<br>存檔時**務必帶 `asset_images`**（漏掉圖片會 404），詳見下節 |
| 送審／排程／發布 | ⚠️ | 狀態機 API 仍未查到，但已知會被**意外觸發**，見下方「⚠️ 狀態機注意事項」 |

**結論（2026-08-05 最終版）：整篇稿件 100% 能純 API 完成，包含本機圖片上傳，不需要開瀏覽器。**

關鍵是圖片上傳除了 `upload-url` → S3 → `images/process` 三步之外，
**存檔的 PUT 一定要帶 `asset_images` 欄位**，否則圖片不會被搬成正式檔（會全程回 200 但圖片 404）。

## 本機圖片上傳（純 API，2026-08-05 實測成功，不需要 file input）

先前誤判為「一定要經過 `<input type=file>`」，實際上有一支專門發 presigned URL 的 API：

```
GET /api/v1/images/upload-url?extension=png
```

回傳：
```json
{
  "uuid": "019fd03c-061a-702c-ac7f-ba5452bb359d",
  "upload_url": {
    "url": "https://t-news-news-images-tvbs-com-tw.s3.ap-northeast-1.amazonaws.com/temp/...?X-Amz-...",
    "headers": {"Host": "..."}
  },
  "expires_in": 180,
  "expires_at": "..."
}
```

**踩雷**：`upload_url` 是**物件** `{url, headers}`，不是字串。直接把它當網址用會變成
`[object Object]` 導致 S3 回 404（我第一次就踩到）。要取 `.url`，並把 `.headers` 原樣帶上。

**踩雷（2026-08-12）：`images/process` 的 `width`/`height` 是輸出尺寸，不是原圖尺寸**——
千萬別把原圖真實解析度（現在的相機/AP圖常常是 5000~8000px 邊長、幾千萬像素）直接當這兩個
參數傳進去。症狀是後端跟 `GET` 都回 200、圖片網址也能正常開啟、`naturalWidth/naturalHeight`
也對得上，**看起來一切正常**，但在後台編輯器的縮圖框裡實際渲染會卡住，畫面呈現「上半段是
照片、下半段是一整塊純灰色矩形」（不是煙霧或陰影，是渲染沒跑完，邊界是筆直的一條線）。
單純呼叫 API 或用 `Image()` 檢查 `complete`/`naturalWidth` 完全看不出這個問題，**一定要
實際截圖看畫面**才抓得到。

正確做法：裁切框（`x,y,right,bottom`）可以照樣用完整原圖範圍，但 `width`/`height` 這兩個
輸出尺寸參數要換算成合理的網頁用尺寸（例如統一縮到寬 1600px，高度依原圖長寬比等比例算出來），
不要照抄原圖的px數：

```js
const TARGET_W = 1600;
const outH = Math.round(TARGET_W * (origH / origW));
const q = new URLSearchParams({uuid, extension:'jpg', width: TARGET_W, height: outH,
  crop_aspect_ratio_type:'4x3', x:0, y:0, right:origW, bottom:origH});
```

### 完整三步驟

```js
// 1. 要一組 presigned URL（presigned URL 有效期僅 180 秒，拿到要盡快用）
const {uuid, upload_url} = (await (await fetch('/api/v1/images/upload-url?extension=png')).json()).data;

// 2. 把檔案 bytes 直接 PUT 上 S3（任何 HTTP client 都可以，不需要 file input）
await fetch(upload_url.url, {method: 'PUT', body: fileBlob, headers: upload_url.headers});

// 3. 裁切／浮水印處理，取得可用圖片
const q = new URLSearchParams({
  uuid, extension: 'png',
  width: 1067, height: 600,           // 產出尺寸
  crop_aspect_ratio_type: '16x9',
  x: 0, y: 0, right: 1280, bottom: 720,  // 原圖上的裁切框
  watermark_position: 'lb'            // lb=左下, lt=左上, rb=右下, rt=右上；不加此參數=不用浮水印
});
const {preview_url} = (await (await fetch('/api/v1/images/process?' + q)).json()).data;
```

第 3 步回傳 `{uuid, preview_url}`，把圖片 URL 填進 `featured_image.url`（主圖）或
Lexical `image` 節點的 `src`（內文插圖）即可。

**限制**：圖片高度需 ≥600px（前端會擋，服務端是否也擋未驗證）。

### ✅ 最終解法：`asset_images`（2026-08-05 反查前端 payload 後破解，實測成功）

前面說「圖片必須走 UI」是**錯的**，真正缺的是 PUT 時要帶 **`asset_images`** 欄位。
它就是告訴後端「這些暫存圖要轉成正式檔」的清單，沒帶就不會搬檔，圖片自然 404。

```js
// 1) presign + S3（同前）
const {uuid: U, upload_url} = (await (await fetch('/api/v1/images/upload-url?extension=jpg')).json()).data;
await fetch(upload_url.url, {method:'PUT', body: fileBlob, headers: upload_url.headers});

// 2) process（產生預覽，同前）
await fetch('/api/v1/images/process?' + new URLSearchParams({
  uuid: U, extension:'jpg', width:1067, height:600,
  crop_aspect_ratio_type:'16x9', x:0, y:115, right:3000, bottom:1802,
  watermark_position:'lb'
}));

// 3) PUT 文章時，圖片網址用 temp 版本，並帶上 asset_images ★關鍵★
const tempUrl = 'https://news-images.tvbs.com.tw/api/v1/image/temp/'+U+'.jpg?w=1067&h=600&t=16x9&wp=l,b&c=0,115,3000,1802';
body.featured_image = {url: tempUrl, caption: CAP, alt: CAP};
body.asset_images = [{
  uuid: U,
  extension: 'jpg',
  source_image_id: 7,        // 圖片來源 id，7=達志影像；清單查 GET /source/image
  remark: '',                // 對應彈窗的「備註」
  width: 3000, height: 2000, // 原圖尺寸
  watermark_position: 'lb',  // 不用浮水印則傳 null
  crop: '0,115,3000,1802'    // "x,y,right,bottom"
}];
```

存檔後後端會把 temp 檔搬成正式檔，並自動把網址裡的 `/temp/` 去掉，圖片即可正常顯示。
**實測驗證**：帶 `asset_images` 後 `IMAGE_GET = 200`（先前不帶一律 404）。

補充：完整的 UI payload 還有 `editor_images`（內文插圖）與 `gallery_images`（圖庫選圖）
兩個同類欄位，內文插圖理論上要放進 `editor_images`，尚未單獨實測。

### ⚠️ 先前的錯誤結論與排查過程（保留紀錄）

上面三步**全部回 200，但圖片最後是壞的**。原因：`images/process` 產出的圖只存在
`news-images.tvbs.com.tw/api/v1/image/**temp/**{uuid}.jpg`（temp 路徑），
存進文章時後端會把 `/temp/` 拿掉改成正式路徑，但**實體檔案沒有被搬過去**，
所以文章存好後圖片網址一律 404（畫面上顯示 TVBS 預設灰圖）。

驗證過、確定無效的做法：
- 直接把 `process` 回傳的 `preview_url` 存進 `featured_image.url` → 404
- 自己組 `/temp/{uuid}.jpg?...` 存進去 → 後端照樣把 `/temp/` 去掉 → 404
- 等待非同步搬檔（輪詢 20 秒）→ 一直 404
- 找 commit/confirm/finalize/persist 類端點 → 全部 404，**不存在這種端點**

（↑ 以上排查全部白忙，真正原因是漏了 `asset_images`，解法見上一節。留著是為了記錄
「回 200 不等於成功」這個教訓，以及怎麼靠反查前端 payload 找出缺少的欄位：
攔截 `XMLHttpRequest.prototype.send`（這個後台用 axios 走 XHR，攔 `window.fetch` 抓不到
文章存檔的請求），把 UI 實際送出的 body 印出來跟自己組的 body 逐欄比對。）

**踩雷**：如果文章的 `featured_image.url` 先被寫入過壞網址，主圖元件會卡住、
重新上傳也不顯示。要先按主圖右上角垃圾桶圖示清空，再重新上傳才會正常。

**內文插圖的圖說**：UI 插入後 `figcaption` 只會帶系統預設的「（圖／來源）」，
要改成檔名完整版本時，用 UI 改容易失敗，直接用 PUT 改 `content` 裡 image 節點的
`figcaption`／`altText` 最快。同理，若 UI 操作不慎插入多張圖，也可以用 PUT
把多餘的 image 節點從 `content.root.children` 移除。

## ⚠️ 狀態機注意事項（2026-08-12 踩雷）

`status` 欄位已確認的值：`0`=草稿、`1`=已發布（`published_at` 會有值）、`3`=審核中（畫面顯示
「檢視文章」+ 唯讀，只剩「預覽」「返回」兩個按鈕，抓不到能退回草稿的 API 或按鈕，目前這帳號
權限下**沒有自助退審手段**）。`2` 是什麼還不知道（可能是排程）。

**踩雷經過**：稿號 4006050 原本是 `status:0` 草稿，在用 `find` 工具找「社群&SEO 自動」按鈕、
反覆點擊/重整頁面的過程中，**沒有任何一次操作記錄或網路攔截明確顯示送審 API 被呼叫**，但事後
`GET /articles/4006050` 卻變成 `status:3`（審核中）。懷疑是 `find` 工具回傳的 `ref` 在畫面剛
渲染完、還沒穩定時抓到過期的座標映射，點擊落到別的按鈕（例如送審鈕）上，而不是預期的目標。

**教訓**：
- 在这个後台操作前，**先用 `read_page`（filter: interactive, 完整掃描）取得當下真實的按鈕清單**，
  不要單純依賴 `find` 的自然語言比對結果，尤其畫面剛切換/剛重整時 ref 可能還沒同步。
- 點擊「自動」「送審」這類會改變文章狀態或觸發真實副作用的按鈕前，**先攔截 XHR 確認實際打的
  endpoint**，如果攔截結果跟預期的 URL 不符要立刻停手，不要接著做更多操作。
- 如果不小心把稿件送進審核，**先跟使用者回報狀態並停手**，不要嘗試用猜測的 endpoint
  （`/cancel`、`/back` 等我猜的路徑都是 404/503，代表根本不存在）硬闖，避免搞出更多真實副作用。

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
