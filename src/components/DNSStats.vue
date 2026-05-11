<template>
  <div class="dns-stats-container">
    <div class="dns-stats-header">
      <h3 class="text-lg font-bold">🔍 DNS 查询统计</h3>
      <div class="dns-stats-controls flex gap-2">
        <button
          @click="refresh"
          class="px-3 py-1 bg-blue-500 text-white rounded hover:bg-blue-600 text-sm"
        >
          🔄 刷新
        </button>
        <button
          @click="clearStats"
          class="px-3 py-1 bg-red-500 text-white rounded hover:bg-red-600 text-sm"
        >
          🗑️ 清空
        </button>
      </div>
    </div>

    <div class="dns-stats-grid grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
      <div class="stat-card bg-white p-4 rounded shadow">
        <div class="text-xs text-gray-500 uppercase mb-2">总查询</div>
        <div class="text-2xl font-bold">{{ summary.total_queries || 0 }}</div>
      </div>
      <div class="stat-card bg-white p-4 rounded shadow">
        <div class="text-xs text-gray-500 uppercase mb-2">成功</div>
        <div class="text-2xl font-bold text-green-600">{{ summary.success_queries || 0 }}</div>
      </div>
      <div class="stat-card bg-white p-4 rounded shadow">
        <div class="text-xs text-gray-500 uppercase mb-2">失败</div>
        <div class="text-2xl font-bold text-red-600">{{ summary.failed_queries || 0 }}</div>
      </div>
      <div class="stat-card bg-white p-4 rounded shadow">
        <div class="text-xs text-gray-500 uppercase mb-2">平均延迟</div>
        <div class="text-2xl font-bold text-blue-600">{{ summary.avg_latency_ms || 0 }}ms</div>
      </div>
    </div>

    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <!-- Top Domains -->
      <div class="bg-white p-4 rounded shadow">
        <h4 class="font-bold mb-4">Top 10 域名</h4>
        <div class="space-y-2 max-h-64 overflow-y-auto">
          <div
            v-for="domain in summary.top_domains"
            :key="domain.domain"
            class="flex items-center gap-2"
          >
            <div class="w-24 text-sm truncate" :title="domain.domain">{{ domain.domain }}</div>
            <div
              class="flex-1 h-5 bg-gradient-to-r from-blue-400 to-purple-500 rounded"
              :style="{ width: `${(domain.count / (summary.top_domains?.[0]?.count || 1)) * 100}%` }"
            ></div>
            <div class="w-12 text-right text-sm font-bold">{{ domain.count }}</div>
          </div>
          <div v-if="!summary.top_domains?.length" class="text-gray-400 text-center py-8">
            暂无数据
          </div>
        </div>
      </div>

      <!-- Top Transports -->
      <div class="bg-white p-4 rounded shadow">
        <h4 class="font-bold mb-4">Top 5 DNS 服务器</h4>
        <div class="space-y-2 max-h-64 overflow-y-auto">
          <div
            v-for="transport in summary.top_transports"
            :key="transport.transport"
            class="flex items-center gap-2"
          >
            <div class="w-24 text-sm truncate">{{ transport.transport }}</div>
            <div
              class="flex-1 h-5 bg-gradient-to-r from-blue-400 to-purple-500 rounded"
              :style="{ width: `${(transport.count / (summary.top_transports?.[0]?.count || 1)) * 100}%` }"
            ></div>
            <div class="w-12 text-right text-sm font-bold">{{ transport.count }}</div>
          </div>
          <div v-if="!summary.top_transports?.length" class="text-gray-400 text-center py-8">
            暂无数据
          </div>
        </div>
      </div>
    </div>

    <!-- Rcode Stats -->
    <div class="bg-white p-4 rounded shadow mt-6">
      <h4 class="font-bold mb-4">响应码分布</h4>
      <div class="flex flex-wrap gap-2">
        <span
          v-for="(count, rcode) in summary.rcode_stats"
          :key="rcode"
          :class="[
            'px-3 py-1 rounded text-sm font-semibold',
            rcode === 'NOERROR'
              ? 'bg-green-100 text-green-800'
              : rcode.includes('ERR')
                ? 'bg-red-100 text-red-800'
                : 'bg-yellow-100 text-yellow-800'
          ]"
        >
          {{ rcode }}: {{ count }}
        </span>
        <div v-if="!Object.keys(summary.rcode_stats || {}).length" class="text-gray-400">
          暂无数据
        </div>
      </div>
    </div>

    <!-- Qtype Stats -->
    <div class="bg-white p-4 rounded shadow mt-6">
      <h4 class="font-bold mb-4">查询类型分布</h4>
      <div class="flex flex-wrap gap-2">
        <span
          v-for="(count, qtype) in summary.qtype_stats"
          :key="qtype"
          class="px-3 py-1 rounded text-sm font-semibold bg-yellow-100 text-yellow-800"
        >
          {{ qtype }}: {{ count }}
        </span>
        <div v-if="!Object.keys(summary.qtype_stats || {}).length" class="text-gray-400">
          暂无数据
        </div>
      </div>
    </div>

    <!-- Recent Queries -->
    <div class="bg-white p-4 rounded shadow mt-6">
      <h4 class="font-bold mb-4">最近查询</h4>
      <div class="overflow-x-auto">
        <table class="w-full text-sm">
          <thead class="bg-gray-100">
            <tr>
              <th class="px-4 py-2 text-left">时间</th>
              <th class="px-4 py-2 text-left">域名</th>
              <th class="px-4 py-2 text-left">类型</th>
              <th class="px-4 py-2 text-left">响应码</th>
              <th class="px-4 py-2 text-left">DNS 服务器</th>
              <th class="px-4 py-2 text-left">延迟 (ms)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(query, idx) in queries" :key="idx" class="border-b hover:bg-gray-50">
              <td class="px-4 py-2">{{ formatTime(query.timestamp) }}</td>
              <td class="px-4 py-2 truncate max-w-xs" :title="query.domain">{{ query.domain }}</td>
              <td class="px-4 py-2">{{ query.qtype }}</td>
              <td class="px-4 py-2">
                <span
                  :class="[
                    'px-2 py-1 rounded text-xs font-semibold',
                    query.rcode === 'NOERROR'
                      ? 'bg-green-100 text-green-800'
                      : 'bg-red-100 text-red-800'
                  ]"
                >
                  {{ query.rcode }}
                </span>
              </td>
              <td class="px-4 py-2">{{ query.transport }}</td>
              <td class="px-4 py-2">{{ query.latency_ms }}</td>
            </tr>
            <tr v-if="!queries.length">
              <td colspan="6" class="px-4 py-8 text-center text-gray-400">暂无查询记录</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface QueryStats {
  domain: string
  qtype: string
  rcode: string
  transport: string
  latency_ms: number
  timestamp: string
  client_ip?: string
}

