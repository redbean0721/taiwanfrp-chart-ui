# TaiwanFRP Chart UI

🚀 一個基於 Vue 3 + TypeScript 的 TaiwanFRP 伺服器監控儀表板，提供即時的連線數據視覺化展示。

![Version](https://img.shields.io/badge/version-0.0.4-blue.svg)
![Vue](https://img.shields.io/badge/Vue.js-3.5.18-green.svg)
![TypeScript](https://img.shields.io/badge/TypeScript-5.8.3-blue.svg)
![Chart.js](https://img.shields.io/badge/Chart.js-4.5.0-orange.svg)

## ✨ 功能特色

- 📊 **雙圓餅圖顯示** - 當前連線數與客戶端數量分佈
- 🔄 **即時資料更新** - 使用 SSE (Server-Sent Events)，每 5 秒接收最新資料並覆蓋上一份
- 🌓 **主題切換** - 支援明亮模式與深色模式
- 📱 **響應式設計** - 完美適配桌面、平板、手機
- 📈 **詳細統計** - 顯示各節點連線數與百分比
- ⏰ **更新時間** - 顯示最後資料更新時間
- 🎨 **美觀介面** - 現代化設計風格

## 🖼️ 畫面預覽

### 明亮模式
- 清爽的白色背景設計
- 彩虹色調色盤圖表
- 直觀的統計資訊卡片

### 深色模式
- 優雅的深色背景
- 護眼的配色方案
- 一鍵切換主題

## 🛠️ 技術棧

- **前端框架**: Vue 3.5.18 (Composition API)
- **開發語言**: TypeScript 5.8.3
- **建置工具**: Vite 7.1.2
- **圖表庫**: Chart.js 4.5.0 + vue-chartjs 5.3.2
- **樣式**: 純 CSS (無額外 UI 框架)

## 📦 安裝與執行

### 前置需求
- Node.js 18+ 
- Yarn 或 npm

### 安裝步驟

1. **克隆專案**
```bash
git clone <repository-url>
cd taiwanfrp-chart-ui
```

2. **安裝依賴**
```bash
yarn install
# 或
npm install
```

3. **啟動開發伺服器**
```bash
yarn dev
# 或
npm run dev
```

4. **開啟瀏覽器**
```
http://localhost:5173
```

### 建置專案

```bash
yarn build
# 或
npm run build
```

建置後的檔案會產生在 `dist/` 目錄中。

### 預覽建置結果

```bash
yarn preview
# 或
npm run preview
```

## 🎯 API 資料來源

本專案使用由 [redbean0721](https://redbean0721.com) 開發的 **CoolAPI** 統計服務，為 TaiwanFRP 提供即時監控數據：

```
https://api.redbean0721.com/api/frp/monitor/query
```

### API 服務特色
- 🔄 **即時資料** - 提供最新的伺服器狀態
- 📊 **完整統計** - 包含連線數與客戶端資訊
- 🚀 **高效能** - 快速回應與穩定服務
- 🛡️ **可靠性** - 24/7 監控與維護

### 資料格式
- `cur_conns`: 當前連線數
- `client_counts`: 客戶端數量  
- `server_name`: 伺服器名稱
- `is_online`: 上線狀態

### 技術支援
- **API 開發者**: [redbean0721](https://redbean0721.com)
- **服務提供**: CoolAPI 統計平台
- **服務對象**: TaiwanFRP 社群

## 📱 響應式設計

| 螢幕尺寸 | 寬度範圍 | 佈局特點 |
|---------|---------|---------|
| 超大螢幕 | ≥1400px | 最大寬度 1200px |
| 大螢幕 | 1200-1399px | 最大寬度 1000px |
| 中等螢幕 | 992-1199px | 最大寬度 900px |
| 小螢幕 | 769-991px | 最大寬度 95% |
| 手機 | ≤768px | 垂直堆疊佈局 |

## 🎨 主題系統

### 切換方式
- 點擊右上角的 🌙/☀️ 按鈕
- 設定會自動儲存到瀏覽器
- 首次訪問會檢測系統偏好

### 色彩設計
- **明亮模式**: 白色背景 + 彩虹調色盤
- **深色模式**: 深灰背景 + 相同調色盤

## 📂 專案結構

```
taiwanfrp-chart-ui/
├── public/                 # 靜態資源
├── src/
│   ├── components/         # Vue 組件
│   │   └── FrpChart.vue   # 主要圖表組件
│   ├── App.vue            # 根組件
│   ├── main.ts            # 應用程式入口
│   └── style.css          # 全域樣式
├── package.json           # 依賴配置
├── tsconfig.json          # TypeScript 配置
├── vite.config.ts         # Vite 配置
├── CHANGELOG.md           # 變更日誌
└── README.md              # 專案說明
```

## 🔧 開發指南

### 新增功能
1. 修改 `src/components/FrpChart.vue`
2. 測試響應式設計
3. 更新 `CHANGELOG.md`
4. 提交程式碼

### 自訂樣式
- 主要樣式在 `FrpChart.vue` 的 `<style>` 區塊
- 使用 CSS 變數便於主題切換
- 遵循 BEM 命名規範

### API 修改
如需更換 API 來源，請修改 `FrpChart.vue` 中的：
```typescript
const API_URL = 'https://your-api-endpoint.com/status'
```

## 📄 授權條款

本專案採用 MIT License - 詳見 [LICENSE](LICENSE) 檔案

## 📞 聯絡資訊

- **專案作者**: [redbean0721](https://redbean0721.com)
- **GitHub**: [@redbean0721](https://github.com/redbean0721)
- **CoolAPI**: 統計服務提供者
- **專案用途**: 為 TaiwanFRP 社群提供監控服務

## ⭐ 致謝

- 感謝 [TaiwanFRP](https://taiwanfrp.me) 提供優質的 FRP 服務
- 感謝 [redbean0721](https://github.com/redbean0721) 開發的 **CoolAPI** 統計平台
- 感謝 Vue.js 和 Chart.js 團隊的優秀工具
- 感謝 TaiwanFRP 社群的支持與回饋
- 感謝所有貢獻者的協助

---

如果覺得這個專案有幫助，請給個 ⭐ Star！
