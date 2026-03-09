<script setup lang="ts">
import { useIntervalFn } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch } from 'vue'
import { useAccountStore } from '@/stores/account'
import { useBagStore } from '@/stores/bag'
import { useFarmStore } from '@/stores/farm'
import { useSettingStore } from '@/stores/setting'
import { useStatusStore } from '@/stores/status'
import { useToastStore } from '@/stores/toast'

const accountStore = useAccountStore()
const bagStore = useBagStore()
const farmStore = useFarmStore()
const settingStore = useSettingStore()
const statusStore = useStatusStore()
const toast = useToastStore()
const { currentAccountId, currentAccount } = storeToRefs(accountStore)
const { items, dashboardItems, mallGoods, sellPreview, loading: bagLoading, mallLoading, sellPreviewLoading, actionLoading } = storeToRefs(bagStore)
const { lands, loading: landsLoading } = storeToRefs(farmStore)
const { settings } = storeToRefs(settingStore)
const { status, loading: statusLoading, error: statusError, realtimeConnected } = storeToRefs(statusStore)

const imageErrors = ref<Record<string | number, boolean>>({})
const selectedItemIds = ref<number[]>([])
const purchaseCounts = ref<Record<number, number>>({})
const useCounts = ref<Record<number, number>>({})
const selectedLandIds = ref<number[]>([])
const targetLandsAccountId = ref('')
const lastUseResult = ref<any | null>(null)
const activityHistory = ref<any[]>([])
const expandedActivityKey = ref('')
const activeTab = ref<'bag' | 'sell' | 'mall'>('bag')
const respectPolicyForSelected = ref(true)
const bagQuery = ref('')
const bagCategory = ref<'all' | 'fruit' | 'seed' | 'fertilizer' | 'pack' | 'pet' | 'item'>('all')
const mallQuery = ref('')
const mallQuickFilter = ref<'all' | 'free' | 'limited' | 'bundle'>('all')
const bagSortBy = ref<'count_desc' | 'name_asc' | 'price_desc' | 'level_desc'>('count_desc')
const mallSortBy = ref<'recommended' | 'price_asc' | 'price_desc' | 'contents_desc' | 'name_asc'>('recommended')
const activityFilter = ref<'all' | 'use' | 'purchase' | 'sell'>('all')
const bagDetailItem = ref<any | null>(null)
const mallDetailGoods = ref<any | null>(null)
const mallPurchaseMemory = ref<Record<string, { count: number, lastPurchasedAt: number, name: string }>>({})

const MALL_PURCHASE_HISTORY_KEY = 'qq-farm.mall.purchase-history.v1'
const LEGACY_BAG_USE_HISTORY_KEY = 'qq-farm.bag.use-history.v1'
const BAG_ACTIVITY_HISTORY_KEY = 'qq-farm.bag.activity-history.v1'
const LAND_TARGET_INTERACTIONS = new Set(['water', 'harvest', 'erase', 'erasegrass', 'killbug', 'plant'])

const bagSortOptions = [
  { key: 'count_desc', label: '数量优先' },
  { key: 'name_asc', label: '名称排序' },
  { key: 'price_desc', label: '单价优先' },
  { key: 'level_desc', label: '等级优先' },
]

const mallSortOptions = [
  { key: 'recommended', label: '推荐排序' },
  { key: 'price_asc', label: '价格从低到高' },
  { key: 'price_desc', label: '价格从高到低' },
  { key: 'contents_desc', label: '内容物优先' },
  { key: 'name_asc', label: '名称排序' },
]

function resolveBagCategory(item: any) {
  if (item?.category === 'fruit')
    return 'fruit'
  if (item?.category === 'seed')
    return 'seed'
  if (item?.interactionType === 'fertilizer' || item?.interactionType === 'fertilizerpro' || Number(item?.itemType || 0) === 7)
    return 'fertilizer'
  if (Number(item?.itemType || 0) === 11)
    return 'pack'
  if (Number(item?.itemType || 0) === 9)
    return 'pet'
  return 'item'
}

function getBagCategoryLabel(category: string) {
  if (category === 'fruit')
    return '果实'
  if (category === 'seed')
    return '种子'
  if (category === 'fertilizer')
    return '化肥'
  if (category === 'pack')
    return '礼包'
  if (category === 'pet')
    return '狗粮'
  return '道具'
}

function getItemFallbackLabel(item: any) {
  const category = resolveBagCategory(item)
  if (category === 'fruit')
    return 'FR'
  if (category === 'seed')
    return 'SD'
  if (category === 'fertilizer')
    return 'FT'
  if (category === 'pack')
    return 'BX'
  if (category === 'pet')
    return 'DG'
  return 'IT'
}

function getItemMetaLine(item: any) {
  const segments: string[] = [getBagCategoryLabel(resolveBagCategory(item))]
  if (Number(item?.level || 0) > 0)
    segments.push(`Lv${Number(item.level || 0)}`)
  if (Number(item?.price || 0) > 0)
    segments.push(`${Number(item.price || 0)}金`)
  return segments.join(' · ')
}

function getItemAccentClass(item: any) {
  const category = resolveBagCategory(item)
  if (category === 'fruit')
    return 'inventory-card-fruit'
  if (category === 'seed')
    return 'inventory-card-seed'
  if (category === 'fertilizer')
    return 'inventory-card-fertilizer'
  if (category === 'pack')
    return 'inventory-card-pack'
  if (category === 'pet')
    return 'inventory-card-pet'
  return 'inventory-card-item'
}

function getRarityLabel(rarity: number) {
  const value = Math.max(0, Number(rarity || 0))
  if (value >= 5)
    return '传说'
  if (value >= 4)
    return '珍稀'
  if (value >= 3)
    return '稀有'
  if (value >= 2)
    return '优良'
  return '普通'
}

function getItemDescription(item: any) {
  return String(item?.effectDesc || item?.desc || '').trim()
}

function getItemLongDescription(item: any) {
  const segments = [String(item?.desc || '').trim(), String(item?.effectDesc || '').trim()].filter(Boolean)
  return Array.from(new Set(segments))
}

function itemNeedsLandSelection(item: any) {
  return LAND_TARGET_INTERACTIONS.has(String(item?.interactionType || '').toLowerCase())
}

function normalizeLandStatusLabel(land: any) {
  const statusValue = String(land?.status || '')
  if (statusValue === 'harvestable')
    return '可收获'
  if (statusValue === 'growing')
    return '生长中'
  if (statusValue === 'dead')
    return '已枯萎'
  if (statusValue === 'empty')
    return '空地'
  if (statusValue === 'locked')
    return '未解锁'
  return statusValue || '未知'
}

function getLandSuggestionReason(item: any, land: any) {
  const type = String(item?.interactionType || '').toLowerCase()
  if (type === 'water')
    return !!land?.needWater
  if (type === 'erasegrass')
    return !!land?.needWeed
  if (type === 'killbug')
    return !!land?.needBug
  if (type === 'harvest')
    return String(land?.status || '') === 'harvestable'
  if (type === 'erase')
    return ['growing', 'dead', 'harvestable'].includes(String(land?.status || ''))
  if (type === 'plant')
    return String(land?.status || '') === 'empty'
  return false
}

const availableTargetLands = computed(() => {
  return (Array.isArray(lands.value) ? lands.value : []).filter((land: any) => !!land?.unlocked && Number(land?.id || 0) > 0)
})

const suggestedTargetLandIds = computed(() => {
  if (!bagDetailItem.value || !itemNeedsLandSelection(bagDetailItem.value))
    return []
  return availableTargetLands.value
    .filter((land: any) => getLandSuggestionReason(bagDetailItem.value, land))
    .map((land: any) => Number(land.id || 0))
})

function getSafeStorage() {
  if (typeof window === 'undefined')
    return null
  return window.localStorage
}

