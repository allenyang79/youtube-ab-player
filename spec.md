# YouTube A-B 重複播放器 規格書

線上版：https://allenyang79.github.io/youtube-ab-player/

## 1. 技術限制（不可改變）

- 純前端，單一 `www/index.html`（CSS / JS 內嵌），不需後端、不需 build 工具
- 原生 JavaScript，不使用框架
- 播放器：YouTube IFrame Player API（https://www.youtube.com/iframe_api）
- 儲存：localStorage，以 video ID 為 key
- 介面語言：繁體中文；需支援桌面與手機（RWD）

## 2. 資料結構

### localStorage

| key | 內容 |
| --- | --- |
| `abplayer:video:<videoId>` | 該影片的區段（見下方） |
| `abplayer:last` | 上次開啟的影片 ID |
| `abplayer:rate` | 播放速度 |

```json
{
  "videoId": "dQw4w9WgXcQ",
  "segments": [
    { "id": "s1", "name": "副歌", "a": 62.5, "b": 80.0, "loops": 0 }
  ]
}
```

- `a`、`b` 單位為秒（浮點數）
- `loops` = 播幾遍後接下一段，`0` = 無限循環

### 匯出 JSON 檔（`abplayer-<videoId>.json`）

與 localStorage 格式相容，另加說明欄位：

```json
{
  "app": "youtube-ab-player",
  "version": 1,
  "videoId": "dQw4w9WgXcQ",
  "title": "影片標題",
  "exportedAt": "2026-11-19T08:00:00.000Z",
  "segments": [ { "id": "s1", "name": "副歌", "a": 62.5, "b": 80, "loops": 0 } ]
}
```

### 分享連結（URL hash）

```
https://…/youtube-ab-player/#v=<videoId>&s=<a>,<b>,<loops>,<name>;<a>,<b>,<loops>,<name>…
```

- `name` 經 `encodeURIComponent`（逗號、分號、& 都會被編碼，不會與分隔符號衝突）
- 只有 `v=` 沒有 `s=` 時僅開啟影片

## 3. 功能

### 階段 1：載入影片與 A-B 循環 ✅

- 網址解析：`watch?v=`、`youtu.be/`、`shorts/`、`embed/`、`live/`、純 11 碼 ID，含 `m.` / `music.` 網域
- 輪詢 100ms，只在 PLAYING 時執行；`seekTo` 後有 350ms 冷卻，避免連續跳轉
- 以 `getVideoData().video_id` 比對來避開廣告；ENDED 時也觸發跳回 A（處理 B 在片尾）
- `onError` 中文訊息（2 / 5 / 100 / 101 / 150 / 153）；以 `file://` 開啟時提示改用 http server
- 換影片用 `cueVideoById`（不自動播放，配合手機自動播放限制）、`playsinline`

### 階段 2：多區段 ✅

- 可命名、▶ 播放、✕ 刪除（會先確認）、目前區段高亮，依 a 排序
- 重新整理後自動載入上次的影片與區段
- 匯出章節文案（標題 + 時間 + 名稱 + `?t=` 連結），一鍵複製，可切換是否顯示結束時間

### 階段 3：編輯與播放控制 ✅

- A/B 編輯區編輯「目前選中的對象」：選了區段就編輯該區段（自動儲存），否則編輯草稿；「＋ 新區段」回到草稿模式
- A/B 微調 ±0.1 與 ±1 秒，調整時自動試聽（調 A 從 A 播，調 B 從 B 前 1.5 秒播）
- 可直接輸入 `mm:ss.s` / `h:mm:ss.s` / 秒數
- B 必須晚於 A 至少 0.3 秒，不合法時提示並拒絕
- 播放速度 0.25x～2x，記住上次設定，並與 YouTube 齒輪選單同步（`onPlaybackRateChange`）
- 循環次數：播 N 遍後自動接下一段，最後一段播完停在開頭並暫停
- 快捷鍵：`A`/`B` 標記、`Space` 播放暫停、`[` `]` 調 A、`,` `.` 調 B（加 `Shift` 為 ±1 秒）、`↑` `↓` 切換區段；文字輸入框聚焦或對話框開啟時不觸發；鍵盤焦點在 iframe 時顯示提示

### 階段 4：匯出 / 匯入與分享 ✅

- **匯出 JSON**：下載目前影片的所有區段
- **匯入 JSON**：
  - 驗證格式：必須是含 `videoId`（11 碼）與 `segments` 陣列的物件；檔案 ≤ 1 MB、區段 ≤ 200 個
  - 每個區段需有數字 `a ≥ 0`、`b ≥ a + 0.3`；`loops` 選填（非負數，上限 99）；`name` 選填字串（上限 60 字）
  - 不合格的區段略過並告知數量；全部不合格則拒絕匯入
  - 檔案屬於另一部影片時，詢問是否切換影片
  - 本機已有區段時，詢問「合併」（保留本機、略過 A/B 皆相差 < 0.05 秒的重複區段）／「取代」／「不匯入」
  - 匯入的區段全部已存在時不跳對話框，直接提示沒有變更
- **分享連結**：把所有區段編碼進 URL hash，顯示在文字框可一鍵複製
  - 開啟分享連結：載入該影片；本機沒有區段就直接匯入，有的話用與 JSON 相同的合併 / 取代 / 不匯入對話框
  - 處理完即移除 hash，重新整理不會重複詢問
  - 支援 `hashchange`，也可以把分享連結貼到上方網址框按「載入」

## 4. 測試方式

用 Playwright 攔截 `https://www.youtube.com/iframe_api`，換成模擬的 `YT.Player`（模擬 `getCurrentTime` / `seekTo` / `getVideoData` / `getDuration` / `getPlayerState` / `playVideo` / `pauseVideo` / `get/setPlaybackRate` / `cueVideoById`，並觸發 `onReady` / `onStateChange` / `onPlaybackRateChange`），在 `www/` 執行 `python3 -m http.server` 開啟頁面，跑完整流程並截圖檢查手機（390px）與桌面版面。

## 5. 部署

GitHub Pages 由 `.github/workflows/pages.yml` 將 `www/` 部署（Settings → Pages → Source 需選「GitHub Actions」）。push 到 main 後約 1 分鐘生效。
