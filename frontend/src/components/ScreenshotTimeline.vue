<script setup lang="ts">
import { ref, computed } from 'vue'
import { useFluidStore } from '../store/fluid'

const store = useFluidStore()
const timelineRef = ref<HTMLDivElement | null>(null)
const editingId = ref<number | null>(null)
const editLabel = ref('')

const currentShot = computed(() => {
  if (!store.isReviewing || store.currentReviewIndex < 0) return null
  return store.screenshots[store.currentReviewIndex] || null
})

function formatTime(ts: number): string {
  const d = new Date(ts)
  return `${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}:${d.getSeconds().toString().padStart(2, '0')}`
}

function clickThumb(index: number) {
  store.startReview(index)
}

function onDelete(id: number, e: Event) {
  e.stopPropagation()
  store.removeScreenshot(id)
}

function onExport(id: number, e: Event) {
  e.stopPropagation()
  store.exportScreenshot(id)
}

function startEdit(id: number, label: string, e: Event) {
  e.stopPropagation()
  editingId.value = id
  editLabel.value = label
}

function finishEdit(id: number) {
  if (editingId.value === id && editLabel.value.trim()) {
    store.updateScreenshotLabel(id, editLabel.value.trim())
  }
  editingId.value = null
}

function scrollToCurrent() {
  if (!timelineRef.value || !store.isReviewing) return
  const thumbs = timelineRef.value.querySelectorAll('.thumb-item')
  const el = thumbs[store.currentReviewIndex] as HTMLElement | undefined
  if (el) {
    el.scrollIntoView({ behavior: 'smooth', inline: 'center', block: 'nearest' })
  }
}

function onAutoInterval(e: Event) {
  store.setAutoCaptureInterval(parseInt((e.target as HTMLInputElement).value))
}
</script>

