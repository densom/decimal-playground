<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import LcdExplorer from './components/LcdExplorer.vue'

const activeTab = ref('explore') // 'explore' | 'lcd'

// ── State ─────────────────────────────────────────────────────────────────────
const mode = ref('hundredths') // 'tenths' | 'hundredths' | 'thousandths'
const filledCells = ref(new Set())
const inputText = ref('')
const isDragging = ref(false)
const dragFill = ref(true) // true = painting, false = erasing

// ── Mode config ───────────────────────────────────────────────────────────────
const MODES = {
  tenths:      { denom: 10,   cols: 1,  rows: 10,  label: 'Tenths',      sub: '÷ 10',  places: 1 },
  hundredths:  { denom: 100,  cols: 10, rows: 10,  label: 'Hundredths',  sub: '÷ 100', places: 2 },
  thousandths: { denom: 1000, cols: 25, rows: 40,  label: 'Thousandths', sub: '÷ 1000',places: 3 },
}

const cfg = computed(() => MODES[mode.value])
const denominator = computed(() => cfg.value.denom)

// ── Derived numbers ───────────────────────────────────────────────────────────
const filledCount = computed(() => filledCells.value.size)

const decimalValue = computed(() => {
  const v = filledCount.value / denominator.value
  return v.toFixed(cfg.value.places)
})

const percentValue = computed(() => {
  return ((filledCount.value / denominator.value) * 100).toFixed(cfg.value.places === 3 ? 1 : 0)
})

function gcd(a, b) { return b === 0 ? a : gcd(b, a % b) }

const simplified = computed(() => {
  const n = filledCount.value
  const d = denominator.value
  if (n === 0 || n === d) return null
  const g = gcd(n, d)
  if (g === 1) return null
  return { n: n / g, d: d / g }
})

// ── Mode switching ────────────────────────────────────────────────────────────
function setMode(m) {
  if (m === mode.value) return
  const oldDenom = denominator.value
  const count = filledCells.value.size
  mode.value = m
  const newDenom = MODES[m].denom
  const scaled = Math.round((count / oldDenom) * newDenom)
  const next = new Set()
  for (let i = 0; i < scaled; i++) next.add(i)
  filledCells.value = next
  inputText.value = (scaled / newDenom).toFixed(MODES[m].places)
}

// ── Cell interaction ──────────────────────────────────────────────────────────
function startDrag(i) {
  isDragging.value = true
  dragFill.value = !filledCells.value.has(i)
  applyCell(i)
}

function onEnter(i) {
  if (isDragging.value) applyCell(i)
}

function applyCell(i) {
  const next = new Set(filledCells.value)
  dragFill.value ? next.add(i) : next.delete(i)
  filledCells.value = next
  inputText.value = (filledCells.value.size / denominator.value).toFixed(cfg.value.places)
}

function stopDrag() { isDragging.value = false }

// ── Typed input ───────────────────────────────────────────────────────────────
function applyInput() {
  const raw = inputText.value.trim()
  let val = null

  // Accept "3/10", "37/100", "0.37", ".37"
  const fracMatch = raw.match(/^(\d+)\s*\/\s*(\d+)$/)
  if (fracMatch) {
    val = parseInt(fracMatch[1]) / parseInt(fracMatch[2])
  } else {
    val = parseFloat(raw)
  }

  if (isNaN(val) || val < 0 || val > 1) return

  const count = Math.round(val * denominator.value)
  const next = new Set()
  for (let i = 0; i < count; i++) next.add(i)
  filledCells.value = next
  inputText.value = (count / denominator.value).toFixed(cfg.value.places)
}

function handleInputKey(e) {
  if (e.key === 'Enter') applyInput()
}

// ── Convenience fills ─────────────────────────────────────────────────────────
function fillHalf() {
  const half = Math.round(denominator.value / 2)
  const next = new Set()
  for (let i = 0; i < half; i++) next.add(i)
  filledCells.value = next
  inputText.value = (half / denominator.value).toFixed(cfg.value.places)
}