function loadMallPurchaseMemory() {
  const storage = getSafeStorage()
  if (!storage || !currentAccountId.value) {
    mallPurchaseMemory.value = {}
    return
  }
  try {
    const raw = storage.getItem(MALL_PURCHASE_HISTORY_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    const accountMemory = parsed?.[currentAccountId.value]
    mallPurchaseMemory.value = accountMemory && typeof accountMemory === 'object' ? accountMemory : {}
  }
  catch {
    mallPurchaseMemory.value = {}
  }
}

function loadActivityHistory() {
  const storage = getSafeStorage()
  if (!storage || !currentAccountId.value) {
    activityHistory.value = []
    return
  }
  try {
    const raw = storage.getItem(BAG_ACTIVITY_HISTORY_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    const accountHistory = parsed?.[currentAccountId.value]
    if (Array.isArray(accountHistory)) {
      activityHistory.value = accountHistory
      return
    }
    const legacyRaw = storage.getItem(LEGACY_BAG_USE_HISTORY_KEY)
    const legacyParsed = legacyRaw ? JSON.parse(legacyRaw) : {}
    const legacyHistory = Array.isArray(legacyParsed?.[currentAccountId.value]) ? legacyParsed[currentAccountId.value] : []
    activityHistory.value = legacyHistory
      .slice(0, 30)
      .map((entry: any) => ({
        ...entry,
        type: 'use',
        title: `${String(entry?.itemName || `物品#${Number(entry?.itemId || 0)}`)} x${Number(entry?.count || 1)}`,
        summary: String(entry?.rewardSummary || '使用成功'),
        details: Array.isArray(entry?.rewardItems)
          ? entry.rewardItems.slice(0, 8).map((reward: any) => ({
              id: Number(reward?.id || 0),
              name: String(reward?.name || `物品#${Number(reward?.id || 0)}`),
              count: Number(reward?.count || 0),
              image: String(reward?.image || ''),
              meta: reward?.effectDesc || reward?.desc || '',
            }))
          : [],
      }))
    if (activityHistory.value.length)
      persistActivityHistory()
  }
  catch {
    activityHistory.value = []
  }
}

function persistMallPurchaseMemory() {
  const storage = getSafeStorage()
  if (!storage || !currentAccountId.value)
    return
  try {
    const raw = storage.getItem(MALL_PURCHASE_HISTORY_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    parsed[currentAccountId.value] = mallPurchaseMemory.value
    storage.setItem(MALL_PURCHASE_HISTORY_KEY, JSON.stringify(parsed))
  }
  catch {
  }
}

function persistActivityHistory() {
  const storage = getSafeStorage()
  if (!storage || !currentAccountId.value)
    return
  try {
    const raw = storage.getItem(BAG_ACTIVITY_HISTORY_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    parsed[currentAccountId.value] = activityHistory.value
    storage.setItem(BAG_ACTIVITY_HISTORY_KEY, JSON.stringify(parsed))
  }
  catch {
  }
}

function getMallPurchaseStats(goodsId: number) {
  const row = mallPurchaseMemory.value[String(Number(goodsId || 0))] || null
  return {
    count: Number(row?.count || 0),
    lastPurchasedAt: Number(row?.lastPurchasedAt || 0),
    name: String(row?.name || ''),
  }
}

function recordMallPurchase(goods: any, count: number) {
  const goodsId = Number(goods?.goodsId || 0)
  if (!goodsId)
    return
  const key = String(goodsId)
  const current = getMallPurchaseStats(goodsId)
  mallPurchaseMemory.value = {
    ...mallPurchaseMemory.value,
    [key]: {
      count: current.count + Math.max(1, Number(count || 1)),
      lastPurchasedAt: Date.now(),
      name: String(goods?.name || current.name || `商品#${goodsId}`),
    },
  }
  persistMallPurchaseMemory()
}

function formatPurchaseTime(ts: number) {
  if (!ts)
    return '未购买'
  const delta = Date.now() - ts
  const minute = 60 * 1000
  const hour = 60 * minute
  const day = 24 * hour
  if (delta < hour)
    return `${Math.max(1, Math.floor(delta / minute) || 1)} 分钟前`
  if (delta < day)
    return `${Math.max(1, Math.floor(delta / hour) || 1)} 小时前`
  return `${Math.max(1, Math.floor(delta / day) || 1)} 天前`
}

function clearActivityHistory() {
  activityHistory.value = []
  persistActivityHistory()
  expandedActivityKey.value = ''
}

function recordActivityEntry(entry: any) {
  activityHistory.value = [entry, ...activityHistory.value].slice(0, 30)
  persistActivityHistory()
}

function recordUseActivity(item: any, result: any) {
  const entry = {
    ts: Date.now(),
    itemId: Number(item?.id || result?.itemId || 0),
    itemName: String(item?.name || `物品#${Number(item?.id || result?.itemId || 0)}`),
    count: Number(result?.count || 1),
    landIds: Array.isArray(result?.landIds) ? result.landIds.map((id: any) => Number(id || 0)).filter((id: number) => id > 0) : [],
    rewardSummary: String(result?.rewardSummary || result?.message || ''),
    rewardItems: Array.isArray(result?.rewardItems) ? result.rewardItems.slice(0, 8) : [],
    interactionType: String(item?.interactionType || ''),
  }
  recordActivityEntry({
    ...entry,
    type: 'use',
    title: `${entry.itemName} x${entry.count}`,
    summary: entry.rewardSummary || '使用成功',
    details: entry.rewardItems.map((reward: any) => ({
      id: Number(reward?.id || 0),
      name: String(reward?.name || `物品#${Number(reward?.id || 0)}`),
      count: Number(reward?.count || 0),
      image: String(reward?.image || ''),
      meta: reward?.effectDesc || reward?.desc || '',
    })),
  })
}

const filteredActivityHistory = computed(() => {
  return activityHistory.value.filter((entry: any) => {
    if (activityFilter.value === 'all')
      return true
    return String(entry?.type || '') === activityFilter.value
  })
})
const recentActivityHistory = computed(() => filteredActivityHistory.value.slice(0, 8))
const activityFilterOptions = computed(() => {
  const counts = {
    all: activityHistory.value.length,
    use: 0,
    purchase: 0,
    sell: 0,
  }
  activityHistory.value.forEach((entry: any) => {
    const type = String(entry?.type || '')
    if (type === 'use' || type === 'purchase' || type === 'sell')
      counts[type] += 1
  })
  return [
    { key: 'all', label: '全部', count: counts.all },
    { key: 'use', label: '使用', count: counts.use },
    { key: 'purchase', label: '购买', count: counts.purchase },
    { key: 'sell', label: '出售', count: counts.sell },
  ] as const
})

function buildSellActivitySummary(result: any) {
  const rows = Array.isArray(result?.plan?.items) ? result.plan.items.filter((item: any) => Number(item?.sellCount || 0) > 0) : []
  const headline = rows
    .slice(0, 3)
    .map((item: any) => `${item.name}x${item.sellCount}`)
    .join(' / ')
  const extra = rows.length > 3 ? ` 等${rows.length}种` : ''
  const gold = Number(result?.goldEarned || 0)
  if (headline)
    return `${headline}${extra}${gold > 0 ? ` · +${gold}金币` : ''}`
  if (gold > 0)
    return `获得 ${gold} 金币`
  return String(result?.message || '出售完成')
}

function buildSellActivityDetails(result: any) {
  const rows = Array.isArray(result?.plan?.items) ? result.plan.items.filter((item: any) => Number(item?.sellCount || 0) > 0) : []
  return rows.slice(0, 8).map((item: any) => ({
    id: Number(item?.id || 0),
    name: String(item?.name || `物品#${Number(item?.id || 0)}`),
    count: Number(item?.sellCount || 0),
    image: String(item?.image || ''),
    meta: `${Number(item?.unitPrice || 0)}金/个 · 小计 ${Number(item?.sellValue || 0)}金`,
  }))
}

function getActivityKey(entry: any) {
  return `${String(entry?.type || 'unknown')}-${Number(entry?.ts || 0)}-${Number(entry?.itemId || entry?.goodsId || 0)}`
}

function toggleActivityExpanded(entry: any) {
  const key = getActivityKey(entry)
  expandedActivityKey.value = expandedActivityKey.value === key ? '' : key
}

function isActivityExpanded(entry: any) {
  return expandedActivityKey.value === getActivityKey(entry)
}

function getLandSelectionHint(item: any) {
  const type = String(item?.interactionType || '').toLowerCase()
  if (type === 'water')
    return '优先选择缺水土地'
  if (type === 'erasegrass')
    return '优先选择有杂草的土地'
  if (type === 'killbug')
    return '优先选择有虫害的土地'
  if (type === 'harvest')
    return '优先选择可收获土地'
  if (type === 'erase')
    return '优先选择正在生长或已枯萎的土地'
  if (type === 'plant')
    return '优先选择空地'
  return '请选择目标土地'
}

function getImageErrorKey(scope: string, primaryId: number | string, secondaryId?: number | string) {
  return `${scope}-${primaryId}${secondaryId === undefined ? '' : `-${secondaryId}`}`
}

const bagCategoryOptions = computed(() => {
  const counts = {
    all: items.value.length,
    fruit: 0,
    seed: 0,
    fertilizer: 0,
    pack: 0,
    pet: 0,
    item: 0,
  }
  for (const item of items.value) {
    counts[resolveBagCategory(item) as keyof typeof counts] += 1
  }
  return [
    { key: 'all', label: '全部', count: counts.all },
    { key: 'fruit', label: '果实', count: counts.fruit },
    { key: 'seed', label: '种子', count: counts.seed },
    { key: 'fertilizer', label: '化肥', count: counts.fertilizer },
    { key: 'pack', label: '礼包', count: counts.pack },
    { key: 'pet', label: '狗粮', count: counts.pet },
    { key: 'item', label: '道具', count: counts.item },
  ]
})

const filteredItems = computed(() => {
  const keyword = bagQuery.value.trim().toLowerCase()
  const list = items.value.filter((item: any) => {
    const category = resolveBagCategory(item)
    if (bagCategory.value !== 'all' && category !== bagCategory.value)
      return false
    if (!keyword)
      return true
    const haystack = [
      String(item?.name || ''),
      String(item?.id || ''),
      getBagCategoryLabel(category),
      String(item?.interactionType || ''),
    ].join(' ').toLowerCase()
    return haystack.includes(keyword)
  })
  list.sort((a: any, b: any) => {
    if (bagSortBy.value === 'name_asc')
      return String(a?.name || '').localeCompare(String(b?.name || ''), 'zh-CN')
    if (bagSortBy.value === 'price_desc')
      return Number(b?.price || 0) - Number(a?.price || 0) || Number(b?.count || 0) - Number(a?.count || 0)
    if (bagSortBy.value === 'level_desc')
      return Number(b?.level || 0) - Number(a?.level || 0) || Number(b?.count || 0) - Number(a?.count || 0)
    return Number(b?.count || 0) - Number(a?.count || 0) || Number(a?.id || 0) - Number(b?.id || 0)
  })
  return list
})

const mallFilterOptions = computed(() => {
  const all = mallGoods.value.length
  const free = mallGoods.value.filter((goods: any) => goods?.isFree).length
  const limited = mallGoods.value.filter((goods: any) => goods?.isLimited).length
  const bundle = mallGoods.value.filter((goods: any) => (goods?.itemIds?.length || 0) > 1).length
  return [
    { key: 'all', label: '全部', count: all },
    { key: 'free', label: '免费', count: free },
    { key: 'limited', label: '限购', count: limited },
    { key: 'bundle', label: '礼包', count: bundle },
  ]
})

function setBagCategory(value: string) {
  if (['all', 'fruit', 'seed', 'fertilizer', 'pack', 'pet', 'item'].includes(value))
    bagCategory.value = value as typeof bagCategory.value
}

function setMallQuickFilter(value: string) {
  if (['all', 'free', 'limited', 'bundle'].includes(value))
    mallQuickFilter.value = value as typeof mallQuickFilter.value
}

function getMallRecommendationScore(goods: any) {
  const stats = getMallPurchaseStats(Number(goods?.goodsId || 0))
  const recencyWindow = 7 * 24 * 60 * 60 * 1000
  const recencyBoost = stats.lastPurchasedAt > 0
    ? Math.max(0, recencyWindow - (Date.now() - stats.lastPurchasedAt)) / (24 * 60 * 60 * 1000)
    : 0
  return (
    stats.count * 1000
    + recencyBoost * 100
    + (goods?.isFree ? 40 : 0)
    + (goods?.isLimited ? 20 : 0)
    + Math.max(0, 20 - Number(goods?.priceValue || 0))
  )
}

const filteredMallGoods = computed(() => {
  const keyword = mallQuery.value.trim().toLowerCase()
  const list = mallGoods.value.filter((goods: any) => {
    if (mallQuickFilter.value === 'free' && !goods?.isFree)
      return false
    if (mallQuickFilter.value === 'limited' && !goods?.isLimited)
      return false
    if (mallQuickFilter.value === 'bundle' && (goods?.itemIds?.length || 0) <= 1)
      return false
    if (!keyword)
      return true
    const previewNames = Array.isArray(goods?.itemPreviews) ? goods.itemPreviews.map((item: any) => item?.name || '').join(' ') : ''
    const haystack = [
      String(goods?.name || ''),
      String(goods?.goodsId || ''),
      String(goods?.type || ''),
      previewNames,
    ].join(' ').toLowerCase()
    return haystack.includes(keyword)
  })
  list.sort((a: any, b: any) => {
    if (mallSortBy.value === 'price_asc')
      return Number(a?.priceValue || 0) - Number(b?.priceValue || 0) || Number(a?.goodsId || 0) - Number(b?.goodsId || 0)
    if (mallSortBy.value === 'price_desc')
      return Number(b?.priceValue || 0) - Number(a?.priceValue || 0) || Number(a?.goodsId || 0) - Number(b?.goodsId || 0)
    if (mallSortBy.value === 'contents_desc')
      return (Number(b?.itemIds?.length || 0) - Number(a?.itemIds?.length || 0)) || Number(a?.goodsId || 0) - Number(b?.goodsId || 0)
    if (mallSortBy.value === 'name_asc')
      return String(a?.name || '').localeCompare(String(b?.name || ''), 'zh-CN')
    const recommendationDelta = getMallRecommendationScore(b) - getMallRecommendationScore(a)
    if (recommendationDelta)
      return recommendationDelta

    const freeDelta = Number(Boolean(b?.isFree)) - Number(Boolean(a?.isFree))
    if (freeDelta)
      return freeDelta
    const limitedDelta = Number(Boolean(b?.isLimited)) - Number(Boolean(a?.isLimited))
    if (limitedDelta)
      return limitedDelta
    const priceDelta = Number(a?.priceValue || 0) - Number(b?.priceValue || 0)
    if (priceDelta)
      return priceDelta
    const contentDelta = Number(b?.itemIds?.length || 0) - Number(a?.itemIds?.length || 0)
    if (contentDelta)
      return contentDelta
    return Number(a?.goodsId || 0) - Number(b?.goodsId || 0)
  })
  return list
})

const recommendedMallGoods = computed(() => {
  return mallGoods.value
    .map((goods: any) => ({
      ...goods,
      purchaseStats: getMallPurchaseStats(Number(goods?.goodsId || 0)),
    }))
    .filter((goods: any) => goods.purchaseStats.count > 0 || goods.purchaseStats.lastPurchasedAt > 0)
    .sort((a: any, b: any) => {
      const scoreDelta = getMallRecommendationScore(b) - getMallRecommendationScore(a)
      if (scoreDelta)
        return scoreDelta
      return Number(a?.goodsId || 0) - Number(b?.goodsId || 0)
    })
    .slice(0, 4)
})

function getMallFallbackLabel(goods: any) {
  if (goods?.isFree)
    return 'FR'
  if ((goods?.itemIds?.length || 0) > 1)
    return 'BX'
  return 'GD'
}

function getMallPriceLabel(goods: any) {
  return goods?.isFree ? '免费领取' : `${Number(goods?.priceValue || 0)} 点券`
}

function getGoodsPreviewGroups(goods: any, limit = Infinity) {
  const groups = new Map<number, any>()
  for (const preview of Array.isArray(goods?.itemPreviews) ? goods.itemPreviews : []) {
    const id = Number(preview?.id || 0)
    if (id <= 0)
      continue
    if (!groups.has(id)) {
      groups.set(id, { ...preview, count: 0 })
    }
    groups.get(id).count += 1
  }
  return Array.from(groups.values()).slice(0, limit)
}

function getGoodsSummary(goods: any) {
  const summary = String(goods?.summary || '').trim()
  if (summary)
    return summary
  const previews = getGoodsPreviewGroups(goods, 3)
  if (!previews.length)
    return '暂无内容说明'
  return previews.map((preview: any) => `${preview.name}${preview.count > 1 ? ` x${preview.count}` : ''}`).join(' / ')
}

function isLandSelected(landId: number) {
  return selectedLandIds.value.includes(landId)
}

function getLandSelectionLimit(item: any) {
  return Math.max(1, Number(item?.count || 1))
}

function pruneSelectedLands() {
  const allowed = new Set(availableTargetLands.value.map((land: any) => Number(land.id || 0)))
  const limit = bagDetailItem.value && itemNeedsLandSelection(bagDetailItem.value)
    ? getLandSelectionLimit(bagDetailItem.value)
    : Number.POSITIVE_INFINITY
  selectedLandIds.value = selectedLandIds.value
    .filter(id => allowed.has(Number(id)))
    .slice(0, limit)
}

function toggleSelectedLand(landId: number) {
  if (!selectedLandIds.value.includes(landId)) {
    const limit = bagDetailItem.value ? getLandSelectionLimit(bagDetailItem.value) : 1
    if (selectedLandIds.value.length >= limit) {
      toast.warning(`当前物品最多只能选择 ${limit} 块土地`)
      return
    }
    selectedLandIds.value = [...selectedLandIds.value, landId]
    return
  }
  selectedLandIds.value = selectedLandIds.value.filter(id => id !== landId)
}

function selectSuggestedLands() {
  const limit = bagDetailItem.value ? getLandSelectionLimit(bagDetailItem.value) : 1
  selectedLandIds.value = [...suggestedTargetLandIds.value].slice(0, limit)
}

function clearSelectedLands() {
  selectedLandIds.value = []
}

const sellableIds = computed(() => new Set(
  items.value
    .filter((it: any) => it?.category === 'fruit')
    .map((it: any) => Number(it.id || 0))
    .filter((id: number) => id > 0),
))

const selectedSellItems = computed(() => {
  const selected = new Set(selectedItemIds.value)
  return items.value.filter((item: any) => selected.has(Number(item.id || 0)) && sellableIds.value.has(Number(item.id || 0)))
})

const sellConfigSummary = computed(() => {
  const sell = settings.value.tradeConfig?.sell || {}
  const rareKeep = sell.rareKeep || {}
  return {
    keepMinEachFruit: Number(sell.keepMinEachFruit || 0),
    keepFruitIds: Array.isArray(sell.keepFruitIds) ? sell.keepFruitIds : [],
    rareKeepEnabled: !!rareKeep.enabled,
    judgeBy: rareKeep.judgeBy || 'either',
    minPlantLevel: Number(rareKeep.minPlantLevel || 0),
    minUnitPrice: Number(rareKeep.minUnitPrice || 0),
    previewBeforeManualSell: !!sell.previewBeforeManualSell,
  }
})

function pruneSelectedItems() {
  const allowed = sellableIds.value
  selectedItemIds.value = selectedItemIds.value.filter(id => allowed.has(Number(id)))
}

function toggleSelectItem(itemId: number) {
  if (!sellableIds.value.has(itemId))
    return
  if (selectedItemIds.value.includes(itemId)) {
    selectedItemIds.value = selectedItemIds.value.filter(id => id !== itemId)
  }
  else {
    selectedItemIds.value = [...selectedItemIds.value, itemId]
  }
}

function selectAllSellable() {
  selectedItemIds.value = items.value
    .filter((item: any) => sellableIds.value.has(Number(item.id || 0)))
    .map((item: any) => Number(item.id || 0))
}

function clearSelection() {
  selectedItemIds.value = []
}

async function ensureConnected() {
  if (!currentAccountId.value)
    return false
  if (!realtimeConnected.value) {
    await statusStore.fetchStatus(currentAccountId.value)
  }
  return !!(currentAccount.value?.running && status.value?.connection?.connected)
}

async function loadBag() {
  if (!currentAccountId.value)
    return
  const connected = await ensureConnected()
  if (!connected) {
    imageErrors.value = {}
    clearSelection()
    return
  }
  await Promise.all([
    settingStore.fetchSettings(currentAccountId.value),
    bagStore.fetchBag(currentAccountId.value),
    bagStore.fetchSellPreview(currentAccountId.value),
    bagStore.fetchMallGoods(currentAccountId.value),
  ])
  imageErrors.value = {}
  pruneSelectedItems()
  pruneSelectedLands()
}

async function handleRefresh() {
  await loadBag()
  toast.success('背包和商城数据已刷新')
}

async function handlePreviewSell() {
  if (!currentAccountId.value)
    return
  await bagStore.fetchSellPreview(currentAccountId.value)
  activeTab.value = 'sell'
  toast.success('已刷新出售预览')
}

async function handleSellByPolicy() {
  if (!currentAccountId.value)
    return
  if (sellConfigSummary.value.previewBeforeManualSell) {
    await bagStore.fetchSellPreview(currentAccountId.value)
  }
  const result = await bagStore.sellByPolicy(currentAccountId.value)
  if (result) {
    recordActivityEntry({
      ts: Date.now(),
      type: 'sell',
      title: `按策略出售 ${Number(result?.soldCount || 0)} 个果实`,
      soldCount: Number(result?.soldCount || 0),
      soldKinds: Number(result?.soldKinds || 0),
      goldEarned: Number(result?.goldEarned || 0),
      summary: buildSellActivitySummary(result),
      details: buildSellActivityDetails(result),
    })
    activeTab.value = 'sell'
    toast.success(result.message || '已按策略出售')
    clearSelection()
  }
}

async function handleSellSelected() {
  if (!currentAccountId.value || selectedItemIds.value.length === 0)
    return
  if (sellConfigSummary.value.previewBeforeManualSell) {
    await bagStore.fetchSellPreview(currentAccountId.value)
  }
  const result = await bagStore.sellSelected(currentAccountId.value, selectedItemIds.value, respectPolicyForSelected.value)
  if (result) {
    recordActivityEntry({
      ts: Date.now(),
      type: 'sell',
      title: `手动出售 ${Number(result?.soldCount || 0)} 个果实`,
      soldCount: Number(result?.soldCount || 0),
      soldKinds: Number(result?.soldKinds || 0),
      goldEarned: Number(result?.goldEarned || 0),
      summary: buildSellActivitySummary(result),
      details: buildSellActivityDetails(result),
    })
    activeTab.value = 'sell'
    toast.success(result.message || '已出售选中果实')
    clearSelection()
  }
}

function getPurchaseCount(goodsId: number) {
  return Math.max(1, Number(purchaseCounts.value[goodsId] || 1))
}

function updatePurchaseCount(goodsId: number, value: number) {
  purchaseCounts.value = {
    ...purchaseCounts.value,
    [goodsId]: Math.max(1, Number(value || 1)),
  }
}

function bumpPurchaseCount(goodsId: number, delta: number) {
  updatePurchaseCount(goodsId, getPurchaseCount(goodsId) + delta)
}

function getUseCount(itemId: number, maxCount = Infinity) {
  const limit = Math.max(1, Number(maxCount || 1))
  return Math.min(limit, Math.max(1, Number(useCounts.value[itemId] || 1)))
}

function updateUseCount(itemId: number, value: number, maxCount = Infinity) {
  const limit = Math.max(1, Number(maxCount || 1))
  useCounts.value = {
    ...useCounts.value,
    [itemId]: Math.min(limit, Math.max(1, Number(value || 1))),
  }
}

function bumpUseCount(itemId: number, delta: number, maxCount = Infinity) {
  updateUseCount(itemId, getUseCount(itemId, maxCount) + delta, maxCount)
}

async function ensureFarmLandsLoaded() {
  if (!currentAccountId.value)
    return
  if (targetLandsAccountId.value === currentAccountId.value && availableTargetLands.value.length > 0)
    return
  await farmStore.fetchLands(currentAccountId.value)
  targetLandsAccountId.value = currentAccountId.value
}

async function handleBuyGoods(goods: any) {
  if (!currentAccountId.value)
    return
  const goodsId = Number(goods?.goodsId || 0)
  if (!goodsId)
    return
  const count = getPurchaseCount(goodsId)
  const result = await bagStore.buyMallGoods(currentAccountId.value, goodsId, count)
  if (result) {
    recordMallPurchase(goods, count)
    recordActivityEntry({
      ts: Date.now(),
      type: 'purchase',
      goodsId,
      goodsName: String(goods?.name || `商品#${goodsId}`),
      title: `${goods?.name || `商品#${goodsId}`} x${count}`,
      count,
      summary: getGoodsSummary(goods),
      priceLabel: getMallPriceLabel(goods),
      itemIds: Array.isArray(goods?.itemIds) ? goods.itemIds.slice(0, 12) : [],
      details: getGoodsPreviewGroups(goods).slice(0, 8).map((preview: any) => ({
        id: Number(preview?.id || 0),
        name: String(preview?.name || `物品#${Number(preview?.id || 0)}`),
        count: Number(preview?.count || 0),
        image: String(preview?.image || ''),
        meta: preview?.effectDesc || preview?.desc || '',
      })),
    })
    toast.success(`已购买 ${goods?.name || `商品#${goodsId}`}`)
  }
}

async function handleUseBagItem(item: any) {
  if (!currentAccountId.value)
    return
  const itemId = Number(item?.id || 0)
  const maxCount = Math.max(1, Number(item?.count || 1))
  if (!itemId || !item?.canUse)
    return
  const needLandSelection = itemNeedsLandSelection(item)
  let landIds = needLandSelection ? [...selectedLandIds.value] : []
  if (needLandSelection && landIds.length > maxCount) {
    landIds = landIds.slice(0, maxCount)
    selectedLandIds.value = landIds
  }
  if (needLandSelection && landIds.length === 0) {
    toast.warning('请至少选择一块土地')
    return
  }
  const count = needLandSelection
    ? landIds.length
    : getUseCount(itemId, maxCount)
  const result = await bagStore.useBagItem(currentAccountId.value, itemId, count, landIds)
  if (result) {
    lastUseResult.value = result
    recordUseActivity(item, result)
    toast.success(result.message || `已使用 ${item?.name || `物品#${itemId}`} x${count}`)
    if (needLandSelection) {
      await farmStore.fetchLands(currentAccountId.value)
      targetLandsAccountId.value = currentAccountId.value
      clearSelectedLands()
    }
  }
}

function isSelected(itemId: number) {
  return selectedItemIds.value.includes(itemId)
}

function formatPreviewReason(item: any) {
  if (!item?.keepReasons?.length)
    return ''
  return item.keepReasons.join(' / ')
}

function openActivityEntry(entry: any) {
  if (String(entry?.type || '') === 'purchase') {
    const goods = mallGoods.value.find((item: any) => Number(item?.goodsId || 0) === Number(entry?.goodsId || 0))
      || { goodsId: entry?.goodsId, name: entry?.goodsName, itemIds: entry?.itemIds || [] }
    openMallDetail(goods)
    return
  }
  if (String(entry?.type || '') === 'sell') {
    activeTab.value = 'sell'
    return
  }
  const item = items.value.find((row: any) => Number(row?.id || 0) === Number(entry?.itemId || 0))
    || { id: entry?.itemId, name: entry?.itemName, interactionType: entry?.interactionType, canUse: true, count: entry?.count || 1 }
  openBagDetail(item)
}

function openBagDetail(item: any) {
  bagDetailItem.value = item
  mallDetailGoods.value = null
  lastUseResult.value = null
  if (itemNeedsLandSelection(item))
    ensureFarmLandsLoaded()
  else
    clearSelectedLands()
}

function openMallDetail(goods: any) {
  mallDetailGoods.value = goods
  bagDetailItem.value = null
}

function closeDetailPanel() {
  bagDetailItem.value = null
  mallDetailGoods.value = null
  lastUseResult.value = null
  clearSelectedLands()
}

// vue-tsc occasionally misses a few late template refs in this large SFC.
const templateUsageBridge = {
  formatPurchaseTime,
  recommendedMallGoods,
  bumpUseCount,
  handleUseBagItem,
}
void templateUsageBridge

onMounted(() => {
  loadMallPurchaseMemory()
  loadActivityHistory()
  loadBag()
})

watch(currentAccountId, () => {
  clearSelection()
  closeDetailPanel()
  targetLandsAccountId.value = ''
  loadMallPurchaseMemory()
  loadActivityHistory()
  loadBag()
})

watch(items, () => {
  pruneSelectedItems()
  if (bagDetailItem.value) {
    const nextItem = items.value.find((item: any) => Number(item?.id || 0) === Number(bagDetailItem.value?.id || 0))
    bagDetailItem.value = nextItem || null
  }
})

watch(lands, () => {
  pruneSelectedLands()
})

watch(mallGoods, () => {
  if (mallDetailGoods.value) {
    const nextGoods = mallGoods.value.find((goods: any) => Number(goods?.goodsId || 0) === Number(mallDetailGoods.value?.goodsId || 0))
    mallDetailGoods.value = nextGoods || null
  }
})

useIntervalFn(loadBag, 60000)
</script>

<template>
  <div class="space-y-4">
    <div class="mb-4 flex flex-col items-start justify-between gap-3 sm:flex-row sm:items-center">
      <div>
        <h2 class="flex items-center gap-2 text-2xl font-bold">
          <div class="i-carbon-inventory-management" />
          背包与交易
        </h2>
        <div class="mt-1 text-sm text-gray-500">
          背包 {{ items.length }} 种物品
          <span v-if="activeTab === 'bag'"> · 当前显示 {{ filteredItems.length }} 项</span>
          <span v-if="activeTab === 'mall'"> · 当前显示 {{ filteredMallGoods.length }} 项商品</span>
          <span v-if="selectedSellItems.length"> · 已选 {{ selectedSellItems.length }} 种果实</span>
        </div>
      </div>

      <div class="flex flex-wrap items-center gap-2">
        <button class="trade-btn trade-btn-secondary" :disabled="actionLoading" @click="handleRefresh">
          刷新
        </button>
        <button class="trade-btn trade-btn-secondary" :disabled="sellPreviewLoading || actionLoading" @click="handlePreviewSell">
          刷新出售预览
        </button>
        <button class="trade-btn trade-btn-primary" :disabled="actionLoading" @click="handleSellByPolicy">
          按策略出售
        </button>
      </div>
    </div>

    <div class="grid gap-3 lg:grid-cols-3">
      <div class="glass-panel rounded-xl p-4 shadow">
        <div class="text-xs text-gray-400 tracking-[0.2em] uppercase">
          交易策略
        </div>
        <div class="mt-2 text-sm font-medium">
          每种果实保留 {{ sellConfigSummary.keepMinEachFruit }} 个
        </div>
        <div class="mt-1 text-xs text-gray-500">
          白名单 {{ sellConfigSummary.keepFruitIds.length }} 项
          <span v-if="sellConfigSummary.rareKeepEnabled">
            · 稀有保留开启
          </span>
          <span v-if="sellConfigSummary.previewBeforeManualSell">
            · 手动出售前强制预览
          </span>
        </div>
        <div class="mt-2 text-xs text-amber-500/90">
          当前仅支持出售果实，种子、化肥、狗粮、礼包等物品不会出现在出售勾选中。
        </div>
      </div>

      <div class="glass-panel rounded-xl p-4 shadow">
        <div class="text-xs text-gray-400 tracking-[0.2em] uppercase">
          预计出售
        </div>
        <div class="mt-2 text-sm font-medium">
          {{ sellPreview?.totalSellCount || 0 }} 个果实
        </div>
        <div class="mt-1 text-xs text-gray-500">
          {{ sellPreview?.totalSellKinds || 0 }} 种 · 预计 {{ sellPreview?.expectedGold || 0 }} 金币
        </div>
      </div>

      <div class="glass-panel rounded-xl p-4 shadow">
        <div class="text-xs text-gray-400 tracking-[0.2em] uppercase">
          快速选择
        </div>
        <div class="mt-3 flex flex-wrap gap-2">
          <button class="trade-btn trade-btn-secondary" @click="selectAllSellable">
            全选可售果实
          </button>
          <button class="trade-btn trade-btn-secondary" @click="clearSelection">
            清空选择
          </button>
          <label class="inline-flex items-center gap-2 text-sm text-gray-500">
            <input v-model="respectPolicyForSelected" type="checkbox">
            选中出售也遵守保留策略
          </label>
        </div>
      </div>
    </div>

    <div class="flex flex-wrap gap-2">
      <button class="trade-tab" :class="{ active: activeTab === 'bag' }" @click="activeTab = 'bag'">
        背包
      </button>
      <button class="trade-tab" :class="{ active: activeTab === 'sell' }" @click="activeTab = 'sell'">
        出售预览
      </button>
      <button class="trade-tab" :class="{ active: activeTab === 'mall' }" @click="activeTab = 'mall'">
        商城
      </button>
    </div>

    <div v-if="activityHistory.length" class="activity-panel glass-panel rounded-2xl p-4 shadow">
      <div class="activity-panel__header">
        <div>
          <div class="activity-panel__title">
            交易动态
          </div>
          <div class="activity-panel__subtitle">
            最近 30 条购买、使用、出售动作，按账号本地保存
          </div>
        </div>
        <div class="activity-panel__actions">
          <div class="activity-panel__summary">
            当前筛选 {{ filteredActivityHistory.length }} 条，展示最近 {{ recentActivityHistory.length }} 条
          </div>
          <div class="inventory-toolbar__chips">
            <button
              v-for="option in activityFilterOptions"
              :key="option.key"
              class="filter-chip"
              :class="{ active: activityFilter === option.key }"
              @click="activityFilter = option.key"
            >
              <span>{{ option.label }}</span>
              <span class="filter-chip__count">{{ option.count }}</span>
            </button>
          </div>
          <button class="trade-btn trade-btn-secondary" @click="clearActivityHistory()">
            清空动态
          </button>
        </div>
      </div>

      <div v-if="recentActivityHistory.length" class="activity-list">
        <div
          v-for="entry in recentActivityHistory"
          :key="`activity-${entry.type}-${entry.ts}-${entry.itemId || entry.goodsId}`"
          class="activity-card"
        >
          <div class="activity-card__body" role="button" tabindex="0" @click="openActivityEntry(entry)">
            <div class="activity-card__type" :class="{ purchase: entry.type === 'purchase', sell: entry.type === 'sell', use: entry.type !== 'purchase' && entry.type !== 'sell' }">
              {{ entry.type === 'purchase' ? '购买' : entry.type === 'sell' ? '出售' : '使用' }}
            </div>
            <div class="activity-card__main">
              <div class="activity-card__title">
                {{ entry.title }}
              </div>
              <div class="activity-card__meta">
                <span>{{ formatPurchaseTime(entry.ts) }}</span>
                <span v-if="entry.priceLabel">{{ entry.priceLabel }}</span>
                <span v-if="entry.goldEarned">+{{ entry.goldEarned }} 金币</span>
                <span v-if="entry.landIds?.length">地块 {{ entry.landIds.join(', ') }}</span>
              </div>
              <div class="activity-card__summary">
                {{ entry.summary || '操作成功' }}
              </div>
            </div>
          </div>
          <div class="activity-card__actions">
            <button
              v-if="entry.details?.length"
              class="trade-btn trade-btn-secondary"
              @click="toggleActivityExpanded(entry)"
            >
              {{ isActivityExpanded(entry) ? '收起明细' : '展开明细' }}
            </button>
          </div>
          <div v-if="isActivityExpanded(entry) && entry.details?.length" class="activity-card__details">
            <div
              v-for="detail in entry.details"
              :key="`activity-detail-${getActivityKey(entry)}-${detail.id}`"
              class="activity-detail-card"
            >
              <div class="activity-detail-card__thumb" :data-fallback="getItemFallbackLabel(detail)">
                <img
                  v-if="detail.image && !imageErrors[getImageErrorKey('activity-detail', getActivityKey(entry), detail.id)]"
                  :src="detail.image"
                  :alt="detail.name"
                  loading="lazy"
                  @error="imageErrors[getImageErrorKey('activity-detail', getActivityKey(entry), detail.id)] = true"
                >
                <div v-else class="inventory-card__fallback activity-detail-card__fallback">
                  {{ getItemFallbackLabel(detail) }}
                </div>
              </div>
              <div class="activity-detail-card__copy">
                <div class="activity-detail-card__title">
                  {{ detail.name }}
                </div>
                <div class="activity-detail-card__meta">
                  <span>x{{ detail.count || 0 }}</span>
                  <span v-if="detail.meta">{{ detail.meta }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="activity-panel__empty glass-text-muted border border-white/20 rounded-lg bg-black/5 p-8 text-center backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
        当前筛选下暂无交易动态
      </div>
    </div>

    <div v-if="bagLoading || statusLoading" class="flex justify-center py-12">
      <div class="i-svg-spinners-90-ring-with-bg text-4xl text-blue-500" />
    </div>

    <div v-else-if="!currentAccountId" class="glass-text-muted border border-white/20 rounded-lg bg-black/5 p-8 text-center backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
      请选择账号后查看背包
    </div>

    <div v-else-if="statusError" class="border border-red-200/50 rounded-lg bg-red-50/50 p-8 text-center text-red-500 shadow backdrop-blur-sm dark:border-red-800 dark:bg-red-900/20">
      <div class="mb-2 text-lg font-bold">
        获取数据失败
      </div>
      <div class="text-sm">
        {{ statusError }}
      </div>
    </div>

    <div v-else-if="!status?.connection?.connected" class="glass-text-muted flex flex-col items-center justify-center gap-4 border border-white/20 rounded-lg bg-black/5 p-12 text-center shadow-sm backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
      <div class="i-carbon-connection-signal-off text-4xl text-gray-400" />
      <div>
        <div class="glass-text-main text-lg font-medium">
          账号未登录
        </div>
        <div class="glass-text-muted mt-1 text-sm">
          请先运行账号或检查网络连接
        </div>
      </div>
    </div>

    <template v-else>
      <div v-if="dashboardItems.length" class="grid gap-3 lg:grid-cols-4 md:grid-cols-2">
        <div
          v-for="item in dashboardItems"
          :key="`dashboard-${item.id}`"
          class="glass-panel rounded-xl p-4 shadow"
        >
          <div class="text-xs text-gray-400 tracking-[0.2em] uppercase">
            {{ item.name }}
          </div>
          <div class="mt-2 text-lg font-semibold">
            {{ item.hoursText || `x${item.count || 0}` }}
          </div>
        </div>
      </div>

      <div v-if="activeTab === 'bag'">
        <div class="inventory-toolbar glass-panel mb-4 rounded-2xl p-4 shadow">
          <div class="inventory-toolbar__row">
            <div class="inventory-toolbar__search">
              <div class="i-carbon-search text-sm opacity-70" />
              <input v-model="bagQuery" type="text" class="inventory-search-input" placeholder="搜索物品名称 / ID / 分类">
            </div>
            <label class="trade-select-wrap">
              <span class="trade-select-wrap__label">排序</span>
              <select v-model="bagSortBy" class="trade-select">
                <option v-for="option in bagSortOptions" :key="option.key" :value="option.key">
                  {{ option.label }}
                </option>
              </select>
            </label>
          </div>
          <div class="inventory-toolbar__chips">
            <button
              v-for="option in bagCategoryOptions"
              :key="option.key"
              class="filter-chip"
              :class="{ active: bagCategory === option.key }"
              @click="setBagCategory(option.key)"
            >
              <span>{{ option.label }}</span>
              <span class="filter-chip__count">{{ option.count }}</span>
            </button>
          </div>
        </div>

        <div v-if="filteredItems.length === 0" class="glass-text-muted border border-white/20 rounded-lg bg-black/5 p-8 text-center backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
          无可展示物品
        </div>

        <div v-else class="grid grid-cols-2 gap-4 lg:grid-cols-4 md:grid-cols-3 xl:grid-cols-5">
          <div
            v-for="item in filteredItems"
            :key="item.id"
            class="inventory-card group"
            :class="getItemAccentClass(item)"
            role="button"
            tabindex="0"
            @click="openBagDetail(item)"
          >
            <label
              v-if="item.category === 'fruit'"
              class="absolute right-3 top-3 z-10 h-6 w-6 flex items-center justify-center rounded-full bg-white/85 text-xs shadow-sm dark:bg-black/50"
              @click.stop
            >
              <input
                :checked="isSelected(Number(item.id || 0))"
                type="checkbox"
                @click.stop
                @change="toggleSelectItem(Number(item.id || 0))"
              >
            </label>

            <div class="absolute left-3 top-3 z-10 rounded-full bg-black/10 px-2 py-1 text-[11px] text-white/80 font-mono backdrop-blur-sm">
              #{{ item.id }}
            </div>

            <div class="inventory-card__art" :data-fallback="getItemFallbackLabel(item)">
              <img
                v-if="item.image && !imageErrors[getImageErrorKey('bag', item.id)]"
                :src="item.image"
                :alt="item.name"
                class="inventory-card__image"
                loading="lazy"
                @error="imageErrors[getImageErrorKey('bag', item.id)] = true"
              >
              <div v-else class="inventory-card__fallback">
                {{ getItemFallbackLabel(item) }}
              </div>
            </div>

            <div class="inventory-card__count">
              {{ item.hoursText || `x${item.count || 0}` }}
            </div>

            <div class="inventory-card__name" :title="item.name">
              {{ item.name || `物品${item.id}` }}
            </div>

            <div class="inventory-card__meta">
              <span class="inventory-pill">
                {{ getBagCategoryLabel(resolveBagCategory(item)) }}
              </span>
              <span>{{ getItemMetaLine(item) }}</span>
              <span v-if="getItemDescription(item)">
                {{ getItemDescription(item) }}
              </span>
              <span v-if="item.category === 'fruit'" class="text-emerald-300">
                可加入手动出售
              </span>
            </div>

            <div class="inventory-card__footer">
              <span>点击查看详情</span>
              <span v-if="item.canUse" class="inventory-card__footer-tag">可使用</span>
            </div>
          </div>
        </div>
      </div>

      <div v-else-if="activeTab === 'sell'" class="space-y-4">
        <div class="glass-panel rounded-xl p-4 shadow">
          <div class="flex flex-wrap items-center justify-between gap-3">
            <div>
              <div class="text-lg font-semibold">
                出售预览
              </div>
              <div class="mt-1 text-sm text-gray-500">
                可出售 {{ sellPreview?.totalSellKinds || 0 }} 种，共 {{ sellPreview?.totalSellCount || 0 }} 个，预计 {{ sellPreview?.expectedGold || 0 }} 金币
              </div>
            </div>
            <div class="flex flex-wrap gap-2">
              <button class="trade-btn trade-btn-secondary" :disabled="sellPreviewLoading || actionLoading" @click="handlePreviewSell">
                刷新预览
              </button>
              <button class="trade-btn trade-btn-primary" :disabled="actionLoading" @click="handleSellByPolicy">
                全部按策略出售
              </button>
              <button class="trade-btn trade-btn-secondary" :disabled="actionLoading || selectedSellItems.length === 0" @click="handleSellSelected">
                出售选中 {{ selectedSellItems.length }} 种
              </button>
            </div>
          </div>
        </div>

        <div v-if="sellPreviewLoading" class="flex justify-center py-10">
          <div class="i-svg-spinners-90-ring-with-bg text-3xl text-blue-500" />
        </div>

        <div v-else-if="!sellPreview?.items?.length" class="glass-text-muted border border-white/20 rounded-lg bg-black/5 p-8 text-center backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
          当前没有可预览的果实出售计划
        </div>

        <div v-else class="space-y-3">
          <div
            v-for="item in sellPreview.items"
            :key="`sell-${item.id}`"
            class="glass-panel flex flex-col gap-3 rounded-xl p-4 shadow sm:flex-row sm:items-center sm:justify-between"
          >
            <div class="min-w-0 flex-1">
              <div class="flex items-center gap-3">
                <label v-if="sellableIds.has(Number(item.id || 0))" class="shrink-0">
                  <input
                    :checked="isSelected(Number(item.id || 0))"
                    type="checkbox"
                    @change="toggleSelectItem(Number(item.id || 0))"
                  >
                </label>
                <div class="min-w-0">
                  <div class="truncate text-base font-semibold">
                    {{ item.name }}
                  </div>
                  <div class="mt-1 text-sm text-gray-500">
                    持有 {{ item.count }} · 出售 {{ item.sellCount }} · 保留 {{ item.keepCount }} · 单价 {{ item.unitPrice }}
                  </div>
                  <div v-if="formatPreviewReason(item)" class="mt-1 text-xs text-amber-500">
                    {{ formatPreviewReason(item) }}
                  </div>
                </div>
              </div>
            </div>
            <div class="text-right text-sm">
              <div class="text-emerald-500 font-semibold">
                预计 {{ item.sellValue }} 金币
              </div>
              <div class="mt-1 text-xs text-gray-400">
                ID #{{ item.id }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="space-y-4">
        <div class="glass-panel rounded-2xl p-4 shadow">
          <div class="flex flex-wrap items-center justify-between gap-4">
            <div>
              <div class="text-lg font-semibold">
                商城商品
              </div>
              <div class="mt-1 text-sm text-gray-500">
                支持按礼包内容、限购与免费状态快速筛选
              </div>
            </div>
            <div class="flex flex-wrap items-center gap-2">
              <div class="inventory-toolbar__search min-w-[220px]">
                <div class="i-carbon-search text-sm opacity-70" />
                <input v-model="mallQuery" type="text" class="inventory-search-input" placeholder="搜索商品 / ID / 内容物">
              </div>
              <label class="trade-select-wrap">
                <span class="trade-select-wrap__label">排序</span>
                <select v-model="mallSortBy" class="trade-select">
                  <option v-for="option in mallSortOptions" :key="option.key" :value="option.key">
                    {{ option.label }}
                  </option>
                </select>
              </label>
              <button class="trade-btn trade-btn-secondary" :disabled="mallLoading || actionLoading" @click="currentAccountId && bagStore.fetchMallGoods(currentAccountId)">
                刷新商城
              </button>
            </div>
          </div>

          <div class="inventory-toolbar__chips mt-4">
            <button
              v-for="option in mallFilterOptions"
              :key="option.key"
              class="filter-chip"
              :class="{ active: mallQuickFilter === option.key }"
              @click="setMallQuickFilter(option.key)"
            >
              <span>{{ option.label }}</span>
              <span class="filter-chip__count">{{ option.count }}</span>
            </button>
          </div>

          <div v-if="recommendedMallGoods.length" class="recommend-strip">
            <div class="recommend-strip__label">
              常买推荐
            </div>
            <div class="recommend-strip__list">
              <button
                v-for="goods in recommendedMallGoods"
                :key="`recommend-${goods.goodsId}`"
                class="recommend-card"
                @click="openMallDetail(goods)"
              >
                <div class="recommend-card__main">
                  <span class="recommend-card__title">{{ goods.name }}</span>
                  <span class="recommend-card__meta">{{ getMallPriceLabel(goods) }} · 已买 {{ goods.purchaseStats.count }} 次</span>
                </div>
                <div class="recommend-card__time">
                  {{ formatPurchaseTime(goods.purchaseStats.lastPurchasedAt) }}
                </div>
              </button>
            </div>
          </div>
        </div>

        <div v-if="mallLoading" class="flex justify-center py-10">
          <div class="i-svg-spinners-90-ring-with-bg text-3xl text-blue-500" />
        </div>

        <div v-else-if="!filteredMallGoods.length" class="glass-text-muted border border-white/20 rounded-lg bg-black/5 p-8 text-center backdrop-blur-sm dark:border-white/10 dark:bg-black/20">
          商城暂无可显示商品
        </div>

        <div v-else class="grid gap-4 lg:grid-cols-2 xl:grid-cols-3">
          <div
            v-for="goods in filteredMallGoods"
            :key="goods.goodsId"
            class="mall-card glass-panel rounded-2xl p-4 shadow"
            role="button"
            tabindex="0"
            @click="openMallDetail(goods)"
          >
            <div class="flex items-start gap-4">
              <div class="mall-card__art" :data-fallback="getMallFallbackLabel(goods)">
                <img
                  v-if="goods.image && !imageErrors[getImageErrorKey('mall', goods.goodsId)]"
                  :src="goods.image"
                  :alt="goods.name"
                  class="mall-card__image"
                  loading="lazy"
                  @error="imageErrors[getImageErrorKey('mall', goods.goodsId)] = true"
                >
                <div v-else class="inventory-card__fallback">
                  {{ getMallFallbackLabel(goods) }}
                </div>
              </div>

              <div class="min-w-0 flex-1">
                <div class="truncate text-base text-white/95 font-semibold">
                  {{ goods.name }}
                </div>
                <div class="mt-1 text-sm text-white/55">
                  商品ID #{{ goods.goodsId }} · 类型 {{ goods.type }}
                </div>
                <div class="mt-3 flex flex-wrap gap-2 text-xs">
                  <span class="inventory-pill inventory-pill-emerald">
                    {{ getMallPriceLabel(goods) }}
                  </span>
                  <span v-if="goods.isLimited" class="inventory-pill inventory-pill-amber">
                    限购
                  </span>
                  <span v-if="goods.discount" class="inventory-pill inventory-pill-sky">
                    {{ goods.discount }}
                  </span>
                </div>
                <div class="mall-card__summary">
                  {{ getGoodsSummary(goods) }}
                </div>
              </div>
              <div class="shrink-0 text-right text-xs text-white/45">
                包含 {{ goods.itemIds?.length || 0 }} 项
              </div>
            </div>

            <div v-if="goods.itemPreviews?.length" class="mall-card__previews">
              <div
                v-for="preview in getGoodsPreviewGroups(goods, 4)"
                :key="`${goods.goodsId}-${preview.id}`"
                class="mall-preview-pill"
                :title="preview.name"
              >
                <div class="mall-preview-pill__thumb" :data-fallback="getItemFallbackLabel(preview)">
                  <img
                    v-if="preview.image && !imageErrors[getImageErrorKey('mall-preview', goods.goodsId, preview.id)]"
                    :src="preview.image"
                    :alt="preview.name"
                    loading="lazy"
                    @error="imageErrors[getImageErrorKey('mall-preview', goods.goodsId, preview.id)] = true"
                  >
                  <div v-else class="inventory-card__fallback mall-preview-pill__fallback">
                    {{ getItemFallbackLabel(preview) }}
                  </div>
                </div>
                <span class="mall-preview-pill__name">{{ preview.name }}<template v-if="preview.count > 1"> x{{ preview.count }}</template></span>
              </div>
              <div v-if="getGoodsPreviewGroups(goods).length > 4" class="mall-preview-pill mall-preview-pill--more">
                另有 {{ getGoodsPreviewGroups(goods).length - 4 }} 项
              </div>
            </div>

            <div class="mall-card__actions">
              <div class="trade-stepper" @click.stop>
                <button class="trade-stepper__btn" @click="bumpPurchaseCount(goods.goodsId, -1)">
                  -
                </button>
                <input
                  :value="getPurchaseCount(goods.goodsId)"
                  type="number"
                  min="1"
                  class="trade-input trade-input-stepper"
                  @input="updatePurchaseCount(goods.goodsId, Number(($event.target as HTMLInputElement).value || 1))"
                >
                <button class="trade-stepper__btn" @click="bumpPurchaseCount(goods.goodsId, 1)">
                  +
                </button>
              </div>
              <button class="trade-btn trade-btn-secondary" :disabled="actionLoading" @click.stop="openMallDetail(goods)">
                详情
              </button>
              <button class="trade-btn trade-btn-primary flex-1" :disabled="actionLoading" @click.stop="handleBuyGoods(goods)">
                购买
              </button>
            </div>
          </div>
        </div>
      </div>
    </template>

    <Teleport to="body">
      <div v-if="bagDetailItem || mallDetailGoods" class="detail-modal-backdrop" @click="closeDetailPanel">
        <div class="detail-modal-panel" @click.stop>
          <div class="detail-modal-header">
            <div class="detail-modal-header__copy">
              <div class="detail-modal-header__eyebrow">
                {{ bagDetailItem ? '物品详情' : '商城商品详情' }}
              </div>
              <div class="detail-modal-header__title">
                {{ bagDetailItem?.name || mallDetailGoods?.name }}
              </div>
              <div class="detail-modal-header__sub">
                <template v-if="bagDetailItem">
                  物品ID #{{ bagDetailItem.id }}
                </template>
                <template v-else>
                  商品ID #{{ mallDetailGoods?.goodsId }} · 槽位 {{ mallDetailGoods?.slotType }}
                </template>
              </div>
            </div>
            <button class="detail-modal-close" @click="closeDetailPanel">
              关闭
            </button>
          </div>

          <template v-if="bagDetailItem">
            <div class="detail-hero">
              <div class="detail-hero__art" :data-fallback="getItemFallbackLabel(bagDetailItem)">
                <img
                  v-if="bagDetailItem.image && !imageErrors[getImageErrorKey('bag-detail', bagDetailItem.id)]"
                  :src="bagDetailItem.image"
                  :alt="bagDetailItem.name"
                  class="detail-hero__image"
                  loading="lazy"
                  @error="imageErrors[getImageErrorKey('bag-detail', bagDetailItem.id)] = true"
                >
                <div v-else class="inventory-card__fallback">
                  {{ getItemFallbackLabel(bagDetailItem) }}
                </div>
              </div>
              <div class="detail-hero__copy">
                <div class="detail-badges">
                  <span class="inventory-pill">{{ getBagCategoryLabel(resolveBagCategory(bagDetailItem)) }}</span>
                  <span class="inventory-pill inventory-pill-sky">稀有度 {{ getRarityLabel(bagDetailItem.rarity) }}</span>
                  <span class="inventory-pill" :class="bagDetailItem.canUse ? 'inventory-pill-emerald' : 'inventory-pill-amber'">
                    {{ bagDetailItem.canUse ? '可使用' : '不可直接使用' }}
                  </span>
                </div>
                <div class="detail-summary">
                  {{ getItemDescription(bagDetailItem) || '暂无物品说明' }}
                </div>
                <div class="detail-actions">
                  <div v-if="bagDetailItem.canUse && !itemNeedsLandSelection(bagDetailItem)" class="trade-stepper">
                    <button class="trade-stepper__btn" @click="bumpUseCount(bagDetailItem.id, -1, bagDetailItem.count)">
                      -
                    </button>
                    <input
                      :value="getUseCount(bagDetailItem.id, bagDetailItem.count)"
                      type="number"
                      min="1"
                      :max="Math.max(1, Number(bagDetailItem.count || 1))"
                      class="trade-input trade-input-stepper"
                      @input="updateUseCount(bagDetailItem.id, Number(($event.target as HTMLInputElement).value || 1), bagDetailItem.count)"
                    >
                    <button class="trade-stepper__btn" @click="bumpUseCount(bagDetailItem.id, 1, bagDetailItem.count)">
                      +
                    </button>
                  </div>
                  <div v-if="bagDetailItem.canUse && itemNeedsLandSelection(bagDetailItem)" class="detail-land-counter">
                    已选 {{ selectedLandIds.length }} / {{ getLandSelectionLimit(bagDetailItem) }} 块，将按选中地块数消耗
                  </div>
                  <button
                    v-if="bagDetailItem.canUse"
                    class="trade-btn trade-btn-primary"
                    :disabled="actionLoading"
                    @click="handleUseBagItem(bagDetailItem)"
                  >
                    立即使用
                  </button>
                  <button
                    v-if="bagDetailItem.category === 'fruit'"
                    class="trade-btn trade-btn-primary"
                    @click="toggleSelectItem(Number(bagDetailItem.id || 0))"
                  >
                    {{ isSelected(Number(bagDetailItem.id || 0)) ? '移出出售选择' : '加入出售选择' }}
                  </button>
                  <button class="trade-btn trade-btn-secondary" @click="closeDetailPanel">
                    返回列表
                  </button>
                </div>
              </div>
            </div>

            <div class="detail-stats">
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">当前持有</span>
                <span class="detail-stat-card__value">{{ bagDetailItem.hoursText || `x${bagDetailItem.count || 0}` }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">等级 / 单价</span>
                <span class="detail-stat-card__value">Lv{{ bagDetailItem.level || 0 }} · {{ bagDetailItem.price || 0 }}金</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">类型 / 交互</span>
                <span class="detail-stat-card__value">{{ bagDetailItem.itemType || 0 }} · {{ bagDetailItem.interactionType || '无' }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">堆叠上限</span>
                <span class="detail-stat-card__value">{{ bagDetailItem.maxCount || 0 }} / {{ bagDetailItem.maxOwn || 0 }}</span>
              </div>
            </div>

            <div class="detail-sections">
              <div v-for="(text, index) in getItemLongDescription(bagDetailItem)" :key="`bag-desc-${index}`" class="detail-section-card">
                {{ text }}
              </div>
            </div>

            <template v-if="bagDetailItem.canUse && itemNeedsLandSelection(bagDetailItem)">
              <div class="detail-section-title">
                目标土地
              </div>
              <div class="detail-land-toolbar">
                <div class="detail-land-toolbar__hint">
                  {{ getLandSelectionHint(bagDetailItem) }}
                </div>
                <div class="detail-land-toolbar__actions">
                  <button class="trade-btn trade-btn-secondary" :disabled="landsLoading" @click="ensureFarmLandsLoaded()">
                    刷新土地
                  </button>
                  <button class="trade-btn trade-btn-secondary" :disabled="suggestedTargetLandIds.length === 0" @click="selectSuggestedLands()">
                    智能选择
                  </button>
                  <button class="trade-btn trade-btn-secondary" :disabled="selectedLandIds.length === 0" @click="clearSelectedLands()">
                    清空选择
                  </button>
                </div>
              </div>
              <div v-if="landsLoading" class="detail-land-empty">
                正在加载土地...
              </div>
              <div v-else-if="availableTargetLands.length === 0" class="detail-land-empty">
                当前没有可用土地数据
              </div>
              <div v-else class="detail-land-grid">
                <button
                  v-for="land in availableTargetLands"
                  :key="`use-land-${land.id}`"
                  class="detail-land-card"
                  :class="{ active: isLandSelected(Number(land.id || 0)), suggested: suggestedTargetLandIds.includes(Number(land.id || 0)) }"
                  @click="toggleSelectedLand(Number(land.id || 0))"
                >
                  <div class="detail-land-card__head">
                    <span class="detail-land-card__id">#{{ land.id }}</span>
                    <span class="detail-land-card__status">{{ normalizeLandStatusLabel(land) }}</span>
                  </div>
                  <div class="detail-land-card__name">
                    {{ land.plantName || land.phaseName || '空地' }}
                  </div>
                  <div class="detail-land-card__meta">
                    <span v-if="land.needWater">缺水</span>
                    <span v-if="land.needWeed">有草</span>
                    <span v-if="land.needBug">有虫</span>
                    <span v-if="Number(land.matureInSec || 0) > 0">{{ Number(land.matureInSec || 0) }}s</span>
                    <span v-if="!land.needWater && !land.needWeed && !land.needBug && Number(land.matureInSec || 0) <= 0">普通</span>
                  </div>
                </button>
              </div>
            </template>

            <template v-if="lastUseResult && Number(lastUseResult.itemId || 0) === Number(bagDetailItem.id || 0)">
              <div class="detail-section-title">
                本次使用结果
              </div>
              <div class="detail-use-summary">
                <div class="detail-use-summary__message">
                  {{ lastUseResult.rewardSummary || lastUseResult.message || '使用成功' }}
                </div>
                <div class="detail-use-summary__meta">
                  <span>使用数量 {{ lastUseResult.count || 1 }}</span>
                  <span v-if="lastUseResult.landIds?.length">目标土地 {{ lastUseResult.landIds.join(', ') }}</span>
                </div>
              </div>
              <div v-if="lastUseResult.rewardItems?.length" class="detail-preview-grid">
                <div
                  v-for="reward in lastUseResult.rewardItems"
                  :key="`use-result-${bagDetailItem.id}-${reward.id}`"
                  class="detail-preview-card"
                >
                  <div class="detail-preview-card__thumb" :data-fallback="getItemFallbackLabel(reward)">
                    <img
                      v-if="reward.image && !imageErrors[getImageErrorKey('use-result', bagDetailItem.id, reward.id)]"
                      :src="reward.image"
                      :alt="reward.name"
                      loading="lazy"
                      @error="imageErrors[getImageErrorKey('use-result', bagDetailItem.id, reward.id)] = true"
                    >
                    <div v-else class="inventory-card__fallback detail-preview-card__fallback">
                      {{ getItemFallbackLabel(reward) }}
                    </div>
                  </div>
                  <div class="detail-preview-card__copy">
                    <div class="detail-preview-card__title">
                      {{ reward.name }}
                    </div>
                    <div class="detail-preview-card__meta">
                      <span>{{ getBagCategoryLabel(resolveBagCategory(reward)) }}</span>
                      <span>x{{ reward.count }}</span>
                      <span v-if="reward.level > 0">Lv{{ reward.level }}</span>
                    </div>
                    <div v-if="reward.effectDesc || reward.desc" class="detail-preview-card__desc">
                      {{ reward.effectDesc || reward.desc }}
                    </div>
                  </div>
                </div>
              </div>
            </template>
          </template>

          <template v-else-if="mallDetailGoods">
            <div class="detail-hero">
              <div class="detail-hero__art" :data-fallback="getMallFallbackLabel(mallDetailGoods)">
                <img
                  v-if="mallDetailGoods.image && !imageErrors[getImageErrorKey('mall-detail', mallDetailGoods.goodsId)]"
                  :src="mallDetailGoods.image"
                  :alt="mallDetailGoods.name"
                  class="detail-hero__image"
                  loading="lazy"
                  @error="imageErrors[getImageErrorKey('mall-detail', mallDetailGoods.goodsId)] = true"
                >
                <div v-else class="inventory-card__fallback">
                  {{ getMallFallbackLabel(mallDetailGoods) }}
                </div>
              </div>
              <div class="detail-hero__copy">
                <div class="detail-badges">
                  <span class="inventory-pill inventory-pill-emerald">{{ getMallPriceLabel(mallDetailGoods) }}</span>
                  <span v-if="mallDetailGoods.isLimited" class="inventory-pill inventory-pill-amber">限购商品</span>
                  <span class="inventory-pill inventory-pill-sky">内容 {{ getGoodsPreviewGroups(mallDetailGoods).length }} 种</span>
                </div>
                <div class="detail-summary">
                  {{ getGoodsSummary(mallDetailGoods) }}
                </div>
                <div class="detail-actions">
                  <div class="trade-stepper">
                    <button class="trade-stepper__btn" @click="bumpPurchaseCount(mallDetailGoods.goodsId, -1)">
                      -
                    </button>
                    <input
                      :value="getPurchaseCount(mallDetailGoods.goodsId)"
                      type="number"
                      min="1"
                      class="trade-input trade-input-stepper"
                      @input="updatePurchaseCount(mallDetailGoods.goodsId, Number(($event.target as HTMLInputElement).value || 1))"
                    >
                    <button class="trade-stepper__btn" @click="bumpPurchaseCount(mallDetailGoods.goodsId, 1)">
                      +
                    </button>
                  </div>
                  <button class="trade-btn trade-btn-primary" :disabled="actionLoading" @click="handleBuyGoods(mallDetailGoods)">
                    立即购买
                  </button>
                </div>
              </div>
            </div>

            <div class="detail-stats">
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">价格</span>
                <span class="detail-stat-card__value">{{ getMallPriceLabel(mallDetailGoods) }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">商品类型</span>
                <span class="detail-stat-card__value">{{ mallDetailGoods.type || 0 }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">内容总项</span>
                <span class="detail-stat-card__value">{{ mallDetailGoods.itemIds?.length || 0 }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">主物品</span>
                <span class="detail-stat-card__value">#{{ mallDetailGoods.primaryItemId || 0 }}</span>
              </div>
              <div class="detail-stat-card">
                <span class="detail-stat-card__label">购买记录</span>
                <span class="detail-stat-card__value">
                  {{ getMallPurchaseStats(mallDetailGoods.goodsId).count || 0 }} 次 · {{ formatPurchaseTime(getMallPurchaseStats(mallDetailGoods.goodsId).lastPurchasedAt) }}
                </span>
              </div>
            </div>

            <div class="detail-section-title">
              内容物清单
            </div>
            <div class="detail-preview-grid">
              <div v-for="preview in getGoodsPreviewGroups(mallDetailGoods)" :key="`detail-preview-${mallDetailGoods.goodsId}-${preview.id}`" class="detail-preview-card">
                <div class="detail-preview-card__thumb" :data-fallback="getItemFallbackLabel(preview)">
                  <img
                    v-if="preview.image && !imageErrors[getImageErrorKey('mall-detail-preview', mallDetailGoods.goodsId, preview.id)]"
                    :src="preview.image"
                    :alt="preview.name"
                    loading="lazy"
                    @error="imageErrors[getImageErrorKey('mall-detail-preview', mallDetailGoods.goodsId, preview.id)] = true"
                  >
                  <div v-else class="inventory-card__fallback detail-preview-card__fallback">
                    {{ getItemFallbackLabel(preview) }}
                  </div>
                </div>
                <div class="detail-preview-card__copy">
                  <div class="detail-preview-card__title">
                    {{ preview.name }}
                  </div>
                  <div class="detail-preview-card__meta">
                    <span>{{ getBagCategoryLabel(resolveBagCategory(preview)) }}</span>
                    <span v-if="preview.count > 1">x{{ preview.count }}</span>
                    <span v-if="preview.level > 0">Lv{{ preview.level }}</span>
                  </div>
                  <div v-if="preview.effectDesc || preview.desc" class="detail-preview-card__desc">
                    {{ preview.effectDesc || preview.desc }}
                  </div>
                </div>
              </div>
            </div>
          </template>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.trade-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 999px;
  padding: 0.55rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  transition: all 0.2s ease;
  border: 1px solid transparent;
}

.trade-btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

.trade-btn-primary {
  background: linear-gradient(135deg, #0f766e, #14b8a6);
  color: #fff;
}

.trade-btn-secondary {
  background: rgba(15, 23, 42, 0.04);
  color: #334155;
  border-color: rgba(148, 163, 184, 0.35);
}

.trade-tab {
  border-radius: 999px;
  border: 1px solid rgba(148, 163, 184, 0.35);
  background: rgba(255, 255, 255, 0.5);
  padding: 0.55rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
}

.trade-tab.active {
  background: linear-gradient(135deg, #0f766e, #14b8a6);
  color: #fff;
  border-color: transparent;
}

.trade-input {
  width: 88px;
  border-radius: 999px;
  border: 1px solid rgba(148, 163, 184, 0.35);
  background: rgba(255, 255, 255, 0.7);
  padding: 0.55rem 0.85rem;
  font-size: 0.875rem;
}

.trade-input-stepper {
  width: 64px;
  padding-left: 0;
  padding-right: 0;
  text-align: center;
}

.inventory-toolbar {
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
}

.inventory-toolbar__row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.9rem;
  align-items: center;
  justify-content: space-between;
}

.inventory-toolbar__search {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.05);
  padding: 0.72rem 0.95rem;
  color: rgba(255, 255, 255, 0.88);
}

.inventory-search-input {
  width: 100%;
  border: 0;
  background: transparent;
  outline: none;
  color: inherit;
  font-size: 0.92rem;
}

.inventory-search-input::placeholder {
  color: rgba(255, 255, 255, 0.38);
}

.trade-select-wrap {
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.05);
  padding: 0.28rem 0.42rem 0.28rem 0.85rem;
  color: rgba(255, 255, 255, 0.82);
}

.trade-select-wrap__label {
  font-size: 0.78rem;
  color: rgba(255, 255, 255, 0.58);
}

.trade-select {
  min-width: 148px;
  border: 0;
  border-radius: 999px;
  background: rgba(15, 23, 42, 0.38);
  color: #fff;
  padding: 0.45rem 0.85rem;
  outline: none;
  font-size: 0.85rem;
}

.inventory-toolbar__chips {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
}

.activity-panel {
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
}

.activity-panel__header {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 0.8rem;
}

.activity-panel__title {
  color: rgba(255, 255, 255, 0.92);
  font-size: 1rem;
  font-weight: 700;
}

.activity-panel__subtitle {
  margin-top: 0.2rem;
  color: rgba(255, 255, 255, 0.52);
  font-size: 0.8rem;
}

.activity-panel__actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.7rem;
  align-items: center;
}

.activity-panel__summary {
  color: rgba(255, 255, 255, 0.56);
  font-size: 0.78rem;
}

.activity-panel__empty {
  color: rgba(255, 255, 255, 0.56);
}

.activity-list {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.75rem;
}

.activity-card {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  border-radius: 18px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: rgba(255, 255, 255, 0.05);
  padding: 0.9rem 1rem;
  transition:
    transform 0.2s ease,
    border-color 0.2s ease;
}

.activity-card:hover {
  transform: translateY(-1px);
  border-color: rgba(45, 212, 191, 0.24);
}

.activity-card__body {
  display: flex;
  align-items: flex-start;
  gap: 0.85rem;
  text-align: left;
  cursor: pointer;
}

.activity-card__actions {
  display: flex;
  justify-content: flex-end;
}

.activity-card__details {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.7rem;
}

.activity-card__type {
  flex-shrink: 0;
  border-radius: 999px;
  padding: 0.28rem 0.58rem;
  font-size: 0.74rem;
  font-weight: 700;
}

.activity-card__type.use {
  background: rgba(45, 212, 191, 0.12);
  color: rgba(153, 246, 228, 0.95);
}

.activity-card__type.purchase {
  background: rgba(251, 191, 36, 0.14);
  color: rgba(253, 224, 71, 0.95);
}

.activity-card__type.sell {
  background: rgba(96, 165, 250, 0.14);
  color: rgba(191, 219, 254, 0.95);
}

.activity-card__main {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.28rem;
}

.activity-card__title {
  color: rgba(255, 255, 255, 0.94);
  font-size: 0.92rem;
  font-weight: 700;
}

.activity-card__meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.45rem;
  color: rgba(255, 255, 255, 0.5);
  font-size: 0.76rem;
}

.activity-card__summary {
  color: rgba(255, 255, 255, 0.68);
  font-size: 0.82rem;
  line-height: 1.5;
}

.activity-detail-card {
  display: flex;
  gap: 0.7rem;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.05);
  padding: 0.75rem;
}

.activity-detail-card__thumb {
  width: 56px;
  height: 56px;
  flex-shrink: 0;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.08);
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.activity-detail-card__thumb img {
  max-width: 80%;
  max-height: 80%;
  object-fit: contain;
}

.activity-detail-card__copy {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.28rem;
}

.activity-detail-card__title {
  color: rgba(255, 255, 255, 0.92);
  font-size: 0.84rem;
  font-weight: 700;
}

.activity-detail-card__meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  color: rgba(255, 255, 255, 0.56);
  font-size: 0.74rem;
  line-height: 1.45;
}

.activity-detail-card__fallback {
  font-size: 0.8rem;
}

.filter-chip {
  display: inline-flex;
  align-items: center;
  gap: 0.55rem;
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.05);
  padding: 0.5rem 0.85rem;
  color: rgba(255, 255, 255, 0.82);
  transition: 0.2s ease;
}

.filter-chip.active {
  border-color: rgba(45, 212, 191, 0.55);
  background: linear-gradient(135deg, rgba(20, 184, 166, 0.25), rgba(13, 148, 136, 0.32));
  color: #fff;
}

.filter-chip__count {
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  padding: 0.1rem 0.45rem;
  font-size: 0.72rem;
}

.inventory-card {
  position: relative;
  overflow: hidden;
  display: flex;
  min-height: 250px;
  flex-direction: column;
  border-radius: 24px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background:
    radial-gradient(circle at top, rgba(255, 255, 255, 0.1), transparent 46%),
    linear-gradient(180deg, rgba(255, 255, 255, 0.08), rgba(15, 23, 42, 0.12));
  padding: 1rem;
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease,
    border-color 0.2s ease;
  cursor: pointer;
}

.inventory-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 16px 34px rgba(2, 6, 23, 0.18);
}

.inventory-card-fruit {
  border-color: rgba(251, 146, 60, 0.28);
}

.inventory-card-seed {
  border-color: rgba(74, 222, 128, 0.26);
}

.inventory-card-fertilizer {
  border-color: rgba(56, 189, 248, 0.24);
}

.inventory-card-pack {
  border-color: rgba(244, 114, 182, 0.24);
}

.inventory-card-pet {
  border-color: rgba(168, 85, 247, 0.24);
}

.inventory-card-item {
  border-color: rgba(148, 163, 184, 0.24);
}

.inventory-card__art,
.mall-card__art {
  position: relative;
  margin-top: 2.3rem;
  height: 112px;
  border-radius: 22px;
  background: radial-gradient(circle at top, rgba(255, 255, 255, 0.16), transparent 60%), rgba(255, 255, 255, 0.06);
  display: flex;
  align-items: center;
  justify-content: center;
}

.inventory-card__image,
.mall-card__image {
  max-width: 78%;
  max-height: 78%;
  object-fit: contain;
  filter: drop-shadow(0 10px 20px rgba(15, 23, 42, 0.24));
}

.inventory-card__fallback {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.55rem;
  font-weight: 800;
  letter-spacing: 0.08em;
  color: rgba(255, 255, 255, 0.88);
}

.inventory-card__count {
  margin-top: 0.9rem;
  color: rgba(255, 255, 255, 0.94);
  font-size: 1.45rem;
  font-weight: 700;
}

.inventory-card__name {
  margin-top: 0.55rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: rgba(255, 255, 255, 0.95);
  font-size: 1rem;
  font-weight: 700;
}

.inventory-card__meta {
  margin-top: 0.55rem;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  color: rgba(255, 255, 255, 0.6);
  font-size: 0.8rem;
}

.inventory-card__footer {
  margin-top: auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding-top: 0.85rem;
  color: rgba(255, 255, 255, 0.44);
  font-size: 0.72rem;
}

.inventory-card__footer-tag {
  border-radius: 999px;
  background: rgba(16, 185, 129, 0.16);
  padding: 0.18rem 0.48rem;
  color: #a7f3d0;
}

.inventory-pill {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  width: fit-content;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.1);
  padding: 0.22rem 0.55rem;
  color: rgba(255, 255, 255, 0.86);
  font-size: 0.72rem;
  font-weight: 600;
}

.inventory-pill-emerald {
  background: rgba(16, 185, 129, 0.16);
  color: #a7f3d0;
}

.inventory-pill-amber {
  background: rgba(245, 158, 11, 0.16);
  color: #fde68a;
}

.inventory-pill-sky {
  background: rgba(14, 165, 233, 0.16);
  color: #bae6fd;
}

.mall-card {
  border: 1px solid rgba(255, 255, 255, 0.12);
  background:
    radial-gradient(circle at top right, rgba(45, 212, 191, 0.14), transparent 34%),
    linear-gradient(180deg, rgba(255, 255, 255, 0.08), rgba(15, 23, 42, 0.16));
  cursor: pointer;
}

.mall-card__summary {
  margin-top: 0.7rem;
  color: rgba(255, 255, 255, 0.58);
  font-size: 0.8rem;
  line-height: 1.5;
}

.mall-card__previews {
  margin-top: 1rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.55rem;
}

.mall-preview-pill {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  max-width: 100%;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.06);
  padding: 0.35rem 0.55rem 0.35rem 0.4rem;
}

.mall-preview-pill--more {
  color: rgba(255, 255, 255, 0.68);
}

.mall-preview-pill__thumb {
  height: 30px;
  width: 30px;
  flex-shrink: 0;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.mall-preview-pill__thumb img {
  max-width: 80%;
  max-height: 80%;
  object-fit: contain;
}

.mall-preview-pill__name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 120px;
  color: rgba(255, 255, 255, 0.78);
  font-size: 0.78rem;
}

.mall-preview-pill__fallback {
  font-size: 0.78rem;
}

.mall-card__actions {
  margin-top: 1rem;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.75rem;
}

.recommend-strip {
  margin-top: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
}

.recommend-strip__label {
  color: rgba(255, 255, 255, 0.55);
  font-size: 0.8rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.recommend-strip__list {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.7rem;
}

.recommend-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.8rem;
  border-radius: 18px;
  border: 1px solid rgba(45, 212, 191, 0.14);
  background: linear-gradient(135deg, rgba(20, 184, 166, 0.12), rgba(255, 255, 255, 0.04));
  padding: 0.85rem 0.95rem;
  text-align: left;
  transition:
    transform 0.2s ease,
    border-color 0.2s ease;
}

.recommend-card:hover {
  transform: translateY(-1px);
  border-color: rgba(45, 212, 191, 0.28);
}

.recommend-card__main {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.recommend-card__title {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: rgba(255, 255, 255, 0.92);
  font-size: 0.92rem;
  font-weight: 700;
}

.recommend-card__meta,
.recommend-card__time {
  color: rgba(255, 255, 255, 0.54);
  font-size: 0.77rem;
}

.trade-stepper {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.06);
  padding: 0.28rem;
}

.trade-stepper__btn {
  width: 32px;
  height: 32px;
  border-radius: 999px;
  border: 0;
  background: rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.9);
  font-size: 1rem;
  font-weight: 700;
}

.detail-modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 80;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.25rem;
  background: rgba(2, 6, 23, 0.56);
  backdrop-filter: blur(12px);
}

