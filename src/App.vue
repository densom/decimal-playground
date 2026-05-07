<script setup>
import { ref, computed, nextTick, onMounted, onBeforeUnmount, watch } from 'vue'
import LcdExplorer from './components/LcdExplorer.vue'
import { trackEvent } from './analytics.js'

const activeTab = ref('explore') // 'explore' | 'lcd'

watch(activeTab, (tab) => {
  trackEvent('tab_view', { tab_name: tab })
})

// ── State ─────────────────────────────────────────────────────────────────────
const mode = ref('hundredths') // 'fourths' | 'eighths' | 'sixteenths' | 'tenths' | 'hundredths' | 'thousandths'
const filledCells = ref(new Set())
const inputText = ref('')
const isDragging = ref(false)
const dragFill = ref(true) // true = painting, false = erasing

// ── Mode config ───────────────────────────────────────────────────────────────
const MODES = {
  fourths:     { denom: 4,    cols: 2,  rows: 2,   label: 'Fourths',     sub: '÷ 4',   places: 2 },
  eighths:     { denom: 8,    cols: 2,  rows: 4,   label: 'Eighths',     sub: '÷ 8',   places: 3 },
  sixteenths:  { denom: 16,   cols: 4,  rows: 4,   label: 'Sixteenths',  sub: '÷ 16',  places: 4 },
  tenths:      { denom: 10,   cols: 1,  rows: 10,  label: 'Tenths',      sub: '÷ 10',  places: 1 },
  hundredths:  { denom: 100,  cols: 10, rows: 10,  label: 'Hundredths',  sub: '÷ 100', places: 2 },
  thousandths: { denom: 1000, cols: 10, rows: 100, label: 'Thousandths', sub: '÷ 1000',places: 3 },
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
  trackEvent('grid_mode_changed', { mode: m, previous_mode: mode.value })
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

// ── Cluster mode ──────────────────────────────────────────────────────────────
const clusterMode = ref(false)

// Flying cell animation state
const flyAnim    = ref(null)
const isFlying   = ref(false)   // lock: ignore clicks during animation
const flyTimer   = ref(null)    // pending setTimeout handle
const gridOuterRef = ref(null)
const cellRefs = ref([])

watch(denominator, () => { cancelFly(); cellRefs.value = [] })

watch(clusterMode, (on) => {
  if (!on) return
  // Snap any scattered cells into a contiguous cluster at the start
  const count = filledCells.value.size
  const next = new Set()
  for (let i = 0; i < count; i++) next.add(i)
  filledCells.value = next
})

function setCellRef(el, i) {
  if (el) cellRefs.value[i] = el
}

function cellCenter(index) {
  const el = cellRefs.value[index]
  const grid = gridOuterRef.value
  if (!el || !grid) return null
  const gr = grid.getBoundingClientRect()
  const cr = el.getBoundingClientRect()
  return {
    x: cr.left - gr.left + cr.width / 2,
    y: cr.top  - gr.top  + cr.height / 2,
  }
}

function cancelFly() {
  if (flyTimer.value) { clearTimeout(flyTimer.value); flyTimer.value = null }
  flyAnim.value = null
  isFlying.value = false
}

async function triggerFly(fromIdx, toIdx) {
  trackEvent('fly_animation_triggered', { mode: mode.value })
  const fromEl = cellRefs.value[fromIdx]
  const toEl   = cellRefs.value[toIdx]
  const grid   = gridOuterRef.value
  if (!fromEl || !toEl || !grid) {
    // No DOM refs — swap instantly
    const next = new Set(filledCells.value)
    next.delete(fromIdx)
    next.add(toIdx)
    filledCells.value = next
    inputText.value = (next.size / denominator.value).toFixed(cfg.value.places)
    return
  }

  isFlying.value = true

  const gr = grid.getBoundingClientRect()
  const fr = fromEl.getBoundingClientRect()
  const tr = toEl.getBoundingClientRect()
  const fx = fr.left - gr.left,  fy = fr.top - gr.top
  const tx = tr.left - gr.left,  ty = tr.top - gr.top
  const w  = fr.width,            h  = fr.height

  // 1. Flash the clicked cell as filled
  const s1 = new Set(filledCells.value)
  s1.add(fromIdx)
  filledCells.value = s1

  // 2. Spawn the flying clone at the exact same position
  flyAnim.value = { x: fx, y: fy, w, h, moving: false }
  await nextTick()

  // 3. Unmark source — clone visually takes its place
  const s2 = new Set(filledCells.value)
  s2.delete(fromIdx)
  filledCells.value = s2

  // 4. Two rAF ticks so the browser paints the clone before the transition fires
  requestAnimationFrame(() => requestAnimationFrame(() => {
    if (flyAnim.value) flyAnim.value = { x: tx, y: ty, w, h, moving: true }
  }))

  // 5. Land: fill target, release lock
  flyTimer.value = setTimeout(() => {
    flyAnim.value = null
    isFlying.value = false
    flyTimer.value = null
    const s3 = new Set(filledCells.value)
    s3.add(toIdx)
    filledCells.value = s3
    inputText.value = (s3.size / denominator.value).toFixed(cfg.value.places)
  }, 480)
}

// ── Cell interaction ──────────────────────────────────────────────────────────
let dragStartCount = 0

function startDrag(i) {
  dragStartCount = filledCells.value.size
  isDragging.value = true
  const alreadyFilled = filledCells.value.has(i)
  dragFill.value = !alreadyFilled

  if (clusterMode.value) {
    isDragging.value = false
    if (isFlying.value) return   // ignore clicks mid-animation
    const clusterSize = filledCells.value.size
    if (!alreadyFilled) {
      const target = clusterSize
      if (target >= denominator.value) return
      if (i === target) {
        // Already adjacent — fill immediately, no animation
        applyCell(target)
      } else {
        triggerFly(i, target)
      }
    } else {
      // Clicking a filled cell → pop the last one off the cluster
      dragFill.value = false
      if (clusterSize > 0) {
        const next = new Set(filledCells.value)
        next.delete(clusterSize - 1)
        filledCells.value = next
        inputText.value = (next.size / denominator.value).toFixed(cfg.value.places)
      }
    }
    return
  }

  applyCell(i)
}

function onEnter(i) {
  if (isDragging.value && !clusterMode.value) applyCell(i)
}

function applyCell(i) {
  const next = new Set(filledCells.value)
  dragFill.value ? next.add(i) : next.delete(i)
  filledCells.value = next
  inputText.value = (filledCells.value.size / denominator.value).toFixed(cfg.value.places)
}

function stopDrag() {
  if (!clusterMode.value) {
    const delta = filledCells.value.size - dragStartCount
    if (delta > 0) trackEvent('grid_painted', { cells_added: delta, mode: mode.value, cluster_mode: false })
    if (delta < 0) trackEvent('grid_erased', { cells_removed: -delta, mode: mode.value })
  }
  isDragging.value = false
}

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

  trackEvent('value_submitted', { value: String(val), format: fracMatch ? 'fraction' : 'decimal', mode: mode.value })

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
  trackEvent('quick_fill_used', { fraction: '1/2', mode: mode.value })
  const half = Math.round(denominator.value / 2)
  const next = new Set()
  for (let i = 0; i < half; i++) next.add(i)
  filledCells.value = next
  inputText.value = (half / denominator.value).toFixed(cfg.value.places)
}

