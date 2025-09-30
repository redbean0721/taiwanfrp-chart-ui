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
                        <td class="server-name-cell">{{ item.server }}</td>
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

/* 歷史資料標頭樣式 */
.history-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
}

.history-header h3 {
    margin: 0;
}

.expand-btn {
    padding: 8px 16px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 600;
    transition: all 0.3s ease;
    box-shadow: 0 2px 4px rgba(102, 126, 234, 0.3);
}

.expand-btn:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 8px rgba(102, 126, 234, 0.4);
}

.expand-btn:active {
    transform: translateY(0);
}

.history-content {
    animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
    from {
        opacity: 0;
        transform: translateY(-10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

/* 折線圖樣式 */
.line-chart-container {
    margin-bottom: 30px;
    padding: 20px;
    background: rgba(248, 249, 250, 0.8);
    border-radius: 8px;
    border: 1px solid rgba(0, 0, 0, 0.08);
}

.line-chart-container h4 {
    color: #2c3e50;
    margin-bottom: 15px;
    font-size: 16px;
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 8px;
    justify-content: center;
}

.line-chart-area {
    height: 400px;
    background: white;
    border-radius: 8px;
    padding: 15px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

/* 圖表顯示控制項樣式 */
.chart-display-controls {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
    padding: 15px;
    background: rgba(248, 249, 250, 0.8);
    border-radius: 8px;
    border: 1px solid rgba(0, 0, 0, 0.08);
}

.chart-display-controls h4 {
    margin: 0;
    color: #2c3e50;
    font-size: 16px;
    font-weight: 600;
    display: flex;
    align-items: center;
    gap: 8px;
}

.display-options {
    display: flex;
    gap: 20px;
    align-items: center;
}

.checkbox-label {
    display: flex;
    align-items: center;
    gap: 8px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
    color: #2c3e50;
    user-select: none;
}

.checkbox-label input[type="checkbox"] {
    appearance: none;
    width: 18px;
    height: 18px;
    border: 2px solid #ddd;
    border-radius: 4px;
    background: white;
    cursor: pointer;
    position: relative;
    transition: all 0.3s ease;
}

.checkbox-label input[type="checkbox"]:checked {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border-color: #667eea;
}

.checkbox-label input[type="checkbox"]:checked::after {
    content: '✓';
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    font-size: 12px;
    font-weight: bold;
}

.checkbox-label:hover input[type="checkbox"] {
    border-color: #667eea;
    box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.2);
}

/* 表格控制項樣式 */
.table-controls {
    display: flex;
    flex-wrap: wrap;
    gap: 15px;
    margin-bottom: 20px;
    padding: 20px;
    background: rgba(248, 249, 250, 0.8);
    border-radius: 8px;
    border: 1px solid rgba(0, 0, 0, 0.08);
}

.control-group {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.control-group label {
    font-size: 12px;
    font-weight: 600;
    color: #2c3e50;
}

.control-group select,
.control-group input {
    padding: 8px 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 14px;
    background-color: white;
    transition: border-color 0.3s ease;
}

.control-group select:focus,
.control-group input:focus {
    outline: none;
    border-color: #3498db;
    box-shadow: 0 0 0 2px rgba(52, 152, 219, 0.2);
}

.reset-btn {
    padding: 8px 16px;
    background: #e74c3c;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 600;
    transition: background-color 0.3s ease;
    align-self: flex-end;
}

.reset-btn:hover {
    background: #c0392b;
}

/* 表格樣式 */
.table-container {
    overflow-x: auto;
    border-radius: 8px;
    border: 1px solid rgba(0, 0, 0, 0.08);
    background: white;
}

.history-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 14px;
}

.history-table th {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    padding: 12px 8px;
    text-align: left;
    font-weight: 600;
    font-size: 12px;
    white-space: nowrap;
    position: sticky;
    top: 0;
    z-index: 1;
    text-align: center;
}

.history-table td {
    padding: 10px 8px;
    border-bottom: 1px solid #f0f0f0;
    font-size: 12px;
}

.history-table tr:hover {
    background-color: #f8f9fa;
}

.history-table tr.offline {
    background-color: #ffeaa7;
}

.history-table tr.offline:hover {
    background-color: #fdcb6e;
}

.server-name-cell {
    font-weight: 600;
    color: #2c3e50;
    min-width: 100px;
}

.time-cell {
    color: #666;
    min-width: 140px;
    font-family: monospace;
}

.number-cell {
    text-align: center;
    color: #2980b9;
    font-weight: 600;
    min-width: 60px;
}

.version-cell {
    color: #8e44ad;
    font-family: monospace;
    min-width: 80px;
}

.status-cell {
    text-align: center;
    min-width: 80px;
}

.status-badge {
    padding: 4px 8px;
    border-radius: 12px;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.status-badge.online {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.status-badge.offline {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
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

/* 深色模式歷史標頭樣式 */
.dark-theme .expand-btn {
    background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
    box-shadow: 0 2px 4px rgba(79, 70, 229, 0.3);
}

.dark-theme .expand-btn:hover {
    box-shadow: 0 4px 8px rgba(79, 70, 229, 0.4);
}

/* 深色模式折線圖樣式 */
.dark-theme .line-chart-container {
    background: rgba(55, 65, 81, 0.8);
    border-color: rgba(255, 255, 255, 0.1);
}

.dark-theme .line-chart-container h4 {
    color: #e2e8f0;
}

.dark-theme .line-chart-area {
    background: #2d3748;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

/* 深色模式圖表顯示控制項樣式 */
.dark-theme .chart-display-controls {
    background: rgba(55, 65, 81, 0.8);
    border-color: rgba(255, 255, 255, 0.1);
}

.dark-theme .chart-display-controls h4 {
    color: #e2e8f0;
}

.dark-theme .checkbox-label {
    color: #e2e8f0;
}

.dark-theme .checkbox-label input[type="checkbox"] {
    border-color: #4a5568;
    background: #374151;
}

.dark-theme .checkbox-label input[type="checkbox"]:checked {
    background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
    border-color: #4f46e5;
}

.dark-theme .checkbox-label:hover input[type="checkbox"] {
    border-color: #4f46e5;
    box-shadow: 0 0 0 2px rgba(79, 70, 229, 0.2);
}

/* 深色模式表格樣式 */
.dark-theme .table-controls {
    background: rgba(55, 65, 81, 0.8);
    border-color: rgba(255, 255, 255, 0.1);
}

.dark-theme .control-group label {
    color: #e2e8f0;
}

.dark-theme .control-group select,
.dark-theme .control-group input {
    background-color: #374151;
    border-color: #4a5568;
    color: #e2e8f0;
}

.dark-theme .control-group select:focus,
.dark-theme .control-group input:focus {
    border-color: #63b3ed;
    box-shadow: 0 0 0 2px rgba(99, 179, 237, 0.2);
}

.dark-theme .reset-btn {
    background: #e53e3e;
}

.dark-theme .reset-btn:hover {
    background: #c53030;
}

.dark-theme .table-container {
    border-color: rgba(255, 255, 255, 0.1);
    background: #2d3748;
}

.dark-theme .history-table th {
    /* background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%); */
    background: #7c3aed;
}

.dark-theme .history-table td {
    border-bottom-color: #4a5568;
}

.dark-theme .history-table tr:hover {
    background-color: #374151;
}

.dark-theme .history-table tr.offline {
    background-color: #374151;
}

.dark-theme .history-table tr.offline:hover {
    background-color: #4a5568;
}

.dark-theme .server-name-cell {
    color: #e2e8f0;
}

.dark-theme .time-cell {
    color: #a0aec0;
}

.dark-theme .number-cell {
    color: #63b3ed;
}

.dark-theme .version-cell {
    color: #d69e2e;
}

.dark-theme .status-badge.online {
    background-color: #276749;
    color: #9ae6b4;
    border-color: #38a169;
}

.dark-theme .status-badge.offline {
    background-color: #742a2a;
    color: #fc8181;
    border-color: #e53e3e;
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

    /* 表格控制項響應式 */
    .table-controls {
        gap: 10px;
        padding: 15px;
    }

    .control-group {
        min-width: 120px;
    }

    .control-group select,
    .control-group input {
        font-size: 13px;
        padding: 6px 8px;
    }

    .reset-btn {
        font-size: 13px;
        padding: 6px 12px;
    }

    .history-table {
        font-size: 12px;
    }

    .history-table th,
    .history-table td {
        padding: 8px 6px;
        font-size: 11px;
    }

    /* 折線圖響應式 */
    .line-chart-container {
        padding: 15px;
        margin-bottom: 20px;
    }

    .line-chart-container h4 {
        font-size: 14px;
    }

    .line-chart-area {
        height: 300px;
        padding: 10px;
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

    /* 小螢幕表格控制項 */
    .history-header {
        flex-direction: column;
        gap: 10px;
        text-align: center;
    }
    
    .expand-btn {
        font-size: 13px;
        padding: 6px 12px;
        align-self: center;
    }
    
    .history-header {
        flex-direction: column;
        gap: 12px;
        text-align: center;
    }
    
    .expand-btn {
        font-size: 14px;
        padding: 8px 16px;
        align-self: center;
        width: 100%;
        max-width: 200px;
    }
    
    /* 小螢幕圖表控制項樣式 */
    .chart-display-controls {
        flex-direction: column;
        gap: 10px;
        padding: 12px;
        text-align: center;
    }
    
    .display-options {
        justify-content: center;
        gap: 15px;
    }
    
    .checkbox-label {
        font-size: 13px;
    }
    
    .table-controls {
        flex-direction: column;
        gap: 12px;
        padding: 12px;
    }

    .control-group {
        min-width: auto;
    }

    .control-group select,
    .control-group input {
        font-size: 14px;
        padding: 8px 10px;
        width: 100%;
    }

    .reset-btn {
        font-size: 14px;
        padding: 8px 16px;
        align-self: stretch;
    }

    .history-table {
        font-size: 11px;
    }

    .history-table th,
    .history-table td {
        padding: 6px 4px;
        font-size: 10px;
    }

    .server-name-cell {
        min-width: 80px;
    }

    .time-cell {
        min-width: 100px;
        font-size: 9px;
    }

    .number-cell {
        min-width: 40px;
    }

    .version-cell {
        min-width: 60px;
    }

    .status-cell {
        min-width: 60px;
    }

    .status-badge {
        font-size: 9px;
        padding: 2px 6px;
    }

    /* 小螢幕折線圖樣式 */
    .line-chart-container {
        padding: 12px;
        margin-bottom: 15px;
    }

    .line-chart-container h4 {
        font-size: 13px;
        margin-bottom: 10px;
    }

    .line-chart-area {
        height: 250px;
        padding: 8px;
    }
}
</style>
