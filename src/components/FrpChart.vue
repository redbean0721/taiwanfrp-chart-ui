<template>
<div class="chart-container" :class="{ 'dark-theme': isDarkMode }">
    <div class="header">
    <h2>TaiwanFRP 伺服器監控面板</h2>
    <button @click="toggleTheme" class="theme-toggle" :title="isDarkMode ? '切換到明亮模式' : '切換到深色模式'">
        <span v-if="isDarkMode">☀️</span>
        <span v-else>🌙</span>
    </button>
    </div>
    <div class="version-info">UI Version: {{ version }}</div>
    <div class="update-time">最後更新時間: {{ lastUpdateTime }}</div>
    <div v-if="loading" class="loading">載入中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else class="content">
    <!-- 當前連線數圖表 -->
    <div class="chart-section">
        <h3>當前連線數分佈</h3>
        <div class="chart-content">
            <div class="chart-area">
                <Pie :data="chartData" :options="chartOptions" />
            </div>
            <div class="stats-card">
                <h4>📊 統計資訊</h4>
                <div class="total-info">
                    <span class="total-label">總連線數</span>
                    <span class="total-value">{{ totalConnections }}</span>
                </div>
                <div class="server-list">
                    <div v-for="item in serverStats" :key="item?.name" class="server-item">
                        <span class="server-name">{{ item?.name }}</span>
                        <span class="server-connections">{{ item?.connections }}</span>
                        <span class="server-percentage">{{ item?.percentage }}%</span>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- 客戶端數量圖表 -->
    <div class="chart-section">
        <h3>客戶端數量分佈</h3>
        <div class="chart-content">
            <div class="chart-area">
                <Pie :data="clientChartData" :options="clientChartOptions" />
            </div>
            <div class="stats-card">
                <h4>👥 統計資訊</h4>
                <div class="total-info">
                    <span class="total-label">總客戶端數</span>
                    <span class="total-value">{{ totalClients }}</span>
                </div>
                <div class="server-list">
                    <div v-for="item in clientStats" :key="item?.name" class="server-item">
                        <span class="server-name">{{ item?.name }}</span>
                        <span class="server-connections">{{ item?.clients }}</span>
                        <span class="server-percentage">{{ item?.percentage }}%</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
    </div>
</div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { Pie } from 'vue-chartjs'
import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    ArcElement,
    CategoryScale
} from 'chart.js'
import packageInfo from '../../package.json'

// 註冊 Chart.js 組件
ChartJS.register(Title, Tooltip, Legend, ArcElement, CategoryScale)

// 版本信息
const version = packageInfo.version

// 定義資料類型
interface ServerData {
    id: number
    server_id: number
    version: string
    is_online: number
    client_counts: number
    cur_conns: number
    tcp_count: number
    udp_count: number
    http_count: number
    https_count: number
    tcpmux_count: number
    stcp_count: number
    sudp_count: number
    total_traffic_in: number
    total_traffic_out: number
    recorded_at: string
}

interface ApiResponse {
    stats: {
        version: Record<string, number>
        server: Record<string, number>
    }
    result: Record<string, ServerData[]>
}

const loading = ref(true)
const error = ref('')
const apiData = ref<ApiResponse | null>(null)
const lastUpdateTime = ref('now')
const isDarkMode = ref(false)

// 主題切換功能
const toggleTheme = () => {
    isDarkMode.value = !isDarkMode.value
    localStorage.setItem('theme', isDarkMode.value ? 'dark' : 'light')
}

// 從 localStorage 載入主題設置
const loadTheme = () => {
    const savedTheme = localStorage.getItem('theme')
    if (savedTheme) {
        isDarkMode.value = savedTheme === 'dark'
    } else {
        // 如果沒有保存的主題，檢查系統偏好
        isDarkMode.value = window.matchMedia('(prefers-color-scheme: dark)').matches
    }
}

// 生成隨機顏色
const generateColors = (count: number) => {
    const colors = [
        '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0',
        '#9966FF', '#FF9F40', '#8BC34A', '#E91E63',
        '#FFC107', '#795548', '#607D8B', '#FF5722',
        '#9C27B0', '#3F51B5', '#009688', '#CDDC39',
        '#FF9800', '#673AB7', '#2196F3', '#4CAF50'
    ]
    return colors.slice(0, count)
}