.detail-modal-panel {
  width: min(980px, 100%);
  max-height: calc(100vh - 2rem);
  overflow: auto;
  border-radius: 28px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background:
    radial-gradient(circle at top right, rgba(45, 212, 191, 0.12), transparent 28%),
    linear-gradient(180deg, rgba(71, 14, 54, 0.98), rgba(43, 11, 34, 0.98));
  padding: 1.25rem;
  box-shadow: 0 28px 60px rgba(2, 6, 23, 0.4);
}

.detail-modal-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  margin-bottom: 1.25rem;
}

.detail-modal-header__eyebrow {
  color: rgba(255, 255, 255, 0.48);
  font-size: 0.74rem;
  letter-spacing: 0.18em;
  text-transform: uppercase;
}

.detail-modal-header__title {
  margin-top: 0.4rem;
  color: rgba(255, 255, 255, 0.96);
  font-size: 1.45rem;
  font-weight: 800;
}

.detail-modal-header__sub {
  margin-top: 0.35rem;
  color: rgba(255, 255, 255, 0.5);
  font-size: 0.86rem;
}

.detail-modal-close {
  border-radius: 999px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.06);
  color: rgba(255, 255, 255, 0.84);
  padding: 0.6rem 0.95rem;
  font-size: 0.85rem;
  font-weight: 600;
}

