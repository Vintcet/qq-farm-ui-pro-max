<script setup lang="ts">
import { useStorage } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { onMounted, ref, watch } from 'vue'
import api from '@/api'
import BaseSelect from '@/components/ui/BaseSelect.vue'
import { useAccountStore } from '@/stores/account'

const accountStore = useAccountStore()
const { currentAccountId } = storeToRefs(accountStore)

const loading = ref(false)
const list = ref<any[]>([])
const sortKey = useStorage('analytics_sort_key', 'exp')
const imageErrors = ref<Record<string | number, boolean>>({})

const sortOptions = [
  { value: 'exp', label: '经验/小时' },
  { value: 'fert', label: '普通肥经验/小时' },
  { value: 'profit', label: '利润/小时' },
  { value: 'fert_profit', label: '普通肥利润/小时' },
  { value: 'level', label: '等级' },
]

async function loadAnalytics() {
  if (!currentAccountId.value)
    return
  loading.value = true
  try {
    const res = await api.get(`/api/analytics`, {
      params: { sort: sortKey.value },
      headers: { 'x-account-id': currentAccountId.value },
    })
    const data = res.data.data
    if (Array.isArray(data)) {
      list.value = data
      // web sort as fallback
      const metricMap: Record<string, string> = {
        exp: 'expPerHour',
        fert: 'normalFertilizerExpPerHour',
        profit: 'profitPerHour',
        fert_profit: 'normalFertilizerProfitPerHour',
        level: 'level',
      }
      const metric = metricMap[sortKey.value]
      if (metric) {
        list.value.sort((a, b) => {
          const av = Number(a[metric])
          const bv = Number(b[metric])
          if (!Number.isFinite(av) && !Number.isFinite(bv))
            return 0
          if (!Number.isFinite(av))
            return 1
          if (!Number.isFinite(bv))
            return -1
          return bv - av
        })
      }
    }
    else {
      list.value = []
    }
  }
  catch (e) {
    console.error(e)
    list.value = []
  }
  finally {
    loading.value = false
  }
}

onMounted(() => {
  loadAnalytics()
})

watch([currentAccountId, sortKey], () => {
  loadAnalytics()
})

function formatLv(level: any) {
  if (level === null || level === undefined || level === '' || Number(level) < 0)
    return '未知'
  return String(level)
}

function formatGrowTime(seconds: any) {
  const s = Number(seconds)
  if (!Number.isFinite(s) || s <= 0)
    return '0秒'
  if (s < 60)
    return `${s}秒`
  if (s < 3600) {
    const mins = Math.floor(s / 60)
    const secs = s % 60
    return secs > 0 ? `${mins}分${secs}秒` : `${mins}分`
  }
  const hours = Math.floor(s / 3600)
  const mins = Math.floor((s % 3600) / 60)
  return mins > 0 ? `${hours}时${mins}分` : `${hours}时`
}
</script>

