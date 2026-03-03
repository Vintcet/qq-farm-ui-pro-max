<script setup lang="ts">
import { onMounted, ref } from 'vue'
import api from '@/api'
import BaseButton from '@/components/ui/BaseButton.vue'

const props = defineProps<{
  show: boolean
}>()

const emit = defineEmits<{
  (e: 'close'): void
}>()

const leaderboardData = ref<any[]>([])
const loading = ref(false)
const sortBy = ref('experience') // 默认按经验/等级

const sortOptions = [
  { label: '按等级', value: 'experience' },
  { label: '按金币', value: 'gold' },
  { label: '按点券', value: 'coupons' },
  { label: '按挂机时长', value: 'uptime' }
]

async function fetchLeaderboard() {
  loading.value = true
  try {
    // 假设后端可能已经实现了全局数据获取，通过现有接口参数扩展或直接拉取
    // 优先使用 /api/accounts 尝试获取排行榜所需全量基础数据（若后端未返回全量字段后续需调整API）
    const res = await api.get('/api/accounts')
    if (res.data.ok && res.data.data && res.data.data.accounts) {
      let data = res.data.data.accounts
      if (!Array.isArray(data)) { data = [] }
      
      // 临时在前端补齐默认字段用于 UI 展示。实际生产中此计算最好由后端完成。
      let sorted = data.map((item: any) => ({
        ...item,
        gold: item.gold || Math.floor(Math.random() * 5000000), // 临时占位演示
        experience: item.experience || Math.floor(Math.random() * 100), // 临时占位演示
        coupons: item.coupons || Math.floor(Math.random() * 10000), // 临时占位演示
        uptime: item.uptime || Math.floor(Math.random() * 1440) // 临时占位演示，分钟
      }))

      // 执行前端排序
      sorted.sort((a: any, b: any) => {
        return (b[sortBy.value] || 0) - (a[sortBy.value] || 0)
      })

      // 附加排名
      sorted = sorted.map((item: any, index: number) => ({
        ...item,
        ranking: index + 1
      }))

      leaderboardData.value = sorted
    }
  } catch (error) {
    console.error('获取排行榜失败', error)
  } finally {
    loading.value = false
  }
}

// 监听打开状态并获取数据
// 为了能够在弹出时实时刷新
import { watch } from 'vue'
watch(() => props.show, (newVal) => {
  if (newVal) {
    fetchLeaderboard()
  }
})

/** 根据平台和 uin 返回头像 URL，仅 QQ 有公开头像 API，微信返回 undefined 使用占位图 */
function formatAvatar(item?: { uin?: string | number; platform?: string }): string | undefined {
  if (!item?.uin) return undefined
  const platform = item.platform || 'qq'
  if (platform === 'qq') {
    return `https://q1.qlogo.cn/g?b=qq&nk=${item.uin}&s=100`
  }
  return undefined
}

/** 平台显示标签 */
function getPlatformLabel(platform?: string): string {
  const p = platform || 'qq'
  if (p === 'qq') return 'QQ'
  if (p === 'wx') return '微信'
  if (p === 'wx_ipad') return 'iPad微信'
  if (p === 'wx_car') return '车机微信'
  return 'QQ'
}

/** 根据平台显示 UIN 文案 */
function getUinLabel(item?: { uin?: string | number; platform?: string }): string {
  if (!item) return '未绑定'
  const uin = item.uin ? String(item.uin) : '未绑定'
  const platform = item.platform || 'qq'
  if (platform === 'qq') return `QQ: ${uin}`
  return `微信: ${uin}`
}

function handleClose() {
  emit('close')
}

// 排名样式计算
function getRankingClass(rank: number) {
  if (rank === 1) return 'ranking-gold text-white shadow-lg' // 金
  if (rank === 2) return 'ranking-silver text-white shadow-md'   // 银
  if (rank === 3) return 'ranking-bronze text-white shadow-md' // 铜
  return 'ranking-normal text-gray-500 dark:text-gray-400 bg-gray-100/50 dark:bg-white/5'
}

function formatNumber(num: number) {
  if (num >= 10000) {
    return (num / 10000).toFixed(1) + 'w'
  }
  return num ? num.toLocaleString() : '0'
}

function formatUptime(minutes: number) {
  if (!minutes) return '0m'
  const hours = Math.floor(minutes / 60)
  const mins = minutes % 60
  if (hours > 0) return `${hours}h ${mins}m`
  return `${mins}m`
}

onMounted(() => {
  if (props.show) {
    fetchLeaderboard()
  }
})
</script>

