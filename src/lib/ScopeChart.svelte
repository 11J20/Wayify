<!-- ScopeChart.svelte — pannable, zoomable, interactive canvas chart with crosshair cursor -->
<script lang="ts">
  import { onMount } from 'svelte';

  // ── Exported types (re-used by App.svelte + VectroScope.svelte) ────
  export interface ScopePoint {
    ts:   string;        // "DDMMYYYY - HH:MM:SS:mmm"
    x:    number;
    y:    number;
    z:    number | null; // null = magnetometer absent
    abs?: number;
  }

  // ── Props ──────────────────────────────────────────────────────────
  interface Props {
    data:     ScopePoint[];
    showAbs?: boolean;
    dark?:    boolean;
    dataKey?: number;    // increment externally to reset view
  }

  let {
    data    = [],
    showAbs = false,
    dark    = false,
    dataKey = 0,
  }: Props = $props();

  // ── Canvas ref ─────────────────────────────────────────────────────
  let canvas: HTMLCanvasElement;

  // ── View range (data-index space, plain vars — only affect canvas) ─
  let vL = 0;   // left  boundary (float data index)
  let vR = 0;   // right boundary (float data index)

  // ── Cursor ($state — drives DOM footer) ───────────────────────────
  interface CursorInfo {
    ts:   string;
    x:    number;
    y:    number;
    z:    number | null;
    abs?: number;
    idx:  number;
  }
  let cursor    = $state<CursorInfo | null>(null);
  let cursorPx: number | null = null;   // CSS pixel X for draw()

  // ── Interaction state ──────────────────────────────────────────────
  let drag  = { on: false, sx: 0, vL0: 0, vR0: 0 };
  let pinch = { on: false, dist0: 0, vL0: 0, vR0: 0, mf: 0.5 };
  let t1Pan = { sx: 0, vL: 0, vR: 0 };

  // ── Layout ─────────────────────────────────────────────────────────
  const PL = 54, PR = 12, PT = 16, PB = 32;

  // ── Channel palette ────────────────────────────────────────────────
  type K = 'x' | 'y' | 'z' | 'abs';
  const CHANNELS = [
    { k: 'x'   as K, c: '#ff5c7a', lbl: 'X' },
    { k: 'y'   as K, c: '#1fd19e', lbl: 'Y' },
    { k: 'z'   as K, c: '#5b96f5', lbl: 'Z' },
  ];
  const ABS_CH = { k: 'abs' as K, c: '#ffc147', lbl: '|A|' };

  // ── Helpers ────────────────────────────────────────────────────────
  function dpr() { return window.devicePixelRatio || 1; }

  function cssWH() {
    const r = canvas.getBoundingClientRect();
    return { w: r.width, h: r.height };
  }

  function resize() {
    const d = dpr(), { w, h } = cssWH();
    canvas.width  = Math.floor(w * d);
    canvas.height = Math.floor(h * d);
  }

  function clamp() {
    const n = data.length;
    if (!n) return;
    const span = Math.max(vR - vL, 3);
    vR = vL + span;
    if (vL < 0)     { vR -= vL; vL = 0; }
    if (vR > n - 1) { vL -= vR - (n - 1); vR = n - 1; }
    vL = Math.max(0, vL);
    vR = Math.min(n - 1, vR);
  }

  function fracToIdx(f: number) { return vL + f * (vR - vL); }

  function snapIdx(f: number) {
    return Math.max(0, Math.min(data.length - 1, Math.round(fracToIdx(Math.max(0, Math.min(1, f))))));
  }

  function idxToX(i: number, plotW: number): number {
    const f = vR === vL ? 0.5 : (i - vL) / (vR - vL);
    return PL + f * plotW;
  }

  // ── Main draw ──────────────────────────────────────────────────────
  function draw() {
    if (!canvas) return;
    const n   = data.length;
    const d   = dpr();
    const ctx = canvas.getContext('2d')!;
    const { w, h } = cssWH();

    ctx.setTransform(d, 0, 0, d, 0, 0);
    ctx.clearRect(0, 0, w, h);

    const plotW = w - PL - PR;
    const plotH = h - PT - PB;

    // Colours
    const gridC  = dark ? 'rgba(255,255,255,0.05)' : 'rgba(0,0,0,0.07)';
    const labelC = dark ? 'rgba(255,255,255,0.28)' : 'rgba(0,0,0,0.35)';
    const bgC    = dark ? 'rgba(255,255,255,0.015)' : 'rgba(0,0,0,0.018)';

    // Plot area background
    ctx.fillStyle = bgC;
    ctx.beginPath();
    if (ctx.roundRect) ctx.roundRect(PL, PT, plotW, plotH, 6);
    else ctx.rect(PL, PT, plotW, plotH);
    ctx.fill();

    if (!n) {
      ctx.fillStyle = labelC;
      ctx.font = '13px Inter, sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('No data loaded', w / 2, h / 2);
      return;
    }

    // Visible slice
    const si  = Math.max(0, Math.floor(vL));
    const ei  = Math.min(n - 1, Math.ceil(vR));
    const vis = data.slice(si, ei + 1);

    // Value range
    const chs: K[] = ['x', 'y', 'z', ...(showAbs ? ['abs' as K] : [])];
    const vals: number[] = [];
    vis.forEach(p => chs.forEach(k => {
      const v = p[k]; if (v !== null && v !== undefined) vals.push(v as number);
    }));
    const rawMin = vals.length ? Math.min(...vals) : -1;
    const rawMax = vals.length ? Math.max(...vals) :  1;
    const pad = Math.max((rawMax - rawMin) * 0.12, 0.5);
    const lo  = rawMin - pad, hi = rawMax + pad, span = hi - lo || 1;

    const toY = (v: number) => PT + plotH * (1 - (v - lo) / span);
    const toX = (i: number) => idxToX(i, plotW);

    // ── Grid + Y labels ────────────────────────
    ctx.font = '9px SF Mono, Consolas, monospace';
    ctx.textAlign = 'right';
    [0, 0.25, 0.5, 0.75, 1].forEach(t => {
      const y = PT + t * plotH, v = hi - t * span;
      ctx.strokeStyle = gridC;
      ctx.lineWidth = 1;
      ctx.setLineDash([3, 6]);
      ctx.beginPath(); ctx.moveTo(PL, y); ctx.lineTo(w - PR, y); ctx.stroke();
      ctx.setLineDash([]);
      ctx.fillStyle = labelC;
      ctx.fillText(v.toFixed(2), PL - 6, y + 3.5);
    });

    // ── X axis time labels ──────────────────────
    ctx.textAlign = 'center';
    ctx.font = '8px Inter, sans-serif';
    ctx.fillStyle = labelC;
    const numLbls = Math.max(2, Math.min(5, Math.floor(plotW / 90)));
    for (let i = 0; i <= numLbls; i++) {
      const f   = i / numLbls;
      const idx = Math.max(0, Math.min(n - 1, Math.round(fracToIdx(f))));
      const ts  = data[idx]?.ts ?? '';
      const t   = ts.includes(' - ') ? ts.split(' - ')[1] : ts;
      ctx.fillText(t, PL + f * plotW, h - PB + 16);
    }

    // ── Vertical grid lines ─────────────────────
    ctx.strokeStyle = gridC;
    ctx.lineWidth = 1;
    ctx.setLineDash([2, 8]);
    for (let i = 1; i < numLbls; i++) {
      const x = PL + (i / numLbls) * plotW;
      ctx.beginPath(); ctx.moveTo(x, PT); ctx.lineTo(x, PT + plotH); ctx.stroke();
    }
    ctx.setLineDash([]);

    // ── Draw channels ───────────────────────────
    const allChs = [...CHANNELS, ...(showAbs ? [ABS_CH] : [])];
    const useBezier = (ei - si) < 250;

    allChs.forEach(({ k, c }) => {
      const pts: { x: number; y: number }[] = [];
      for (let i = si; i <= ei; i++) {
        const v = data[i][k];
        if (v !== null && v !== undefined) pts.push({ x: toX(i), y: toY(v as number) });
      }
      if (pts.length < 2) return;

      const buildPath = () => {
        ctx.beginPath();
        ctx.moveTo(pts[0].x, pts[0].y);
        if (useBezier) {
          for (let i = 1; i < pts.length; i++) {
            const mx = (pts[i-1].x + pts[i].x) / 2;
            ctx.bezierCurveTo(mx, pts[i-1].y, mx, pts[i].y, pts[i].x, pts[i].y);
          }
        } else {
          for (let i = 1; i < pts.length; i++) ctx.lineTo(pts[i].x, pts[i].y);
        }
      };

      // Gradient fill
      const grad = ctx.createLinearGradient(0, PT, 0, PT + plotH);
      grad.addColorStop(0, c + '45');
      grad.addColorStop(1, c + '00');
      buildPath();
      ctx.lineTo(pts[pts.length-1].x, PT + plotH);
      ctx.lineTo(pts[0].x, PT + plotH);
      ctx.closePath();
      ctx.fillStyle = grad;
      ctx.fill();

      // Line stroke
      buildPath();
      ctx.strokeStyle = c;
      ctx.lineWidth   = pts.length > 500 ? 1 : 1.5;
      ctx.lineJoin    = 'round';
      ctx.lineCap     = 'round';
      ctx.stroke();
    });

    // ── Crosshair ───────────────────────────────
    if (cursorPx !== null) {
      const f   = (cursorPx - PL) / plotW;
      const idx = snapIdx(f);
      const sx  = toX(idx);  // snap to nearest data point X

      // Vertical dashed line
      ctx.strokeStyle = dark ? 'rgba(255,255,255,0.55)' : 'rgba(0,0,0,0.45)';
      ctx.lineWidth   = 1;
      ctx.setLineDash([5, 3]);
      ctx.beginPath(); ctx.moveTo(sx, PT); ctx.lineTo(sx, PT + plotH); ctx.stroke();
      ctx.setLineDash([]);

      // Channel dots + horizontal guide lines
      allChs.forEach(({ k, c }) => {
        const v = data[idx]?.[k];
        if (v !== null && v !== undefined) {
          const dotY = toY(v as number);

          // Horizontal dotted guideline to Y axis
          ctx.strokeStyle = c + '55';
          ctx.lineWidth   = 1;
          ctx.setLineDash([3, 5]);
          ctx.beginPath(); ctx.moveTo(PL, dotY); ctx.lineTo(sx, dotY); ctx.stroke();
          ctx.setLineDash([]);

          // Y axis tick value
          ctx.fillStyle  = c;
          ctx.font       = '9px SF Mono, Consolas, monospace';
          ctx.textAlign  = 'right';
          ctx.fillText((v as number).toFixed(2), PL - 6, dotY + 3.5);

          // Glowing dot
          ctx.shadowColor = c;
          ctx.shadowBlur  = 10;
          ctx.beginPath(); ctx.arc(sx, dotY, 5.5, 0, Math.PI * 2);
          ctx.fillStyle = c; ctx.fill();
          ctx.shadowBlur = 0;
          ctx.beginPath(); ctx.arc(sx, dotY, 2.5, 0, Math.PI * 2);
          ctx.fillStyle = dark ? '#000' : '#fff'; ctx.fill();
        }
      });
    }

    // ── Zoom indicator ──────────────────────────
    const cnt    = ei - si + 1;
    const zoomPct = Math.round((cnt / n) * 100);
    ctx.fillStyle  = labelC;
    ctx.font       = '9px Inter, sans-serif';
    ctx.textAlign  = 'right';
    ctx.fillText(`${cnt} / ${n} pts · ${zoomPct}%`, w - PR, PT - 4);
  }

  // ── Cursor update ──────────────────────────────────────────────────
  function updateCursor(clientX: number | null) {
    if (!canvas || !data.length) { cursor = null; cursorPx = null; return; }
    if (clientX === null)        { cursor = null; cursorPx = null; draw(); return; }

    const rect  = canvas.getBoundingClientRect();
    const plotW = rect.width - PL - PR;
    cursorPx = Math.max(PL, Math.min(rect.width - PR, clientX - rect.left));

    const f   = (cursorPx - PL) / plotW;
    const idx = snapIdx(f);
    if (idx >= 0 && idx < data.length) cursor = { ...data[idx], idx };
    draw();
  }

  // ── Mouse handlers ─────────────────────────────────────────────────
  function onMouseDown(e: MouseEvent) {
    drag = { on: true, sx: e.clientX, vL0: vL, vR0: vR };
    canvas.style.cursor = 'grabbing';
  }

  function onMouseMove(e: MouseEvent) {
    if (drag.on) {
      const rect  = canvas.getBoundingClientRect();
      const plotW = rect.width - PL - PR;
      const dx    = (e.clientX - drag.sx) / plotW;
      const span  = drag.vR0 - drag.vL0;
      vL = drag.vL0 - dx * span;
      vR = drag.vR0 - dx * span;
      clamp();
      cursor = null; cursorPx = null;
      draw();
    } else {
      updateCursor(e.clientX);
    }
  }

  function onMouseUp()    { drag.on = false; canvas.style.cursor = 'crosshair'; }

  function onMouseLeave() {
    drag.on = false;
    cursor = null; cursorPx = null;
    canvas.style.cursor = 'crosshair';
    draw();
  }

  function onWheel(e: WheelEvent) {
    e.preventDefault();
    const rect  = canvas.getBoundingClientRect();
    const plotW = rect.width - PL - PR;
    const f     = Math.max(0, Math.min(1, (e.clientX - rect.left - PL) / plotW));
    const span  = vR - vL;
    const pivot = vL + f * span;
    const fac   = e.deltaY > 0 ? 1.22 : 0.82;
    vL = pivot - f * span * fac;
    vR = pivot + (1 - f) * span * fac;
    clamp();
    draw();
  }

  // ── Touch handlers ─────────────────────────────────────────────────
  function td(e: TouchEvent) {
    return Math.abs(e.touches[0].clientX - e.touches[1].clientX);
  }

  function onTouchStart(e: TouchEvent) {
    e.preventDefault();
    if (e.touches.length === 1) {
      pinch.on = false;
      t1Pan = { sx: e.touches[0].clientX, vL, vR };
    } else if (e.touches.length === 2) {
      const rect  = canvas.getBoundingClientRect();
      const plotW = rect.width - PL - PR;
      const mid   = (e.touches[0].clientX + e.touches[1].clientX) / 2;
      pinch = {
        on: true, dist0: td(e), vL0: vL, vR0: vR,
        mf: Math.max(0, Math.min(1, (mid - rect.left - PL) / plotW)),
      };
    }
  }

  function onTouchMove(e: TouchEvent) {
    e.preventDefault();
    if (e.touches.length === 1 && !pinch.on) {
      const rect  = canvas.getBoundingClientRect();
      const plotW = rect.width - PL - PR;
      const dx    = (e.touches[0].clientX - t1Pan.sx) / plotW;
      const span  = t1Pan.vR - t1Pan.vL;
      vL = t1Pan.vL - dx * span;
      vR = t1Pan.vR - dx * span;
      clamp(); draw();
    } else if (e.touches.length === 2 && pinch.on) {
      const origSpan = pinch.vR0 - pinch.vL0;
      const factor   = pinch.dist0 / Math.max(td(e), 1);
      const span     = origSpan * factor;
      const pivot    = pinch.vL0 + pinch.mf * origSpan;
      vL = pivot - pinch.mf * span;
      vR = pivot + (1 - pinch.mf) * span;
      clamp(); draw();
    }
  }

  function onTouchEnd(e: TouchEvent) {
    if (e.touches.length < 2) pinch.on = false;
    if (e.touches.length === 1) t1Pan = { sx: e.touches[0].clientX, vL, vR };
    if (e.touches.length === 0) { cursor = null; cursorPx = null; draw(); }
  }

  // ── Reactive redraw ────────────────────────────────────────────────
  let prevKey = -1;
  $effect(() => {
    void dark;
    const n = data.length;
    if (dataKey !== prevKey) {
      prevKey = dataKey;
      vL = 0; vR = Math.max(0, n - 1);
    }
    draw();
  });

  // ── Mount ──────────────────────────────────────────────────────────
  onMount(() => {
    resize();
    vL = 0; vR = Math.max(0, data.length - 1);
    draw();
    const ro = new ResizeObserver(() => { resize(); draw(); });
    ro.observe(canvas);
    return () => ro.disconnect();
  });
