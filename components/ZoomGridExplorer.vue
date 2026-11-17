<script setup>
// Toy picture of why zooms need a second cell grid (mid-plane slice). A 1 Gpc
// periodic box is split into bkg^2 top-level cells; the high-res region is a
// disc at the box centre.
//   zoom off: background grid only, so the region lands in a handful of cells.
//   zoom on:  the background cells covering the region become the "void" and
//             each is split `depth` times (2^depth zoom cells per side).
import { computed, ref } from 'vue'

const L = 1000 // box side [Mpc]

const zoomOn = ref(false)
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
// ponytail: zoom grid capped at 128 cells per side so the SVG stays light; the
// depth slider's max shrinks as the void grows
const maxDepth = computed(() => Math.max(0, Math.min(6, Math.floor(Math.log2(128 / nVoid.value)))))
const dep = computed(() => Math.min(depth.value, maxDepth.value))
const zoom = computed(() => nVoid.value * 2 ** dep.value) // zoom cells per void side
const zw = computed(() => w.value / 2 ** dep.value) // zoom cell side

// panels
const S = 240
const fsc = S / L
const bkgLines = computed(() => Array.from({ length: bkg.value - 1 }, (_, i) => (i + 1) * w.value))
// inset: the void plus one background cell either side (clamped to the box)
const inLo = computed(() => Math.max(0, x0.value - w.value))
const inHi = computed(() => Math.min(L, x0.value + V.value + w.value))
const isc = computed(() => S / (inHi.value - inLo.value))
const ix = (x) => (x - inLo.value) * isc.value

const zoomLines = computed(() => Array.from({ length: zoom.value - 1 }, (_, i) => x0.value + (i + 1) * zw.value))

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
</script>

<template>
  <div class="zge">
    <div class="pan">
      <svg :viewBox="`0 0 ${S} ${S}`">
        <rect :width="S" :height="S" class="bg" />
        <circle v-for="(p, k) in field" :key="`b${k}`" :cx="p[0] * fsc" :cy="p[1] * fsc" r="0.9" class="lo" />
        <circle :cx="C * fsc" :cy="C * fsc" :r="Math.max(R * fsc, 1.2)" class="hr" />
        <g v-if="zoomOn && zw * fsc > 1.5">
          <line v-for="z in zoomLines" :key="`zv${z}`" :x1="z * fsc" :x2="z * fsc" :y1="x0 * fsc" :y2="(x0 + V) * fsc" class="zl" />
          <line v-for="z in zoomLines" :key="`zh${z}`" :y1="z * fsc" :y2="z * fsc" :x1="x0 * fsc" :x2="(x0 + V) * fsc" class="zl" />
        </g>
        <line v-for="l in bkgLines" :key="`v${l}`" :x1="l * fsc" :x2="l * fsc" y1="0" :y2="S" class="bl" />
        <line v-for="l in bkgLines" :key="`h${l}`" :y1="l * fsc" :y2="l * fsc" x1="0" :x2="S" class="bl" />
        <rect v-if="zoomOn" :x="x0 * fsc" :y="x0 * fsc" :width="V * fsc" :height="V * fsc" class="void" />
        <rect :x="inLo * fsc" :y="inLo * fsc" :width="(inHi - inLo) * fsc" :height="(inHi - inLo) * fsc" class="crop" />
      </svg>
      <div class="cap">whole box · {{ L }} Mpc</div>
    </div>

    <div class="pan">
      <svg :viewBox="`0 0 ${S} ${S}`">
        <rect :width="S" :height="S" class="bg" />
        <circle v-for="(p, k) in fieldIn" :key="`ib${k}`" :cx="ix(p[0])" :cy="ix(p[1])" r="1.2" class="lo" />
        <circle v-for="(p, k) in disc" :key="`d${k}`" :cx="ix(C + p[0] * R)" :cy="ix(C + p[1] * R)" r="1" class="hrp" />
        <circle :cx="ix(C)" :cy="ix(C)" :r="R * isc" class="hr" />
        <g v-if="zoomOn">
          <line v-for="z in zoomLines" :key="`izv${z}`" :x1="ix(z)" :x2="ix(z)" :y1="ix(x0)" :y2="ix(x0 + V)" class="zl" />
          <line v-for="z in zoomLines" :key="`izh${z}`" :y1="ix(z)" :y2="ix(z)" :x1="ix(x0)" :x2="ix(x0 + V)" class="zl" />
        </g>
        <template v-for="i in voidHi - voidLo + 3" :key="`ig${i}`">
          <line :x1="ix((voidLo - 2 + i) * w)" :x2="ix((voidLo - 2 + i) * w)" y1="0" :y2="S" class="bl" />
          <line :y1="ix((voidLo - 2 + i) * w)" :y2="ix((voidLo - 2 + i) * w)" x1="0" :x2="S" class="bl" />
        </template>
        <rect v-if="zoomOn" :x="ix(x0)" :y="ix(x0)" :width="V * isc" :height="V * isc" class="void" />
      </svg>
      <div class="cap">around the high-res region · {{ fmtL(inHi - inLo) }} Mpc</div>
    </div>

    <div class="ctl">
      <div class="msg" v-if="!zoomOn">SWIFT splits work over a <b class="warn">grid of cells</b>. A zoom piles nearly every particle into a handful of them.</div>
      <div class="msg" v-else>Fix: turn the cells around the region into a <b class="accent">void</b> and nest a <b class="accent">second grid</b> of zoom cells inside.</div>

      <label>high-res region <b>{{ fmtL(D) }} Mpc</b></label>
      <input v-model.number="logD" type="range" :min="Math.log10(5)" :max="Math.log10(700)" step="0.01" />
      <label>background cdim <b>{{ bkg }}³</b> <span class="muted">· {{ fmtL(w) }} Mpc</span></label>
      <input v-model.number="bkg" type="range" min="3" max="100" step="1" />
      <label class="chk"><input v-model="zoomOn" type="checkbox" /> nested zoom grid</label>
      <label :class="{ off: !zoomOn }">zoom depth <b>{{ dep }}</b> <span class="muted">· {{ 2 ** dep }}³ per bkg cell · {{ zoom }}³ total · {{ fmtL(zw) }} Mpc</span></label>
      <input v-model.number="depth" type="range" min="0" :max="maxDepth" step="1" :disabled="!zoomOn" />

      <div class="note">void: {{ nVoid }}³ background cell{{ nVoid > 1 ? 's' : '' }}</div>
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
.chk { margin-top: 10px; cursor: pointer; }
.note { margin-top: 10px; font-size: 0.64rem; color: var(--muted); }
.chk input { accent-color: var(--accent); margin-right: 4px; vertical-align: -1px; }
</style>
