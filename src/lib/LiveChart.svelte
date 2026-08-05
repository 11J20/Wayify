<!-- LiveChart.svelte – Canvas-based real-time line chart (no external deps) -->
<script lang="ts">
  import { onMount } from 'svelte';

  // ── Types ──────────────────────────────────────────────────────────
  export interface ChartPoint {
    x:    number;
    y:    number;
    z:    number | null;
    abs?: number;
  }

  interface Props {
    points:    ChartPoint[];
    showAbs?:  boolean;
    dark?:     boolean;
    maxPoints?: number;
    label?:    string;
  }

  let {
    points,
    showAbs   = false,
    dark      = false,
    maxPoints = 100,
    label     = '',
  }: Props = $props();

  // ── Canvas ref ─────────────────────────────────────────────────────
  let canvas: HTMLCanvasElement;

  // ── Channel definitions ────────────────────────────────────────────
  const CHANNELS = [
    { key: 'x',  color: '#ff5c7a', glyph: 'X' },
    { key: 'y',  color: '#1fd19e', glyph: 'Y' },
    { key: 'z',  color: '#5b96f5', glyph: 'Z' },
  ] as const;
  const ABS_CH = { key: 'abs', color: '#ffc147', glyph: '|A|' };

  // ── Resize canvas to match CSS size @ device pixel ratio ───────────
  function resize() {
    if (!canvas) return;
    const dpr  = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    canvas.width  = Math.floor(rect.width  * dpr);
    canvas.height = Math.floor(rect.height * dpr);
  }

  // ── Main draw routine ──────────────────────────────────────────────
  function draw() {
    if (!canvas) return;
    const dpr = window.devicePixelRatio || 1;
    const ctx = canvas.getContext('2d')!;

    // CSS dimensions (logical pixels)
    const CW = canvas.width  / dpr;
    const CH = canvas.height / dpr;

    // Reset transform so scale never accumulates between frames
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    ctx.clearRect(0, 0, CW, CH);

    const visible = points.slice(-maxPoints);
    const n = visible.length;

    // ── Layout ─────────────────────────────────
    const PAD_L = 46;
    const PAD_R = 10;
    const PAD_T = 14;
    const PAD_B = 36;
    const plotW = CW - PAD_L - PAD_R;
    const plotH = CH - PAD_T - PAD_B;

    // ── Value range ────────────────────────────
    const allVals: number[] = [];
    visible.forEach(p => {
      allVals.push(p.x, p.y);
      if (p.z !== null && p.z !== undefined) allVals.push(p.z);
      if (showAbs && p.abs !== undefined)    allVals.push(p.abs);
    });

    const rawMin = allVals.length ? Math.min(...allVals) : -1;
    const rawMax = allVals.length ? Math.max(...allVals) :  1;
    const pad    = Math.max((rawMax - rawMin) * 0.12, 0.5);
    const lo     = rawMin - pad;
    const hi     = rawMax + pad;
    const span   = hi - lo || 1;

    const toY = (v: number) => PAD_T + plotH * (1 - (v - lo) / span);
    const toX = (i: number) => PAD_L + (n <= 1 ? plotW / 2 : (i / (n - 1)) * plotW);

    // ── Colour tokens ──────────────────────────
    const gridCol  = dark ? 'rgba(255,255,255,0.05)' : 'rgba(0,0,0,0.06)';
    const labelCol = dark ? 'rgba(255,255,255,0.28)' : 'rgba(0,0,0,0.32)';
    const bgPanel  = dark ? 'rgba(255,255,255,0.02)' : 'rgba(0,0,0,0.02)';

    // ── Plot area background ───────────────────
    ctx.fillStyle = bgPanel;
    ctx.beginPath();
    ctx.roundRect(PAD_L, PAD_T, plotW, plotH, 4);
    ctx.fill();

    // ── Grid lines & Y-axis labels ─────────────
    ctx.font      = '9px SF Mono, Consolas, monospace';
    ctx.textAlign = 'right';

    [0, 0.25, 0.5, 0.75, 1].forEach(t => {
      const y   = PAD_T + t * plotH;
      const val = hi - t * span;

      ctx.strokeStyle = gridCol;
      ctx.lineWidth   = 1;
      ctx.setLineDash([3, 4]);
      ctx.beginPath(); ctx.moveTo(PAD_L, y); ctx.lineTo(CW - PAD_R, y); ctx.stroke();
      ctx.setLineDash([]);

      ctx.fillStyle = labelCol;
      ctx.fillText(val.toFixed(1), PAD_L - 5, y + 3.5);
    });

    // ── "Waiting" message ──────────────────────
    if (n < 2) {
      ctx.fillStyle  = labelCol;
      ctx.font       = '12px Inter, sans-serif';
      ctx.textAlign  = 'center';
      ctx.fillText('Waiting for sensor data…', PAD_L + plotW / 2, PAD_T + plotH / 2);
    }

    // ── Draw each channel ──────────────────────
    const channels = [
      ...CHANNELS,
      ...(showAbs ? [ABS_CH] : []),
    ];

    channels.forEach(({ key, color }) => {
      type K = 'x' | 'y' | 'z' | 'abs';
      const pts: { x: number; y: number }[] = [];
      visible.forEach((p, i) => {
        const v = p[key as K] as number | null | undefined;
        if (v !== null && v !== undefined) {
          pts.push({ x: toX(i), y: toY(v) });
        }
      });
      if (pts.length < 2) return;

      // Build bezier path helper
      const buildPath = () => {
        ctx.beginPath();
        ctx.moveTo(pts[0].x, pts[0].y);
        for (let i = 1; i < pts.length; i++) {
          const mx = (pts[i-1].x + pts[i].x) / 2;
          ctx.bezierCurveTo(mx, pts[i-1].y, mx, pts[i].y, pts[i].x, pts[i].y);
        }
      };

      // Gradient fill under the line
      const grad = ctx.createLinearGradient(0, PAD_T, 0, PAD_T + plotH);
      grad.addColorStop(0, color + '55');
      grad.addColorStop(1, color + '00');

      buildPath();
      ctx.lineTo(pts[pts.length-1].x, PAD_T + plotH);
      ctx.lineTo(pts[0].x,            PAD_T + plotH);
      ctx.closePath();
      ctx.fillStyle = grad;
      ctx.fill();

      // Stroke line
      buildPath();
      ctx.strokeStyle = color;
      ctx.lineWidth   = 2;
      ctx.lineJoin    = 'round';
      ctx.lineCap     = 'round';
      ctx.stroke();

      // Glowing endpoint dot
      const last = pts[pts.length - 1];
      ctx.shadowColor = color;
      ctx.shadowBlur  = 8;
      ctx.beginPath();
      ctx.arc(last.x, last.y, 4, 0, Math.PI * 2);
      ctx.fillStyle = color;
      ctx.fill();
      ctx.shadowBlur = 0;

      // Inner white dot
      ctx.beginPath();
      ctx.arc(last.x, last.y, 1.8, 0, Math.PI * 2);
      ctx.fillStyle = dark ? '#000' : '#fff';
      ctx.fill();
    });

    // ── Legend ─────────────────────────────────
    const legendItems = channels;
    const legendY = CH - 12;
    let legendX = PAD_L;

    ctx.font      = '10px Inter, sans-serif';
    ctx.textAlign = 'left';

    legendItems.forEach(({ glyph, color }) => {
      // swatch
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.roundRect(legendX, legendY - 7, 14, 7, 3);
      ctx.fill();
      legendX += 18;

      ctx.fillStyle = labelCol;
      ctx.fillText(glyph, legendX, legendY);
      legendX += ctx.measureText(glyph).width + 14;
    });

    // ── X axis label ───────────────────────────
    ctx.fillStyle  = labelCol;
    ctx.font       = '9px Inter, sans-serif';
    ctx.textAlign  = 'right';
    ctx.fillText(`${n} pts`, CW - PAD_R, CH - 3);
  }

  // ── Reactive redraw ────────────────────────────────────────────────
  $effect(() => {
    void points;   // track reactive dep
    void dark;
    draw();
  });

  // ── Mount ──────────────────────────────────────────────────────────
  onMount(() => {
    resize();
    draw();

    const ro = new ResizeObserver(() => { resize(); draw(); });
    ro.observe(canvas);
    return () => ro.disconnect();
  });
</script>

<!-- Chart canvas fills its parent -->
<canvas bind:this={canvas} aria-label={label} style="width:100%;height:100%;display:block;"></canvas>