function fillQuarter() {
  trackEvent('quick_fill_used', { fraction: '1/4', mode: mode.value })
  const q = Math.round(denominator.value / 4)
  const next = new Set()
  for (let i = 0; i < q; i++) next.add(i)
  filledCells.value = next
  inputText.value = (q / denominator.value).toFixed(cfg.value.places)
}

function reset() {
  trackEvent('grid_cleared', { cells_cleared: filledCells.value.size, mode: mode.value })
  filledCells.value = new Set()
  inputText.value = ''
}

function toggleCluster() {
  clusterMode.value = !clusterMode.value
  trackEvent('cluster_mode_toggled', { enabled: clusterMode.value, mode: mode.value })
}

// ── Cell visual grouping ──────────────────────────────────────────────────────
// Hundredths (10×10): each row of 10 = 1 tenth → alternate tint per row
// Thousandths (10×100): rows from tenths visible, squares from hundredths visible.
//   - tenths group = every 10 rows → bold bottom border
//   - hundredths group = every 1 row → medium border (CSS), alternate tint per 10-row band
function getCellClasses(i) {
  const classes = { 'cell--filled': filledCells.value.has(i) }

  if (mode.value === 'sixteenths') {
    // Alternate tint on odd rows (each row of 4 = 1 quarter)
    if (Math.floor(i / 4) % 2 === 1) classes['cell--alt-tenth'] = true
  }

  if (mode.value === 'hundredths') {
    // Alternate tint on odd tenth-rows so the 10 strips pop visually
    if (Math.floor(i / 10) % 2 === 1) classes['cell--alt-tenth'] = true
  }

  if (mode.value === 'thousandths') {
    const row = Math.floor(i / 10)          // 0–99
    // Bold border at the bottom of each tenth group (every 10 rows)
    if (row % 10 === 9) classes['cell--tenth-bottom'] = true
    // Alternate tint on odd tenth-bands (same visual language as hundredths)
    if (Math.floor(i / 100) % 2 === 1) classes['cell--alt-tenth'] = true
  }

  return classes
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
    <div v-if="activeTab === 'lcd'" class="lcd-scroll">
      <LcdExplorer />
    </div>

    <!-- ── Main layout ────────────────────────────────────────────── -->
    <template v-if="activeTab === 'explore'">
    <div class="layout">

      <!-- ── Left: Grid ─────────────────────────────────────────── -->
      <section class="grid-section">
        <div class="grid-label">
          <span class="whole-label">1 whole</span>
        </div>

        <div class="grid-outer" ref="gridOuterRef">
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
                :ref="el => setCellRef(el, i - 1)"
                class="cell"
                :class="getCellClasses(i - 1)"
                @mousedown.prevent="startDrag(i - 1)"
                @mouseenter="onEnter(i - 1)"
              />
            </div>
          </Transition>

          <!-- Flying cell for cluster mode -->
          <div
            v-if="flyAnim"
            class="fly-cell"
            :class="{ 'fly-cell--moving': flyAnim.moving }"
            :style="{
              left:   flyAnim.x + 'px',
              top:    flyAnim.y + 'px',
              width:  flyAnim.w + 'px',
              height: flyAnim.h + 'px',
            }"
          />
        </div>

        <!-- Hint + grid controls sit below the square, matching its width -->
        <div class="grid-footer">
          <div class="grid-hint">
            <span v-if="mode === 'tenths'">Each strip = <strong>1 tenth</strong> = <strong>0.1</strong></span>
            <span v-else-if="mode === 'hundredths'">Each square = <strong>1 hundredth</strong> = <strong>0.01</strong></span>
            <span v-else>Each tiny square = <strong>1 thousandth</strong> = <strong>0.001</strong></span>
          </div>

          <!-- Controls row: type input · quick fills · cluster toggle -->
          <div class="grid-controls">

            <div class="gc-input">
              <input
                v-model="inputText"
                @keydown="handleInputKey"
                @blur="applyInput"
                placeholder="Type 0.5 or 3/4…"
                class="number-input"
                type="text"
                inputmode="decimal"
                autocomplete="off"
              />
              <button class="show-btn" @click="applyInput">Show it!</button>
            </div>

            <div class="gc-divider" />

            <div class="gc-fills">
              <button class="quick-btn" @click="fillHalf">½</button>
              <button class="quick-btn" @click="fillQuarter">¼</button>
              <button class="quick-btn reset" @click="reset">Clear</button>
            </div>

            <div class="gc-divider" />

            <div class="cluster-toggle" @click="toggleCluster" :class="{ 'cluster-toggle--on': clusterMode }">
              <div class="cluster-toggle-text">
                <span class="cluster-toggle-label">Keep together</span>
                <span class="cluster-toggle-sub">{{ clusterMode ? 'On' : 'Off' }}</span>
              </div>
              <div class="toggle-pill">
                <div class="toggle-knob" />
              </div>
            </div>

          </div>
        </div>
      </section>

      <!-- ── Right: "What is this number?" panel ────────────────── -->
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
  overflow: hidden;
  background: var(--cream);
  color: var(--text);
  font-family: 'Nunito', sans-serif;
  -webkit-font-smoothing: antialiased;
}

