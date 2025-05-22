<script setup>
import { Analytics } from '@vercel/analytics/vue'
import { ref, computed, watch, onMounted } from 'vue'
import chroma from 'chroma-js'

const color = ref('#FF5733')
const errorMsg = ref('')
const savedSchemes = ref([])
const isRestoring = ref(false)
const copyToast = ref({ show: false, message: '', color: '' })
const isDark = ref(false)

const adjustments = ref([
  { hue: 0, saturation: 0, lightness: 0 },
  { hue: 0, saturation: 0, lightness: 0 },
  { hue: 0, saturation: 0, lightness: 0 },
])

watch(color, () => {
  if (!isRestoring.value) {
    adjustments.value = [
      { hue: 0, saturation: 0, lightness: 0 },
      { hue: 0, saturation: 0, lightness: 0 },
      { hue: 0, saturation: 0, lightness: 0 },
    ]
  }
})

function onInput() {
  if (!/^#([0-9A-Fa-f]{6})$/.test(color.value)) {
    errorMsg.value = '请输入合法的 HEX 颜色值，例如 #FF5733'
  } else {
    errorMsg.value = ''
  }
}

function generateMonochromatic(hex) {
  try {
    return chroma
      .scale([chroma(hex).brighten(2), hex, chroma(hex).darken(2)])
      .mode('lab')
      .colors(5)
  } catch {
    return [hex]
  }
}

function generateComplementary(hex) {
  try {
    const base = chroma(hex)
    const comp = base.set('hsl.h', (base.get('hsl.h') + 180) % 360)
    return [hex, comp.hex()]
  } catch {
    return [hex]
  }
}

function generateTriadic(hex) {
  try {
    const base = chroma(hex)
    const h = base.get('hsl.h')
    const s = base.get('hsl.s')
    const l = base.get('hsl.l')
    const color2 = chroma.hsl((h + 120) % 360, s, l).hex()
    const color3 = chroma.hsl((h + 240) % 360, s, l).hex()
    return [hex, color2, color3]
  } catch {
    return [hex]
  }
}

function applyAdjust(colors, adj) {
  return colors.map((c) => {
    try {
      let hsl = chroma(c).hsl()
      hsl[0] = (hsl[0] + adj.hue + 360) % 360
      hsl[1] = Math.min(1, Math.max(0, hsl[1] + adj.saturation / 100))
      hsl[2] = Math.min(1, Math.max(0, hsl[2] + adj.lightness / 100))
      return chroma.hsl(hsl[0], hsl[1], hsl[2]).hex()
    } catch {
      return c
    }
  })
}

const schemeDescriptions = {
  单色方案: '基于同一色相，通过调整亮度和饱和度生成深浅变化，风格统一，适合简洁设计。',
  互补色方案: '选择基础色的对立色，形成强烈对比，适合突出重点内容。',
  三色调和方案: '在色轮上等距选择三种颜色，色彩丰富且协调，适合活泼、平衡的设计。',
}

const colorSchemes = computed(() => [
  {
    name: '单色方案',
    colors: applyAdjust(generateMonochromatic(color.value), adjustments.value[0]),
    desc: schemeDescriptions['单色方案'],
  },
  {
    name: '互补色方案',
    colors: applyAdjust(generateComplementary(color.value), adjustments.value[1]),
    desc: schemeDescriptions['互补色方案'],
  },
  {
    name: '三色调和方案',
    colors: applyAdjust(generateTriadic(color.value), adjustments.value[2]),
    desc: schemeDescriptions['三色调和方案'],
  },
])

function exportCSS(scheme) {
  const css = scheme.colors.map((color, index) => `--color${index + 1}: ${color};`).join('\n')
  navigator.clipboard
    .writeText(css)
    .then(() => alert('CSS 代码已复制到剪贴板'))
    .catch((err) => console.error('复制失败:', err))
}

