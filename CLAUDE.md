# CLAUDE.md

本檔提供 Claude Code 在此 repo 工作時的指引。完整規格見 `spec.md`。

## 專案概要

YouTube A-B 重複播放器：在 YouTube 影片上標記多個 A-B 區段並循環播放。純前端、單一檔案。

- 程式：`www/index.html`（HTML + 內嵌 CSS + 內嵌 JS，一個 IIFE）
- 部署：`.github/workflows/pages.yml` 把 `www/` 發佈到 GitHub Pages（https://allenyang79.github.io/youtube-ab-player/）

## 不可改變的限制

- 保持單一 `www/index.html`，不加 build 工具、不加框架、不加後端
- 播放器只用 YouTube IFrame Player API
- 資料只存 localStorage（`abplayer:video:<id>`、`abplayer:last`、`abplayer:rate`）
- 介面文字一律繁體中文；桌面與手機（390px）都要可用

## 程式結構（`www/index.html` 的 `<script>`）

依區塊註解（`// ===== … =====`）分段：常數 → `state` → DOM 參照 `el` → 工具（網址 / 時間解析）→ 儲存 → 目標（`editTarget` / `loopTarget`）→ 畫面渲染 → 章節文案 / 文字框 → 對話框 `choose()` → 匯入（`sanitizeSegments`、`importSegments`）→ JSON 匯出入 → 分享連結（hash）→ 輪詢 → A/B 設定 → 區段操作 → 播放速度 → 播放器事件 → 載入影片 → 事件綁定 → 快捷鍵 → 啟動。

重點：
- 所有外部輸入（JSON 檔、URL hash、localStorage）都要經過驗證 / 正規化
- 區段永遠依 `a` 排序；修改後呼叫 `saveStore()` 與 `renderAll()`
- `window.__ab` 暴露內部狀態與部分函式，供測試使用

## 本機執行與測試

```bash
cd www && python3 -m http.server 8000
```

測試用 Playwright：攔截 `https://www.youtube.com/iframe_api`，回傳模擬的 `YT.Player`（getCurrentTime / seekTo / getVideoData / getDuration / getPlayerState / playVideo / pauseVideo / get/setPlaybackRate / cueVideoById，並觸發 onReady / onStateChange / onPlaybackRateChange），跑完整流程並截圖檢查 390px 手機與桌面版面。

## 工作慣例

- commit 訊息用繁體中文
- 每個階段完成後先讓使用者確認，再進下一階段
- 完成功能後同步更新 `spec.md`