function fillQuarter() {
  const q = Math.round(denominator.value / 4)
  const next = new Set()
  for (let i = 0; i < q; i++) next.add(i)
  filledCells.value = next
  inputText.value = (q / denominator.value).toFixed(cfg.value.places)
}

function reset() {
  filledCells.value = new Set()
  inputText.value = ''
}

// ── Global drag stop ──────────────────────────────────────────────────────────
onMounted(() => window.addEventListener('mouseup', stopDrag))
onBeforeUnmount(() => window.removeEventListener('mouseup', stopDrag))
</script>

<template>
  <div class="app" @mouseup="stopDrag">

    <!-- ── Header ─────────────────────────────────────────────────── -->
    <header class="header">
      <div class="logo">
        <span class="logo-icon">🔢</span>
        <div>
          <h1>Decimal Playground</h1>
          <p class="tagline">Click · Drag · Discover</p>
        </div>
      </div>
      <nav class="tabs">
        <button class="tab" :class="{ 'tab--active': activeTab === 'explore' }" @click="activeTab = 'explore'">Explore Decimals</button>
        <button class="tab" :class="{ 'tab--active': activeTab === 'lcd' }" @click="activeTab = 'lcd'">LCD Explorer</button>
      </nav>
    </header>

    <!-- ── LCD tab ────────────────────────────────────────────────── -->
    <LcdExplorer v-if="activeTab === 'lcd'" />

    <!-- ── Main layout ────────────────────────────────────────────── -->
    <template v-if="activeTab === 'explore'">
    <div class="layout">

      <!-- ── Left: Grid ─────────────────────────────────────────── -->
      <section class="grid-section">
        <div class="grid-label">
          <span class="whole-label">1 whole</span>
        </div>

        <div class="grid-outer">
          <Transition name="grid-swap" mode="out-in">
            <div
              class="grid"
              :key="mode"
              :style="{
                gridTemplateColumns: `repeat(${cfg.cols}, 1fr)`,
                gridTemplateRows: `repeat(${cfg.rows}, 1fr)`,
              }"
              :class="[`grid--${mode}`, { 'grid--small': mode === 'thousandths' }]"
              @dragstart.prevent
            >
              <div
                v-for="i in denominator"
                :key="i"
                class="cell"
                :class="{ 'cell--filled': filledCells.has(i - 1) }"
                @mousedown.prevent="startDrag(i - 1)"
                @mouseenter="onEnter(i - 1)"
              />
            </div>
          </Transition>
        </div>

        <!-- Quick hints below grid -->
        <div class="grid-hint">
          <span v-if="mode === 'tenths'">Each strip = <strong>1 tenth</strong> = <strong>0.1</strong></span>
          <span v-else-if="mode === 'hundredths'">Each square = <strong>1 hundredth</strong> = <strong>0.01</strong></span>
          <span v-else>Each tiny square = <strong>1 thousandth</strong> = <strong>0.001</strong></span>
        </div>
      </section>

      <!-- ── Right: Controls ────────────────────────────────────── -->
      <aside class="panel">

        <!-- Mode switcher -->
        <div class="mode-switcher">
          <p class="section-label">Cut the square into…</p>
          <div class="mode-buttons">
            <button
              v-for="(mcfg, key) in MODES"
              :key="key"
              class="mode-btn"
              :class="{ 'mode-btn--active': mode === key }"
              @click="setMode(key)"
            >
              <span class="mode-name">{{ mcfg.label }}</span>
              <span class="mode-denom">{{ mcfg.sub }}</span>
            </button>
          </div>
        </div>

        <!-- Big number readout -->
        <div class="readout">
          <div class="readout-fraction">
            <span class="frac-num">{{ filledCount }}</span>
            <span class="frac-bar"></span>
            <span class="frac-den">{{ denominator }}</span>
          </div>

          <div class="readout-equals">=</div>

          <div class="readout-right">
            <div class="decimal-big">{{ decimalValue }}</div>
            <div class="percent-tag">{{ percentValue }}%</div>
          </div>
        </div>

        <!-- Simplified fraction bonus -->
        <Transition name="fade">
          <div v-if="simplified" class="simplified">
            <span class="simplified-label">also written as</span>
            <span class="simplified-frac">
              <span>{{ simplified.n }}</span>
              <span class="sfrac-bar">/</span>
              <span>{{ simplified.d }}</span>
            </span>
          </div>
        </Transition>

        <!-- Type a number -->
        <div class="input-section">
          <p class="section-label">Type a decimal or fraction:</p>
          <div class="input-row">
            <input
              v-model="inputText"
              @keydown="handleInputKey"
              @blur="applyInput"
              placeholder="e.g. 0.5 or 1/4"
              class="number-input"
              type="text"
              inputmode="decimal"
              autocomplete="off"
            />
            <button class="show-btn" @click="applyInput">Show it!</button>
          </div>
        </div>

        <!-- Quick fills -->
        <div class="quick-fills">
          <p class="section-label">Quick fills:</p>
          <div class="quick-buttons">
            <button class="quick-btn" @click="fillHalf">½</button>
            <button class="quick-btn" @click="fillQuarter">¼</button>
            <button class="quick-btn reset" @click="reset">Clear</button>
          </div>
        </div>

      </aside>
    </div>
    </template>
  </div>
