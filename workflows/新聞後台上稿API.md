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

## 尚未拓測（下一步待補）

- 更新既有文章：應為 `PATCH` 或 `PUT /api/v1/articles/{id}`，欄位結構應與建立時相同，未實測
- 送審（草稿→審核中）：狀態變更 API，未拓測
- 排程/發布：`scheduled_at` 欄位與/或另一支狀態切換 API，未拓測
- 圖片上傳：主圖是走 `news-images.tvbs.com.tw` 的另一套圖床服務，上傳流程未拓測
- `credit_id` 對照表（角色 × 人名 → id）未完整列出，需要時可查 `GET /credits`

## 注意事項

- 建立文章是**真實動作**，會在正式後台留下一筆草稿紀錄（稿號可查）
- 2026-08-05 測試建立的稿號 **4002520**（【AI測試】DenisAutoAPI拓測草稿）待清除，
  目前後台介面未找到刪除按鈕，需要人工確認清除方式
