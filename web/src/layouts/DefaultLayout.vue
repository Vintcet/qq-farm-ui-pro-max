<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { computed, ref } from 'vue'
import LeaderboardModal from '@/components/LeaderboardModal.vue'
import NotificationBell from '@/components/NotificationBell.vue'
import Sidebar from '@/components/Sidebar.vue'
import ThemeSettingDrawer from '@/components/ThemeSettingDrawer.vue'
import { useAppStore } from '@/stores/app'

const appStore = useAppStore()
const { sidebarOpen } = storeToRefs(appStore)

const showThemeDrawer = ref(false)
const showLeaderboard = ref(false)
const workspaceShellClass = computed(() => `workspace-shell--${appStore.workspaceVisualPreset}`)
</script>

<template>
  <div class="workspace-shell w-screen flex overflow-hidden bg-transparent" :class="workspaceShellClass" style="height: 100dvh;">
    <div class="workspace-shell__wash" />
    <div class="workspace-shell__glow workspace-shell__glow--left" />
    <div class="workspace-shell__glow workspace-shell__glow--right" />
    <!-- Mobile Sidebar Overlay -->
    <div
      v-if="sidebarOpen"
      class="fixed inset-0 z-40 bg-gray-900/50 backdrop-blur-sm transition-opacity xl:hidden"
      @click="appStore.closeSidebar"
    />

    <Sidebar />

    <main class="relative z-10 h-full min-w-0 flex flex-1 flex-col overflow-hidden">
      <!-- Top Bar (Mobile/Tablet only or for additional actions) -->
      <header class="glass-panel h-14 flex shrink-0 items-center justify-between border-b border-gray-100/50 px-4 xl:hidden dark:border-gray-700/50">
        <button
          class="flex items-center justify-center rounded-lg p-2 text-gray-500 hover:bg-gray-100/50 dark:text-gray-400 dark:hover:bg-gray-700/50"
          @click="appStore.toggleSidebar"
        >
          <div class="i-carbon-menu text-xl" />
        </button>
        <div class="truncate text-sm font-semibold">
          QQ 农场智能助手
        </div>
        <!-- 右侧占位，保持标题居中 -->
        <div class="w-9" />
      </header>

      <!-- Main Content Area -->
      <div class="relative flex flex-1 flex-col overflow-hidden">
        <!-- 浮动操作区域 (配置与通知) -->
        <div class="absolute right-4 top-4 z-40 flex items-center gap-3">
          <button
            class="glass-panel h-10 w-10 flex items-center justify-center border border-amber-200/50 rounded-full from-amber-50 to-orange-100 bg-gradient-to-br shadow-md transition-all duration-300 hover:rotate-12 hover:scale-110 dark:border-amber-700/50 dark:from-amber-900/40 dark:to-orange-900/40 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-amber-500 dark:focus:ring-offset-gray-900"
            title="平台排行榜"
            @click="showLeaderboard = true"
          >
            <div class="i-carbon-trophy text-xl text-amber-600 dark:text-amber-400" />
          </button>
          <NotificationBell />
          <button
            class="glass-panel glass-text-main h-10 w-10 flex items-center justify-center border rounded-full shadow-md transition-all duration-300 hover:rotate-90 hover:text-primary focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary dark:focus:ring-offset-gray-900"
            title="系统配置"
            @click="showThemeDrawer = true"
          >
            <div class="i-carbon-settings text-xl" />
          </button>
        </div>

        <div class="workspace-scroll custom-scrollbar mt-12 flex flex-1 flex-col overflow-y-auto p-3 md:mt-0 md:p-6 sm:p-4 xl:p-7">
          <RouterView v-slot="{ Component, route }">
            <Transition name="slide-fade" mode="out-in">
              <component :is="Component" v-if="Component" :key="route.fullPath" />
              <div v-else :key="`loading-${route.fullPath}`" class="flex flex-1 items-center justify-center py-20 text-gray-400">
                <div class="i-svg-spinners-ring-resize text-3xl" />
              </div>
            </Transition>
          </RouterView>
        </div>
      </div>
    </main>

    <!-- 主题设置抽屉 -->
    <ThemeSettingDrawer :show="showThemeDrawer" @close="showThemeDrawer = false" />

    <!-- 全平台账号排行榜弹窗 -->
    <LeaderboardModal :show="showLeaderboard" @close="showLeaderboard = false" />
  </div>
</template>

<style scoped>
.workspace-shell {
  position: relative;
  --workspace-panel-bg: rgba(255, 255, 255, 0.74);
  --workspace-panel-border: rgba(255, 255, 255, 0.22);
  --workspace-panel-blur: 18px;
  --workspace-panel-shadow: 0 18px 48px rgba(15, 23, 42, 0.12);
  --workspace-field-bg: rgba(255, 255, 255, 0.14);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.24);
}

:global(.dark) .workspace-shell {
  --workspace-panel-bg: rgba(15, 23, 42, 0.62);
  --workspace-panel-border: rgba(255, 255, 255, 0.12);
  --workspace-panel-shadow: 0 18px 48px rgba(2, 6, 23, 0.32);
  --workspace-field-bg: rgba(255, 255, 255, 0.08);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.14);
}

