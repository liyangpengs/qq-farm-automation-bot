<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch } from 'vue'
import { useAccountStore } from '@/stores/account'
import { useFriendStore } from '@/stores/friend'
import { useStatusStore } from '@/stores/status'

const accountStore = useAccountStore()
const friendStore = useFriendStore()
const statusStore = useStatusStore()
const { currentAccountId, currentAccount } = storeToRefs(accountStore)
const { interactRecords, interactLoading, interactError } = storeToRefs(friendStore)
const { status, loading: statusLoading, realtimeConnected } = storeToRefs(statusStore)

const avatarErrorKeys = ref<Set<string>>(new Set())
const interactFilter = ref('all')
const interactFilters = [
  { key: 'all', label: '全部' },
  { key: 'steal', label: '偷菜' },
  { key: 'help', label: '帮忙' },
  { key: 'bad', label: '捣乱' },
]

async function loadVisitors() {
  if (currentAccountId.value) {
    const acc = currentAccount.value
    if (!acc)
      return

    if (!realtimeConnected.value) {
      await statusStore.fetchStatus(currentAccountId.value)
    }

    if (acc.running && status.value?.connection?.connected) {
      avatarErrorKeys.value.clear()
      friendStore.fetchInteractRecords(currentAccountId.value)
    }
  }
}

onMounted(() => {
  loadVisitors()
})

watch(currentAccountId, () => {
  loadVisitors()
})

const filteredInteractRecords = computed(() => {
  if (interactFilter.value === 'all')
    return interactRecords.value

  const actionTypeMap: Record<string, number> = {
    steal: 1,
    help: 2,
    bad: 3,
  }
  const targetActionType = actionTypeMap[interactFilter.value] || 0
  return interactRecords.value.filter((record: any) => Number(record?.actionType) === targetActionType)
})

const visibleInteractRecords = computed(() => filteredInteractRecords.value.slice(0, 50))

async function refreshInteractRecords() {
  if (!currentAccountId.value)
    return
  await friendStore.fetchInteractRecords(currentAccountId.value)
}

function getInteractAvatar(record: any) {
  return String(record?.avatarUrl || '').trim()
}

function getInteractAvatarKey(record: any) {
  const key = String(record?.visitorGid || record?.key || record?.nick || '').trim()
  return key ? `interact:${key}` : ''
}

function canShowInteractAvatar(record: any) {
  const key = getInteractAvatarKey(record)
  if (!key)
    return false
  return !!getInteractAvatar(record) && !avatarErrorKeys.value.has(key)
}

function handleInteractAvatarError(record: any) {
  const key = getInteractAvatarKey(record)
  if (!key)
    return
  avatarErrorKeys.value.add(key)
}

function getInteractBadgeClass(actionType: number) {
  if (Number(actionType) === 1)
    return 'bg-blue-100 text-blue-700 dark:bg-blue-900/30 dark:text-blue-300'
  if (Number(actionType) === 2)
    return 'bg-green-100 text-green-700 dark:bg-green-900/30 dark:text-green-300'
  if (Number(actionType) === 3)
    return 'bg-red-100 text-red-700 dark:bg-red-900/30 dark:text-red-300'
  return 'bg-gray-100 text-gray-600 dark:bg-gray-700 dark:text-gray-300'
}

function formatInteractTime(timestamp: number) {
  const ts = Number(timestamp) || 0
  if (!ts)
    return '--'

  const date = new Date(ts)
  const now = new Date()
  const diff = now.getTime() - date.getTime()
  const minute = 60 * 1000
  const hour = 60 * minute

  if (diff >= 0 && diff < minute)
    return '刚刚'
  if (diff >= minute && diff < hour)
    return `${Math.floor(diff / minute)} 分钟前`

  const sameDay = now.getFullYear() === date.getFullYear()
      && now.getMonth() === date.getMonth()
      && now.getDate() === date.getDate()

  if (sameDay) {
    return `今天 ${date.toLocaleTimeString('zh-CN', {
      hour: '2-digit',
      minute: '2-digit',
      hour12: false,
    })}`
  }

  if (now.getFullYear() === date.getFullYear()) {
    return `${date.getMonth() + 1}-${date.getDate()} ${date.toLocaleTimeString('zh-CN', {
      hour: '2-digit',
      minute: '2-digit',
      hour12: false,
    })}`
  }

  return date.toLocaleString('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    hour12: false,
  })
}
</script>