</template>

<style>
/* ── Reset & base ─────────────────────────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --cream:      #FAF5FF;
  --cream-dark: #EFE6FF;
  --tan:        #DDD0F5;
  --cell-empty: #E6DAFF;
  --cell-filled:#7C3AED;
  --cell-filled-deep: #5B21B6;
  --blue:       #7C3AED;
  --blue-light: #F3EEFF;
  --navy:       #2E1065;
  --text:       #2E1065;
  --text-muted: #7C5EAD;
  --yellow:     #E879F9;
  --green:      #A855F7;
  --white:      #FFFFFF;
  --radius:     12px;
  --shadow:     0 4px 20px rgba(46,16,101,0.12);
}

html, body {
  height: 100%;
  background: var(--cream);
  color: var(--text);
  font-family: 'Nunito', sans-serif;
  -webkit-font-smoothing: antialiased;
}

#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* ── App shell ────────────────────────────────────────────────────────────── */
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background:
    radial-gradient(ellipse at 10% 0%, rgba(124,58,237,0.1) 0%, transparent 50%),
    radial-gradient(ellipse at 90% 100%, rgba(168,85,247,0.08) 0%, transparent 50%),
    var(--cream);
}

/* ── Header ───────────────────────────────────────────────────────────────── */
.header {
  padding: 20px 32px 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 2px solid var(--tan);
  flex-wrap: wrap;
  gap: 12px;
}

/* ── Tabs ─────────────────────────────────────────────────────────────────── */
.tabs {
  display: flex;
  gap: 6px;
  background: var(--cream-dark);
  padding: 4px;
  border-radius: 12px;
}

.tab {
  padding: 8px 20px;
  background: transparent;
  border: none;
  border-radius: 9px;
  font-family: 'Nunito', sans-serif;
  font-size: 14px;
  font-weight: 800;
  color: var(--text-muted);
  cursor: pointer;
  transition: all 0.15s ease;
}

.tab:hover { color: var(--navy); }

.tab--active {
  background: var(--white);
  color: var(--navy);
  box-shadow: 0 2px 8px rgba(46,16,101,0.12);
}

.logo {
  display: flex;
  align-items: center;
  gap: 14px;
}

.logo-icon {
  font-size: 36px;
  line-height: 1;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.15));
}

.logo h1 {
  font-family: 'Fredoka One', cursive;
  font-size: 28px;
  color: var(--navy);
  letter-spacing: 0.5px;
  line-height: 1;
}

.tagline {
  font-size: 13px;
  color: var(--text-muted);
  font-weight: 600;
  margin-top: 2px;
  letter-spacing: 1px;
  text-transform: uppercase;
}

/* ── Layout ───────────────────────────────────────────────────────────────── */
.layout {
  flex: 1;
  display: flex;
  gap: 0;
  padding: 32px;
  align-items: flex-start;
  max-width: 1100px;
  margin: 0 auto;
  width: 100%;
}