.workspace-shell--console {
  --workspace-panel-bg: rgba(255, 255, 255, 0.68);
  --workspace-panel-border: rgba(255, 255, 255, 0.22);
  --workspace-panel-blur: 18px;
  --workspace-field-bg: rgba(255, 255, 255, 0.14);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.24);
}

:global(.dark) .workspace-shell--console {
  --workspace-panel-bg: rgb(var(--color-primary-950) / 0.5);
  --workspace-panel-border: rgb(var(--color-primary-300) / 0.12);
  --workspace-panel-shadow: 0 20px 52px rgba(2, 6, 23, 0.32);
  --workspace-field-bg: rgba(255, 255, 255, 0.08);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.14);
}

.workspace-shell--poster {
  --workspace-panel-bg: rgba(255, 255, 255, 0.42);
  --workspace-panel-border: rgba(255, 255, 255, 0.34);
  --workspace-panel-blur: 22px;
  --workspace-panel-shadow: 0 22px 60px rgba(15, 23, 42, 0.14);
  --workspace-field-bg: rgba(255, 255, 255, 0.12);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.22);
}

:global(.dark) .workspace-shell--poster {
  --workspace-panel-bg: rgb(var(--color-primary-950) / 0.36);
  --workspace-panel-border: rgb(var(--color-primary-200) / 0.18);
  --workspace-panel-shadow: 0 22px 68px rgba(2, 6, 23, 0.34);
  --workspace-field-bg: rgba(255, 255, 255, 0.06);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.12);
}

.workspace-shell--pure_glass {
  --workspace-panel-bg: rgba(255, 255, 255, 0.28);
  --workspace-panel-border: rgba(255, 255, 255, 0.4);
  --workspace-panel-blur: 28px;
  --workspace-panel-shadow: 0 24px 72px rgba(15, 23, 42, 0.12);
  --workspace-field-bg: rgba(255, 255, 255, 0.1);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.18);
}

:global(.dark) .workspace-shell--pure_glass {
  --workspace-panel-bg: rgba(10, 14, 24, 0.28);
  --workspace-panel-border: rgb(var(--color-primary-200) / 0.22);
  --workspace-panel-shadow: 0 24px 72px rgba(2, 6, 23, 0.38);
  --workspace-field-bg: rgba(255, 255, 255, 0.05);
  --workspace-field-focus-bg: rgba(255, 255, 255, 0.1);
}

.workspace-shell__wash {
  position: absolute;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.06), transparent 18%, transparent 72%, rgba(0, 0, 0, 0.1)),
    radial-gradient(circle at 12% 12%, rgba(255, 255, 255, 0.1), transparent 26%);
}

:global(.dark) .workspace-shell__wash {
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.04), transparent 18%, transparent 70%, rgba(2, 6, 23, 0.16)),
    radial-gradient(circle at 12% 12%, rgb(var(--color-primary-300) / 0.08), transparent 26%);
}

.workspace-shell__glow {
  position: absolute;
  z-index: 0;
  pointer-events: none;
  border-radius: 9999px;
  filter: blur(64px);
  opacity: 0.34;
  background: rgb(var(--color-primary-400) / 0.16);
}

.workspace-shell__glow--left {
  top: 7%;
  left: 4%;
  width: 22rem;
  height: 22rem;
}

.workspace-shell__glow--right {
  right: 5%;
  bottom: 8%;
  width: 18rem;
  height: 18rem;
}

.workspace-shell :deep(.glass-panel) {
  background-color: var(--workspace-panel-bg) !important;
  border-color: var(--workspace-panel-border) !important;
  backdrop-filter: blur(var(--workspace-panel-blur)) !important;
  -webkit-backdrop-filter: blur(var(--workspace-panel-blur)) !important;
  box-shadow: var(--workspace-panel-shadow);
}

.workspace-shell
  :deep(input:not([type='checkbox']):not([type='radio']):not([type='range']):not([type='file']):not([type='color'])),
.workspace-shell :deep(textarea),
.workspace-shell :deep(select),
.workspace-shell :deep(.base-select-trigger) {
  background-color: var(--workspace-field-bg) !important;
  backdrop-filter: blur(calc(var(--workspace-panel-blur) * 0.65)) !important;
  -webkit-backdrop-filter: blur(calc(var(--workspace-panel-blur) * 0.65)) !important;
}

.workspace-shell
  :deep(
    input:not([type='checkbox']):not([type='radio']):not([type='range']):not([type='file']):not([type='color']):focus
  ),
.workspace-shell :deep(textarea:focus),
.workspace-shell :deep(select:focus),
.workspace-shell :deep(.base-select-trigger.ring-2),
.workspace-shell :deep(.base-select-trigger:hover) {
  background-color: var(--workspace-field-focus-bg) !important;
}

.workspace-scroll {
  position: relative;
  z-index: 1;
}

/* Slide Fade Transition */
.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.2s ease-out;
}

.slide-fade-enter-from {
  opacity: 0;
  transform: translateY(10px);
}

.slide-fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.3);
  border-radius: 3px;
}
.custom-scrollbar:hover::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.5);
}
</style>
