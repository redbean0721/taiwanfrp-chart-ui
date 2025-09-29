# 變更日誌 (Changelog)

所有對此專案的重要變更都會記錄在此檔案中。

格式基於 [Keep a Changelog](https://keepachangelog.com/zh-TW/1.0.0/)，
並且此專案遵循 [語意化版本](https://semver.org/lang/zh-TW/)。

## [0.0.2] - 2025-09-29

### ⚡ 即時資料更新改進 (SSE 實作)

#### 核心功能
- 🔄 將資料更新方式從 30 秒輪詢改為 SSE (Server-Sent Events) 即時推送
- ⏱ 每 5 秒自動接收最新資料，並覆蓋上一份，減少前端記憶體消耗
- 🛠 自動重連機制：當 SSE 連線中斷時，自動嘗試重新建立連線

#### 技術實作
- FastAPI SSE API (`/frp/monitor/query/sse`)
- Vue 3 + EventSource 接收 SSE
- 移除原本的 fetch 輪詢機制


## [0.0.1] - 2025-09-09

### ✨ 初始版本發布 (Initial Release)

#### 核心功能
- 🎨 TaiwanFRP 伺服器監控儀表板
- 📊 即時連線數圓餅圖顯示
- 👥 客戶端數量圓餅圖顯示
- � 30 秒自動重新整理資料
- ⏰ 最後更新時間顯示

#### 使用者介面
- � 明亮/深色主題切換功能
- 📱 完整響應式設計 (支援 5 個中斷點)
- 📈 詳細統計資訊卡片顯示
- 🌈 20 色調色盤確保圖表清晰度

#### 技術實作
- Vue 3 + TypeScript + Composition API
- Chart.js 4.5.0 + vue-chartjs 5.3.2
- Vite 7.1.5 建置工具
- CSS 響應式設計
- localStorage 主題持久化
- Fetch API 資料擷取

#### 響應式支援
- 桌面版：≥1400px (寬螢幕)
- 筆電版：1200-1399px
- 平板版：992-1199px
- 小平板：769-991px
- 手機版：≤768px

---

*此專案旨在提供 TaiwanFRP 服務的即時監控視覺化介面*
