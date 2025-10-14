<template>
<div class="chart-container" :class="{ 'dark-theme': isDarkMode }">
    <div class="header">
    <h2>TaiwanFRP 伺服器監控面板</h2>
    <button @click="toggleTheme" class="theme-toggle" :title="isDarkMode ? '切換到明亮模式' : '切換到深色模式'">
        <span v-if="isDarkMode">☀️</span>
        <span v-else>🌙</span>
    </button>
    </div>
    <div class="version-info"><a href="https://github.com/redbean0721/taiwanfrp-chart-ui" target="_blank">UI Version: {{ version }}</a></div>
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
        <h3>FRP 客戶端數量分佈</h3>
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

    <!-- 歷史連線數據表格 -->
    <div class="chart-section">
        <div class="history-header">
            <h3>伺服器連線數歷史資料</h3>
            <button @click="toggleHistoryExpanded" class="expand-btn">
                <span v-if="isHistoryExpanded">🔼 收起</span>
                <span v-else>🔽 展開</span>
            </button>
        </div>
        
        <div v-if="isHistoryExpanded" class="history-content">
            <div class="table-controls">
            <div class="control-group">
                <label for="versionSelect">版本:</label>
                <select id="versionSelect" v-model="selectedVersion" @change="fetchHistoryData">
                    <option value="0.63.0">0.63.0</option>
                </select>
            </div>
            <div class="control-group">
                <label for="nodeSelect">節點:</label>
                <select id="nodeSelect" v-model="selectedNode" @change="fetchHistoryData">
                    <option value="all">全部</option>
                    <option v-for="node in availableNodes" :key="node" :value="node">{{ node }}</option>
                </select>
            </div>
            <div class="control-group">
                <label for="numInput">資料筆數:</label>
                <input id="numInput" type="number" v-model.number="dataCount" @change="fetchHistoryData" min="1" max="100" />
            </div>
            <div class="control-group">
                <label for="startTime">開始時間:</label>
                <input id="startTime" type="datetime-local" v-model="startTime" @change="fetchHistoryData" />
            </div>
            <div class="control-group">
                <label for="endTime">結束時間:</label>
                <input id="endTime" type="datetime-local" v-model="endTime" @change="fetchHistoryData" />
            </div>
            <button @click="resetTimeRange" class="reset-btn">重置時間</button>
        </div>
        
        <div v-if="historyLoading" class="loading">載入歷史資料中...</div>
        <div v-else-if="historyError" class="error">{{ historyError }}</div>
        <div v-else-if="historyData">
            <!-- 歷史資料折線圖 -->
            <div class="line-chart-container">
                <!-- 桌面版或啟用彩蛋後：顯示完整圖表 -->
                <div class="desktop-chart" :class="{ 'mobile-chart-enabled': showMobileChart }">
                    <div class="chart-display-controls">
                        <h4>📈 連線數趨勢圖</h4>
                        <div class="display-options">
                            <label class="checkbox-label">
                                <input type="checkbox" v-model="chartDisplayOptions" value="connections" />
                                <span class="checkmark"></span>
                                連線數
                            </label>
                            <label class="checkbox-label">
                                <input type="checkbox" v-model="chartDisplayOptions" value="clients" />
                                <span class="checkmark"></span>
                                FRP客戶端數
                            </label>
                        </div>
                    </div>
                    <div class="line-chart-area">
                        <Line :data="historyLineChartData" :options="historyLineChartOptions" />
                    </div>
                    <!-- 彩蛋啟用提示 -->
                    <div v-if="showMobileChart" class="easter-egg-notice">
                        <!-- 🎉 隱藏功能已啟用！現在可以在手機上查看趨勢圖了 -->
                        既然你這麼想看，那就給你看好了
                        <br>
                        <img src="https://cdn.discordapp.com/emojis/1142407657890787411.webp?size=48" alt="" srcset="">
                    </div>
                </div>
                
                <!-- 手機版：顯示提示訊息（除非彩蛋被啟用） -->
                <div class="mobile-chart-notice" v-show="!showMobileChart">
                    <div class="notice-icon">💻</div>
                    <h4>趨勢圖僅支援電腦版瀏覽</h4>
                    <p>為了獲得最佳的圖表顯示效果，<br>請使用電腦瀏覽器查看連線數趨勢圖</p>
                    <div class="notice-hint" @click="handleEasterEggClick" style="cursor: pointer; user-select: none;">
                        📊 歷史資料表格仍可在下方查看
                        <span v-if="easterEggClickCount > 0" class="click-indicator"> ({{ easterEggClickCount }}/3)</span>
                    </div>
                </div>
            </div>
            
            <div class="table-container">
            <table class="history-table">
                <thead>
                    <tr>
                        <th>伺服器</th>
                        <th>記錄時間</th>
                        <th>當前連線數</th>
                        <th>客戶端數量</th>
                        <th>TCP</th>
                        <th>UDP</th>
                        <th>HTTP</th>
                        <th>HTTPS</th>
                        <th>STCP</th>
                        <th>SUDP</th>
                        <th>版本</th>
                        <th>線上狀態</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="item in flattenedHistoryData" :key="`${item.server}-${item.recorded_at}`" 
                        :class="{ 'offline': !item.is_online }">
                        <td class="server-name-cell" style="padding-left: 10px;">{{ item.server }}</td>
                        <td class="time-cell">{{ formatTime(item.recorded_at) }}</td>
                        <td class="number-cell">{{ item.cur_conns }}</td>
                        <td class="number-cell">{{ item.client_counts }}</td>
                        <td class="number-cell">{{ item.tcp_count }}</td>
                        <td class="number-cell">{{ item.udp_count }}</td>
                        <td class="number-cell">{{ item.http_count }}</td>
                        <td class="number-cell">{{ item.https_count }}</td>
                        <td class="number-cell">{{ item.stcp_count }}</td>
                        <td class="number-cell">{{ item.sudp_count }}</td>
                        <td class="version-cell">{{ item.version }}</td>
                        <td class="status-cell">
                            <span :class="['status-badge', item.is_online ? 'online' : 'offline']">
                                {{ item.is_online ? '線上' : '離線' }}
                            </span>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
        </div>
    </div>
    </div>
