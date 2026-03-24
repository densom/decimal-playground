<script setup>
import { ref, computed, h, defineComponent } from 'vue'

// ── Math helpers ──────────────────────────────────────────────────────────────
function gcd(a, b) { return b === 0 ? a : gcd(b, a % b) }
function lcm(a, b) { return (a * b) / gcd(a, b) }

function bestGrid(n) {
  if (n <= 1) return { rows: 1, cols: Math.max(1, n) }
  let best = { rows: 1, cols: n }
  let minDiff = n - 1
  for (let i = 2; i <= Math.sqrt(n); i++) {
    if (n % i === 0) {
      const diff = (n / i) - i
      if (diff < minDiff) { minDiff = diff; best = { rows: i, cols: n / i } }
    }
  }
  return best
}

// ── Inline FracSquare component ───────────────────────────────────────────────
// Renders a fraction as a grid of filled/empty cells.
const FracSquare = defineComponent({
  props: {
    n:     { type: Number, default: 0 },
    d:     { type: Number, default: 1 },
    color: { type: String, default: '#7C3AED' },
    size:  { type: Number, default: 140 },
  },
  setup(props) {
    return () => {
      const { rows, cols } = bestGrid(props.d)
      const cells = Array.from({ length: props.d }, (_, i) =>
        h('div', {
          class: 'fsq-cell',
          style: {
            background: i < props.n ? props.color : '#E6DAFF',
            borderRight: '1px solid rgba(46,16,101,0.2)',
            borderBottom: '1px solid rgba(46,16,101,0.2)',
          },
        })
      )
      return h('div', {
        style: {
          display: 'grid',
          width: '100%',
          aspectRatio: '1',
          gridTemplateColumns: `repeat(${cols}, 1fr)`,
          gridTemplateRows: `repeat(${rows}, 1fr)`,
          border: '2px solid #2E1065',
          borderRadius: '4px',
          overflow: 'hidden',
        },
      }, cells)
    }
  },
})

// ── State ─────────────────────────────────────────────────────────────────────
const inputA = ref({ n: '1', d: '3' })
const inputB = ref({ n: '1', d: '4' })
const result = ref(null)
const animKey = ref(0)

const fracA = computed(() => ({
  n: Math.max(0, parseInt(inputA.value.n) || 0),
  d: Math.max(1, parseInt(inputA.value.d) || 1),
}))
const fracB = computed(() => ({
  n: Math.max(0, parseInt(inputB.value.n) || 0),
  d: Math.max(1, parseInt(inputB.value.d) || 1),
}))

const validA = computed(() => fracA.value.n <= fracA.value.d)
const validB = computed(() => fracB.value.n <= fracB.value.d)
const canFind = computed(() => validA.value && validB.value)

const decA = computed(() => fracA.value.d > 0
  ? (fracA.value.n / fracA.value.d).toFixed(4).replace(/\.?0+$/, '')
  : '—'
)
const decB = computed(() => fracB.value.d > 0
  ? (fracB.value.n / fracB.value.d).toFixed(4).replace(/\.?0+$/, '')
  : '—'
)

// ── Find LCD ──────────────────────────────────────────────────────────────────
function findLcd() {
  if (!canFind.value) return
  const { n: na, d: da } = fracA.value
  const { n: nb, d: db } = fracB.value

  if (da === db) {
    result.value = {
      lcd: da, same: true,
      a: { orig: { n: na, d: da }, equiv: { n: na, d: da }, mult: 1 },
      b: { orig: { n: nb, d: db }, equiv: { n: nb, d: db }, mult: 1 },
    }
  } else {
    const l = lcm(da, db)
    const multA = l / da
    const multB = l / db
    result.value = {
      lcd: l, same: false,
      a: { orig: { n: na, d: da }, equiv: { n: na * multA, d: l }, mult: multA },
      b: { orig: { n: nb, d: db }, equiv: { n: nb * multB, d: l }, mult: multB },
    }
  }
  animKey.value++
}

function reset() {
  result.value = null
  inputA.value = { n: '1', d: '3' }
  inputB.value = { n: '1', d: '4' }
}

