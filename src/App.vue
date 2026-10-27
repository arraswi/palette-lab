<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

type Mode = 'analogous' | 'complementary' | 'mono'
type Swatch = { tone: string; hex: string; locked: boolean }
type SavedPalette = { id: string; name: string; colors: string[] }

const tones = ['050', '200', '400', '600', '800']
const mode = ref<Mode>('analogous')
const notice = ref('')
const colors = ref<Swatch[]>([])
const saved = ref<SavedPalette[]>(readSaved())
const progress = computed(() => colors.value.length)

function readSaved(): SavedPalette[] {
  try {
    return JSON.parse(localStorage.getItem('palette-lab.saved.v1') ?? '[]') as SavedPalette[]
  } catch {
    return []
  }
}

function hslToHex(hue: number, saturation: number, lightness: number): string {
  const s = saturation / 100
  const l = lightness / 100
  const chroma = (1 - Math.abs(2 * l - 1)) * s
  const segment = hue / 60
  const second = chroma * (1 - Math.abs(segment % 2 - 1))
  const [red, green, blue] = segment < 1 ? [chroma, second, 0]
    : segment < 2 ? [second, chroma, 0]
      : segment < 3 ? [0, chroma, second]
        : segment < 4 ? [0, second, chroma]
          : segment < 5 ? [second, 0, chroma] : [chroma, 0, second]
  const match = l - chroma / 2
  return '#' + [red, green, blue].map((value) => Math.round((value + match) * 255).toString(16).padStart(2, '0')).join('').toUpperCase()
}

function offsetsFor(currentMode: Mode): number[] {
  if (currentMode === 'complementary') return [0, 22, 180, 202, 225]
  if (currentMode === 'mono') return [0, 0, 0, 0, 0]
  return [-32, -16, 0, 16, 32]
}

function generatePalette() {
  const baseHue = Math.floor(Math.random() * 360)
  const offsets = offsetsFor(mode.value)
  const lightness = [92, 76, 60, 43, 27]
  colors.value = tones.map((tone, index) => {
    if (colors.value[index]?.locked) return colors.value[index]
    const hue = (baseHue + offsets[index] + 360) % 360
    const saturation = mode.value === 'mono' ? 8 + (index % 2) * 4 : 62 + ((baseHue + index * 7) % 18)
    return { tone, hex: hslToHex(hue, saturation, lightness[index]), locked: false }
  })
  notice.value = 'Nova paleta gerada'
}

function toggleLock(index: number) {
  colors.value[index].locked = !colors.value[index].locked
}

async function copyText(text: string, message: string) {
  try {
    await navigator.clipboard.writeText(text)
    notice.value = message
  } catch {
    notice.value = 'Não foi possível acessar a área de transferência'
  }
}

function copyCss() {
  const lines = [':root {', ...colors.value.map((swatch) => '  --color-' + swatch.tone + ': ' + swatch.hex + ';'), '}']
  void copyText(lines.join(String.fromCharCode(10)), 'Variáveis CSS copiadas')
}

function exportJson() {
  const content = JSON.stringify({ mode: mode.value, colors: colors.value.map(({ tone, hex }) => ({ tone, hex })) }, null, 2)
  const url = URL.createObjectURL(new Blob([content], { type: 'application/json' }))
  const link = document.createElement('a')
  link.href = url
  link.download = 'palette-lab.json'
  link.click()
  URL.revokeObjectURL(url)
  notice.value = 'Arquivo JSON exportado'
}

function savePalette() {
  const name = 'Paleta ' + new Intl.DateTimeFormat('pt-BR', { hour: '2-digit', minute: '2-digit' }).format(new Date())
  const item = { id: crypto.randomUUID(), name, colors: colors.value.map((swatch) => swatch.hex) }
  saved.value = [item, ...saved.value].slice(0, 6)
  notice.value = 'Paleta guardada neste navegador'
}

function loadPalette(item: SavedPalette) {
  colors.value = item.colors.map((hex, index) => ({ tone: tones[index] ?? String(index), hex, locked: false }))
  notice.value = item.name + ' carregada'
}