function exportPNG(scheme) {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  const blockSize = 48
  const padding = 8
  const width = scheme.colors.length * (blockSize + padding) - padding
  const height = blockSize

  canvas.width = width
  canvas.height = height

  scheme.colors.forEach((color, index) => {
    ctx.fillStyle = color
    ctx.fillRect(index * (blockSize + padding), 0, blockSize, blockSize)
  })

  const link = document.createElement('a')
  link.download = `${scheme.name}.png`
  link.href = canvas.toDataURL('image/png')
  link.click()
}

function exportPalette(scheme) {
  const gpl = `GIMP Palette
Name: ${scheme.name}
Columns: 0
#
${scheme.colors
  .map((color) => {
    const [r, g, b] = chroma(color).rgb()
    return `${r} ${g} ${b} ${color}`
  })
  .join('\n')}`

  const blob = new Blob([gpl], { type: 'text/plain' })
  const link = document.createElement('a')
  link.download = `${scheme.name}.gpl`
  link.href = URL.createObjectURL(blob)
  link.click()
}

function saveScheme(scheme) {
  const id = Date.now() + Math.random().toString(36).slice(2)
  const schemeIndex = colorSchemes.value.findIndex((s) => s.name === scheme.name)

  const newScheme = {
    id,
    name: scheme.name,
    colors: scheme.colors,
    date: Date.now(),
    baseColor: color.value,
    adjustment: adjustments.value[schemeIndex],
  }
  savedSchemes.value.push(newScheme)
  localStorage.setItem('savedSchemes', JSON.stringify(savedSchemes.value))
  alert('方案已保存！')
}

function restoreBaseColor(scheme) {
  color.value = scheme.baseColor
  alert('已恢复基础色！')
}

function restoreAdjustment(scheme) {
  if (!scheme.adjustment) {
    alert('该方案没有保存调整参数！')
    return
  }

  isRestoring.value = true
  const schemeIndex = colorSchemes.value.findIndex((s) => s.name === scheme.name)
  if (schemeIndex !== -1) {
    adjustments.value[schemeIndex] = { ...scheme.adjustment }
  }
  isRestoring.value = false
  alert('已恢复调整参数！')
}

function deleteScheme(id) {
  savedSchemes.value = savedSchemes.value.filter((s) => s.id !== id)
  localStorage.setItem('savedSchemes', JSON.stringify(savedSchemes.value))
}

function formatDate(ts) {
  const d = new Date(ts)
  return `${d.getFullYear()}-${(d.getMonth() + 1).toString().padStart(2, '0')}-${d.getDate().toString().padStart(2, '0')} ${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}`
}

function copyColorToClipboard(colorValue) {
  try {
    navigator.clipboard
      .writeText(colorValue)
      .then(() => {
        copyToast.value = {
          show: true,
          message: `已复制颜色：${colorValue}`,
          color: colorValue,
        }
        setTimeout(() => {
          copyToast.value.show = false
        }, 2000)
      })
      .catch((err) => {
        console.error('复制失败:', err)
        copyToast.value = {
          show: true,
          message: '复制失败，请手动复制',
          color: '#e74c3c',
        }
        setTimeout(() => {
          copyToast.value.show = false
        }, 2000)
      })
  } catch (err) {
    console.error('复制失败:', err)
    copyToast.value = {
      show: true,
      message: '复制失败，请手动复制',
      color: '#e74c3c',
    }
    setTimeout(() => {
      copyToast.value.show = false
    }, 2000)
  }
}

onMounted(() => {
  const data = localStorage.getItem('savedSchemes')
  if (data) {
    savedSchemes.value = JSON.parse(data)
  }
  const saved = localStorage.getItem('color-matching-dark')
  if (saved) {
    isDark.value = saved === 'true'
    updateDarkClass()
  } else {
    isDark.value = window.matchMedia('(prefers-color-scheme: dark)').matches
    updateDarkClass()
  }
})

function toggleDark() {
  isDark.value = !isDark.value
  localStorage.setItem('color-matching-dark', isDark.value)
  updateDarkClass()
}

function updateDarkClass() {
  if (isDark.value) {
    document.documentElement.classList.add('dark-mode')
  } else {
    document.documentElement.classList.remove('dark-mode')
  }
}
</script>