const PURPLE_A = '#7C3AED'
const PURPLE_B = '#DB2777'
</script>

<template>
  <div class="lcd">

    <!-- ── Intro ────────────────────────────────────────────────── -->
    <div class="lcd-intro">
      <h2>LCD Explorer</h2>
      <p>Enter two fractions. We'll find the <strong>Lowest Common Denominator</strong> — the smallest number both denominators divide into evenly — so you can compare or add them!</p>
    </div>

    <!-- ── Fraction inputs ──────────────────────────────────────── -->
    <div class="fractions-row">

      <div class="frac-card frac-card--a">
        <div class="frac-card-label">Fraction A</div>
        <FracSquare :n="fracA.n" :d="fracA.d" :color="PURPLE_A" :size="120" :key="`a-${fracA.n}-${fracA.d}`" />
        <div class="frac-input">
          <input v-model="inputA.n" type="number" min="0" :max="fracA.d" class="num-input" />
          <div class="input-bar"></div>
          <input v-model="inputA.d" type="number" min="1" max="20" class="num-input" />
        </div>
        <div class="frac-decimal">= {{ decA }}</div>
      </div>

      <div class="center-op">
        <div class="op-symbol">?</div>
        <div class="op-hint">which is<br>bigger?</div>
      </div>

      <div class="frac-card frac-card--b">
        <div class="frac-card-label">Fraction B</div>
        <FracSquare :n="fracB.n" :d="fracB.d" :color="PURPLE_B" :size="120" :key="`b-${fracB.n}-${fracB.d}`" />
        <div class="frac-input">
          <input v-model="inputB.n" type="number" min="0" :max="fracB.d" class="num-input" />
          <div class="input-bar"></div>
          <input v-model="inputB.d" type="number" min="1" max="20" class="num-input" />
        </div>
        <div class="frac-decimal">= {{ decB }}</div>
      </div>

    </div>

    <!-- ── Find button ───────────────────────────────────────────── -->
    <div class="find-row">
      <button class="find-btn" :disabled="!canFind" @click="findLcd">
        Find the LCD!
      </button>
    </div>

    <!-- ── Results ──────────────────────────────────────────────── -->
    <Transition name="result-slide">
      <div v-if="result" class="results" :key="animKey">

        <!-- Same denominator -->
        <div v-if="result.same" class="same-denom">
          <span class="same-icon">🎉</span>
          <div>
            <strong>They already have the same denominator!</strong>
            <p>Both use {{ result.lcd }} — you can compare them directly.</p>
          </div>
        </div>

        <!-- LCD badge -->
        <template v-else>
          <div class="lcd-badge">
            <div class="lcd-badge-label">Lowest Common Denominator</div>
            <div class="lcd-badge-number">{{ result.lcd }}</div>
            <div class="lcd-badge-sub">
              {{ result.a.orig.d }} × {{ result.a.mult }} = {{ result.lcd }}
              &nbsp;·&nbsp;
              {{ result.b.orig.d }} × {{ result.b.mult }} = {{ result.lcd }}
            </div>
          </div>

          <!-- Equivalent squares side by side -->
          <div class="equiv-row">

            <div class="equiv-card">
              <div class="equiv-transform">
                <span class="orig-frac">{{ result.a.orig.n }}/{{ result.a.orig.d }}</span>
                <span class="arrow">→</span>
                <span class="equiv-frac">{{ result.a.equiv.n }}/{{ result.a.equiv.d }}</span>
              </div>
              <div class="equiv-explain">× <strong>{{ result.a.mult }}</strong> on top and bottom</div>
              <FracSquare :n="result.a.equiv.n" :d="result.a.equiv.d" :color="PURPLE_A" :size="200" />
            </div>

            <div class="comparison-col">
              <div class="comparison-sign">
                {{ result.a.equiv.n > result.b.equiv.n ? '>' : result.a.equiv.n < result.b.equiv.n ? '<' : '=' }}
              </div>
              <div class="comparison-hint">
                {{ result.a.equiv.n }}/{{ result.lcd }}
                {{ result.a.equiv.n > result.b.equiv.n ? '>' : result.a.equiv.n < result.b.equiv.n ? '<' : '=' }}
                {{ result.b.equiv.n }}/{{ result.lcd }}
              </div>
            </div>

            <div class="equiv-card">
              <div class="equiv-transform">
                <span class="orig-frac">{{ result.b.orig.n }}/{{ result.b.orig.d }}</span>
                <span class="arrow">→</span>
                <span class="equiv-frac">{{ result.b.equiv.n }}/{{ result.b.equiv.d }}</span>
              </div>
              <div class="equiv-explain">× <strong>{{ result.b.mult }}</strong> on top and bottom</div>
              <FracSquare :n="result.b.equiv.n" :d="result.b.equiv.d" :color="PURPLE_B" :size="200" />
            </div>

          </div>

          <!-- Addition bonus -->
          <div class="addition-bonus">
            <span class="bonus-label">✨ Add them!</span>
            <span class="bonus-math">
              {{ result.a.equiv.n }}/{{ result.lcd }} + {{ result.b.equiv.n }}/{{ result.lcd }}
              = {{ result.a.equiv.n + result.b.equiv.n }}/{{ result.lcd }}
            </span>
          </div>
        </template>

        <button class="reset-link" @click="reset">↩ Try different fractions</button>
      </div>
    </Transition>

  </div>