/* ── Grid section ─────────────────────────────────────────────────────────── */
.grid-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.grid-label {
  display: flex;
  align-items: center;
  gap: 8px;
}

.whole-label {
  font-size: 13px;
  font-weight: 700;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 1.5px;
  padding: 4px 12px;
  background: var(--tan);
  border-radius: 20px;
}

.grid-outer {
  width: 420px;
  height: 420px;
  border-radius: 4px;
  overflow: hidden;
  box-shadow:
    0 0 0 3px var(--navy),
    0 8px 32px rgba(46,16,101,0.2);
  background: var(--cell-empty);
  position: relative;
}

/* ── The grid ─────────────────────────────────────────────────────────────── */
.grid {
  width: 100%;
  height: 100%;
  display: grid;
  cursor: crosshair;
  user-select: none;
}

.grid--tenths {
  gap: 0;
  padding: 0;
  border: 2px solid var(--navy);
}

.grid--hundredths {
  gap: 0;
  padding: 0;
  border: 2px solid var(--navy);
}

.grid--thousandths {
  gap: 0;
  padding: 0;
  border: 2px solid var(--navy);
}

/* ── Cells ────────────────────────────────────────────────────────────────── */
.cell {
  background: var(--cell-empty);
  border-radius: 0;
  border-right: 1px solid rgba(46,16,101,0.25);
  border-bottom: 1px solid rgba(46,16,101,0.25);
  transition: background-color 0.12s ease, box-shadow 0.12s ease;
  position: relative;
  overflow: hidden;
}

.grid--thousandths .cell {
  border-right: 1px solid rgba(46,16,101,0.15);
  border-bottom: 1px solid rgba(46,16,101,0.15);
}

.cell::after {
  content: '';
  position: absolute;
  inset: 0;
  background: linear-gradient(135deg, rgba(255,255,255,0.2) 0%, transparent 60%);
  opacity: 0;
  transition: opacity 0.12s ease;
  border-radius: inherit;
}

.cell--filled {
  background: var(--cell-filled);
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.15), inset 0 -1px 2px rgba(255,255,255,0.1);
}

.cell--filled::after {
  opacity: 1;
}

.cell:hover {
  filter: brightness(1.08);
}

/* ── Grid transition ──────────────────────────────────────────────────────── */
.grid-swap-enter-active {
  transition: opacity 0.25s ease, transform 0.25s ease;
}
.grid-swap-leave-active {
  transition: opacity 0.15s ease, transform 0.15s ease;
}
.grid-swap-enter-from {
  opacity: 0;
  transform: scale(0.94);
}
.grid-swap-leave-to {
  opacity: 0;
  transform: scale(1.04);
}

.grid-hint {
  font-size: 14px;
  color: var(--text-muted);
  font-weight: 600;
  text-align: center;
}

.grid-hint strong {
  color: var(--cell-filled);
}