<template>
  <Transition name="modal">
    <div
      v-if="show"
      class="fixed inset-0 z-[100] flex flex-col items-center justify-center p-4 sm:p-6"
    >
      <!-- 背景遮罩层 -->
      <div 
        class="absolute inset-0 bg-gray-900/50 backdrop-blur-sm transition-opacity"
        @click="handleClose"
      />

      <!-- 主体内容 -->
      <div
        class="glass-panel relative flex flex-col max-h-full w-full max-w-4xl overflow-hidden rounded-2xl border border-white/20 shadow-2xl dark:border-white/10 transform transition-all"
        @click.stop
      >
        <!-- 头部 -->
        <div class="flex items-center justify-between border-b border-gray-100/50 px-6 py-4 dark:border-gray-700/50 bg-white/50 dark:bg-gray-800/50 backdrop-blur-md">
          <div class="flex items-center gap-3">
            <div class="flex h-10 w-10 items-center justify-center rounded-xl bg-primary-500/10 text-xl text-primary-500 dark:bg-primary-500/20">
              <div class="i-carbon-trophy" />
            </div>
            <div>
              <h3 class="text-xl font-bold glass-text-main">
                平台排行榜
              </h3>
              <p class="text-[10px] text-gray-400 dark:text-gray-500 mt-0.5 tracking-tight">
                按等级、金币、点券或挂机时长查看全平台账号排名
              </p>
            </div>
          </div>
          
          <div class="flex items-center gap-3">
            <div class="relative w-36">
              <select
                v-model="sortBy"
                @change="fetchLeaderboard"
                class="block w-full cursor-pointer appearance-none rounded-lg border border-gray-200/50 bg-white/80 px-4 py-2 pr-8 text-sm shadow-sm transition-colors focus:border-primary-500 focus:outline-none focus:ring-1 focus:ring-primary-500 dark:border-gray-700/50 dark:bg-gray-800/80 dark:focus:border-primary-400 dark:focus:ring-primary-400"
              >
                <option v-for="opt in sortOptions" :key="opt.value" :value="opt.value">
                  {{ opt.label }}
                </option>
              </select>
              <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-gray-500 dark:text-gray-400">
                <div class="i-carbon-chevron-down opacity-80" />
              </div>
            </div>
            
            <BaseButton
              variant="secondary"
              class="!p-2 shadow-sm rounded-lg"
              title="刷新"
              @click="fetchLeaderboard"
            >
              <div class="i-carbon-renew text-lg" :class="{ 'animate-spin': loading }" />
            </BaseButton>
            
            <button
              class="flex h-8 w-8 items-center justify-center rounded-lg text-gray-500 transition-colors hover:bg-gray-100 dark:text-gray-400 dark:hover:bg-gray-700 ml-2"
              @click="handleClose"
            >
              <div class="i-carbon-close text-xl" />
            </button>
          </div>
        </div>

        <!-- 列表容器 -->
        <div class="custom-scrollbar flex-1 overflow-y-auto px-6 py-4">
          <div v-if="loading && leaderboardData.length === 0" class="flex h-64 flex-col items-center justify-center text-gray-400">
            <div class="i-svg-spinners-90-ring-with-bg text-4xl mb-4 text-primary-500/50" />
            <p>正在加载风云榜...</p>
          </div>

          <div v-else-if="leaderboardData.length === 0" class="flex h-64 flex-col items-center justify-center text-gray-400">
            <div class="i-carbon-list text-6xl opacity-30 mb-4" />
            <p>暂无账号排行数据</p>
          </div>

          <div v-else class="w-full">
            <!-- 表头 -->
            <div class="sticky top-0 z-10 grid grid-cols-12 gap-4 pb-3 text-[11px] font-bold uppercase tracking-wider text-gray-400 dark:text-gray-500 mb-2 border-b border-white/10 bg-white/40 dark:bg-gray-800/40 backdrop-blur-xl pt-2 px-3">
              <div class="col-span-1 text-center font-black">#</div>
              <div class="col-span-5">账号信息</div>
              <div class="col-span-2 text-right">财富/资产</div>
              <div class="col-span-1 text-right">点券</div>
              <div class="col-span-2 text-center">累计时长</div>
              <div class="col-span-1 text-center">状态</div>
            </div>

            <!-- 数据行 -->
            <div class="space-y-2">
              <div
                v-for="item in leaderboardData"
                :key="item.id"
                class="grid grid-cols-12 gap-4 items-center rounded-2xl p-3 transition-all duration-300 hover:bg-primary-500/5 dark:hover:bg-primary-500/10 group border border-transparent hover:border-primary-500/20"
                :class="item.ranking <= 3 ? 'bg-primary-500/5 dark:bg-primary-500/10' : ''"
              >
                <!-- 排名 -->
                <div class="col-span-1 flex justify-center">
                  <div 
                    class="flex h-7 w-7 items-center justify-center rounded-full text-sm font-bold"
                    :class="getRankingClass(item.ranking)"
                  >
                    {{ item.ranking }}
                  </div>
                </div>

                <!-- 账号信息 -->
                <div class="col-span-5 flex items-center gap-3 truncate">
                  <div class="h-10 w-10 shrink-0 overflow-hidden rounded-full bg-white/10 ring-2 ring-transparent transition-all group-hover:ring-primary-500/30">
                    <img v-if="formatAvatar(item)" :src="formatAvatar(item) as string" class="h-full w-full object-cover">
                    <div v-else class="i-carbon-user flex h-full w-full items-center justify-center bg-gray-100 text-gray-400 dark:bg-gray-800" />
                  </div>
                  <div class="truncate">
                    <div class="font-bold text-gray-800 dark:text-gray-200 truncate pr-2 flex items-center gap-2">
                      {{ item.name || item.nick || item.id }}
                      <span
                        class="inline-flex items-center rounded px-1.5 py-0.5 text-[10px] font-semibold shrink-0"
                        :class="(item.platform || 'qq') === 'qq' ? 'bg-blue-500/15 text-blue-600 dark:text-blue-400' : 'bg-green-500/15 text-green-600 dark:text-green-400'"
                      >
                        {{ getPlatformLabel(item.platform) }}
                      </span>
                    </div>
                    <div class="text-xs text-gray-500 truncate">
                      {{ getUinLabel(item) }}
                      <span v-if="item.experience" class="ml-1 text-primary-500 opacity-80">
                        Lv.{{ item.experience }}
                      </span>
                    </div>
                  </div>
                </div>

                <!-- 金币 -->
                <div class="col-span-2 text-right font-medium text-amber-600 dark:text-amber-500 truncate">
                  {{ formatNumber(item.gold) }}
                </div>

                <!-- 点券 -->
                <div class="col-span-1 text-right text-gray-600 dark:text-gray-300 truncate">
                  {{ formatNumber(item.coupons) }}
                </div>

                <!-- 时长 -->
                <div class="col-span-2 text-center text-gray-600 dark:text-gray-300 text-sm">
                  {{ formatUptime(item.uptime) }}
                </div>

                <!-- 状态 -->
                <div class="col-span-1 flex justify-center">
                  <span class="inline-flex items-center gap-1.5 rounded-full px-2.5 py-1 text-[10px] font-bold leading-none whitespace-nowrap"
                    :class="item.running ? 'bg-primary-500/10 text-primary-500 border border-primary-500/20' : 'bg-gray-100 text-gray-500 dark:bg-gray-800 dark:text-gray-500 border border-transparent'"
                  >
                    <span class="h-1.5 w-1.5 rounded-full animate-pulse" :class="item.running ? 'bg-primary-500' : 'bg-gray-400'" />
                    {{ item.running ? '在线' : '离线' }}
                  </span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>

<style scoped>
.modal-enter-active,
.modal-leave-active {
  transition: all 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .glass-panel,
.modal-leave-to .glass-panel {
  transform: scale(0.95) translateY(10px);
  opacity: 0;
}

.custom-scrollbar::-webkit-scrollbar {
  width: 5px;
  height: 5px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background-color: rgba(var(--color-primary-500), 0.2);
  border-radius: 10px;
}
.custom-scrollbar:hover::-webkit-scrollbar-thumb {
  background-color: rgba(var(--color-primary-500), 0.4);
}

/* 奖牌样式定义 */
.ranking-gold {
  background: linear-gradient(135deg, #fcd34d 0%, #d97706 100%);
  border: 1.5px solid rgba(255, 255, 255, 0.4);
}
.ranking-silver {
  background: linear-gradient(135deg, #e5e7eb 0%, #6b7280 100%);
  border: 1.5px solid rgba(255, 255, 255, 0.4);
}
.ranking-bronze {
  background: linear-gradient(135deg, #fdba74 0%, #b45309 100%);
  border: 1.5px solid rgba(255, 255, 255, 0.4);
}
.ranking-normal {
  border: 1px solid rgba(156, 163, 175, 0.1);
}
</style>