.detail-hero {
  display: grid;
  grid-template-columns: 220px minmax(0, 1fr);
  gap: 1rem;
}

.detail-hero__art {
  min-height: 220px;
  border-radius: 24px;
  background: radial-gradient(circle at top, rgba(255, 255, 255, 0.14), transparent 56%), rgba(255, 255, 255, 0.05);
  display: flex;
  align-items: center;
  justify-content: center;
}

.detail-hero__image {
  max-width: 78%;
  max-height: 78%;
  object-fit: contain;
  filter: drop-shadow(0 12px 24px rgba(15, 23, 42, 0.28));
}

.detail-hero__copy {
  display: flex;
  min-width: 0;
  flex-direction: column;
  gap: 0.9rem;
}

.detail-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 0.55rem;
}

.detail-summary {
  color: rgba(255, 255, 255, 0.72);
  line-height: 1.7;
  font-size: 0.94rem;
}

.detail-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.detail-land-counter {
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  background: rgba(45, 212, 191, 0.12);
  padding: 0.6rem 0.95rem;
  color: rgba(153, 246, 228, 0.95);
  font-size: 0.84rem;
  font-weight: 600;
}

.detail-stats {
  margin-top: 1rem;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 0.8rem;
}

.detail-stat-card {
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.05);
  padding: 0.95rem 1rem;
}

