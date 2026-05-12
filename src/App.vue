<script setup lang="ts">
import { computed, nextTick, reactive, ref } from 'vue'

type Layout = 'vertical' | 'horizontal'

const layout = ref<Layout>('vertical')
const fileInput = ref<HTMLInputElement | null>(null)
const imageElement = ref<HTMLImageElement | null>(null)
const previewImage = ref('')

const photoInfo = reactive({
  fileName: '未选择图片',
  shootTime: '2026-05-12 18:36',
  location: '北京 / 朝阳区',
  note: '柔和色彩的日常，用极简风格记录当下。'
})

const palette = reactive({
  main: '#8f7dff',
  secondary: '#7c9eff',
  light: '#f3efff',
  dark: '#312f48',
  pure: '#5750de'
})

const isReady = computed(() => !!previewImage.value)
const layoutLabel = computed(() => (layout.value === 'vertical' ? '上下布局' : '左右布局'))
const displayPureColor = computed(() => softenColor(palette.pure))
const posterTextColor = computed(() => getSoftTextColor(displayPureColor.value))

const openFile = () => fileInput.value?.click()
const toggleLayout = () => {
  layout.value = layout.value === 'vertical' ? 'horizontal' : 'vertical'
}

const handleFile = async (event: Event) => {
  const target = event.target as HTMLInputElement
  if (!target.files?.length) return
  const file = target.files[0]
  const url = URL.createObjectURL(file)
  previewImage.value = url
  photoInfo.fileName = file.name

  await nextTick()
  if (!imageElement.value) return

  const img = imageElement.value
  if (img.complete && img.naturalWidth > 0) {
    analyzeImage(img)
    return
  }

  await new Promise<void>((resolve) => {
    img.onload = () => {
      analyzeImage(img)
      resolve()
    }
  })
}