<template>
  <Analytics />
  <div class="app-container">
    <header class="site-header">
      <h1>配色生成器<span>快速生成和谐配色方案</span></h1>
      <button @click="toggleDark" class="dark-toggle-btn">
        {{ isDark ? '☀️ 日间模式' : '🌙 夜间模式' }}
      </button>
      <p class="site-description">专业的在线配色工具，为设计师和开发者提供灵感</p>
    </header>

    <main>
      <section class="color-selector-section" aria-labelledby="select-color-heading">
        <h2 id="select-color-heading">选择基础颜色</h2>
        <div class="color-picker">
          <label>
            颜色选择器：
            <input type="color" v-model="color" aria-label="颜色选择器" />
          </label>
          <input
            type="text"
            v-model="color"
            maxlength="7"
            placeholder="#RRGGBB"
            @input="onInput"
            aria-label="十六进制颜色代码"
          />
          <div class="color-preview" :style="{ backgroundColor: color }" :title="color"></div>
        </div>
        <div v-if="errorMsg" class="error-msg" role="alert">{{ errorMsg }}</div>
      </section>

      <section class="schemes-section" aria-labelledby="color-schemes-heading">
        <h2 id="color-schemes-heading">配色方案</h2>
        <div class="schemes-grid">
          <article v-for="(scheme, idx) in colorSchemes" :key="scheme.name" class="scheme-card">
            <h3 class="scheme-title">{{ scheme.name }}</h3>
            <div class="scheme-colors">
              <div
                v-for="c in scheme.colors"
                :key="c"
                class="color-block"
                :style="{ backgroundColor: c }"
                :title="c"
                @click="copyColorToClipboard(c)"
              ></div>
            </div>
            <p class="scheme-desc">{{ scheme.desc }}</p>

            <div class="adjustment-controls">
              <h4>调整参数</h4>
              <div class="sliders">
                <label>
                  色相
                  <input
                    type="range"
                    min="-180"
                    max="180"
                    v-model.number="adjustments[idx].hue"
                    aria-label="调整色相"
                  />
                  <span>{{ adjustments[idx].hue }}</span>
                </label>
                <label>
                  饱和度
                  <input
                    type="range"
                    min="-100"
                    max="100"
                    v-model.number="adjustments[idx].saturation"
                    aria-label="调整饱和度"
                  />
                  <span>{{ adjustments[idx].saturation }}</span>
                </label>
                <label>
                  亮度
                  <input
                    type="range"
                    min="-100"
                    max="100"
                    v-model.number="adjustments[idx].lightness"
                    aria-label="调整亮度"
                  />
                  <span>{{ adjustments[idx].lightness }}</span>
                </label>
              </div>
            </div>

            <div class="scheme-actions">
              <div class="button-group export-buttons">
                <button @click="exportCSS(scheme)" class="btn">导出 CSS</button>
                <button @click="exportPNG(scheme)" class="btn">导出 PNG</button>
                <button @click="exportPalette(scheme)" class="btn">导出调色板</button>
              </div>
              <div class="button-group save-buttons">
                <button @click="saveScheme(scheme)" class="btn btn-primary">保存方案</button>
              </div>
            </div>
          </article>
        </div>
      </section>

      <section
        v-if="savedSchemes.length > 0"
        class="saved-schemes-section"
        aria-labelledby="saved-schemes-heading"
      >
        <h2 id="saved-schemes-heading">已保存的方案</h2>
        <div class="saved-schemes-grid">
          <article v-for="scheme in savedSchemes" :key="scheme.id" class="saved-scheme-card">
            <header class="saved-scheme-header">
              <h3 class="scheme-name">{{ scheme.name }}</h3>
              <time class="scheme-date" :datetime="scheme.date">{{ formatDate(scheme.date) }}</time>
            </header>

            <div class="scheme-colors">
              <div
                v-for="color in scheme.colors"
                :key="color"
                class="color-block"
                :style="{ backgroundColor: color }"
                :title="color"
                @click="copyColorToClipboard(color)"
              ></div>
            </div>

            <div class="button-group scheme-actions">
              <button @click="restoreBaseColor(scheme)" class="btn btn-secondary">
                恢复基础色
              </button>
              <button
                @click="restoreAdjustment(scheme)"
                :disabled="!scheme.adjustment"
                class="btn btn-secondary"
              >
                恢复调整参数
              </button>
              <button @click="deleteScheme(scheme.id)" class="btn btn-danger">删除</button>
            </div>
          </article>
        </div>
      </section>
    </main>

    <footer class="site-footer">
      <p>&copy; {{ new Date().getFullYear() }} 配色生成器 | 设计师和开发者的色彩助手</p>
      <p class="footer-links">
        <a href="#" title="关于我们">关于我们</a> | <a href="#" title="使用帮助">使用帮助</a> |
        <a href="#" title="联系我们">联系我们</a>
      </p>
    </footer>
  </div>

  <!-- 复制成功提示 -->
  <div v-if="copyToast.show" class="copy-toast" :style="{ backgroundColor: copyToast.color }">
    {{ copyToast.message }}
  </div>
