<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { OnFileDrop } from '../../wailsjs/runtime/runtime'

const props = defineProps({
  accept: { type: String, default: '*' },
  multiple: { type: Boolean, default: false },
  disabled: { type: Boolean, default: false }
})

const emit = defineEmits(['drop', 'click', 'error'])
const isDragging = ref(false)
const rootEl = ref(null)

// ---------- 全局唯一 drop 分发器（模块级单例） ----------
// Wails 的 OnFileDrop 全局只能注册一个回调，且 OnFileDropOff 会移除所有监听。
// 因此这里集中管理：注册一次、永不注销，再按命中区域分发给各组件。
const zones = new Set()
let dispatcherStarted = false

function parseExts(accept) {
  if (!accept || accept === '*') return []
  return String(accept)
    .split(',')
    .map(t => t.trim().toLowerCase())
    .filter(t => t.startsWith('.'))
}

function filterPaths(exts, paths) {
  if (exts.length === 0) return [...paths]
  return paths.filter(p => exts.some(ext => p.toLowerCase().endsWith(ext)))
}

function findRoot(node) {
  let cur = node
  while (cur) {
    if (cur.__wailsDropZone) return cur.__wailsDropZone
    cur = cur.parentElement
  }
  return null
}

function deliver(zone, paths) {
  const exts = parseExts(zone.accept)
  let filtered = filterPaths(exts, paths)
  if (filtered.length === 0) {
    zone.error?.('不支持的文件类型')
    return
  }
  if (!zone.multiple) filtered = [filtered[0]]
  zone.drop?.(zone.multiple ? filtered : filtered[0])
}

function dispatch(x, y, paths) {
  if (!paths || paths.length === 0) return

  // 1) 坐标命中检测：找到落点所在的启用区域
  const hits = []
  for (const zone of zones.values()) {
    if (zone.disabled) continue
    const el = zone.el
    if (!el || !el.isConnected) continue
    const r = el.getBoundingClientRect()
    if (r.width === 0 || r.height === 0) continue
    if (x >= r.left && x <= r.right && y >= r.top && y <= r.bottom) hits.push({ zone, area: r.width * r.height })
  }

  // 命中多个时取最上层（面积最小者为最具体的可视区域）
  if (hits.length > 0) {
    hits.sort((a, b) => a.area - b.area)
    deliver(hits[0].zone, paths)
    return
  }

  // 2) 兜底：坐标未命中（如 DPI 缩放导致坐标系不一致），
  //    若当前只有一个启用的区域，则直接投递给它，保证拖拽始终可用。
  const avail = []
  for (const zone of zones.values()) {
    if (zone.disabled) continue
    const el = zone.el
    if (!el || !el.isConnected) continue
    const r = el.getBoundingClientRect()
    if (r.width === 0 || r.height === 0) continue
    avail.push(zone)
  }
  if (avail.length === 1) deliver(avail[0], paths)
}

function startDispatcher() {
  if (dispatcherStarted || typeof window === 'undefined' || !window.runtime?.OnFileDrop) return
  OnFileDrop((x, y, paths) => dispatch(x, y, paths), false)
  dispatcherStarted = true
  initContentFallback()
}

// ---------- 兜底通道 ----------
// WebView2 < 1.0.1774.30 缺少 postMessageWithAdditionalObjects，
// Wails 无法把拖放文件解析为路径。此时前端直接读取文件内容，
// 经 Go 端 SaveDroppedFiles 落盘后得到路径，再走统一分发。
function toBase64(buffer) {
  const bytes = new Uint8Array(buffer)
  let bin = ''
  const chunk = 0x8000
  for (let i = 0; i < bytes.length; i += chunk) {
    bin += String.fromCharCode.apply(null, bytes.subarray(i, Math.min(i + chunk, bytes.length)))
  }
  return btoa(bin)
}

function initContentFallback() {
  const w = window
  if (w.__epubDropFallbackInstalled) return
  if (w.chrome?.webview?.postMessageWithAdditionalObjects) return // 原生路径通道可用
  w.addEventListener('drop', async (e) => {
    const dt = e.dataTransfer
    if (!dt || !Array.from(dt.types || []).includes('Files')) return
    const files = Array.from(dt.files || [])
    if (files.length === 0) return
    e.preventDefault()
    e.stopPropagation()
    try {
      const app = w.go?.main?.App
      if (!app?.SaveDroppedFiles) {
        console.error('SaveDroppedFiles binding unavailable')
        return
      }
      const payload = []
      for (const f of files) {
        payload.push({ name: f.name, data: toBase64(await f.arrayBuffer()) })
      }
      const paths = await app.SaveDroppedFiles(payload)
      if (paths && paths.length > 0) dispatch(e.clientX, e.clientY, paths)
    } catch (err) {
      console.error('拖放兜底通道失败:', err)
    }
  }, true)
  w.__epubDropFallbackInstalled = true
}

let unregisterZone = null

onMounted(() => {
  startDispatcher()
  const node = rootEl.value
  node.__wailsDropZone = {
    el: node,
    get accept() { return props.accept },
    get multiple() { return props.multiple },
    get disabled() { return props.disabled },
    drop: (v) => emit('drop', v),
    error: (msg) => emit('error', msg)
  }
  // 每次 render 都要刷新 latest props（用 getter 已处理），仅记录用于清理
  unregisterZone = () => delete node.__wailsDropZone
})

onUnmounted(() => {
  if (unregisterZone) unregisterZone()
  unregisterZone = null
})

const handleDragOver = (e) => { if (props.disabled) return; e.preventDefault(); e.stopPropagation(); isDragging.value = true }
const handleDragLeave = (e) => { if (props.disabled) return; e.preventDefault(); e.stopPropagation(); isDragging.value = false }
const handleDrop = (e) => { e.preventDefault(); e.stopPropagation(); isDragging.value = false }
const handleClick = () => { if (!props.disabled) emit('click') }
</script>

<template>
  <div ref="rootEl" @dragover="handleDragOver" @dragleave="handleDragLeave" @drop="handleDrop" @click="handleClick" :class="['relative border-2 border-dashed rounded-xl transition-all duration-200 cursor-pointer', isDragging ? 'border-indigo-500 bg-indigo-50 dark:bg-indigo-900/20 scale-[1.02]' : 'border-gray-300 dark:border-gray-600 hover:border-indigo-400 dark:hover:border-indigo-500 hover:bg-gray-50 dark:hover:bg-gray-800/50', disabled && 'opacity-50 cursor-not-allowed pointer-events-none']">
    <slot>
      <div class="flex flex-col items-center justify-center py-8 px-4 text-center">
        <div :class="['w-12 h-12 rounded-full flex items-center justify-center mb-3 transition-colors', isDragging ? 'bg-indigo-100 dark:bg-indigo-800/30 text-indigo-600 dark:text-indigo-400' : 'bg-gray-100 dark:bg-gray-700 text-gray-400 dark:text-gray-500']">
          <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1.5"><path stroke-linecap="round" stroke-linejoin="round" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" /></svg>
        </div>
        <p class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">拖拽文件到此处</p>
        <p class="text-xs text-gray-400 dark:text-gray-500">或点击选择文件</p>
      </div>
    </slot>
    <div v-if="isDragging" class="absolute inset-0 bg-indigo-500/5 dark:bg-indigo-500/10 rounded-xl pointer-events-none"></div>
  </div>
</template>