const rgbToHex = ([r, g, b]: number[]) => {
  const hex = ((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)
  return `#${hex}`
}

const hexToRgb = (hex: string): [number, number, number] => {
  const clean = hex.replace('#', '')
  const normalized = clean.length === 3
    ? clean.split('').map((char) => char + char).join('')
    : clean
  const value = parseInt(normalized, 16)
  return [
    (value >> 16) & 255,
    (value >> 8) & 255,
    value & 255
  ]
}

const softenColor = (hex: string) => {
  const [r, g, b] = hexToRgb(hex)
  const { h, s, l } = rgbToHsl([r, g, b])
  const newS = Math.max(0.18, Math.min(0.6, s * 0.85))
  const newL = l < 0.25 ? l + 0.18 : l > 0.78 ? l - 0.16 : l
  return `hsl(${h}, ${Math.round(newS * 100)}%, ${Math.round(newL * 100)}%)`
}

const getSoftTextColor = (hex: string) => {
  const [r, g, b] = hexToRgb(hex)
  const { h, s, l } = rgbToHsl([r, g, b])
  const complement = (h + 180) % 360
  if (l > 0.75) {
    return `hsl(${complement}, ${Math.min(30, s * 100)}%, 20%)`
  }
  if (l < 0.28) {
    return `hsl(${complement}, ${Math.min(30, s * 100)}%, 92%)`
  }
  const lightness = l > 0.55 ? 20 : 90
  return `hsl(${complement}, ${Math.max(18, Math.min(28, s * 100))}%, ${lightness}%)`
}

const rgbToHsl = ([r, g, b]: number[]) => {
  const rr = r / 255
  const gg = g / 255
  const bb = b / 255
  const max = Math.max(rr, gg, bb)
  const min = Math.min(rr, gg, bb)
  const delta = max - min
  let h = 0
  const l = (max + min) / 2
  const s = delta === 0 ? 0 : delta / (1 - Math.abs(2 * l - 1))
  if (delta) {
    switch (max) {
      case rr:
        h = ((gg - bb) / delta) % 6
        break
      case gg:
        h = (bb - rr) / delta + 2
        break
      case bb:
        h = (rr - gg) / delta + 4
        break
    }
  }
  return {
    h: Math.round((h * 60 + 360) % 360),
    s: Math.round(s * 100) / 100,
    l: Math.round(l * 100) / 100
  }
}

const normalizeRgb = ([r, g, b]: number[]) => [r / 255, g / 255, b / 255]

const colorDistance = (a: number[], b: number[]) => {
  const [ar, ag, ab] = normalizeRgb(a)
  const [br, bg, bb] = normalizeRgb(b)
  return Math.sqrt((ar - br) ** 2 + (ag - bg) ** 2 + (ab - bb) ** 2)
}

const chooseShade = (colors: Array<{ r: number; g: number; b: number; h: number; s: number; l: number; count: number }>) => {
  const main = colors[0]
  const dark = colors.find((item) => item.l < 0.38) || colors[colors.length - 1]
  const light = colors.find((item) => item.l > 0.72) || colors.find((item) => item.l > 0.58) || colors[0]
  const secondary = colors.find((item) => colorDistance([item.r, item.g, item.b], [main.r, main.g, main.b]) > 0.18 && item.s > 0.18) || colors[1] || main
  const pure = colors.find((item) => item.s > 0.68 && colorDistance([item.r, item.g, item.b], [main.r, main.g, main.b]) > 0.18) || secondary || main
  return { main, secondary, light, dark, pure }
}

const analyzeImage = (img: HTMLImageElement) => {
  const sampleSize = 120
  const canvas = document.createElement('canvas')
  canvas.width = sampleSize
  canvas.height = sampleSize
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const scale = Math.max(sampleSize / img.naturalWidth, sampleSize / img.naturalHeight)
  const drawW = Math.round(img.naturalWidth * scale)
  const drawH = Math.round(img.naturalHeight * scale)
  const dx = Math.round((sampleSize - drawW) / 2)
  const dy = Math.round((sampleSize - drawH) / 2)

  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, sampleSize, sampleSize)
  ctx.drawImage(img, dx, dy, drawW, drawH)

  const data = ctx.getImageData(0, 0, sampleSize, sampleSize).data
  const buckets = new Map<string, { count: number; sumR: number; sumG: number; sumB: number }>()

  for (let i = 0; i < data.length; i += 4) {
    const alpha = data[i + 3]
    if (alpha < 100) continue
    const r = data[i]
    const g = data[i + 1]
    const b = data[i + 2]
    const key = `${r >> 4},${g >> 4},${b >> 4}`
    const existing = buckets.get(key)
    if (existing) {
      existing.count += 1
      existing.sumR += r
      existing.sumG += g
      existing.sumB += b
    } else {
      buckets.set(key, { count: 1, sumR: r, sumG: g, sumB: b })
    }
  }

  const colors = Array.from(buckets.values())
    .map((item) => {
      const r = Math.round(item.sumR / item.count)
      const g = Math.round(item.sumG / item.count)
      const b = Math.round(item.sumB / item.count)
      const { h, s, l } = rgbToHsl([r, g, b])
      return { r, g, b, h, s, l, count: item.count }
    })
    .sort((a, b) => b.count - a.count)
    .slice(0, 24)

  if (!colors.length) return

  const { main, secondary, light, dark, pure } = chooseShade(colors)
  palette.main = rgbToHex([main.r, main.g, main.b])
  palette.secondary = rgbToHex([secondary.r, secondary.g, secondary.b])
  palette.light = rgbToHex([light.r, light.g, light.b])
  palette.dark = rgbToHex([dark.r, dark.g, dark.b])
  palette.pure = rgbToHex([pure.r, pure.g, pure.b])
}

const drawRoundRect = (ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) => {
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.lineTo(x + w - r, y)
  ctx.quadraticCurveTo(x + w, y, x + w, y + r)
  ctx.lineTo(x + w, y + h - r)
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h)
  ctx.lineTo(x + r, y + h)
  ctx.quadraticCurveTo(x, y + h, x, y + h - r)
  ctx.lineTo(x, y + r)
  ctx.quadraticCurveTo(x, y, x + r, y)
  ctx.closePath()
}

const drawImageCover = (ctx: CanvasRenderingContext2D, img: HTMLImageElement, x: number, y: number, w: number, h: number) => {
  const ratio = Math.max(w / img.naturalWidth, h / img.naturalHeight)
  const drawW = img.naturalWidth * ratio
  const drawH = img.naturalHeight * ratio
  const dx = x - (drawW - w) / 2
  const dy = y - (drawH - h) / 2
  ctx.drawImage(img, dx, dy, drawW, drawH)
}