// 計算圖表資料
const chartData = computed(() => {
    if (!apiData.value) return { labels: [], datasets: [] }

    const servers = Object.entries(apiData.value.result)
    const labels: string[] = []
    const data: number[] = []

    servers.forEach(([serverName, serverArray]) => {
        if (serverArray.length > 0) {
        const serverData = serverArray[0]
        if (serverData.cur_conns > 0) {
            labels.push(serverName)
            data.push(serverData.cur_conns)
        }
        }
    })

    const colors = generateColors(labels.length)

    return {
        labels,
        datasets: [
        {
            label: '當前連線數',
            data,
            backgroundColor: colors,
            borderColor: colors.map(color => color + '80'),
            borderWidth: 2
        }
        ]
    }
})

// 計算總連線數
const totalConnections = computed(() => {
    if (!apiData.value) return 0

    return Object.values(apiData.value.result).reduce((total, serverArray) => {
        if (serverArray.length > 0) {
        return total + serverArray[0].cur_conns
        }
        return total
    }, 0)
})

// 計算總客戶端數
const totalClients = computed(() => {
    if (!apiData.value) return 0

    return Object.values(apiData.value.result).reduce((total, serverArray) => {
        if (serverArray.length > 0) {
        return total + serverArray[0].client_counts
        }
        return total
    }, 0)
})

// 計算伺服器統計
const serverStats = computed(() => {
    if (!apiData.value) return []

    const servers = Object.entries(apiData.value.result)
    return servers
        .map(([serverName, serverArray]) => {
        if (serverArray.length > 0) {
            const connections = serverArray[0].cur_conns
            const percentage = totalConnections.value > 0 
            ? ((connections / totalConnections.value) * 100).toFixed(1)
            : '0'
            return {
            name: serverName,
            connections,
            percentage
            }
        }
        return null
        })
        .filter(item => item !== null)
        .sort((a, b) => b!.connections - a!.connections)
})

// 計算客戶端統計
const clientStats = computed(() => {
    if (!apiData.value) return []

    const servers = Object.entries(apiData.value.result)
    return servers
        .map(([serverName, serverArray]) => {
        if (serverArray.length > 0) {
            const clients = serverArray[0].client_counts
            const percentage = totalClients.value > 0 
            ? ((clients / totalClients.value) * 100).toFixed(1)
            : '0'
            return {
            name: serverName,
            clients,
            percentage
            }
        }
        return null
        })
        .filter(item => item !== null)
        .sort((a, b) => b!.clients - a!.clients)
})

// 計算客戶端圖表資料
const clientChartData = computed(() => {
    if (!apiData.value) return { labels: [], datasets: [] }

    const servers = Object.entries(apiData.value.result)
    const labels: string[] = []
    const data: number[] = []

    servers.forEach(([serverName, serverArray]) => {
        if (serverArray.length > 0) {
        const serverData = serverArray[0]
        if (serverData.client_counts > 0) {
            labels.push(serverName)
            data.push(serverData.client_counts)
        }
        }
    })

    const colors = generateColors(labels.length)

    return {
        labels,
        datasets: [
        {
            label: '客戶端數量',
            data,
            backgroundColor: colors,
            borderColor: colors.map(color => color + '80'),
            borderWidth: 2
        }
        ]
    }
})

// 圖表選項
const chartOptions = computed(() => ({
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
        legend: {
        position: 'bottom' as const,
        labels: {
            padding: 15,
            usePointStyle: true,
            color: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
            boxWidth: 12,
            font: {
                size: 12
            }
        }
        },
        tooltip: {
        backgroundColor: isDarkMode.value ? '#374151' : '#ffffff',
        titleColor: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
        bodyColor: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
        borderColor: isDarkMode.value ? '#4b5563' : '#e5e7eb',
        borderWidth: 1,
        callbacks: {
            label: function(context: any) {
            const label = context.label || ''
            const value = context.raw || 0
            const percentage = totalConnections.value > 0 
                ? ((value / totalConnections.value) * 100).toFixed(1)
                : '0'
            return `${label}: ${value} 連線 (${percentage}%)`
            }
        }
        }
    }
}))

