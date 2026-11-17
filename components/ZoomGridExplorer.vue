<script setup>
// Toy picture of why zooms need a second cell grid. A (1 Gpc)^3 periodic box
// is split into bkg^3 top-level cells. The high-res region is a sphere at the
// box centre holding N_HR particles; the rest of the box holds N_BG low-res
// particles spread uniformly. Work per cell ~ particles per cell.
//   step 0: background grid only. The cells overlapping the sphere carry all
//           of N_HR between them.
//   step 1: the background cells covering the sphere become the "void" and
//           each is split `depth` times (2^depth zoom cells per side), so
//           N_HR is shared over many more, smaller cells.
// Load counts are 3D (cells whose cube intersects the sphere); the panels are
// mid-plane slices.
import { computed, ref } from 'vue'

const props = defineProps({ step: { type: Number, default: 1 } })

const L = 1000 // box side [Mpc]
const N_HR = 512 ** 3
const N_BG = 512 ** 3

const bkg = ref(15)
const depth = ref(2)
const logD = ref(Math.log10(30)) // region diameter [Mpc]
const D = computed(() => 10 ** logD.value)
const R = computed(() => D.value / 2)
const C = L / 2

const w = computed(() => L / bkg.value)
const voidLo = computed(() => Math.max(0, Math.floor((C - R.value) / w.value)))
const voidHi = computed(() => Math.min(bkg.value - 1, Math.floor((C + R.value) / w.value - 1e-9)))
const nVoid = computed(() => voidHi.value - voidLo.value + 1)
const x0 = computed(() => voidLo.value * w.value) // void lower edge
const V = computed(() => nVoid.value * w.value) // void side
const zoom = computed(() => nVoid.value * 2 ** depth.value) // zoom cells per void side
const zw = computed(() => w.value / 2 ** depth.value) // zoom cell side

// does the cube [lo, lo + s]^k intersect the sphere? (k = 2 for drawing, 3 for counting)
const d2 = (lo, s) => {
  const c = Math.min(Math.max(C, lo), lo + s)
  return (c - C) ** 2
}
const hits = (lo, s, k) => {
  let t = 0
  for (let i = 0; i < k; i++) t += d2(lo[i], s)
  return t < R.value ** 2
}
// number of cells of side s, starting at x0, n per side, that intersect the sphere (3D)
const count3 = (o, s, n) => {
  const ax = []
  for (let i = 0; i < n; i++) ax.push(d2(o + i * s, s))
  let c = 0
  const r2 = R.value ** 2
  for (const a of ax) for (const b of ax) { if (a + b >= r2) continue; for (const e of ax) if (a + b + e < r2) c++ }
  return Math.max(c, 1)
}

const bgPerCell = computed(() => N_BG / bkg.value ** 3)
const hotBkg = computed(() => N_HR / count3(x0.value, w.value, nVoid.value) + bgPerCell.value)
const hotZoom = computed(() => N_HR / count3(x0.value, zw.value, zoom.value))
const hot = computed(() => (props.step >= 1 ? hotZoom.value : hotBkg.value))
const ratio = computed(() => hot.value / bgPerCell.value)
const ratioCls = computed(() => (ratio.value > 4 ? 'bad' : ratio.value < 0.25 ? 'theory' : 'accent'))

// colour a loaded cell by how far its load is from a background cell's
const heat = (load) => {
  const r = Math.log10(load / bgPerCell.value)
  if (Math.abs(r) < Math.log10(2)) return { fill: 'var(--accent)', 'fill-opacity': 0.25 }
  return { fill: r > 0 ? 'var(--warn)' : 'var(--theory)', 'fill-opacity': Math.min(0.85, 0.2 + Math.abs(r) / 3) }
}

// panels
const S = 240
const fsc = S / L
const bkgLines = computed(() => Array.from({ length: bkg.value - 1 }, (_, i) => (i + 1) * w.value))
// inset: the void plus one background cell either side (clamped to the box)
const inLo = computed(() => Math.max(0, x0.value - w.value))
const inHi = computed(() => Math.min(L, x0.value + V.value + w.value))
const isc = computed(() => S / (inHi.value - inLo.value))
const ix = (x) => (x - inLo.value) * isc.value