</template>

<style>
/* 全局样式与变量 */
:root {
  --primary-color: #ff5733;
  --primary-light: #ff8c61;
  --primary-dark: #e04121;
  --secondary-color: #4388cc;
  --success-color: #2ecc71;
  --danger-color: #e74c3c;
  --text-color: #333;
  --text-light: #666;
  --background: #fff;
  --background-alt: #f5f7fa;
  --border-color: #eaecef;
  --shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  --radius: 8px;
  --transition: all 0.2s ease;
  --spacing: 16px;
  --container-width: 1200px;
  --header-height: 120px;
  --footer-height: 80px;
}

/* 基础重置 */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  scroll-behavior: smooth;
}

body {
  margin: 0;
  padding: 0;
  font-family: 'Noto Sans SC', sans-serif;
  color: var(--text-color);
  background-color: var(--background-alt);
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

h1,
h2,
h3,
h4,
h5,
h6 {
  margin: 0 0 var(--spacing) 0;
  line-height: 1.3;
  color: var(--text-color);
  font-weight: 700;
}

p {
  margin: 0 0 var(--spacing) 0;
}

a {
  color: var(--primary-color);
  text-decoration: none;
  transition: var(--transition);
}

a:hover {
  color: var(--primary-dark);
}

button,
.btn {
  cursor: pointer;
  border: none;
  border-radius: 4px;
  background: #f0f0f0;
  color: var(--text-color);
  padding: 8px 16px;
  font-size: 14px;
  transition: var(--transition);
}

button:hover,
.btn:hover {
  background: #e0e0e0;
}

.btn-primary {
  background: var(--primary-color);
  color: white;
}

.btn-primary:hover {
  background: var(--primary-dark);
}

.btn-secondary {
  background: var(--secondary-color);
  color: white;
}

.btn-secondary:hover {
  background: #3476b5;
}

.btn-danger {
  background: var(--danger-color);
  color: white;
}

.btn-danger:hover {
  background: #c0392b;
}

button:disabled,
.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 布局容器 */
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  width: 100%;
  max-width: 100%;
  margin: 0 auto;
  padding: 0;
}

main {
  flex: 1;
  width: 100%;
  padding: 0;
  max-width: 1200px;
  margin: 0 auto;
}

section {
  margin-bottom: 60px;
  width: 100%;
  max-width: var(--container-width);
  margin-left: auto;
  margin-right: auto;
}

/* 头部样式 */
.site-header {
  background: var(--background);
  box-shadow: var(--shadow);
  padding: 32px 0;
  text-align: center;
  margin-bottom: 40px;
}

.site-header h1 {
  font-size: 2.6rem;
  color: var(--primary-color);
  margin-bottom: 8px;
}

.site-header h1 span {
  display: block;
  font-size: 1.2rem;
  font-weight: 400;
  color: var(--text-light);
  margin-top: 8px;
}