#app {
  height: 100vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

/* ── App shell ────────────────────────────────────────────────────────────── */
.app {
  height: 100vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  background:
    radial-gradient(ellipse at 10% 0%, rgba(124,58,237,0.1) 0%, transparent 50%),
    radial-gradient(ellipse at 90% 100%, rgba(168,85,247,0.08) 0%, transparent 50%),
    var(--cream);
}

/* ── Header ───────────────────────────────────────────────────────────────── */
.header {
  padding: 10px 28px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 2px solid var(--tan);
  flex-wrap: wrap;
  gap: 8px;
  flex-shrink: 0;
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
  font-size: 28px;
  line-height: 1;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.15));
}

.logo h1 {
  font-family: 'Fredoka One', cursive;
  font-size: 22px;
  color: var(--navy);
  letter-spacing: 0.5px;
  line-height: 1;
}

.tagline {
  font-size: 11px;
  color: var(--text-muted);
  font-weight: 600;
  margin-top: 2px;
  letter-spacing: 1px;
  text-transform: uppercase;
}

/* ── LCD scroll wrapper ───────────────────────────────────────────────────── */
.lcd-scroll {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
}

/* ── Layout ───────────────────────────────────────────────────────────────── */
.layout {
  flex: 1;
  min-height: 0;
  display: flex;
  padding: 16px 24px 20px;
  gap: 0;
  width: 100%;
  overflow: hidden;
}