const hotBkgCells = computed(() => {
  const out = []
  for (let i = voidLo.value; i <= voidHi.value; i++)
    for (let j = voidLo.value; j <= voidHi.value; j++)
      if (hits([i * w.value, j * w.value], w.value, 2)) out.push([i * w.value, j * w.value])
  return out
})
const zoomCells = computed(() => {
  const out = []
  for (let i = 0; i < zoom.value; i++)
    for (let j = 0; j < zoom.value; j++) {
      const lo = [x0.value + i * zw.value, x0.value + j * zw.value]
      if (hits(lo, zw.value, 2)) out.push(lo)
    }
  return out
})
const zoomLines = computed(() => Array.from({ length: zoom.value - 1 }, (_, i) => x0.value + (i + 1) * zw.value))
const cells = computed(() => (props.step >= 1 ? zoomCells.value : hotBkgCells.value))
const cellW = computed(() => (props.step >= 1 ? zw.value : w.value))
const cellStyle = computed(() => heat(hot.value))

// particles: a fixed scatter in the unit disc (high-res) and the box (low-res)
let seed = 7
const rnd = () => ((seed = (seed * 16807) % 2147483647) / 2147483647)
const disc = Array.from({ length: 500 }, () => {
  const r = Math.sqrt(rnd())
  const t = 2 * Math.PI * rnd()
  return [r * Math.cos(t), r * Math.sin(t)]
})
const field = Array.from({ length: 400 }, () => [rnd() * L, rnd() * L])
const fieldIn = computed(() => field.filter(([x, y]) => x > inLo.value && x < inHi.value && y > inLo.value && y < inHi.value))

const fmtL = (x) => (x >= 100 ? x.toFixed(0) : x >= 10 ? x.toFixed(1) : x.toFixed(2))
const fmtN = (n) => (n >= 1e6 ? `${(n / 1e6).toFixed(1)}M` : n >= 1e3 ? `${(n / 1e3).toFixed(1)}k` : `${Math.round(n)}`)
</script>

<template>
  <div class="zge">
    <div class="pan">
      <svg :viewBox="`0 0 ${S} ${S}`">
        <rect :width="S" :height="S" class="bg" />
        <rect v-for="(c, k) in cells" :key="`f${k}`" :x="c[0] * fsc" :y="c[1] * fsc" :width="cellW * fsc" :height="cellW * fsc" v-bind="cellStyle" />
        <circle v-for="(p, k) in field" :key="`b${k}`" :cx="p[0] * fsc" :cy="p[1] * fsc" r="0.9" class="lo" />
        <circle :cx="C * fsc" :cy="C * fsc" :r="Math.max(R * fsc, 1.2)" class="hr" />
        <g v-if="step >= 1 && zw * fsc > 1.5">
          <line v-for="z in zoomLines" :key="`zv${z}`" :x1="z * fsc" :x2="z * fsc" :y1="x0 * fsc" :y2="(x0 + V) * fsc" class="zl" />
          <line v-for="z in zoomLines" :key="`zh${z}`" :y1="z * fsc" :y2="z * fsc" :x1="x0 * fsc" :x2="(x0 + V) * fsc" class="zl" />
        </g>
        <line v-for="l in bkgLines" :key="`v${l}`" :x1="l * fsc" :x2="l * fsc" y1="0" :y2="S" class="bl" />
        <line v-for="l in bkgLines" :key="`h${l}`" :y1="l * fsc" :y2="l * fsc" x1="0" :x2="S" class="bl" />
        <rect v-if="step >= 1" :x="x0 * fsc" :y="x0 * fsc" :width="V * fsc" :height="V * fsc" class="void" />
        <rect :x="inLo * fsc" :y="inLo * fsc" :width="(inHi - inLo) * fsc" :height="(inHi - inLo) * fsc" class="crop" />
      </svg>
      <div class="cap">whole box · {{ L }} Mpc</div>
    </div>

    <div class="pan">
      <svg :viewBox="`0 0 ${S} ${S}`">
        <rect :width="S" :height="S" class="bg" />
        <rect v-for="(c, k) in cells" :key="`i${k}`" :x="ix(c[0])" :y="ix(c[1])" :width="cellW * isc" :height="cellW * isc" v-bind="cellStyle" />
        <circle v-for="(p, k) in fieldIn" :key="`ib${k}`" :cx="ix(p[0])" :cy="ix(p[1])" r="1.2" class="lo" />
        <circle v-for="(p, k) in disc" :key="`d${k}`" :cx="ix(C + p[0] * R)" :cy="ix(C + p[1] * R)" r="1" class="hrp" />
        <circle :cx="ix(C)" :cy="ix(C)" :r="R * isc" class="hr" />
        <g v-if="step >= 1">
          <line v-for="z in zoomLines" :key="`izv${z}`" :x1="ix(z)" :x2="ix(z)" :y1="ix(x0)" :y2="ix(x0 + V)" class="zl" />
          <line v-for="z in zoomLines" :key="`izh${z}`" :y1="ix(z)" :y2="ix(z)" :x1="ix(x0)" :x2="ix(x0 + V)" class="zl" />
        </g>
        <template v-for="i in voidHi - voidLo + 3" :key="`ig${i}`">
          <line :x1="ix((voidLo - 2 + i) * w)" :x2="ix((voidLo - 2 + i) * w)" y1="0" :y2="S" class="bl" />
          <line :y1="ix((voidLo - 2 + i) * w)" :y2="ix((voidLo - 2 + i) * w)" x1="0" :x2="S" class="bl" />
        </template>
        <rect v-if="step >= 1" :x="ix(x0)" :y="ix(x0)" :width="V * isc" :height="V * isc" class="void" />
      </svg>
      <div class="cap">around the high-res region · {{ fmtL(inHi - inLo) }} Mpc</div>
    </div>

    <div class="ctl">
      <div class="msg" v-if="step < 1">SWIFT splits work over a <b class="warn">grid of cells</b>. A zoom piles nearly every particle into a handful of them.</div>
      <div class="msg" v-else>Fix: turn the cells around the region into a <b class="accent">void</b> and nest a <b class="accent">second grid</b> of zoom cells inside.</div>

      <label>high-res region <b>{{ fmtL(D) }} Mpc</b></label>
      <input v-model.number="logD" type="range" :min="Math.log10(5)" :max="Math.log10(700)" step="0.01" />
      <label>background cdim <b>{{ bkg }}³</b> <span class="muted">· {{ fmtL(w) }} Mpc</span></label>
      <input v-model.number="bkg" type="range" min="3" max="48" step="1" />
      <label :class="{ off: step < 1 }">zoom depth <b>{{ depth }}</b> <span class="muted">· {{ 2 ** depth }}³ per bkg cell · {{ zoom }}³ total · {{ fmtL(zw) }} Mpc</span></label>
      <input v-model.number="depth" type="range" min="0" max="6" step="1" :disabled="step < 1" />

      <div class="stats">
        <div>void: <b>{{ nVoid }}³</b> background cell{{ nVoid > 1 ? 's' : '' }}</div>
        <div>background cell: <b>{{ fmtN(bgPerCell) }}</b> particles</div>
        <div>busiest {{ step >= 1 ? 'zoom' : 'background' }} cell: <b>{{ fmtN(hot) }}</b></div>
        <div class="big">load imbalance <b :class="ratioCls">{{ ratio >= 10 ? ratio.toFixed(0) : ratio.toFixed(1) }}×</b></div>
      </div>
      <div class="key"><span class="sw warn" /> overloaded <span class="sw accent" /> balanced <span class="sw theory" /> underloaded</div>
    </div>
  </div>