</template>

<style scoped>
.lcd {
  max-width: 860px;
  margin: 0 auto;
  padding: 32px;
  display: flex;
  flex-direction: column;
  gap: 28px;
}

/* Intro */
.lcd-intro { text-align: center; }
.lcd-intro h2 {
  font-family: 'Fredoka One', cursive;
  font-size: 32px;
  color: #2E1065;
  margin-bottom: 8px;
}
.lcd-intro p { font-size: 15px; color: #7C5EAD; max-width: 560px; margin: 0 auto; line-height: 1.6; }
.lcd-intro strong { color: #7C3AED; }

/* Fraction cards */
.fractions-row { display: flex; align-items: center; justify-content: center; gap: 16px; flex-wrap: wrap; }

.frac-card {
  background: white;
  border-radius: 16px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  width: clamp(140px, 28vw, 200px);
  box-shadow: 0 4px 20px rgba(46,16,101,0.1);
}
.frac-card--a { border-top: 4px solid #7C3AED; }
.frac-card--b { border-top: 4px solid #DB2777; }

.frac-card-label {
  font-size: 11px; font-weight: 800; text-transform: uppercase;
  letter-spacing: 1.5px; color: #7C5EAD;
}

/* Numeric inputs */
.frac-input { display: flex; flex-direction: column; align-items: center; gap: 4px; }

.num-input {
  width: 64px; text-align: center;
  font-family: 'Fredoka One', cursive; font-size: 28px; color: #2E1065;
  border: 2px solid #DDD0F5; border-radius: 8px; padding: 4px 0;
  background: #FAF5FF; outline: none;
  -moz-appearance: textfield;
}
.num-input::-webkit-inner-spin-button,
.num-input::-webkit-outer-spin-button { -webkit-appearance: none; }
.num-input:focus { border-color: #7C3AED; }

.input-bar { width: 56px; height: 3px; background: #2E1065; border-radius: 2px; }
.frac-decimal { font-size: 13px; font-weight: 700; color: #7C5EAD; }

/* Center operator */
.center-op { display: flex; flex-direction: column; align-items: center; gap: 6px; flex-shrink: 0; }
.op-symbol { font-family: 'Fredoka One', cursive; font-size: 48px; color: #DDD0F5; }
.op-hint { font-size: 11px; font-weight: 700; color: #7C5EAD; text-align: center; text-transform: uppercase; letter-spacing: 0.5px; line-height: 1.4; }

/* Find button */
.find-row { display: flex; justify-content: center; }
.find-btn {
  padding: 14px 40px; background: #2E1065; color: white;
  font-family: 'Nunito', sans-serif; font-size: 18px; font-weight: 800;
  border: none; border-radius: 50px; cursor: pointer;
  transition: all 0.2s ease; letter-spacing: 0.5px;
  box-shadow: 0 4px 16px rgba(46,16,101,0.25);
}
.find-btn:hover:not(:disabled) {
  background: #7C3AED; transform: translateY(-2px);
  box-shadow: 0 8px 24px rgba(124,58,237,0.35);
}
.find-btn:active:not(:disabled) { transform: translateY(0); }
.find-btn:disabled { opacity: 0.4; cursor: default; }

/* Results */
.results { display: flex; flex-direction: column; gap: 24px; align-items: center; }

.same-denom {
  display: flex; align-items: center; gap: 16px;
  background: #E8FFEF; border: 2px solid #06BA7A;
  border-radius: 14px; padding: 16px 24px; max-width: 480px;
}
.same-icon { font-size: 32px; }

/* LCD badge */
.lcd-badge {
  background: #2E1065; border-radius: 16px; padding: 20px 48px;
  text-align: center; box-shadow: 0 8px 32px rgba(46,16,101,0.25);
}
.lcd-badge-label { font-size: 11px; font-weight: 800; text-transform: uppercase; letter-spacing: 2px; color: rgba(255,255,255,0.45); margin-bottom: 4px; }
.lcd-badge-number { font-family: 'Fredoka One', cursive; font-size: 72px; color: #E879F9; line-height: 1; }
.lcd-badge-sub { font-size: 13px; font-weight: 600; color: rgba(255,255,255,0.35); margin-top: 6px; }

/* Equivalent cards */
.equiv-row { display: flex; align-items: center; justify-content: center; gap: 16px; flex-wrap: wrap; }

.equiv-card {
  background: white; border-radius: 16px; padding: 20px;
  display: flex; flex-direction: column; align-items: center; gap: 10px;
  box-shadow: 0 4px 20px rgba(46,16,101,0.1);
  width: clamp(160px, 38vw, 260px);
}

.equiv-transform { display: flex; align-items: center; gap: 8px; font-family: 'Fredoka One', cursive; font-size: 22px; }
.orig-frac { color: #BBB; }
.arrow { color: #DDD; font-size: 18px; }
.equiv-frac { color: #2E1065; }

.equiv-explain { font-size: 13px; font-weight: 600; color: #7C5EAD; }
.equiv-explain strong { color: #7C3AED; }

/* Comparison */
.comparison-col { display: flex; flex-direction: column; align-items: center; gap: 6px; }
.comparison-sign { font-family: 'Fredoka One', cursive; font-size: 56px; color: #2E1065; line-height: 1; }
.comparison-hint { font-size: 13px; font-weight: 700; color: #7C5EAD; text-align: center; }

/* Addition bonus */
.addition-bonus {
  display: flex; align-items: center; gap: 12px;
  background: #F3EEFF; border: 2px solid #7C3AED;
  border-radius: 12px; padding: 12px 20px;
}
.bonus-label { font-size: 12px; font-weight: 800; text-transform: uppercase; letter-spacing: 1px; color: #7C3AED; white-space: nowrap; }
.bonus-math { font-family: 'Fredoka One', cursive; font-size: 20px; color: #2E1065; }

.reset-link {
  background: none; border: none; font-family: 'Nunito', sans-serif;
  font-size: 14px; font-weight: 700; color: #7C5EAD;
  cursor: pointer; text-decoration: underline; padding: 4px;
}
.reset-link:hover { color: #7C3AED; }

/* Transition */
.result-slide-enter-active { transition: opacity 0.4s ease, transform 0.4s ease; }
.result-slide-enter-from { opacity: 0; transform: translateY(16px); }

/* ── Responsive ──────────────────────────────────────────────────────────── */
@media (max-width: 600px) {
  .lcd { padding: 16px; gap: 20px; }
  .lcd-intro h2 { font-size: 24px; }
  .lcd-intro p { font-size: 13px; }
  .frac-card { width: clamp(120px, 36vw, 180px); padding: 14px; }
  .center-op { display: none; }
  .lcd-badge { padding: 16px 24px; }
  .lcd-badge-number { font-size: 52px; }
  .equiv-card { width: clamp(140px, 42vw, 200px); padding: 14px; }
  .addition-bonus { flex-direction: column; text-align: center; gap: 6px; }
  .find-btn { width: 100%; }
}
</style>