.detail-stat-card__label {
  color: rgba(255, 255, 255, 0.45);
  font-size: 0.76rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.detail-stat-card__value {
  color: rgba(255, 255, 255, 0.92);
  font-size: 0.95rem;
  font-weight: 700;
}

.detail-sections {
  margin-top: 1rem;
  display: grid;
  gap: 0.8rem;
}

.detail-section-card {
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.05);
  padding: 1rem 1.05rem;
  color: rgba(255, 255, 255, 0.72);
  line-height: 1.72;
}

.detail-section-title {
  margin-top: 1.15rem;
  color: rgba(255, 255, 255, 0.9);
  font-size: 1rem;
  font-weight: 700;
}

.detail-land-toolbar {
  margin-top: 0.85rem;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 0.8rem;
}

.detail-land-toolbar__hint {
  color: rgba(255, 255, 255, 0.56);
  font-size: 0.84rem;
}

.detail-land-toolbar__actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.55rem;
}

.detail-land-empty {
  margin-top: 0.85rem;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.05);
  padding: 1rem;
  color: rgba(255, 255, 255, 0.58);
  text-align: center;
}

.detail-land-grid {
  margin-top: 0.85rem;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.8rem;
}

.detail-land-card {
  display: flex;
  min-width: 0;
  flex-direction: column;
  gap: 0.45rem;
  border-radius: 18px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: rgba(255, 255, 255, 0.05);
  padding: 0.9rem;
  text-align: left;
  transition:
    transform 0.2s ease,
    border-color 0.2s ease,
    background 0.2s ease;
}