// 客戶端圖表選項
const clientChartOptions = computed(() => ({
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
        legend: {
        position: 'bottom' as const,
        labels: {
            padding: 15,
            usePointStyle: true,
            color: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
            boxWidth: 12,
            font: {
                size: 12
            }
        }
        },
        tooltip: {
        backgroundColor: isDarkMode.value ? '#374151' : '#ffffff',
        titleColor: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
        bodyColor: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
        borderColor: isDarkMode.value ? '#4b5563' : '#e5e7eb',
        borderWidth: 1,
        callbacks: {
            label: function(context: any) {
            const label = context.label || ''
            const value = context.raw || 0
            const percentage = totalClients.value > 0 
                ? ((value / totalClients.value) * 100).toFixed(1)
                : '0'
            return `${label}: ${value} 客戶端 (${percentage}%)`
            }
        }
        }
    }
}))

// 獲取 API 資料
const fetchData = async () => {
    try {
        loading.value = true
        const response = await fetch('https://api.redbean0721.com/api/frp/monitor/query?node=all&num=8')
        
        if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`)
        }
        
        const data = await response.json()
        apiData.value = data
        lastUpdateTime.value = new Date().toLocaleTimeString('zh-TW')
    } catch (err) {
        error.value = err instanceof Error ? err.message : '獲取資料時發生錯誤'
        console.error('Error fetching data:', err)
    } finally {
        loading.value = false
    }
}

// 組件掛載時獲取資料
onMounted(() => {
    loadTheme()
    fetchData()

    // 每 30 秒更新一次資料
    setInterval(fetchData, 30000)
})
</script>

<style scoped>
.chart-container {
    padding: 20px;
    max-width: 1800px;
    margin: 0 auto;
    transition: all 0.3s ease;
    background-color: #f5f5f5;
    min-height: 100vh;
    width: 100%;
    box-sizing: border-box;
}

.header {
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    margin-bottom: 10px;
    width: 100%;
}

.chart-container h2 {
    text-align: center;
    color: #2c3e50;
    margin: 0;
    flex: 1;
    font-size: clamp(18px, 4vw, 24px);
    word-wrap: break-word;
}

.theme-toggle {
    position: absolute;
    right: 0;
    background: none;
    border: 2px solid #ddd;
    border-radius: 50%;
    width: 50px;
    height: 50px;
    cursor: pointer;
    font-size: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.3s ease;
    background-color: white;
    flex-shrink: 0;
}

.theme-toggle:hover {
    transform: scale(1.1);
    border-color: #3498db;
}

.version-info {
    text-align: center;
    color: #888;
    font-size: clamp(11px, 2.2vw, 13px);
    margin-bottom: 10px;
    font-weight: 500;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
}

.update-time {
    text-align: center;
    color: #666;
    font-size: clamp(12px, 2.5vw, 14px);
    margin-bottom: 30px;
    font-style: italic;
    word-wrap: break-word;
}

.loading {
    text-align: center;
    padding: 40px;
    font-size: clamp(16px, 3vw, 18px);
    color: #666;
}

.error {
    text-align: center;
    padding: 40px;
    font-size: clamp(16px, 3vw, 18px);
    color: #e74c3c;
    background: #fdf2f2;
    border: 1px solid #e74c3c;
    border-radius: 8px;
    word-wrap: break-word;
}

.content {
    display: flex;
    flex-direction: column;
    gap: 40px;
    width: 100%;
}

.chart-section {
    background: white;
    border-radius: 12px;
    padding: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    transition: all 0.3s ease;
    width: 100%;
    box-sizing: border-box;
}

.chart-section h3 {
    text-align: center;
    color: #2c3e50;
    margin-bottom: 20px;
    font-size: clamp(16px, 3.5vw, 20px);
    word-wrap: break-word;
}

.chart-content {
    display: flex;
    gap: 30px;
    align-items: flex-start;
    width: 100%;
    box-sizing: border-box;
}

.chart-area {
    flex: 2;
    min-height: 450px;
    background: rgba(248, 249, 250, 0.5);
    border-radius: 8px;
    padding: 20px;
    display: flex;
    flex-direction: column;
    box-sizing: border-box;
    width: 100%;
}

.stats-card {
    flex: 1;
    min-width: 280px;
    background: rgba(248, 249, 250, 0.8);
    border-radius: 12px;
    padding: 20px;
    border: 1px solid rgba(0, 0, 0, 0.08);
    backdrop-filter: blur(10px);
    box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
    box-sizing: border-box;
    width: 100%;
}

.stats-card h4 {
    color: #2c3e50;
    margin-bottom: 15px;
    font-size: clamp(14px, 3vw, 16px);
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 8px;
    justify-content: center;
}

.total-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 8px;
    margin-bottom: 20px;
    box-shadow: 0 4px 8px rgba(102, 126, 234, 0.3);
    box-sizing: border-box;
}

.total-label {
    font-size: clamp(12px, 2.5vw, 14px);
    opacity: 0.9;
}

.total-value {
    font-size: clamp(20px, 5vw, 24px);
    font-weight: bold;
}

.server-list {
    max-height: 300px;
    overflow-y: auto;
    width: 100%;
}

.server-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 15px;
    background: white;
    border-radius: 8px;
    margin-bottom: 8px;
    transition: all 0.3s ease;
    border: 1px solid rgba(0, 0, 0, 0.05);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
    box-sizing: border-box;
    width: 100%;
}

.server-item:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
}

.server-name {
    font-weight: 500;
    color: #2c3e50;
    flex: 1;
    font-size: clamp(12px, 2.5vw, 14px);
    word-wrap: break-word;
    overflow-wrap: break-word;
    min-width: 0;
    margin-right: 10px;
}

.server-connections {
    color: #666;
    margin-right: 8px;
    font-weight: 600;
    background: #e3f2fd;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: clamp(10px, 2vw, 12px);
    white-space: nowrap;
    flex-shrink: 0;
}

.server-percentage {
    color: #666;
    font-size: clamp(10px, 2vw, 12px);
    background: #f3e5f5;
    padding: 4px 8px;
    border-radius: 4px;
    min-width: 45px;
    text-align: center;
    white-space: nowrap;
    flex-shrink: 0;
}

/* 深色模式樣式 */
.chart-container.dark-theme {
    background-color: #1a202c;
    color: #e2e8f0;
}

.dark-theme h2 {
    color: #e2e8f0;
}

.dark-theme .theme-toggle {
    background-color: #2d3748;
    border-color: #4a5568;
    color: #e2e8f0;
}

.dark-theme .theme-toggle:hover {
    border-color: #63b3ed;
    background-color: #374151;
}

.dark-theme .version-info {
    color: #a0aec0;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
}

.dark-theme .update-time {
    color: #a0aec0;
}

.dark-theme .loading {
    color: #a0aec0;
}

.dark-theme .error {
    background: #742a2a;
    border-color: #e53e3e;
    color: #fed7d7;
}

.dark-theme .chart-section {
    background: #2d3748;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
}

.dark-theme .chart-section h3 {
    color: #e2e8f0;
}

.dark-theme .chart-area {
    background: rgba(55, 65, 81, 0.5);
}

.dark-theme .stats-card {
    background: rgba(55, 65, 81, 0.8);
    border-color: rgba(255, 255, 255, 0.1);
}

.dark-theme .stats-card h4 {
    color: #e2e8f0;
}

.dark-theme .total-info {
    background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
}

.dark-theme .server-item {
    background: #374151;
    border-color: rgba(255, 255, 255, 0.1);
}

.dark-theme .server-item:hover {
    background: #4b5563;
}

.dark-theme .server-name {
    color: #e2e8f0;
}

.dark-theme .server-connections {
    color: #a0aec0;
    background: #374151;
}

.dark-theme .server-percentage {
    color: #a0aec0;
    background: #374151;
}

@media (min-width: 1400px) {
    .chart-container {
        max-width: 1800px;
        padding: 40px;
    }
}

@media (min-width: 1200px) and (max-width: 1399px) {
    .chart-container {
        max-width: 1600px;
        padding: 30px;
    }
    
    .chart-content {
        gap: 40px;
    }
    
    .chart-area {
        min-height: 500px;
        padding: 25px;
    }
    
    .stats-card {
        min-width: 320px;
        padding: 25px;
    }
    
    .total-value {
        font-size: 28px;
    }
}

@media (min-width: 992px) and (max-width: 1199px) {
    .chart-container {
        padding: 25px;
        max-width: 1400px;
    }
    
    .chart-content {
        gap: 25px;
    }
    
    .chart-area {
        min-height: 480px;
        padding: 20px;
    }
    
    .stats-card {
        min-width: 280px;
    }
}

@media (min-width: 769px) and (max-width: 991px) {
    .chart-container {
        padding: 20px;
        max-width: 1200px;
    }
    
    .chart-content {
        gap: 20px;
    }
    
    .chart-area {
        min-height: 420px;
        padding: 18px;
    }
    
    .stats-card {
        min-width: 250px;
        padding: 18px;
    }
    
    .total-value {
        font-size: 22px;
    }
    
    .server-item {
        padding: 10px 12px;
    }
    
    .server-name {
        font-size: 13px;
    }
    
    .server-connections,
    .server-percentage {
        font-size: 11px;
        padding: 3px 6px;
    }
}

@media (min-width: 576px) and (max-width: 768px) {
    .chart-container {
        padding: 15px;
    }
    
    .chart-content {
        flex-direction: column;
        gap: 20px;
    }

    .chart-area {
        min-height: 380px;
        padding: 15px;
    }

    .stats-card {
        min-width: auto;
        padding: 18px;
    }

    .content {
        gap: 25px;
    }

    .chart-section {
        padding: 18px;
    }

    .header {
        flex-direction: row;
        justify-content: space-between;
        align-items: center;
    }
    
    .chart-container h2 {
        font-size: 18px;
    }

    .theme-toggle {
        position: static;
        width: 40px;
        height: 40px;
        font-size: 16px;
    }

    .server-item {
        padding: 12px 15px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-wrap: nowrap;
    }
    
    .server-name {
        font-size: 14px;
        flex: 1;
        margin-right: 10px;
        min-width: 0;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    
    .server-connections,
    .server-percentage {
        font-size: 12px;
        padding: 4px 8px;
        flex-shrink: 0;
        margin-left: 5px;
    }
    
    .total-info {
        flex-direction: column;
        text-align: center;
        gap: 5px;
    }
    
    .total-value {
        font-size: 20px;
    }
}

@media (max-width: 575px) {
    .chart-container {
        padding: 10px;
    }
    
    .chart-content {
        flex-direction: column;
        gap: 15px;
    }

    .chart-area {
        min-height: 320px;
        padding: 12px;
    }

    .stats-card {
        min-width: auto;
        padding: 15px;
    }

    .content {
        gap: 20px;
    }

    .chart-section {
        padding: 12px;
    }

    .header {
        flex-direction: column;
        gap: 15px;
        text-align: center;
    }
    
    .chart-container h2 {
        font-size: 16px;
        margin: 0;
    }

    .theme-toggle {
        position: static;
        width: 45px;
        height: 45px;
        font-size: 18px;
        margin: 0 auto;
    }
    
    .update-time {
        font-size: 12px;
        margin-bottom: 20px;
    }

    .chart-section h3 {
        font-size: 16px;
        margin-bottom: 15px;
    }

    .server-item {
        padding: 10px 12px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-wrap: nowrap;
        min-height: 50px;
    }
    
    .server-name {
        font-size: 13px;
        font-weight: 600;
        flex: 1;
        margin-right: 8px;
        min-width: 0;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    
    .server-connections,
    .server-percentage {
        font-size: 11px;
        padding: 3px 8px;
        flex-shrink: 0;
        margin-left: 4px;
    }
    
    .total-info {
        flex-direction: column;
        text-align: center;
        gap: 8px;
        padding: 12px;
    }
    
    .total-label {
        font-size: 12px;
    }
    
    .total-value {
        font-size: 18px;
    }
    
    .stats-card h4 {
        font-size: 14px;
        text-align: center;
        margin-bottom: 12px;
    }
    
    .server-list {
        max-height: 250px;
    }
}
</style>