</script>

<!-- ── Markup ─────────────────────────────────────────────────────── -->
<div class="scope-wrap">
  <canvas
    bind:this={canvas}
    onmousedown={onMouseDown}
    onmousemove={onMouseMove}
    onmouseup={onMouseUp}
    onmouseleave={onMouseLeave}
    onwheel={onWheel}
    ontouchstart={onTouchStart}
    ontouchmove={onTouchMove}
    ontouchend={onTouchEnd}
    style="width:100%;display:block;cursor:crosshair;touch-action:none;user-select:none;"
  ></canvas>

  <!-- Cursor / hint footer -->
  <div class="scope-footer" class:active={cursor !== null}>
    {#if cursor}
      <span class="sf-ts">⏱ {cursor.ts}</span>
      <span class="sf-div">│</span>
      <span class="sf-val x">X&thinsp;<b>{cursor.x.toFixed(4)}</b></span>
      <span class="sf-val y">Y&thinsp;<b>{cursor.y.toFixed(4)}</b></span>
      <span class="sf-val z">Z&thinsp;<b>{cursor.z !== null ? cursor.z.toFixed(4) : 'N/A'}</b></span>
      {#if cursor.abs !== undefined && showAbs}
        <span class="sf-val abs">|A|&thinsp;<b>{cursor.abs.toFixed(4)}</b></span>
      {/if}
    {:else}
      <span class="sf-hint">← Drag to pan &nbsp;·&nbsp; Scroll to zoom &nbsp;·&nbsp; Hover to inspect &nbsp;·&nbsp; Pinch on mobile</span>
    {/if}
  </div>
</div>

<!-- ── Styles ─────────────────────────────────────────────────────── -->
<style>
  .scope-wrap {
    display: flex;
    flex-direction: column;
    height: 100%;
  }

  canvas { flex: 1; min-height: 0; }

  .scope-footer {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 7px 14px;
    font-size: 11px;
    font-family: var(--font-mono);
    border-top: 1px solid var(--color-border-subtle);
    min-height: 34px;
    flex-wrap: wrap;
    background: transparent;
    transition: background var(--transition-fast);
  }

  .scope-footer.active { background: var(--color-border-subtle); }

  .sf-ts   { color: var(--color-text-muted); white-space: nowrap; }
  .sf-div  { color: var(--color-border); flex-shrink: 0; }
  .sf-hint { color: var(--color-text-muted); font-family: var(--font-family); font-style: italic; }

  .sf-val { white-space: nowrap; font-size: 11px; }
  .sf-val b { font-weight: 700; margin-left: 2px; }
  .sf-val.x   { color: var(--color-x); }
  .sf-val.y   { color: var(--color-y); }
  .sf-val.z   { color: var(--color-z); }
  .sf-val.abs { color: var(--color-abs); }
</style>