</template>

<style scoped>
.zge { display: grid; grid-template-columns: 240px 240px 1fr; gap: 18px; align-items: start; font-size: 0.72rem; }
.pan svg { width: 240px; height: 240px; display: block; border: 1px solid var(--line); }
.bg { fill: #0a0a0a; }
.bl { stroke: #555; stroke-width: 0.7; }
.zl { stroke: color-mix(in srgb, var(--accent) 45%, transparent); stroke-width: 0.5; }
.void { fill: none; stroke: var(--accent); stroke-width: 1.5; stroke-dasharray: 4 2; }
.crop { fill: none; stroke: var(--muted); stroke-width: 0.8; stroke-dasharray: 2 2; }
.lo { fill: #777; }
.hrp { fill: var(--warn); fill-opacity: 0.8; }
.hr { fill: none; stroke: var(--warn); stroke-width: 1.2; }
.cap { font-size: 0.6rem; color: var(--muted); text-align: center; margin-top: 4px; }
.msg { line-height: 1.45; min-height: 4.4em; margin-bottom: 8px; }
label { display: block; margin-top: 4px; }
label b { color: var(--accent); }
label.off { opacity: 0.35; }
input[type='range'] { width: 100%; accent-color: var(--accent); }
input:disabled { opacity: 0.3; }
.stats { margin-top: 8px; border: 1px solid var(--line); border-radius: 6px; padding: 6px 8px; background: #0a0a0a; font-size: 0.64rem; line-height: 1.6; }
.stats b { color: var(--ink); }
.stats .big { font-size: 0.8rem; margin-top: 2px; }
.stats b.bad { color: var(--bad); }
.stats b.accent { color: var(--accent); }
.stats b.theory { color: var(--theory); }
.key { margin-top: 6px; font-size: 0.58rem; color: var(--muted); display: flex; gap: 6px; align-items: center; }
.sw { display: inline-block; width: 9px; height: 9px; }
.sw.warn { background: var(--warn); }
.sw.accent { background: var(--accent); opacity: 0.6; }
.sw.theory { background: var(--theory); }
</style>