</div>
</div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { Pie, Line } from 'vue-chartjs'
import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    ArcElement,
    CategoryScale,
    LinearScale,
    PointElement,
    LineElement
} from 'chart.js'
import packageInfo from '../../package.json'

// 註冊 Chart.js 組件
ChartJS.register(Title, Tooltip, Legend, ArcElement, CategoryScale, LinearScale, PointElement, LineElement)

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

interface HistoryItem extends ServerData {
    server: string
}

const loading = ref(true)
const error = ref('')
const apiData = ref<ApiResponse | null>(null)
const lastUpdateTime = ref('now')
const isDarkMode = ref(false)

// 歷史資料相關狀態
const historyLoading = ref(false)
const historyError = ref('')
const historyData = ref<ApiResponse | null>(null)
const selectedVersion = ref('0.63.0')
const selectedNode = ref('all')
const dataCount = ref(36)
const startTime = ref('')
const endTime = ref('')
const isHistoryExpanded = ref(false)
const chartDisplayOptions = ref(['connections', 'clients']) // 圖表顯示選項

// 彩蛋功能：點擊計數器
const easterEggClickCount = ref(0)
const showMobileChart = ref(false) // 控制是否在手機版顯示圖表

// 彩蛋點擊處理
const handleEasterEggClick = () => {
    easterEggClickCount.value++
    
    if (easterEggClickCount.value >= 3) {
        showMobileChart.value = true
        // 3秒後重置計數器（但保持圖表顯示）
        setTimeout(() => {
            easterEggClickCount.value = 0
        }, 3000)
    } else {
        // 如果1.5秒內沒有繼續點擊，重置計數器
        setTimeout(() => {
            if (easterEggClickCount.value < 3) {
                easterEggClickCount.value = 0
            }
        }, 1500)
    }
}

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

// 計算可用節點列表
const availableNodes = computed(() => {
    if (!apiData.value) return []
    return Object.keys(apiData.value.result)
})

