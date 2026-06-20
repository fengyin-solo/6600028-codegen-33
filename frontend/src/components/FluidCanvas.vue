<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick, computed } from 'vue'
import { useFluidStore } from '../store/fluid'

const store = useFluidStore()
const canvas = ref<HTMLCanvasElement | null>(null)

const W = 800
const H = 500

const currentReviewScreenshot = computed(() => {
  if (!store.isReviewing || store.currentReviewIndex < 0) return null
  return store.screenshots[store.currentReviewIndex] || null
})

watch(
  () => store._screenshotRequestCount,
  () => {
    if (store._screenshotRequestCount > 0) {
      nextTick(() => takeScreenshot())
    }
  }
)

function velocityToColor(speed: number): string {
  // Blue (slow) -> Green (medium) -> Red (fast)
  const maxSpeed = 200
  const t = Math.min(speed / maxSpeed, 1)
  const hue = (1 - t) * 240 // 240=blue, 120=green, 0=red
  const sat = 80
  const light = 40 + t * 20
  return `hsl(${hue}, ${sat}%, ${light}%)`
}

function draw() {
  const ctx = canvas.value?.getContext('2d')
  if (!ctx) return

  if (store.isReviewing && currentReviewScreenshot.value) {
    const img = new Image()
    img.onload = () => {
      ctx.drawImage(img, 0, 0, W, H)
      drawReviewOverlay(ctx)
    }
    img.src = currentReviewScreenshot.value.dataUrl
    return
  }

  // Clear
  ctx.fillStyle = '#0c1222'
  ctx.fillRect(0, 0, W, H)

  // Draw boundary walls
  ctx.strokeStyle = '#475569'
  ctx.lineWidth = 3
  ctx.strokeRect(2, 2, W - 4, H - 4)

  // Draw grid (faint)
  ctx.strokeStyle = '#1e293b'
  ctx.lineWidth = 0.3
  for (let x = 0; x < W; x += 50) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, H)
    ctx.stroke()
  }
  for (let y = 0; y < H; y += 50) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(W, y)
    ctx.stroke()
  }

  if (!store.engine) return

  // Draw density heatmap background (low-res)
  const gridSize = 20
  const gw = Math.ceil(W / gridSize)
  const gh = Math.ceil(H / gridSize)
  const densityGrid = new Float32Array(gw * gh)
  for (const p of store.engine.particles) {
    const gx = Math.floor(p.x / gridSize)
    const gy = Math.floor(p.y / gridSize)
    if (gx >= 0 && gx < gw && gy >= 0 && gy < gh) {
      densityGrid[gy * gw + gx] += p.density
    }
  }
  const maxDens = Math.max(...densityGrid, 1)
  for (let gy = 0; gy < gh; gy++) {
    for (let gx = 0; gx < gw; gx++) {
      const d = densityGrid[gy * gw + gx]
      if (d > 0) {
        const alpha = Math.min(d / maxDens * 0.15, 0.15)
        ctx.fillStyle = `rgba(59, 130, 246, ${alpha})`
        ctx.fillRect(gx * gridSize, gy * gridSize, gridSize, gridSize)
      }
    }
  }

  // Draw particles
  const particles = store.engine.particles
  for (const p of particles) {
    const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy)
    const color = velocityToColor(speed)
    const radius = 4

    ctx.beginPath()
    ctx.arc(p.x, p.y, radius, 0, Math.PI * 2)
    ctx.fillStyle = color
    ctx.fill()
  }

  // FPS overlay
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 80, 5, 75, 22)
  ctx.fillStyle = '#22c55e'
  ctx.font = 'bold 12px monospace'
  ctx.fillText(`FPS: ${store.fps}`, W - 74, 20)

  // Frame count
  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 120, 30, 115, 22)
  ctx.fillStyle = '#94a3b8'
  ctx.font = '11px monospace'
  ctx.fillText(`Frame: ${store.frameCount}`, W - 114, 44)

  if (store.autoCapture && store.isRunning) {
    checkAutoCapture()
  }
}

function drawReviewOverlay(ctx: CanvasRenderingContext2D) {
  const shot = currentReviewScreenshot.value
  if (!shot) return

  ctx.fillStyle = 'rgba(0, 0, 0, 0.7)'
  ctx.fillRect(10, 10, 200, 50)
  ctx.fillStyle = '#fbbf24'
  ctx.font = 'bold 12px monospace'
  ctx.fillText('回看模式', 20, 30)
  ctx.fillStyle = '#94a3b8'
  ctx.font = '10px monospace'
  ctx.fillText(`${shot.label} · 帧 ${shot.frame}`, 20, 50)
}

let _lastCaptureFrame = -1
function checkAutoCapture() {
  if (store.frameCount > 0 && store.frameCount % store.autoCaptureInterval === 0 && store.frameCount !== _lastCaptureFrame) {
    _lastCaptureFrame = store.frameCount
    takeScreenshot()
  }
}

function takeScreenshot(label?: string): string | null {
  if (!canvas.value) return null
  const dataUrl = canvas.value.toDataURL('image/png')
  store.addScreenshot(dataUrl, label)
  return dataUrl
}

defineExpose({ takeScreenshot })


let raf: number | null = null
function animate() {
  draw()
  raf = requestAnimationFrame(animate)
}

function onClick(e: MouseEvent) {
  if (!store.engine || !canvas.value) return
  const rect = canvas.value.getBoundingClientRect()
  const scaleX = W / rect.width
  const scaleY = H / rect.height
  const x = (e.clientX - rect.left) * scaleX
  const y = (e.clientY - rect.top) * scaleY
  store.engine.applyImpulse(x, y, 300)
}

onMounted(() => {
  animate()
})

onUnmounted(() => {
  if (raf) cancelAnimationFrame(raf)
})
</script>

<template>
  <div class="relative">
    <canvas
      ref="canvas"
      :width="W"
      :height="H"
      class="rounded-lg border border-gray-700 cursor-crosshair w-full max-w-[800px]"
      @click="onClick"
    />
  </div>
</template>