.site-description {
  color: var(--text-light);
  font-size: 1rem;
  max-width: 600px;
  margin: 8px auto;
}

/* 颜色选择器部分 */
.color-selector-section {
  background: var(--background);
  border-radius: var(--radius);
  padding: 24px;
  box-shadow: var(--shadow);
  max-width: 800px;
  margin: 0 auto 40px;
}

.color-selector-section h2 {
  color: var(--primary-color);
  margin-bottom: 16px;
  padding-bottom: 8px;
  border-bottom: 2px solid var(--border-color);
}

.color-picker {
  display: flex;
  align-items: center;
  gap: var(--spacing);
  flex-wrap: wrap;
}

.color-picker label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 500;
}

.color-picker input[type='color'] {
  width: 40px;
  height: 40px;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  background: none;
  cursor: pointer;
}

.color-picker input[type='text'] {
  padding: 8px 12px;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  font-size: 16px;
  width: 100px;
}

.color-preview {
  width: 40px;
  height: 40px;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  box-shadow: var(--shadow);
}

.error-msg {
  color: var(--danger-color);
  margin-top: 8px;
  font-size: 14px;
  padding: 8px;
  background: rgba(231, 76, 60, 0.1);
  border-radius: 4px;
}

/* 配色方案部分 */
.schemes-section h2,
.saved-schemes-section h2 {
  text-align: center;
  margin-bottom: 32px;
  color: var(--primary-color);
  font-size: 1.8rem;
}

.schemes-grid,
.saved-schemes-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 24px;
}

.scheme-card,
.saved-scheme-card {
  background: var(--background);
  border-radius: var(--radius);
  padding: 24px;
  box-shadow: var(--shadow);
  transition: var(--transition);
  border: 1px solid var(--border-color);
}

.scheme-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.1);
}

.scheme-title {
  font-size: 1.2rem;
  color: var(--primary-color);
  margin-bottom: 16px;
}