// 扁平化歷史資料
const flattenedHistoryData = computed(() => {
    if (!historyData.value) return []
    
    const flattened: HistoryItem[] = []
    Object.entries(historyData.value.result).forEach(([serverName, serverArray]) => {
        serverArray.forEach(item => {
            flattened.push({
                ...item,
                server: serverName
            })
        })
    })
    
    // 按時間排序，最新的在前
    return flattened.sort((a, b) => new Date(b.recorded_at).getTime() - new Date(a.recorded_at).getTime())
})

// 計算歷史折線圖資料
const historyLineChartData = computed(() => {
    if (!historyData.value) return { labels: [], datasets: [] }
    
    // 按伺服器分組資料
    const serverDataMap: Record<string, HistoryItem[]> = {}
    Object.entries(historyData.value.result).forEach(([serverName, serverArray]) => {
        serverDataMap[serverName] = serverArray.map(item => ({
            ...item,
            server: serverName
        })).sort((a, b) => new Date(a.recorded_at).getTime() - new Date(b.recorded_at).getTime())
    })
    
    // 取得所有時間點
    const allTimes = new Set<string>()
    Object.values(serverDataMap).forEach(serverArray => {
        serverArray.forEach(item => {
            allTimes.add(item.recorded_at)
        })
    })
    
    const sortedTimes = Array.from(allTimes).sort()
    const labels = sortedTimes.map(time => {
        const date = new Date(time)
        return date.toLocaleString('zh-TW', {
            month: '2-digit',
            day: '2-digit',
            hour: '2-digit',
            minute: '2-digit'
        })
    })
    
    // 為每個伺服器和每個資料類型生成資料集
    const datasets: any[] = []
    const serverNames = Object.keys(serverDataMap)
    const colors = generateColors(serverNames.length * 2) // 為連線數和客戶端數預留顏色
    
    serverNames.forEach((serverName, serverIndex) => {
        const serverArray = serverDataMap[serverName]
        
        // 連線數資料集
        if (chartDisplayOptions.value.includes('connections')) {
            const connectionsData = sortedTimes.map(time => {
                const item = serverArray.find(s => s.recorded_at === time)
                return item ? item.cur_conns : null
            })
            
            datasets.push({
                label: `${serverName} - 連線數`,
                data: connectionsData,
                borderColor: colors[serverIndex * 2],
                backgroundColor: colors[serverIndex * 2] + '20',
                fill: false,
                tension: 0.1,
                pointRadius: 0,
                pointHoverRadius: 4,
                pointBackgroundColor: colors[serverIndex * 2],
                pointBorderColor: '#fff',
                pointBorderWidth: 2
            })
        }
        
        // FRP客戶端數資料集
        if (chartDisplayOptions.value.includes('clients')) {
            const clientsData = sortedTimes.map(time => {
                const item = serverArray.find(s => s.recorded_at === time)
                return item ? item.client_counts : null
            })
            
            datasets.push({
                label: `${serverName} - FRP客戶端`,
                data: clientsData,
                borderColor: colors[serverIndex * 2 + 1],
                backgroundColor: colors[serverIndex * 2 + 1] + '20',
                fill: false,
                tension: 0.1,
                pointRadius: 0,
                pointHoverRadius: 4,
                pointBackgroundColor: colors[serverIndex * 2 + 1],
                pointBorderColor: '#fff',
                pointBorderWidth: 2,
                borderDash: [5, 5] // 使用虛線來區分客戶端數
            })
        }
    })
    
    return {
        labels,
        datasets
    }
})