const downloadPNG = () => {
  if (!imageElement.value) return
  const width = layout.value === 'vertical' ? 1080 : 1480
  const height = layout.value === 'vertical' ? 1760 : 920
  const canvas = document.createElement('canvas')
  canvas.width = width
  canvas.height = height
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.fillStyle = '#f4f2f9'
  ctx.fillRect(0, 0, width, height)

  const padding = 56
  const innerW = width - padding * 2
  const innerH = height - padding * 2
  const isVertical = layout.value === 'vertical'
  const topArea = isVertical
    ? { x: padding, y: padding, w: innerW, h: Math.round(innerH * 0.38) }
    : { x: padding, y: padding, w: Math.round(innerW * 0.42), h: innerH }
  const photoArea = isVertical
    ? { x: padding, y: topArea.y + topArea.h + 32, w: innerW, h: innerH - topArea.h - 32 }
    : { x: topArea.x + topArea.w + 32, y: padding, w: innerW - topArea.w - 32, h: innerH }

  ctx.fillStyle = displayPureColor.value
  drawRoundRect(ctx, topArea.x, topArea.y, topArea.w, topArea.h, 42)
  ctx.fill()

  ctx.fillStyle = posterTextColor.value
  ctx.font = '700 58px Inter'
  ctx.textBaseline = 'top'
  const titleX = topArea.x + 64
  let offsetY = topArea.y + 72
  ctx.fillText(photoInfo.location, titleX, offsetY)
  offsetY += 78
  ctx.font = '600 36px Inter'
  ctx.fillText(photoInfo.shootTime, titleX, offsetY)
  offsetY += 64
  ctx.font = '500 28px Inter'
  const noteLines = photoInfo.note.match(/.{1,20}(\s|$)/g) || [photoInfo.note]
  noteLines.slice(0, 3).forEach((line) => {
    ctx.fillText(line.trim(), titleX, offsetY)
    offsetY += 40
  })

  ctx.fillStyle = 'rgba(255,255,255,0.22)'
  drawRoundRect(ctx, topArea.x + 40, topArea.y + 40, 130, 42, 999)
  ctx.fill()
  ctx.fillStyle = posterTextColor.value
  ctx.font = '600 18px Inter'
  ctx.fillText('色彩拼图', topArea.x + 60, topArea.y + 52)

  ctx.fillStyle = '#ffffff'
  drawRoundRect(ctx, photoArea.x, photoArea.y, photoArea.w, photoArea.h, 42)
  ctx.fill()
  ctx.save()
  ctx.beginPath()
  drawRoundRect(ctx, photoArea.x, photoArea.y, photoArea.w, photoArea.h, 42)
  ctx.clip()
  drawImageCover(ctx, imageElement.value, photoArea.x, photoArea.y, photoArea.w, photoArea.h)
  ctx.restore()

  const a = document.createElement('a')
  a.href = canvas.toDataURL('image/png')
  a.download = 'photocolors-mvp.png'
  a.click()
}
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="title-block">
        <span class="eyebrow">PhotoColors</span>
        <h1>移动修图拼图 MVP</h1>
        <p>导入照片后自动提取主色、亮色、暗色、辅助色，生成极简拼图并支持导出 PNG。</p>
      </div>
      <div class="controls">
        <button type="button" class="button secondary" @click="openFile">导入本地图片</button>
        <button type="button" class="button secondary" @click="toggleLayout">切换{{ layoutLabel }}</button>
        <button type="button" class="button primary" @click="downloadPNG" :disabled="!isReady">导出 PNG</button>
      </div>
    </header>

    <main class="workspace">
      <section class="preview" :class="layout">
        <article class="card poster-panel" :class="layout">
          <div class="poster-top" :style="{ background: displayPureColor, color: posterTextColor }">
            <div class="poster-tag">Pure Color</div>
            <div class="poster-location">{{ photoInfo.location }}</div>
            <div class="poster-time">{{ photoInfo.shootTime }}</div>
            <p class="poster-note">{{ photoInfo.note }}</p>
          </div>
          <div class="poster-photo">
            <template v-if="previewImage">
              <img ref="imageElement" :src="previewImage" alt="Imported photo" class="poster-image" />
            </template>
            <template v-else>
              <div class="poster-empty">
                <div>请导入图片，生成专属色彩海报</div>
              </div>
            </template>
          </div>
        </article>

        <article class="card edit-panel">
          <span class="panel-label">文字编辑区域</span>
          <div class="field">
            <label>拍摄时间</label>
            <input v-model="photoInfo.shootTime" type="text" placeholder="例如 2026-05-12 18:36" />
          </div>
          <div class="field">
            <label>地点</label>
            <input v-model="photoInfo.location" type="text" placeholder="例如 北京 / 朝阳区" />
          </div>
          <div class="field">
            <label>一句话</label>
            <textarea rows="4" v-model="photoInfo.note" placeholder="记录一段当下心情或场景。" />
          </div>
        </article>
      </section>

      <section class="palette-panel">
        <div class="swatches">
          <div class="swatch" :style="{ background: palette.main }">
            <span>主色</span>
            <strong>{{ palette.main }}</strong>
          </div>
          <div class="swatch" :style="{ background: palette.secondary }">
            <span>辅助色</span>
            <strong>{{ palette.secondary }}</strong>
          </div>
          <div class="swatch" :style="{ background: palette.light }">
            <span>亮色</span>
            <strong>{{ palette.light }}</strong>
          </div>
          <div class="swatch" :style="{ background: palette.dark }">
            <span>暗色</span>
            <strong>{{ palette.dark }}</strong>
          </div>
        </div>
        <p class="hint">自动识别图片主色、辅助色、亮色、暗色，并基于图片风格挑选最适合的纯色色块。</p>
      </section>
    </main>

    <input ref="fileInput" type="file" accept="image/*" class="hidden-input" @change="handleFile" />
  </div>