.detail-land-card:hover {
  transform: translateY(-1px);
  border-color: rgba(255, 255, 255, 0.18);
}

.detail-land-card.active {
  border-color: rgba(45, 212, 191, 0.52);
  background: linear-gradient(135deg, rgba(20, 184, 166, 0.16), rgba(255, 255, 255, 0.04));
}

.detail-land-card.suggested:not(.active) {
  border-color: rgba(125, 211, 252, 0.28);
}

.detail-land-card__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.7rem;
}

.detail-land-card__id {
  color: rgba(255, 255, 255, 0.84);
  font-size: 0.78rem;
  font-weight: 700;
}

.detail-land-card__status {
  color: rgba(255, 255, 255, 0.52);
  font-size: 0.74rem;
}

.detail-land-card__name {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: rgba(255, 255, 255, 0.94);
  font-size: 0.9rem;
  font-weight: 700;
}

.detail-land-card__meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35rem;
  color: rgba(255, 255, 255, 0.62);
  font-size: 0.76rem;
}

.detail-use-summary {
  margin-top: 0.85rem;
  border-radius: 20px;
  background: linear-gradient(135deg, rgba(20, 184, 166, 0.12), rgba(255, 255, 255, 0.04));
  padding: 1rem 1.05rem;
}

.detail-use-summary__message {
  color: rgba(255, 255, 255, 0.92);
  font-size: 0.94rem;
  font-weight: 700;
}