/* ── Panel ────────────────────────────────────────────────────────────────── */
.panel {
  width: 320px;
  flex-shrink: 0;
  margin-left: 40px;
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.section-label {
  font-size: 11px;
  font-weight: 800;
  text-transform: uppercase;
  letter-spacing: 1.5px;
  color: var(--text-muted);
  margin-bottom: 8px;
}

/* ── Mode switcher ────────────────────────────────────────────────────────── */
.mode-buttons {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.mode-btn {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: var(--white);
  border: 2px solid var(--tan);
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: 'Nunito', sans-serif;
}

.mode-btn:hover {
  border-color: var(--cell-filled);
  background: #FFF5F0;
  transform: translateX(2px);
}

.mode-btn--active {
  background: var(--navy);
  border-color: var(--navy);
  color: var(--white);
  transform: translateX(4px);
  box-shadow: 4px 0 0 var(--cell-filled);
}

.mode-name {
  font-weight: 800;
  font-size: 15px;
}

.mode-denom {
  font-size: 12px;
  opacity: 0.6;
  font-weight: 600;
}

/* ── Readout ──────────────────────────────────────────────────────────────── */
.readout {
  background: var(--navy);
  border-radius: var(--radius);
  padding: 20px 24px;
  display: flex;
  align-items: center;
  gap: 16px;
  box-shadow: var(--shadow);
}

.readout-fraction {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  flex-shrink: 0;
}

.frac-num, .frac-den {
  font-family: 'Fredoka One', cursive;
  font-size: 32px;
  color: var(--white);
  line-height: 1;
}

.frac-bar {
  width: 40px;
  height: 3px;
  background: var(--yellow);
  border-radius: 2px;
  display: block;
}

.readout-equals {
  font-family: 'Fredoka One', cursive;
  font-size: 28px;
  color: var(--yellow);
  flex-shrink: 0;
}

.readout-right {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.decimal-big {
  font-family: 'Fredoka One', cursive;
  font-size: 42px;
  color: var(--yellow);
  line-height: 1;
  letter-spacing: 1px;
}

.percent-tag {
  font-size: 14px;
  font-weight: 700;
  color: rgba(255,255,255,0.5);
  background: rgba(255,255,255,0.1);
  padding: 2px 10px;
  border-radius: 20px;
}

/* ── Simplified fraction ──────────────────────────────────────────────────── */
.simplified {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  background: #F3EEFF;
  border: 2px solid var(--green);
  border-radius: 10px;
}

.simplified-label {
  font-size: 12px;
  font-weight: 700;
  color: var(--green);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.simplified-frac {
  font-family: 'Fredoka One', cursive;
  font-size: 22px;
  color: var(--navy);
  display: flex;
  align-items: center;
  gap: 2px;
}

.sfrac-bar {
  color: var(--green);
  font-size: 26px;
  line-height: 1;
}

/* ── Input section ────────────────────────────────────────────────────────── */
.input-row {
  display: flex;
  gap: 8px;
}

.number-input {
  flex: 1;
  padding: 10px 14px;
  font-family: 'Nunito', sans-serif;
  font-size: 16px;
  font-weight: 700;
  border: 2px solid var(--tan);
  border-radius: 10px;
  background: var(--white);
  color: var(--text);
  outline: none;
  transition: border-color 0.15s;
}

.number-input:focus {
  border-color: var(--blue);
  box-shadow: 0 0 0 3px rgba(26,63,255,0.1);
}

.number-input::placeholder {
  color: var(--tan);
  font-weight: 600;
}

.show-btn {
  padding: 10px 18px;
  background: var(--blue);
  color: var(--white);
  font-family: 'Nunito', sans-serif;
  font-size: 14px;
  font-weight: 800;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.show-btn:hover {
  background: #0A2FEF;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(26,63,255,0.3);
}

.show-btn:active {
  transform: translateY(0);
}

/* ── Quick fills ──────────────────────────────────────────────────────────── */
.quick-buttons {
  display: flex;
  gap: 8px;
}

.quick-btn {
  flex: 1;
  padding: 10px 8px;
  background: var(--white);
  border: 2px solid var(--tan);
  border-radius: 10px;
  font-family: 'Fredoka One', cursive;
  font-size: 20px;
  color: var(--navy);
  cursor: pointer;
  transition: all 0.15s ease;
}

.quick-btn:hover {
  background: var(--cell-filled);
  border-color: var(--cell-filled);
  color: var(--white);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(255,107,53,0.3);
}

.quick-btn.reset {
  font-family: 'Nunito', sans-serif;
  font-size: 14px;
  font-weight: 800;
  color: var(--text-muted);
}

.quick-btn.reset:hover {
  background: var(--navy);
  border-color: var(--navy);
  color: var(--white);
  box-shadow: 0 4px 12px rgba(20,33,61,0.2);
}

/* ── Fade transition ──────────────────────────────────────────────────────── */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
  transform: translateY(-4px);
}

/* ── Responsive ───────────────────────────────────────────────────────────── */
@media (max-width: 860px) {
  .layout {
    flex-direction: column;
    align-items: center;
    padding: 20px 16px;
  }
  .panel {
    width: 100%;
    max-width: 420px;
    margin-left: 0;
  }
  .grid-outer {
    width: 360px;
    height: 360px;
  }
}
</style>