watch(saved, (value) => localStorage.setItem('palette-lab.saved.v1', JSON.stringify(value)), { deep: true })
function handleShortcut(event: KeyboardEvent) {
  if (event.code === 'Space' && event.target === document.body) {
    event.preventDefault()
    generatePalette()
  }
}
onMounted(() => window.addEventListener('keydown', handleShortcut))
onUnmounted(() => window.removeEventListener('keydown', handleShortcut))
generatePalette()
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Palette Lab início"><span class="brand-symbol">◉</span><span>palette<span class="brand-light">lab</span></span></a>
      <div class="topbar-meta"><span class="online-dot"></span> FERRAMENTA DE CORES <span class="version">v1.0</span></div>
    </header>

    <main>
      <section class="intro">
        <div>
          <p class="eyebrow">CORES COM INTENÇÃO</p>
          <h1>Encontre sua próxima<br /><span>paleta favorita.</span></h1>
          <p class="intro-copy">Gere combinações harmoniosas, ajuste os tons e leve a paleta direto para o seu projeto.</p>
        </div>
        <div class="intro-actions">
          <label class="mode-select-label" for="mode">HARMONIA</label>
          <select id="mode" v-model="mode" class="mode-select" @change="generatePalette">
            <option value="analogous">Análoga</option>
            <option value="complementary">Complementar</option>
            <option value="mono">Monocromática</option>
          </select>
          <button class="generate-button" @click="generatePalette"><span>✦</span> Gerar paleta</button>
        </div>
      </section>

      <section class="palette-area" aria-label="Paleta atual">
        <div class="palette-heading"><div><span class="eyebrow">PALETA ATUAL</span><span class="palette-subtitle">Clique no código para copiar · trave cores para mantê-las</span></div><span class="swatch-count">{{ progress }} CORES</span></div>
        <div class="swatch-grid">
          <article v-for="(swatch, index) in colors" :key="swatch.tone" class="swatch-card">
            <button class="color-surface" :style="{ backgroundColor: swatch.hex }" :aria-label="'Copiar cor ' + swatch.hex" @click="copyText(swatch.hex, swatch.hex + ' copiada')">
              <span class="tone-label">{{ swatch.tone }}</span>
              <span class="copy-hint">COPIAR HEX</span>
            </button>
            <div class="swatch-details">
              <button class="hex-value" @click="copyText(swatch.hex, swatch.hex + ' copiada')">{{ swatch.hex }}</button>
              <div class="swatch-tools">
                <label class="color-picker" :aria-label="'Editar cor ' + swatch.tone"><input v-model="swatch.hex" type="color" /></label>
                <button class="lock-button" :class="{ locked: swatch.locked }" :aria-label="swatch.locked ? 'Destravar cor' : 'Travar cor'" :title="swatch.locked ? 'Destravar' : 'Travar'" @click="toggleLock(index)">{{ swatch.locked ? '●' : '○' }}</button>
              </div>
            </div>
            <span class="tone-caption">TONE {{ swatch.tone }}</span>
          </article>
        </div>
      </section>

      <section class="action-row">
        <div class="keyboard-hint"><span class="keycap">SPACE</span><span>Gere novas ideias com a barra de espaço</span></div>
        <div class="action-buttons">
          <button class="secondary-button" @click="savePalette">♡ <span>Salvar</span></button>
          <button class="secondary-button" @click="copyCss">⌘ <span>Copiar CSS</span></button>
          <button class="primary-small" @click="exportJson">Exportar JSON <span>↗</span></button>
        </div>
      </section>

      <section v-if="saved.length" class="saved-section">
        <div class="saved-heading"><div><p class="eyebrow">SUA BIBLIOTECA</p><h2>Paletas salvas</h2></div><span class="saved-count">{{ saved.length }} / 6</span></div>
        <div class="saved-list">
          <button v-for="item in saved" :key="item.id" class="saved-palette" @click="loadPalette(item)">
            <span class="mini-colors"><i v-for="hex in item.colors" :key="hex" :style="{ backgroundColor: hex }"></i></span><span>{{ item.name }}</span><span class="saved-arrow">↗</span>
          </button>
        </div>
      </section>

      <footer class="footer"><span>PALETTE LAB <b>·</b> FEITO PARA QUEM CRIA</span><span class="notice" aria-live="polite">{{ notice }}</span><span>As paletas ficam no seu navegador.</span></footer>
    </main>
  </div>
</template>