interface Summary {
  total_queries: number
  success_queries: number
  failed_queries: number
  avg_latency_ms: number
  top_domains: Array<{ domain: string; count: number }>
  top_transports: Array<{ transport: string; count: number }>
  rcode_stats: Record<string, number>
  qtype_stats: Record<string, number>
  uptime_seconds: number
}

const API_BASE = '/api/dns/stats'
const summary = ref<Summary>({
  total_queries: 0,
  success_queries: 0,
  failed_queries: 0,
  avg_latency_ms: 0,
  top_domains: [],
  top_transports: [],
  rcode_stats: {},
  qtype_stats: {},
  uptime_seconds: 0
})
const queries = ref<QueryStats[]>([])
let refreshTimer: NodeJS.Timeout | null = null

const fetchData = async () => {
  try {
    const [summaryRes, queriesRes] = await Promise.all([
      fetch(`${API_BASE}/summary`),
      fetch(`${API_BASE}/queries?limit=30`)
    ])

    if (summaryRes.ok) {
      summary.value = await summaryRes.json()
    }

    if (queriesRes.ok) {
      const data = await queriesRes.json()
      queries.value = data.queries || []
    }
  } catch (err) {
    console.error('DNS Stats Error:', err)
  }
}

const refresh = () => {
  fetchData()
}

const clearStats = async () => {
  if (!confirm('确定要清空所有统计数据吗？')) return
  try {
    const res = await fetch(`${API_BASE}/`, { method: 'DELETE' })
    if (res.ok) {
      await fetchData()
      alert('统计数据已清空')
    }
  } catch (err) {
    console.error('Clear stats error:', err)
  }
}

const formatTime = (timestamp: string) => {
  return new Date(timestamp).toLocaleTimeString()
}

onMounted(() => {
  fetchData()
  refreshTimer = setInterval(fetchData, 2000)
})

onUnmounted(() => {
  if (refreshTimer) {
    clearInterval(refreshTimer)
  }
})
</script>

<style scoped>
.dns-stats-container {
  padding: 1.5rem;
  background: #f9fafb;
  border-radius: 0.5rem;
  margin: 1.5rem 0;
}

.dns-stats-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  border-bottom: 2px solid #e5e7eb;
  padding-bottom: 0.75rem;
}
</style>