<template>
  <div class="bg-gray-800 rounded-lg border border-gray-700 p-3 flex flex-col gap-3">
    <!-- Toolbar -->
    <div class="flex items-center justify-between gap-2">
      <div class="flex items-center gap-2">
        <span class="text-xs font-semibold text-gray-400 uppercase tracking-wider">截图时间轴</span>
        <span class="text-xs text-gray-500">({{ store.screenshots.length }} 张)</span>
      </div>
      <div class="flex items-center gap-2">
        <button
          @click="store.clearScreenshots()"
          :disabled="store.screenshots.length === 0"
          class="text-xs px-2 py-1 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-30 text-gray-300 transition"
        >
          清空
        </button>
        <button
          @click="store.exportAllScreenshots()"
          :disabled="store.screenshots.length === 0"
          class="text-xs px-2 py-1 rounded bg-blue-600 hover:bg-blue-500 disabled:opacity-30 text-white transition"
        >
          导出全部
        </button>
      </div>
    </div>

    <!-- Auto Capture Controls -->
    <div class="flex items-center gap-3 bg-gray-900 rounded px-3 py-2">
      <button
        @click="store.toggleAutoCapture()"
        class="text-xs px-3 py-1.5 rounded font-medium transition flex-shrink-0"
        :class="store.autoCapture
          ? 'bg-amber-600 hover:bg-amber-500 text-white'
          : 'bg-gray-700 hover:bg-gray-600 text-gray-300'"
      >
        {{ store.autoCapture ? '自动截图中' : '自动截图' }}
      </button>
      <div class="flex-1 flex items-center gap-2">
        <label class="text-xs text-gray-500 flex-shrink-0">间隔</label>
        <input
          type="range" min="10" max="300" step="10"
          :value="store.autoCaptureInterval"
          @input="onAutoInterval"
          class="flex-1 accent-amber-500 h-1.5"
        />
        <span class="text-xs text-gray-400 font-mono w-12 text-right">{{ store.autoCaptureInterval }}帧</span>
      </div>
    </div>

    <!-- Review Controls -->
    <div v-if="store.isReviewing" class="flex items-center justify-center gap-2 bg-amber-900/30 border border-amber-700/50 rounded px-3 py-2">
      <button
        @click="store.prevScreenshot()"
        :disabled="store.currentReviewIndex <= 0"
        class="text-xs px-3 py-1.5 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-30 text-gray-200 transition"
      >
        ← 上一张
      </button>
      <span class="text-xs text-amber-300 font-mono">
        {{ store.currentReviewIndex + 1 }} / {{ store.screenshots.length }}
      </span>
      <button
        @click="store.nextScreenshot()"
        :disabled="store.currentReviewIndex >= store.screenshots.length - 1"
        class="text-xs px-3 py-1.5 rounded bg-gray-700 hover:bg-gray-600 disabled:opacity-30 text-gray-200 transition"
      >
        下一张 →
      </button>
      <button
        @click="store.exitReview()"
        class="text-xs px-3 py-1.5 rounded bg-amber-600 hover:bg-amber-500 text-white transition ml-2"
      >
        退出回看
      </button>
    </div>

    <!-- Timeline Thumbnails -->
    <div
      ref="timelineRef"
      class="flex gap-2 overflow-x-auto pb-2 scrollbar-thin"
      :class="{ 'items-stretch': store.screenshots.length > 0 }"
    >
      <div
        v-if="store.screenshots.length === 0"
        class="flex-1 flex items-center justify-center py-6 text-xs text-gray-500"
      >
        暂无截图，运行模拟后点击"截图"按钮或开启自动截图
      </div>
      <div
        v-for="(shot, index) in store.screenshots"
        :key="shot.id"
        class="thumb-item flex-shrink-0 cursor-pointer group relative"
        :class="{ 'ring-2 ring-amber-400 ring-offset-2 ring-offset-gray-800 rounded': store.isReviewing && index === store.currentReviewIndex }"
        @click="clickThumb(index)"
      >
        <div class="w-28 h-[88px] bg-gray-900 rounded overflow-hidden relative">
          <img :src="shot.dataUrl" :alt="shot.label" class="w-full h-full object-cover" />
          <div class="absolute bottom-0 left-0 right-0 bg-black/70 px-1.5 py-0.5">
            <div v-if="editingId !== shot.id" class="text-[10px] text-white truncate">
              {{ shot.label }}
            </div>
            <input
              v-else
              v-model="editLabel"
              @blur="finishEdit(shot.id)"
              @keyup.enter="finishEdit(shot.id)"
              class="w-full text-[10px] bg-transparent text-white border-b border-amber-400 outline-none"
              @click.stop
              autofocus
            />
          </div>
          <div class="absolute top-1 left-1 bg-black/60 px-1 py-0.5 rounded text-[9px] text-gray-300 font-mono">
            #{{ index + 1 }} F{{ shot.frame }}
          </div>
        </div>
        <div class="flex items-center justify-between mt-1 opacity-0 group-hover:opacity-100 transition">
          <span class="text-[9px] text-gray-500">{{ formatTime(shot.timestamp) }}</span>
          <div class="flex gap-1">
            <button
              @click="startEdit(shot.id, shot.label, $event)"
              class="text-[9px] text-blue-400 hover:text-blue-300"
              title="重命名"
            >
              ✎
            </button>
            <button
              @click="onExport(shot.id, $event)"
              class="text-[9px] text-green-400 hover:text-green-300"
              title="导出"
            >
              ↓
            </button>
            <button
              @click="onDelete(shot.id, $event)"
              class="text-[9px] text-red-400 hover:text-red-300"
              title="删除"
            >
              ✕
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Keyboard hints -->
    <div class="text-[10px] text-gray-600 text-center">
      提示：点击缩略图回看，左右方向键切换，ESC 退出回看
    </div>
  </div>
</template>

<style scoped>
.scrollbar-thin::-webkit-scrollbar {
  height: 6px;
}
.scrollbar-thin::-webkit-scrollbar-track {
  background: #1f2937;
  border-radius: 3px;
}
.scrollbar-thin::-webkit-scrollbar-thumb {
  background: #4b5563;
  border-radius: 3px;
}
.scrollbar-thin::-webkit-scrollbar-thumb:hover {
  background: #6b7280;
}
</style>