.scheme-colors {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.color-block {
  width: 48px;
  height: 48px;
  border-radius: 6px;
  border: 1px solid var(--border-color);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  cursor: pointer;
  transition: var(--transition);
  position: relative;
}

.color-block:hover {
  transform: scale(1.1);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.color-block:hover::after {
  content: '点击复制';
  position: absolute;
  bottom: -24px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  white-space: nowrap;
  pointer-events: none;
}

.scheme-desc {
  color: var(--text-light);
  font-size: 14px;
  margin-bottom: 16px;
  line-height: 1.5;
}

/* 调整参数部分 */
.adjustment-controls {
  background: var(--background-alt);
  padding: 16px;
  border-radius: var(--radius);
  margin-bottom: 16px;
}

.adjustment-controls h4 {
  margin-bottom: 12px;
  font-size: 1rem;
  color: var(--primary-color);
}

.sliders {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.sliders label {
  display: flex;
  flex-direction: column;
  font-size: 14px;
  color: var(--text-light);
  position: relative;
}

.sliders input[type='range'] {
  width: 100%;
  margin: 8px 0;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e0e0e0;
  border-radius: 3px;
  outline: none;
}

.sliders input[type='range']::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  background: var(--primary-color);
  border-radius: 50%;
  cursor: pointer;
}

.sliders span {
  position: absolute;
  right: 0;
  top: 0;
  font-size: 12px;
  color: var(--text-light);
}

/* 按钮组 */
.scheme-actions {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.button-group {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

/* 已保存方案部分 */
.saved-scheme-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  align-items: center;
}

.scheme-date {
  font-size: 12px;
  color: var(--text-light);
}

/* 页脚 */
.site-footer {
  background: var(--background);
  border-top: 1px solid var(--border-color);
  padding: 24px;
  text-align: center;
  margin-top: 60px;
  color: var(--text-light);
}

.footer-links {
  margin-top: 8px;
}

.footer-links a {
  margin: 0 8px;
  color: var(--text-light);
}

.footer-links a:hover {
  color: var(--primary-color);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .color-picker {
    flex-direction: column;
    align-items: flex-start;
  }

  .schemes-grid,
  .saved-schemes-grid {
    grid-template-columns: 1fr;
  }

  .site-header h1 {
    font-size: 2rem;
  }

  .site-header h1 span {
    font-size: 1rem;
  }

  .button-group {
    flex-direction: column;
  }

  .btn {
    width: 100%;
  }
}

/* 暗色模式支持 */
@media (prefers-color-scheme: dark) {
  :root {
    --text-color: #f0f0f0;
    --text-light: #aaa;
    --background: #222;
    --background-alt: #1a1a1a;
    --border-color: #333;
  }

  .scheme-card,
  .saved-scheme-card {
    border-color: #333;
  }

  .color-block {
    border-color: #444;
  }

  .sliders input[type='range'] {
    background: #444;
  }

  .adjustment-controls {
    background: #2a2a2a;
  }
}

/* 复制成功提示样式 */
.copy-toast {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 12px 24px;
  border-radius: 4px;
  color: white;
  font-size: 14px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 1000;
  animation: fadeInOut 2s ease-in-out;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

@keyframes fadeInOut {
  0% {
    opacity: 0;
    transform: translate(-50%, 20px);
  }
  15% {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  85% {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -20px);
  }
}

/* 夜间按钮样式 */
.dark-toggle-btn {
  margin-left: 1rem;
  padding: 6px 16px;
  border: none;
  border-radius: 4px;
  background: var(--color-background-mute, #eee);
  color: var(--color-text, #222);
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.3s;
}
.dark-toggle-btn:hover {
  background: var(--color-border-hover, #ccc);
}

.dark-mode {
  /* 基础变量 */
  --vt-c-white: #181818;
  --vt-c-white-soft: #222222;
  --vt-c-white-mute: #282828;
  --vt-c-black: #ffffff;
  --vt-c-black-soft: #f8f8f8;
  --vt-c-black-mute: #f2f2f2;
  --vt-c-indigo: #c9def1;

  --vt-c-divider-light-1: rgba(84, 84, 84, 0.65);
  --vt-c-divider-light-2: rgba(84, 84, 84, 0.48);
  --vt-c-divider-dark-1: rgba(60, 60, 60, 0.29);
  --vt-c-divider-dark-2: rgba(60, 60, 60, 0.12);

  --vt-c-text-light-1: #ffffff;
  --vt-c-text-light-2: rgba(235, 235, 235, 0.64);
  --vt-c-text-dark-1: var(--vt-c-indigo);
  --vt-c-text-dark-2: rgba(60, 60, 60, 0.66);

  /* 语义变量 */
  --color-background: var(--vt-c-black);
  --color-background-soft: var(--vt-c-black-soft);
  --color-background-mute: var(--vt-c-black-mute);
  --color-border: var(--vt-c-divider-dark-2);
  --color-border-hover: var(--vt-c-divider-dark-1);
  --color-heading: var(--vt-c-text-dark-1);
  --color-text: var(--vt-c-text-dark-2);

  /* 应用特定变量 */
  --text-color: #f0f0f0;
  --text-light: #aaa;
  --background: #222;
  --background-alt: #1a1a1a;
  --border-color: #333;
  --primary-color: #5c9ce6;
}

/* 按钮高对比度样式 */
.dark-mode button,
.dark-mode .btn {
  background: #333;
  color: #fff;
  border: 1px solid #444;
}

.dark-mode button:hover,
.dark-mode .btn:hover {
  background: #444;
  border-color: #555;
}

.dark-mode .btn-primary {
  background: var(--primary-color);
  color: white;
  border: none;
}

.dark-mode .btn-primary:hover {
  background: #4a8cd6;
}

.dark-mode .btn-secondary {
  background: #3476b5;
  color: white;
  border: none;
}

.dark-mode .btn-secondary:hover {
  background: #2a6299;
}

.dark-mode .btn-danger {
  background: #c0392b;
  color: white;
  border: none;
}

.dark-mode .btn-danger:hover {
  background: #a93226;
}
</style>