// 歷史折線圖選項
const historyLineChartOptions = computed(() => ({
    responsive: true,
    maintainAspectRatio: false,
    interaction: {
        intersect: false,
        mode: 'index' as const
    },
    plugins: {
        title: {
            display: true,
            text: '伺服器連線數歷史趨勢',
            color: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
            font: {
                size: 16,
                weight: 'bold' as const
            }
        },
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
                title: function(context: any) {
                    if (context.length > 0) {
                        const dataIndex = context[0].dataIndex
                        const sortedTimes = Array.from(new Set(
                            Object.values(historyData.value?.result || {}).flat().map(item => item.recorded_at)
                        )).sort()
                        return new Date(sortedTimes[dataIndex]).toLocaleString('zh-TW')
                    }
                    return ''
                },
                label: function(context: any) {
                    const label = context.dataset.label || ''
                    const value = context.raw
                    return value !== null ? `${label}: ${value} 連線` : `${label}: 無資料`
                }
            }
        }
    },
    scales: {
        x: {
            title: {
                display: true,
                text: '時間',
                color: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
                font: {
                    size: 14,
                    weight: 'bold' as const
                }
            },
            ticks: {
                color: isDarkMode.value ? '#a0aec0' : '#666',
                maxTicksLimit: 8
            },
            grid: {
                color: isDarkMode.value ? '#4a5568' : '#e5e7eb'
            }
        },
        y: {
            title: {
                display: true,
                text: '連線數',
                color: isDarkMode.value ? '#e2e8f0' : '#2c3e50',
                font: {
                    size: 14,
                    weight: 'bold' as const
                }
            },
            ticks: {
                color: isDarkMode.value ? '#a0aec0' : '#666',
                stepSize: 1
            },
            grid: {
                color: isDarkMode.value ? '#4a5568' : '#e5e7eb'
            },
            beginAtZero: true
        }
    }
}))

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
// @ts-ignore
const _fetchData = async () => {
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

// 獲取歷史資料
const fetchHistoryData = async () => {
    try {
        historyLoading.value = true
        historyError.value = ''
        
        let url = 'https://api.redbean0721.com/api/frp/monitor/query?'
        const params = new URLSearchParams()
        
        if (selectedVersion.value) {
            params.append('version', selectedVersion.value)
        }
        if (selectedNode.value) {
            params.append('node', selectedNode.value)
        }
        if (dataCount.value) {
            params.append('num', dataCount.value.toString())
        }
        if (startTime.value) {
            const formattedStartTime = startTime.value.replace('T', ' ') + ':00'
            params.append('start_time', formattedStartTime)
        }
        if (endTime.value) {
            const formattedEndTime = endTime.value.replace('T', ' ') + ':00'
            params.append('end_time', formattedEndTime)
        }
        
        url += params.toString()
        
        const response = await fetch(url)
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`)
        }
        
        const data = await response.json()
        historyData.value = data
    } catch (err) {
        historyError.value = err instanceof Error ? err.message : '獲取歷史資料時發生錯誤'
        console.error('Error fetching history data:', err)
    } finally {
        historyLoading.value = false
    }
}

// 重置時間範圍
const resetTimeRange = () => {
    startTime.value = ''
    endTime.value = ''
    fetchHistoryData()
}

// 切換歷史資料展開/收起狀態
const toggleHistoryExpanded = () => {
    isHistoryExpanded.value = !isHistoryExpanded.value
    // 如果是第一次展開且還沒有資料，則獲取資料
    if (isHistoryExpanded.value && !historyData.value) {
        fetchHistoryData()
    }
}

// 格式化時間
const formatTime = (timeString: string) => {
    const date = new Date(timeString)
    return date.toLocaleString('zh-TW', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        second: '2-digit'
    })
}

// 獲取 SSE 資料
const initSSE = () => {
    const eventSource = new EventSource("https://api.redbean0721.com/api/frp/monitor/query/sse?node=all&num=8")

    eventSource.onmessage = (event) => {
        try {
            const data = JSON.parse(event.data)
            apiData.value = data    // 覆蓋上一份資料, 防止記憶體洩漏
            lastUpdateTime.value = new Date().toLocaleTimeString("zh-TW")
            loading.value = false
        } catch (err) {
            console.error("Error parsing SSE data:", err)
            error.value = "解析 SSE 資料時發生錯誤"
        }
    }

    eventSource.onerror = (err) => {
        console.error("SSE error:", err)
        error.value = "SSE 連線錯誤，正在重試..."
        eventSource.close()
        // 簡單自動重連機制
        setTimeout(initSSE, 5000)
    }
}

// 組件掛載時獲取資料
onMounted(() => {
    loadTheme()
    // fetchData()
    initSSE()
    
    // 歷史資料在用戶展開時才獲取
    // fetchHistoryData()

    // 每 30 秒更新一次資料
    // setInterval(fetchData, 30000)
})
</script>

<style scoped src="../assets/FrpChart.css"></style>