.detail-use-summary__meta {
  margin-top: 0.45rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.55rem;
  color: rgba(255, 255, 255, 0.56);
  font-size: 0.8rem;
}

.detail-preview-grid {
  margin-top: 0.85rem;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 0.8rem;
}

.detail-preview-card {
  display: flex;
  gap: 0.8rem;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.05);
  padding: 0.85rem;
}

.detail-preview-card__thumb {
  width: 72px;
  height: 72px;
  flex-shrink: 0;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.08);
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.detail-preview-card__thumb img {
  max-width: 82%;
  max-height: 82%;
  object-fit: contain;
}

.detail-preview-card__copy {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.detail-preview-card__title {
  color: rgba(255, 255, 255, 0.94);
  font-weight: 700;
}

.detail-preview-card__meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.45rem;
  color: rgba(255, 255, 255, 0.52);
  font-size: 0.78rem;
}

.detail-preview-card__desc {
  color: rgba(255, 255, 255, 0.68);
  font-size: 0.82rem;
  line-height: 1.55;
}

.detail-preview-card__fallback {
  font-size: 0.85rem;
}

@media (max-width: 768px) {
  .inventory-toolbar__row {
    flex-direction: column;
    align-items: stretch;
  }

  .use-history-list {
    grid-template-columns: 1fr;
  }

  .activity-list {
    grid-template-columns: 1fr;
  }

  .activity-card__details {
    grid-template-columns: 1fr;
  }

  .trade-select-wrap {
    width: 100%;
    justify-content: space-between;
  }

  .trade-select {
    min-width: 0;
    width: 100%;
  }

  .inventory-card {
    min-height: 222px;
    padding: 0.9rem;
  }

  .inventory-card__art,
  .mall-card__art {
    height: 96px;
  }

  .mall-preview-pill__name {
    max-width: 88px;
  }

  .mall-card__actions {
    flex-direction: column;
    align-items: stretch;
  }

  .recommend-strip__list {
    grid-template-columns: 1fr;
  }

  .trade-stepper {
    width: 100%;
    justify-content: space-between;
  }

  .detail-modal-panel {
    padding: 1rem;
  }

  .detail-hero {
    grid-template-columns: 1fr;
  }

  .detail-hero__art {
    min-height: 176px;
  }

  .detail-stats,
  .detail-preview-grid {
    grid-template-columns: 1fr;
  }

  .detail-land-grid {
    grid-template-columns: 1fr;
  }
}
</style>