<template>
  <div class="p-4">
    <div class="mb-4 flex flex-col justify-between gap-4 sm:flex-row sm:items-center">
      <h2 class="flex items-center gap-2 text-2xl font-bold">
        <div class="i-carbon-catalog" />
        作物图鉴
      </h2>

      <div class="flex items-center gap-2">
        <label class="whitespace-nowrap text-sm font-medium">排序方式:</label>
        <BaseSelect
          v-model="sortKey"
          :options="sortOptions"
          class="w-40"
        />
      </div>
    </div>

    <div v-if="loading" class="flex justify-center py-12">
      <div class="i-svg-spinners-90-ring-with-bg text-4xl text-blue-500" />
    </div>

    <div v-else-if="!currentAccountId" class="glass-panel glass-text-muted rounded-xl p-8 text-center shadow-md">
      请选择账号后查看数据分析
    </div>

    <div v-else-if="list.length === 0" class="glass-panel glass-text-muted rounded-xl p-8 text-center shadow-md">
      暂无数据
    </div>

    <div v-else>
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
        <div
          v-for="(item, idx) in list"
          :key="idx"
          class="glass-panel group overflow-hidden rounded-xl shadow transition-all hover:-translate-y-1 hover:shadow-lg dark:hover:bg-white/5 flex flex-col"
        >
          <!-- 卡片内容主体: 允许点击放大或高亮交互 -->
          <div class="p-4 cursor-pointer flex flex-col flex-1 gap-4 transition bg-transparent">
            
            <!-- 头部：图鉴图标 + 名称 + 核心状态 -->
            <div class="flex flex-row items-center gap-3">
              <!-- 作物图片 -->
              <div class="relative h-12 w-12 flex shrink-0 items-center justify-center overflow-hidden border border-white/20 rounded-lg bg-primary-500/10 dark:border-white/10 dark:bg-black/20 group-hover:bg-primary-500/20 transition-colors">
                <img
                  v-if="item.image && !imageErrors[item.seedId]"
                  :src="item.image"
                  class="h-10 w-10 object-contain drop-shadow-sm group-hover:scale-110 transition-transform"
                  loading="lazy"
                  @error="imageErrors[item.seedId] = true"
                >
                <div v-else class="i-carbon-sprout text-2xl text-primary-500/50" />
              </div>
              
              <!-- 文本信息 -->
              <div class="min-w-0 flex-1">
                <div class="flex items-center justify-between">
                  <div class="glass-text-main truncate font-bold text-base group-hover:text-primary-500 transition-colors">
                    {{ item.name }}
                  </div>
                  <div class="glass-text-muted text-[10px] font-mono">
                    ID:{{ item.seedId }}
                  </div>
                </div>
                <div class="mt-1 flex flex-wrap items-center gap-2">
                  <span class="border border-primary-500/20 rounded bg-primary-500/10 px-1.5 py-0.5 text-[10px] font-semibold text-primary-600 dark:text-primary-400">Lv {{ formatLv(item.level) }}</span>
                  <span class="border border-yellow-500/20 rounded bg-yellow-500/10 px-1.5 py-0.5 text-[10px] font-semibold text-yellow-600 dark:text-yellow-400">{{ item.seasons }}季</span>
                  <span class="glass-text-muted text-[10px] flex items-center gap-0.5 ml-auto">
                    <div class="i-carbon-time" />
                    {{ formatGrowTime(item.growTime) }}
                  </span>
                </div>
              </div>
            </div>

            <!-- 数据列 (2x2 网格) -->
            <div class="grid grid-cols-2 gap-3 text-sm mt-auto pt-2 border-t border-gray-200/50 dark:border-white/10">
              <div class="flex flex-col">
                <span class="glass-text-muted mb-0.5 text-[10px] uppercase opacity-80">经验/小时</span>
                <span class="text-purple-600 font-bold dark:text-purple-400 leading-none">{{ item.expPerHour }}</span>
              </div>
              <div class="flex flex-col text-right">
                <span class="glass-text-muted mb-0.5 text-[10px] uppercase opacity-80">净利润/小时</span>
                <span class="text-amber-500 font-bold dark:text-amber-400 leading-none">{{ item.profitPerHour ?? '-' }}</span>
              </div>
              <div class="flex flex-col">
                <span class="glass-text-muted mb-0.5 text-[10px] uppercase opacity-80">普肥经验/小时</span>
                <span class="text-blue-600 font-bold dark:text-blue-400 leading-none">{{ item.normalFertilizerExpPerHour ?? '-' }}</span>
              </div>
              <div class="flex flex-col text-right">
                <span class="glass-text-muted mb-0.5 text-[10px] uppercase opacity-80">普肥净利润/小时</span>
                <span class="text-primary-600 font-bold dark:text-primary-400 leading-none">{{ item.normalFertilizerProfitPerHour ?? '-' }}</span>
              </div>
            </div>
            
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