</template>

<style scoped>
.app-shell {
  width: min(100%, 1100px);
  margin: 0 auto;
  padding-bottom: 40px;
}
.topbar {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 18px;
  margin-bottom: 24px;
}
.title-block {
  max-width: 640px;
}
.eyebrow {
  display: inline-block;
  margin-bottom: 14px;
  color: #6c63ff;
  font-size: 0.86rem;
  letter-spacing: 0.26em;
  text-transform: uppercase;
}
h1 {
  margin: 0;
  font-size: clamp(2rem, 3.5vw, 3.2rem);
  line-height: 1.05;
  letter-spacing: -0.04em;
  color: var(--text);
}
p {
  margin: 18px 0 0;
  color: var(--muted);
  max-width: 720px;
  line-height: 1.75;
}
.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}
.button {
  border: none;
  border-radius: 999px;
  padding: 14px 22px;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease, filter 0.2s ease;
}
.button:hover {
  transform: translateY(-1px);
}
.button:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}
.button.primary {
  background: var(--text);
  color: #fff;
  box-shadow: 0 18px 48px rgba(36, 35, 71, 0.18);
}
.button.secondary {
  background: rgba(255, 255, 255, 0.96);
  color: var(--text);
  border: 1px solid rgba(15, 23, 42, 0.08);
}
.workspace {
  display: grid;
  gap: 24px;
}
.preview {
  display: grid;
  gap: 24px;
  grid-template-columns: 1fr;
}
.preview.horizontal {
  grid-template-columns: minmax(380px, 1.4fr) minmax(320px, 0.8fr);
}
.poster-panel {
  display: grid;
  grid-template-rows: auto 1fr;
  min-height: 720px;
}
.poster-top {
  padding: 48px 40px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  gap: 16px;
  min-height: 380px;
}
.poster-panel.horizontal {
  grid-template-columns: minmax(320px, 1.1fr) minmax(320px, 1fr);
  grid-template-rows: 1fr;
  min-height: 520px;
}
.poster-panel.horizontal .poster-top,
.poster-panel.horizontal .poster-photo {
  min-height: auto;
}
.poster-tag {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  font-size: 0.8rem;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: inherit;
  background: rgba(255, 255, 255, 0.12);
  padding: 10px 16px;
  border-radius: 999px;
}
.poster-location {
  font-size: clamp(2rem, 3vw, 3rem);
  line-height: 1.05;
  color: inherit;
  font-weight: 700;
}
.poster-time {
  font-size: 1.05rem;
  letter-spacing: 0.1em;
  color: inherit;
}
.poster-note {
  max-width: 560px;
  color: inherit;
  font-size: 1rem;
  line-height: 1.8;
}
.poster-photo {
  min-height: 520px;
  background: #f7f5ff;
  overflow: hidden;
  position: relative;
}
.poster-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
.poster-empty {
  min-height: 420px;
  display: grid;
  place-items: center;
  color: var(--muted);
  text-align: center;
  font-size: 1rem;
  padding: 40px;
}
.edit-panel {
  padding: 36px 36px 32px;
  display: grid;
  gap: 22px;
}
.card {
  border-radius: 34px;
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(15, 23, 42, 0.08);
  box-shadow: 0 20px 60px rgba(18, 22, 54, 0.08);
  overflow: hidden;
  position: relative;
}
.panel-label {
  position: absolute;
  z-index: 1;
  top: 18px;
  left: 18px;
  padding: 10px 16px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.88);
  color: #6b7280;
  font-size: 0.82rem;
  letter-spacing: 0.12em;
  text-transform: uppercase;
}
.photo-frame {
  min-height: 460px;
  display: grid;
  place-items: center;
  background: linear-gradient(180deg, #f7f5ff 0%, #ffffff 100%);
}
.photo-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
.empty-state {
  display: grid;
  place-items: center;
  gap: 12px;
  color: var(--muted);
  text-align: center;
  padding: 72px 20px;
}
.empty-hint {
  color: rgba(107, 114, 128, 0.88);
}
.photo-meta {
  padding: 18px 24px 22px;
  color: var(--muted);
  font-size: 0.95rem;
}
.file-name {
  display: inline-flex;
  padding: 8px 14px;
  border-radius: 999px;
  background: rgba(99, 102, 241, 0.08);
  color: #3f3c99;
}
.accent-panel {
  min-height: 260px;
  display: flex;
  align-items: flex-end;
  padding: 38px 34px;
  color: #ffffff;
}
.accent-meta {
  display: grid;
  gap: 16px;
}
.accent-meta h2 {
  margin: 0;
  font-size: clamp(2rem, 2.8vw, 2.8rem);
  letter-spacing: -0.06em;
}
.accent-meta p {
  margin: 0;
  max-width: 360px;
  color: rgba(255, 255, 255, 0.82);
  line-height: 1.7;
}
.color-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}
.tag {
  border-radius: 999px;
  padding: 10px 14px;
  color: #fff;
  font-size: 0.82rem;
}
.edit-panel {
  padding: 36px 36px 32px;
  display: grid;
  gap: 22px;
}
.field {
  display: grid;
  gap: 10px;
}
.field label {
  color: var(--muted);
  font-size: 0.82rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}
.field input,
.field textarea {
  width: 100%;
  border: none;
  border-radius: 22px;
  padding: 18px 20px;
  background: #f6f4ff;
  color: var(--text);
  outline: none;
  font-size: 1rem;
  line-height: 1.6;
}
.field textarea {
  min-height: 130px;
}
.palette-panel {
  display: grid;
  gap: 18px;
}
.swatches {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 18px;
}
.swatch {
  border-radius: 26px;
  padding: 26px 24px;
  min-height: 128px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  color: #fff;
  box-shadow: 0 20px 45px rgba(15, 23, 42, 0.1);
}
.swatch span {
  font-size: 0.9rem;
  opacity: 0.95;
}
.swatch strong {
  font-size: 1.3rem;
  letter-spacing: 0.02em;
}
.hint {
  color: var(--muted);
  max-width: 760px;
  line-height: 1.7;
}
.hidden-input {
  display: none;
}
@media (max-width: 900px) {
  .topbar {
    flex-direction: column;
    align-items: stretch;
  }
  .workspace {
    gap: 20px;
  }
  .preview.horizontal {
    grid-template-columns: 1fr;
    grid-template-areas: none;
  }
  .edit-panel {
    order: 3;
  }
}
</style>