/* ── Grid section ─────────────────────────────────────────────────────────── */
.grid-section {
  /* Define --grid-size here so both grid-outer and grid-footer can share it */
  --panel-w: clamp(300px, 27vw, 380px);
  --grid-size: min(calc(100vh - 320px), calc(100vw - var(--panel-w) - 80px), 820px);
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding-top: 8px;
  gap: 10px;
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
  width: var(--grid-size);
  aspect-ratio: 1;
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

.grid--fourths {
  gap: 4px;
  padding: 4px;
  border: 2px solid var(--navy);
  background: var(--navy);
}

.grid--eighths {
  gap: 0;
  padding: 0;
  border: 2px solid var(--navy);
}

.grid--sixteenths {
  gap: 0;
  padding: 0;
  border: 2px solid var(--navy);
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

/* ── Grouping indicators ─────────────────────────────────────────────────── */

/* Fourths: 2×2 — each cell is one quarter, use rounded corners for a "tile" feel */
.grid--fourths .cell {
  border-radius: 4px;
  border: none;
}

/* Eighths: 2×4 — each row of 2 cells = 1 quarter → bold border after each row */
.grid--eighths .cell {
  border-bottom: 3px solid rgba(46, 16, 101, 0.5);
  border-right: 1px solid rgba(46, 16, 101, 0.25);
}

/* Sixteenths: 4×4 — each row of 4 cells = 1 quarter → bold border after each row */
.grid--sixteenths .cell {
  border-bottom: 3px solid rgba(46, 16, 101, 0.5);
  border-right: 1px solid rgba(46, 16, 101, 0.25);
}

/* Hundredths: each row of 10 cells = 1 tenth → bold bottom border on every row */
.grid--hundredths .cell {
  border-bottom: 3px solid rgba(46, 16, 101, 0.5);
}

/* Thousandths: each row = 1 hundredth → light bottom border on every row */
.grid--thousandths .cell {
  border-right:  1px solid rgba(46, 16, 101, 0.12);
  border-bottom: 1px solid rgba(46, 16, 101, 0.22);
}

/* Thousandths: bold border at the bottom of every tenth-group (every 10 rows) */
.cell--tenth-bottom {
  border-bottom: 3px solid rgba(46, 16, 101, 0.55) !important;
}

/* Alternating tenth-band tint (both hundredths and thousandths modes) */
.cell--alt-tenth:not(.cell--filled) {
  background: rgba(109, 40, 217, 0.1);
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

.grid-footer {
  width: var(--grid-size);
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.grid-hint {
  font-size: clamp(13px, 1.4vw, 18px);
  color: var(--text-muted);
  font-weight: 600;
  text-align: center;
}

.grid-hint strong {
  color: var(--cell-filled);
}

/* ── Grid controls (below the square) ────────────────────────────────────── */
.grid-controls {
  display: flex;
  align-items: center;
  gap: 10px;
  background: var(--white);
  border: 2px solid var(--tan);
  border-radius: 14px;
  padding: 10px 14px;
}

.gc-input {
  display: flex;
  gap: 6px;
  flex: 1;
  min-width: 0;
}

.gc-divider {
  width: 1px;
  height: 32px;
  background: var(--tan);
  flex-shrink: 0;
}

.gc-fills {
  display: flex;
  gap: 6px;
  flex-shrink: 0;
}

/* ── Panel ────────────────────────────────────────────────────────────────── */
.panel {
  width: clamp(280px, 27vw, 380px);
  flex-shrink: 0;
  margin-left: clamp(20px, 2.5vw, 44px);
  display: flex;
  flex-direction: column;
  gap: clamp(16px, 2vh, 28px);
  overflow-y: auto;
  max-height: 100%;
  padding-right: 4px;
  justify-content: flex-start;
  padding-top: 8px;
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
  font-size: clamp(14px, 1.4vw, 18px);
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
  font-size: clamp(28px, 3.5vw, 52px);
  color: var(--white);
  line-height: 1;
}

.frac-bar {
  width: clamp(36px, 4vw, 56px);
  height: 3px;
  background: var(--yellow);
  border-radius: 2px;
  display: block;
}

.readout-equals {
  font-family: 'Fredoka One', cursive;
  font-size: clamp(24px, 3vw, 44px);
  color: var(--yellow);
  flex-shrink: 0;
}

.readout-right {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.decimal-big {
  font-family: 'Fredoka One', cursive;
  font-size: clamp(24px, 3vw, 48px);
  color: var(--yellow);
  line-height: 1;
  letter-spacing: 1px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

.percent-tag {
  font-size: clamp(13px, 1.4vw, 20px);
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
  font-size: clamp(20px, 2.2vw, 30px);
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
  min-width: 80px;
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
  flex-shrink: 0;
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

/* ── Flying cell ─────────────────────────────────────────────────────────── */
.fly-cell {
  position: absolute;
  background: var(--cell-filled);
  border-radius: 3px;
  pointer-events: none;
  z-index: 20;
  box-shadow:
    0 0 0 2px rgba(124,58,237,0.5),
    0 4px 16px rgba(124,58,237,0.5);
}

.fly-cell--moving {
  transition:
    left   0.42s cubic-bezier(0.34, 1.56, 0.64, 1),
    top    0.42s cubic-bezier(0.34, 1.56, 0.64, 1);
  animation: cell-fly 0.42s ease-out forwards;
}

@keyframes cell-fly {
  0%   { box-shadow: 0 0 0 2px rgba(124,58,237,0.5), 0 4px 16px rgba(124,58,237,0.5); transform: scale(1.1); }
  50%  { box-shadow: 0 0 0 4px rgba(124,58,237,0.3), 0 8px 28px rgba(124,58,237,0.4); transform: scale(1.15); }
  100% { box-shadow: 0 0 0 2px rgba(124,58,237,0.5), 0 4px 16px rgba(124,58,237,0.5); transform: scale(1); }
}

/* ── Cluster toggle ───────────────────────────────────────────────────────── */
.cluster-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 2px 4px;
  border-radius: 10px;
  cursor: pointer;
  transition: background 0.2s ease;
  user-select: none;
  flex-shrink: 0;
}

.cluster-toggle:hover {
  background: var(--cream-dark);
}

.cluster-toggle--on {
  background: var(--blue-light);
}

.cluster-toggle-text {
  display: flex;
  flex-direction: column;
  gap: 1px;
}

.cluster-toggle-label {
  font-size: clamp(11px, 1.1vw, 13px);
  font-weight: 800;
  color: var(--navy);
  white-space: nowrap;
}

.cluster-toggle-sub {
  font-size: 10px;
  font-weight: 600;
  color: var(--text-muted);
}

.toggle-pill {
  width: 44px;
  height: 24px;
  background: var(--tan);
  border-radius: 12px;
  flex-shrink: 0;
  position: relative;
  transition: background 0.2s ease;
}

.cluster-toggle--on .toggle-pill {
  background: var(--cell-filled);
}

.toggle-knob {
  position: absolute;
  top: 3px;
  left: 3px;
  width: 18px;
  height: 18px;
  background: white;
  border-radius: 50%;
  transition: transform 0.2s cubic-bezier(0.34, 1.56, 0.64, 1);
  box-shadow: 0 1px 4px rgba(0,0,0,0.2);
}

.cluster-toggle--on .toggle-knob {
  transform: translateX(20px);
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
  html, body, #app, .app { overflow: auto; height: auto; }
  .layout {
    flex-direction: column;
    align-items: center;
    padding: 16px;
    overflow: visible;
  }
  .panel {
    width: 100%;
    max-width: 520px;
    margin-left: 0;
    overflow-y: visible;
    max-height: none;
    justify-content: flex-start;
  }
  .grid-section {
    --grid-size: min(90vw, 520px) !important;
  }
  .grid-controls {
    flex-wrap: wrap;
  }
  .gc-divider {
    display: none;
  }
}
</style>
