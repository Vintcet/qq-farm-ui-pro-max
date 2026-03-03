<script setup lang="ts">
import { useIntervalFn } from '@vueuse/core'
import { storeToRefs } from 'pinia'
import { onMounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import { adminToken } from '@/utils/auth'
import AccountModal from '@/components/AccountModal.vue'
import ConfirmModal from '@/components/ConfirmModal.vue'
import BaseButton from '@/components/ui/BaseButton.vue'
import { useAccountStore } from '@/stores/account'
import { useAvatar } from '@/utils/avatar'

const { getAvatarUrl, markFailed } = useAvatar()

const router = useRouter()
const accountStore = useAccountStore()
const { accounts, loading } = storeToRefs(accountStore)

const showModal = ref(false)
const showDeleteConfirm = ref(false)
const deleteLoading = ref(false)
const editingAccount = ref<any>(null)
const accountToDelete = ref<any>(null)

onMounted(() => {
  accountStore.fetchAccounts()
})

// 监听 Token 变化，当用户重新登录或 Token 被赋值时强制同步一次
watch(adminToken, (val) => {
  if (val) {
    accountStore.fetchAccounts()
  }
})

useIntervalFn(() => {
  accountStore.fetchAccounts()
}, 3000)

function openSettings(account: any) {
  accountStore.selectAccount(account.id)
  router.push('/settings')
}

function openAddModal() {
  editingAccount.value = null
  showModal.value = true
}

function openEditModal(account: any) {
  editingAccount.value = { ...account }
  showModal.value = true
}

async function handleDelete(account: any) {
  accountToDelete.value = account
  showDeleteConfirm.value = true
}

async function confirmDelete() {
  if (accountToDelete.value) {
    try {
      deleteLoading.value = true
      await accountStore.deleteAccount(accountToDelete.value.id)
      accountToDelete.value = null
      showDeleteConfirm.value = false
    }
    finally {
      deleteLoading.value = false
    }
  }
}

async function toggleAccount(account: any) {
  if (account.running) {
    await accountStore.stopAccount(account.id)
  }
  else {
    await accountStore.startAccount(account.id)
  }
}

function handleSaved() {
  accountStore.fetchAccounts()
}

async function handleModeChange(acc: any, mode: string) {
  try {
    await accountStore.updateAccountMode(acc.id, mode)
  } catch (e: any) {
    alert('切换模式失败: ' + e.message)
  }
}

const safeCheckingId = ref('')
async function handleSafeCheck(acc: any) {
  if (confirm(`是否分析 ${acc.name || acc.id} 的历史封禁日志并自动补充黑名单？`)) {
    try {
      safeCheckingId.value = acc.id
      const res = await accountStore.applySafeModeBlacklist(acc.id)
      if (res && res.ok && res.data && res.data.length >= 0) {
        alert(`一键生成成功！\n共新增 ${res.data.length} 个黑名单好友。\n${res.data.join(', ')}`)
      } else {
        alert('可能无新增黑名单。')
      }
    } catch (e: any) {
      alert('生成失败: ' + e.message)
    } finally {
      safeCheckingId.value = ''
    }
  }
}
</script>

<template>
  <div class="mx-auto max-w-6xl w-full p-4">
    <div class="mb-6 flex items-center justify-between">
      <h1 class="text-2xl font-bold">
        账号管理
      </h1>
      <BaseButton
        variant="primary"
        @click="openAddModal"
      >
        <div class="i-carbon-add mr-2" />
        添加账号
      </BaseButton>
    </div>

    <div v-if="loading && accounts.length === 0" class="glass-panel min-h-[300px] flex flex-col items-center justify-center rounded-lg py-12 text-center shadow">
      <div class="relative mb-6">
        <div class="i-svg-spinners-ring-resize text-5xl text-primary-500" />
        <div class="absolute inset-0 animate-ping rounded-full bg-primary-400/20 blur-xl" />
      </div>
      <h3 class="text-lg font-semibold tracking-tight">
        正在同步云端账号...
      </h3>
      <p class="glass-text-muted mt-2 text-sm">
        正在为您的容器分配安全资源，请稍候
      </p>
    </div>

    <div v-else-if="accounts.length === 0" class="glass-panel rounded-lg py-12 text-center shadow">
      <div i-carbon-user-avatar class="mb-4 inline-block text-4xl glass-text-muted" />
      <p class="glass-text-muted mb-4">
        暂无账号
      </p>
      <BaseButton
        variant="text"
        @click="openAddModal"
      >
        立即添加
      </BaseButton>
    </div>

    <div v-else class="grid grid-cols-1 items-start gap-4 lg:grid-cols-3 sm:grid-cols-2">
      <div
        v-for="acc in accounts"
        :key="acc.id"
        class="glass-panel cursor-pointer rounded-lg p-4 shadow transition-all duration-300 border border-transparent hover:border-primary-500/50 hover:shadow-[0_0_15px_rgba(var(--color-primary-500),0.1)]"
        :class="acc.id === accountStore.currentAccountId ? 'border-primary-500/50 bg-primary-500/[0.03] shadow-[0_0_20px_rgba(var(--color-primary-500),0.15)] dark:border-primary-400/50' : ''"
        @click="accountStore.selectAccount(acc.id)"
      >
        <div class="mb-4 flex items-start justify-between">
          <div class="flex items-center gap-3">
            <div class="h-12 w-12 flex items-center justify-center overflow-hidden rounded-full bg-black/5 dark:bg-white/10">
              <img v-if="getAvatarUrl(acc)" :src="getAvatarUrl(acc)" class="h-full w-full object-cover" @error="(e) => markFailed((e.target as HTMLImageElement).src)">
              <div v-else class="i-carbon-user text-2xl glass-text-muted" />
            </div>
            <div>
              <h3 class="text-lg font-bold">
                {{ acc.name || acc.nick || acc.id }}
              </h3>
              <div class="glass-text-muted text-sm flex items-center gap-2 mt-1">
                <span>QQ: {{ acc.uin || '未绑定' }}</span>
                <select
                  v-model="(acc as any).accountMode"
                  @change="(e) => handleModeChange(acc, (e.target as HTMLSelectElement).value)"
                  class="bg-transparent text-xs border border-gray-200/50 dark:border-gray-700/50 rounded px-1 py-0.5 outline-none focus:ring-1 focus:ring-primary-500 cursor-pointer"
                  :class="{
                    'text-primary-500 dark:text-primary-400 font-medium': (acc.accountMode || (acc as any).account_mode || 'main') === 'main',
                    'text-amber-500 dark:text-amber-400': (acc.accountMode || (acc as any).account_mode) === 'alt',
                    'text-emerald-500 dark:text-emerald-400 font-bold': (acc.accountMode || (acc as any).account_mode) === 'safe'
                  }"
                  @click.stop
                >
                  <option value="main">主号</option>
                  <option value="alt">小号</option>
                  <option value="safe">风险规避</option>
                </select>
              </div>
            </div>
          </div>
          <div class="flex flex-col items-end gap-2">
            <BaseButton
              variant="secondary"
              size="sm"
              class="w-20 border rounded-full shadow-sm transition-all duration-500 ease-in-out active:scale-95"
              :class="acc.running ? 'border-red-500/20 bg-red-500/10 text-red-600 hover:bg-red-500/20 focus:ring-red-500/50 dark:border-red-500/30 dark:bg-red-500/10 dark:text-red-400 dark:hover:bg-red-500/20' : 'border-green-500/20 bg-green-500/10 text-green-600 hover:bg-green-500/20 focus:ring-green-500/50 dark:border-green-500/30 dark:bg-green-500/10 dark:text-green-400 dark:hover:bg-green-500/20'"
              @click.stop="toggleAccount(acc)"
            >
              <div :class="acc.running ? 'i-carbon-stop-filled' : 'i-carbon-play-filled'" class="mr-1" />
              {{ acc.running ? '停止' : '启动' }}
            </BaseButton>
          </div>
        </div>

        <div class="mt-2 flex items-center justify-between border-t border-gray-100/50 pt-4 dark:border-white/10">
          <div class="glass-text-muted flex items-center gap-2 text-sm">
            <span class="flex items-center gap-1">
              <div class="h-2 w-2 rounded-full" :class="acc.running ? 'bg-green-500' : 'bg-gray-300'" />
              {{ acc.running ? '运行中' : '已停止' }}
            </span>
            <transition name="fade">
              <BaseButton
                v-if="(acc.accountMode || (acc as any).account_mode) === 'safe'"
                variant="ghost"
                size="sm"
                class="!p-1 !text-xs !ml-2 text-emerald-600 dark:text-emerald-400 bg-emerald-500/10 hover:bg-emerald-500/20 rounded"
                title="一键分析封禁日志并加入黑名单"
                :loading="safeCheckingId === acc.id"
                @click.stop="handleSafeCheck(acc)"
              >
                <div class="i-carbon-security mr-1" />
                防封扫描
              </BaseButton>
            </transition>
          </div>

          <div class="flex gap-2">
            <BaseButton
              variant="ghost"
              class="!p-2"
              title="设置"
              @click.stop="openSettings(acc)"
            >
              <div class="i-carbon-settings text-lg" />
            </BaseButton>
            <BaseButton
              variant="ghost"
              class="!p-2"
              title="编辑"
              @click.stop="openEditModal(acc)"
            >
              <div class="i-carbon-edit text-lg" />
            </BaseButton>
            <BaseButton
              variant="ghost"
              class="text-red-500 !p-2 dark:text-red-400 hover:text-red-600 dark:hover:text-red-300"
              title="删除"
              @click.stop="handleDelete(acc)"
            >
              <div class="i-carbon-trash-can text-lg" />
            </BaseButton>
          </div>
        </div>
      </div>
    </div>

    <AccountModal
      :show="showModal"
      :edit-data="editingAccount"
      @close="showModal = false"
      @saved="handleSaved"
    />

    <ConfirmModal
      :show="showDeleteConfirm"
      :loading="deleteLoading"
      title="删除账号"
      :message="accountToDelete ? `确定要删除账号 ${accountToDelete.name || accountToDelete.id} 吗?` : ''"
      confirm-text="删除"
      type="danger"
      @close="!deleteLoading && (showDeleteConfirm = false)"
      @cancel="!deleteLoading && (showDeleteConfirm = false)"
      @confirm="confirmDelete"
    />
  </div>
</template>