<template>
  <div class="p-4">
    <div class="mb-4 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <h2 class="flex items-center gap-2 text-2xl font-bold">
        <div class="i-carbon-user-activity text-amber-500" />
        最近访客
      </h2>
      <div class="flex items-center gap-3">
        <div class="flex flex-wrap items-center gap-2">
          <button
            v-for="item in interactFilters"
            :key="item.key"
            class="rounded-full px-3 py-1 text-xs transition"
            :class="interactFilter === item.key
              ? 'bg-amber-500 text-white'
              : 'bg-gray-100 text-gray-600 dark:bg-gray-700 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600'"
            @click="interactFilter = item.key"
          >
            {{ item.label }}
          </button>
          <button
            v-if="status?.connection?.connected && currentAccountId"
            class="rounded bg-gray-100 px-3 py-1.5 text-xs text-gray-600 transition dark:bg-gray-700 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600 disabled:cursor-not-allowed disabled:opacity-60"
            :disabled="interactLoading"
            @click="refreshInteractRecords"
          >
            {{ interactLoading ? '刷新中...' : '刷新' }}
          </button>
        </div>
        <div v-if="interactRecords.length" class="text-sm text-gray-500">
          共 {{ filteredInteractRecords.length }}/{{ interactRecords.length }} 条记录
        </div>
      </div>
    </div>

    <div v-if="interactLoading || statusLoading" class="flex justify-center py-12">
      <div class="i-svg-spinners-90-ring-with-bg text-4xl text-amber-500" />
    </div>

    <div v-else-if="!currentAccountId" class="flex flex-col items-center justify-center gap-4 rounded-lg bg-white p-12 text-center text-gray-500 shadow dark:bg-gray-800">
      <div class="i-carbon-user-offline text-4xl text-gray-400" />
      <div>
        <div class="text-lg text-gray-700 font-medium dark:text-gray-300">
          未登录账号
        </div>
        <div class="mt-1 text-sm text-gray-400">
          请先添加农场账号
        </div>
      </div>
    </div>

    <div v-else-if="!status?.connection?.connected" class="flex flex-col items-center justify-center gap-4 rounded-lg bg-white p-12 text-center text-gray-500 shadow dark:bg-gray-800">
      <div class="i-carbon-connection-signal-off text-4xl text-gray-400" />
      <div>
        <div class="text-lg text-gray-700 font-medium dark:text-gray-300">
          账号未登录
        </div>
        <div class="mt-1 text-sm text-gray-400">
          请先运行账号或检查网络连接
        </div>
      </div>
    </div>

    <div v-else-if="!!interactError" class="rounded-lg bg-red-50 px-4 py-6 text-center text-sm text-red-600 dark:bg-red-900/20 dark:text-red-300">
      {{ interactError }}
    </div>

    <div v-else-if="visibleInteractRecords.length === 0" class="rounded-lg bg-white p-8 text-center text-gray-500 shadow dark:bg-gray-800">
      <div class="i-carbon-user-activity mx-auto mb-3 text-4xl text-gray-300" />
      暂无访客记录
    </div>

    <div v-else class="space-y-3">
      <div
        v-for="record in visibleInteractRecords"
        :key="record.key"
        class="flex items-start gap-3 rounded-lg bg-white p-4 shadow dark:bg-gray-800"
      >
        <div class="h-12 w-12 flex shrink-0 items-center justify-center overflow-hidden rounded-full bg-gray-200 ring-1 ring-gray-100 dark:bg-gray-700 dark:ring-gray-600">
          <img
            v-if="canShowInteractAvatar(record)"
            :src="getInteractAvatar(record)"
            class="h-full w-full object-cover"
            loading="lazy"
            @error="handleInteractAvatarError(record)"
          >
          <div v-else class="i-carbon-user-avatar text-gray-400 text-xl" />
        </div>
        <div class="min-w-0 flex-1">
          <div class="mb-1 flex flex-wrap items-center gap-2">
            <span class="max-w-full truncate text-base text-gray-800 font-medium dark:text-gray-100">
              {{ record.nick || `GID:${record.visitorGid}` }}
            </span>
            <span
              class="rounded-full px-2 py-0.5 text-xs font-medium"
              :class="getInteractBadgeClass(record.actionType)"
            >
              {{ record.actionLabel }}
            </span>
            <span v-if="record.level" class="rounded bg-gray-100 px-2 py-0.5 text-xs text-gray-500 dark:bg-gray-700 dark:text-gray-300">
              Lv.{{ record.level }}
            </span>
            <span v-if="record.visitorGid" class="text-xs text-gray-400">
              GID {{ record.visitorGid }}
            </span>
          </div>
          <div class="text-sm text-gray-600 dark:text-gray-300">
            {{ record.actionDetail || record.actionLabel }}
          </div>
        </div>
        <div class="shrink-0 text-right text-xs text-gray-400">
          {{ formatInteractTime(record.serverTimeMs) }}
        </div>
      </div>

      <div v-if="filteredInteractRecords.length > visibleInteractRecords.length" class="text-center text-xs text-gray-400">
        仅展示最近 {{ visibleInteractRecords.length }} 条
      </div>
    </div>
  </div>
</template>
