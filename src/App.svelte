<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import LiveChart from './lib/LiveChart.svelte';
  import type { ChartPoint } from './lib/LiveChart.svelte';
  import VectroScope from './VectroScope.svelte';

  // ── Types ─────────────────────────────────────────────────────────
  interface SensorEntry {
    ts:  string;   // DDMMYYYY - HH:MM
    x:   number;
    y:   number;
    z:   number | null;  // null when device has no magnetometer (alpha)
    abs?: number;
  }

  // ── Raw High-Frequency Memory Buffers ─────────────────────────────
  // These are plain JS objects/arrays. They prevent Svelte reactivity and GC 
  // pressure when updating 100+ times per second.
  const fullGyroLog:  SensorEntry[] = [];
  const fullAccelLog: SensorEntry[] = [];
  const rawGyroBuf:   ChartPoint[] = [];
  const rawAccelBuf:  ChartPoint[] = [];

  let rawGyroLive = { x: 0, y: 0, z: null as number | null };
  let rawAccelLive = { x: 0, y: 0, z: 0, abs: 0 };
  let rawAccelSource: 'gravity' | 'linear' | 'none' = 'none';

  // ── Svelte State (UI synced via RAF) ──────────────────────────────
  let currentPage = $state<'live' | 'vectroscope'>('live');

  // We only render the latest 200 items in the DOM to prevent browser hang
  let gyroLogPreview:  SensorEntry[] = $state([]);
  let accelLogPreview: SensorEntry[] = $state([]);
  let gyroLogCount  = $state(0);
  let accelLogCount = $state(0);
  let sessionStartTime = $state<string | null>(null);
  let sessionEndTime   = $state<string | null>(null);

  // Always-live chart buffers (synced via RAF)
  let gyroBuf:  ChartPoint[] = $state([]);
  let accelBuf: ChartPoint[] = $state([]);

  let gyroLive  = $state({ x: 0, y: 0, z: null as number | null });
  let accelLive = $state({ x: 0, y: 0, z: 0, abs: 0 });
  let accelSource = $state<'gravity' | 'linear' | 'none'>('none');

  let isCapturing    = $state(false);
  let permGranted    = $state(false);
  let permPending    = $state(false);
  let errorMsg       = $state('');
  let sensorSupported = $state(true);

  // Dark mode – respects OS preference, persists manual override
  let darkMode = $state(
    localStorage.getItem('wayify-dark') !== null
      ? localStorage.getItem('wayify-dark') === 'true'
      : window.matchMedia('(prefers-color-scheme: dark)').matches
  );

  function toggleDark() {
    darkMode = !darkMode;
    localStorage.setItem('wayify-dark', String(darkMode));
  }

  let gyroRef:  HTMLDivElement;
  let accelRef: HTMLDivElement;

  // ── Helpers ───────────────────────────────────────────────────────
  function formatTs(): string {
    const now  = new Date();
    const dd   = String(now.getDate()).padStart(2, '0');
    const mm   = String(now.getMonth() + 1).padStart(2, '0');
    const yyyy = now.getFullYear();
    const hh   = String(now.getHours()).padStart(2, '0');
    const min  = String(now.getMinutes()).padStart(2, '0');
    const ss   = String(now.getSeconds()).padStart(2, '0');
    const ms   = String(now.getMilliseconds()).padStart(3, '0');
    return `${dd}${mm}${yyyy} - ${hh}:${min}:${ss}:${ms}`;
  }

  function fix(n: number) { return n.toFixed(4); }

  function scrollBottom(el?: HTMLDivElement) {
    if (el) el.scrollTop = el.scrollHeight;
  }

  // ── RAF Throttle Loop ─────────────────────────────────────────────
  let rafId: number;
  let lastFrame = 0;
  let lastTableUpdate = 0;

  function tick(now: number) {
    rafId = requestAnimationFrame(tick);

    if (now - lastFrame < 33) return; // Cap at 30fps for graphs/live-readouts
    lastFrame = now;

    // 1. Sync live readouts
    gyroLive.x = rawGyroLive.x;
    gyroLive.y = rawGyroLive.y;
    gyroLive.z = rawGyroLive.z;
    
    accelLive.x = rawAccelLive.x;
    accelLive.y = rawAccelLive.y;
    accelLive.z = rawAccelLive.z;
    accelLive.abs = rawAccelLive.abs;
    accelSource = rawAccelSource;

    // 2. Sync buffers (slice creates a fresh array reference for Svelte)
    gyroBuf = rawGyroBuf.slice();
    accelBuf = rawAccelBuf.slice();

    // 3. Sync UI log counts
    gyroLogCount = fullGyroLog.length;
    accelLogCount = fullAccelLog.length;

    if (isCapturing) {
      if (now - lastTableUpdate > 250) { // Throttle heavy DOM updates to 4fps
        gyroLogPreview = fullGyroLog.slice(-200);
        accelLogPreview = fullAccelLog.slice(-200);
        
        // Wait a microtask for DOM to update the table before scrolling
        Promise.resolve().then(() => {
          scrollBottom(gyroRef);
          scrollBottom(accelRef);
        });
        
        lastTableUpdate = now;
      }
    }
  }

  // ── Sensor handlers (Raw Array Mutations) ─────────────────────────
  function onGyro(e: DeviceOrientationEvent) {
    const x = e.beta ?? 0;
    const y = e.gamma ?? 0;
    const z = e.alpha;

    rawGyroLive.x = x;
    rawGyroLive.y = y;
    rawGyroLive.z = z;

    rawGyroBuf.push({ x, y, z });
    if (rawGyroBuf.length > 120) rawGyroBuf.shift();

    if (isCapturing) {
      fullGyroLog.push({ ts: formatTs(), x, y, z });
    }
  }

  function onAccel(e: DeviceMotionEvent) {
    const aG  = e.accelerationIncludingGravity;
    const aL  = e.acceleration;
    const useLinear = aL !== null && (aL.x !== null || aL.y !== null || aL.z !== null);
    const src = useLinear ? aL : aG;
    const x   = src?.x ?? 0;
    const y   = src?.y ?? 0;
    const z   = src?.z ?? 0;
    const abs = Math.sqrt(x*x + y*y + z*z);

    rawAccelSource = useLinear ? 'linear' : (aG ? 'gravity' : 'none');
    rawAccelLive.x = x;
    rawAccelLive.y = y;
    rawAccelLive.z = z;
    rawAccelLive.abs = abs;

    rawAccelBuf.push({ x, y, z, abs });
    if (rawAccelBuf.length > 120) rawAccelBuf.shift();

    if (isCapturing) {
      fullAccelLog.push({ ts: formatTs(), x, y, z, abs });
    }
  }

  // ── Permission & start/stop ───────────────────────────────────────
  async function requestPermission() {
    permPending = true;
    errorMsg = '';
    try {
      // iOS 13+ requires explicit permission
      const DOE = DeviceOrientationEvent as unknown as {
        requestPermission?: () => Promise<string>;
      };
      const DME = DeviceMotionEvent as unknown as {
        requestPermission?: () => Promise<string>;
      };

      if (typeof DOE.requestPermission === 'function') {
        const res = await DOE.requestPermission();
        if (res !== 'granted') throw new Error('Orientation permission denied.');
      }
      if (typeof DME.requestPermission === 'function') {
        const res = await DME.requestPermission();
        if (res !== 'granted') throw new Error('Motion permission denied.');
      }

      window.addEventListener('deviceorientation', onGyro, true);
      window.addEventListener('devicemotion', onAccel, true);
      permGranted = true;
    } catch (err: unknown) {
      errorMsg = (err as Error).message ?? 'Permission error.';
      sensorSupported = false;
    } finally {
      permPending = false;
    }
  }

  function toggleCapture() {
    if (!permGranted) return;
    isCapturing = !isCapturing;
    if (isCapturing) {
      if (fullGyroLog.length === 0 && fullAccelLog.length === 0) {
        sessionStartTime = formatTs();
        sessionEndTime = null;
      }
    } else {
      sessionEndTime = formatTs();
    }
  }

  function clearAll() {
    fullGyroLog.length = 0;
    fullAccelLog.length = 0;
    gyroLogCount = 0;
    accelLogCount = 0;
    gyroLogPreview = [];
    accelLogPreview = [];
    sessionStartTime = null;
    sessionEndTime = null;
  }

  // ── Ripple effect ───────────────────────────────────────────────
  function ripple(e: MouseEvent) {
    const btn  = e.currentTarget as HTMLElement;
    const rect = btn.getBoundingClientRect();
    const span = document.createElement('span');
    span.className    = 'ripple-wave';
    span.style.left   = `${e.clientX - rect.left}px`;
    span.style.top    = `${e.clientY - rect.top}px`;
    btn.appendChild(span);
    span.addEventListener('animationend', () => span.remove(), { once: true });
  }

  function exportCSV() {
    // UTF-8 BOM so Excel opens without encoding issues
    const BOM = '\uFEFF';

    // Gyroscope section
    // Z (alpha) may be null when device has no magnetometer — shown as blank
    const gyroHeader = 'Section,Timestamp,Gyro_X_deg,Gyro_Y_deg,Gyro_Z_deg';
    const gyroRows   = fullGyroLog.map(
      r => `GYROSCOPE,${r.ts},${r.x.toFixed(4)},${r.y.toFixed(4)},${r.z !== null ? r.z.toFixed(4) : ''}`
    );

    // Accelerometer section
    const accelHeader = 'Section,Timestamp,Accel_X_ms2,Accel_Y_ms2,Accel_Z_ms2,Abs_ms2';
    const accelRows   = fullAccelLog.map(
      r => `ACCELEROMETER,${r.ts},${r.x.toFixed(4)},${r.y.toFixed(4)},${r.z.toFixed(4)},${(r.abs ?? 0).toFixed(4)}`
    );

    const lines = [
      '=== WAYIFY SENSOR EXPORT ===',
      `Session Start: ${sessionStartTime ?? 'N/A'}`,
      `Session End: ${sessionEndTime ?? 'N/A'}`,
      '',
      gyroHeader,
      ...gyroRows,
      '',
      accelHeader,
      ...accelRows,
    ];

    const csv  = BOM + lines.join('\n');
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href     = url;
    a.download = `wayify_${formatTs().replace(/[\s:]/g, '')}.wag`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }

  // ── Lifecycle ─────────────────────────────────────────────────────
  onMount(() => {
    rafId = requestAnimationFrame(tick);

    if (typeof DeviceOrientationEvent === 'undefined' &&
        typeof DeviceMotionEvent === 'undefined') {
      sensorSupported = false;
      errorMsg = 'Sensors not available on this device / browser.';
    } else {
      const DOE = DeviceOrientationEvent as unknown as {
        requestPermission?: () => Promise<string>;
      };
      if (typeof DOE.requestPermission !== 'function') {
        window.addEventListener('deviceorientation', onGyro, true);
        window.addEventListener('devicemotion', onAccel, true);
        permGranted = true;
      }
    }


  });

  onDestroy(() => {
    cancelAnimationFrame(rafId);
    window.removeEventListener('deviceorientation', onGyro, true);
    window.removeEventListener('devicemotion', onAccel, true);
  });

  // ── Svelte Action for Scroll Reveal ───────────────────────────────
  function reveal(node: HTMLElement) {
    node.classList.add('reveal');
    const io = new IntersectionObserver((entries) => {
      entries.forEach(e => {
        if (e.isIntersecting) {
          node.classList.add('visible');
          io.unobserve(node);
        }
      });
    }, { threshold: 0.12 });
    io.observe(node);
    return {
      destroy() {
        io.disconnect();
      }
    };
  }
</script>

<!-- ── Markup ──────────────────────────────────────────────────────── -->

<div class="shell" class:dark={darkMode}>

  <!-- Top Bar -->
  <header class="topbar">
    <div class="topbar__brand">
      <svg class="topbar__logo" viewBox="0 0 28 28" fill="none" aria-hidden="true">
        <circle cx="14" cy="14" r="13" stroke="currentColor" stroke-width="2"/>
        <path d="M8 14 L12 10 L16 18 L20 14" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
      <span class="topbar__name">Wayify</span>
    </div>
    <div class="topbar__right">
      <!-- Nav toggle -->
      <button
        class="nav-toggle"
        onclick={(e) => { ripple(e); currentPage = currentPage === 'live' ? 'vectroscope' : 'live'; }}
        aria-label="Switch page"
        title={currentPage === 'live' ? 'Go to VectroScope' : 'Go to Live Capture'}
      >
        {#if currentPage === 'live'}
          <svg style="margin-right: 6px; width:16px; height:16px;" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <polyline points="22 12 18 12 15 21 9 3 6 12 2 12"></polyline>
          </svg>
          VectroScope
        {:else}
          <svg style="margin-right: 6px; width:16px; height:16px;" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="2"></circle>
            <path d="M16.24 7.76a6 6 0 0 1 0 8.49m-8.48-.01a6 6 0 0 1 0-8.49m11.31-2.82a10 10 0 0 1 0 14.14m-14.14 0a10 10 0 0 1 0-14.14"></path>
          </svg>
          Live Data
        {/if}
      </button>

      <!-- Dark mode toggle -->
      <button
        id="btn-dark"
        class="dark-toggle"
        onclick={toggleDark}
        aria-label="Toggle dark mode"
        title={darkMode ? 'Switch to light mode' : 'Switch to dark mode'}
      >
        {#if darkMode}
          <!-- Sun icon -->
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="5"/>
            <line x1="12" y1="1" x2="12" y2="3"/><line x1="12" y1="21" x2="12" y2="23"/>
            <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"/><line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/>
            <line x1="1" y1="12" x2="3" y2="12"/><line x1="21" y1="12" x2="23" y2="12"/>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"/><line x1="18.36" y1="5.64" x2="19.78" y2="4.22"/>
          </svg>
        {:else}
          <!-- Moon icon -->
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
          </svg>
        {/if}
      </button>
      <div class="topbar__status">
        <span class="status-dot" class:active={isCapturing}></span>
        <span class="status-label">{isCapturing ? 'LIVE' : 'IDLE'}</span>
      </div>
    </div>
  </header>

  {#if currentPage === 'live'}
    <!-- Page -->
    <main class="page">

    <!-- ── Sensor Live Cards ─────────────────── -->
    <section class="section">
      <h2 class="section__title">Live Readout</h2>
      <div class="card-row">

        <!-- Gyroscope card -->
        <div class="sensor-card">
          <div class="sensor-card__header">
            <span class="sensor-card__icon gyro-icon">⟳</span>
            <div>
              <div class="sensor-card__title">Gyroscope</div>
              <div class="sensor-card__subtitle">Orientation · deg</div>
            </div>
          </div>
          <div class="axis-grid">
            <div class="axis-item">
              <span class="axis-label x">X</span>
              <span class="axis-value">{fix(gyroLive.x)}</span>
            </div>
            <div class="axis-item">
              <span class="axis-label y">Y</span>
              <span class="axis-value">{fix(gyroLive.y)}</span>
            </div>
            <div class="axis-item">
              <span class="axis-label z">Z</span>
              <span class="axis-value">{gyroLive.z !== null ? fix(gyroLive.z) : 'N/A'}</span>
            </div>
          </div>
        </div>

        <!-- Accelerometer card -->
        <div class="sensor-card">
          <div class="sensor-card__header">
            <span class="sensor-card__icon accel-icon">↗</span>
            <div>
              <div class="sensor-card__title">Accelerometer</div>
              <div class="sensor-card__subtitle">
                {accelSource === 'linear' ? 'Linear · m/s²' : accelSource === 'gravity' ? 'incl. Gravity · m/s²' : 'Motion · m/s²'}
              </div>
            </div>
          </div>
          <div class="axis-grid">
            <div class="axis-item">
              <span class="axis-label x">X</span>
              <span class="axis-value">{fix(accelLive.x)}</span>
            </div>
            <div class="axis-item">
              <span class="axis-label y">Y</span>
              <span class="axis-value">{fix(accelLive.y)}</span>
            </div>
            <div class="axis-item">
              <span class="axis-label z">Z</span>
              <span class="axis-value">{fix(accelLive.z)}</span>
            </div>
            <div class="axis-item span-full">
              <span class="axis-label abs">|A|</span>
              <span class="axis-value abs-value">{fix(accelLive.abs)}</span>
            </div>
          </div>
        </div>

      </div>
    </section>

    <!-- ── Controls ──────────────────────────── -->
    <section class="controls card" use:reveal>
      {#if !permGranted}
        <button
          id="btn-permission"
          class="btn btn--primary"
          style="width: 100%;"
          disabled={permPending}
          onclick={(e) => { ripple(e); requestPermission(); }}
        >
          <span class="btn__icon">🔓</span>
          {permPending ? 'Requesting…' : 'Enable Sensors'}
        </button>
      {:else}
        <div class="controls-row">
          <!-- Modern pill start/stop button -->
          <button
            id="btn-toggle"
            class="btn-capture"
            class:is-recording={isCapturing}
            onclick={(e) => { ripple(e); toggleCapture(); }}
            aria-label={isCapturing ? 'Stop capture' : 'Start capture'}
          >
            <span class="btn-capture__ring"></span>
            <span class="btn-capture__icon" style="display:flex;align-items:center;">
              {#if isCapturing}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="6" width="12" height="12" rx="2"/></svg>
              {:else}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><circle cx="12" cy="12" r="8"/></svg>
              {/if}
            </span>
            <span class="btn-capture__label">{isCapturing ? 'Stop' : 'Start'}</span>
          </button>

          <div class="btn-group">
            <button id="btn-clear" class="btn btn--ghost" onclick={(e) => { ripple(e); clearAll(); }}>
              <span style="display:flex;align-items:center;">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="margin-right:6px;"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
              </span>
              Clear
            </button>
            <button id="btn-export" class="btn btn--outline" onclick={(e) => { ripple(e); exportCSV(); }}
              disabled={gyroLogCount === 0 && accelLogCount === 0}>
              <span style="display:flex;align-items:center;">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="margin-right:6px;"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path><polyline points="7 10 12 15 17 10"></polyline><line x1="12" y1="15" x2="12" y2="3"></line></svg>
              </span>
              Export
            </button>
          </div>
        </div>

        {#if sessionStartTime}
          <div class="session-times">
            <div class="time-pill">
              <span class="time-label">Start:</span>
              <span class="time-val">{sessionStartTime.split(' - ')[1]}</span>
            </div>
            {#if sessionEndTime}
              <div class="time-pill">
                <span class="time-label">End:</span>
                <span class="time-val">{sessionEndTime.split(' - ')[1]}</span>
              </div>
            {/if}
          </div>
        {/if}
      {/if}
    </section>

    <!-- Error banner -->
    {#if errorMsg}
      <div class="error-banner" use:reveal role="alert">⚠ {errorMsg}</div>
    {/if}

    <!-- Sensor not supported notice -->
    {#if !sensorSupported}
      <div class="info-banner" use:reveal>
        ℹ  Sensors not available. Open on a mobile browser for live data.
      </div>
    {/if}

    <!-- ── Live Charts ─────────────────────────── -->
    <section class="charts-section" use:reveal>
      <h2 class="section__title">Live Graphs</h2>
      <div class="chart-row">

        <!-- Gyroscope chart -->
        <div class="chart-card">
          <div class="chart-card__head">
            <div class="chart-card__label">
              <span class="chart-card__dot gyro-dot"></span>
              Gyroscope · X Y Z
            </div>
            <div class="chart-legend">
              <span class="leg-item x">X</span>
              <span class="leg-item y">Y</span>
              <span class="leg-item z">Z</span>
            </div>
          </div>
          <div class="chart-canvas-wrap">
            <LiveChart points={gyroBuf} dark={darkMode} maxPoints={100} label="Gyroscope XYZ chart" />
          </div>
        </div>

        <!-- Accelerometer chart -->
        <div class="chart-card">
          <div class="chart-card__head">
            <div class="chart-card__label">
              <span class="chart-card__dot accel-dot"></span>
              Accelerometer · X Y Z |A|
            </div>
            <div class="chart-legend">
              <span class="leg-item x">X</span>
              <span class="leg-item y">Y</span>
              <span class="leg-item z">Z</span>
              <span class="leg-item abs">|A|</span>
            </div>
          </div>
          <div class="chart-canvas-wrap">
            <LiveChart points={accelBuf} showAbs={true} dark={darkMode} maxPoints={100} label="Accelerometer XYZ chart" />
          </div>
        </div>

      </div>
    </section>

    <!-- ── Logs ────────────────────────────────── -->
    <div class="log-row">
      <!-- Gyroscope log -->
      <section class="log-panel" use:reveal>
        <div class="log-panel__head">
          <h3 class="log-panel__title">Gyroscope Log</h3>
          <span class="log-panel__count">{gyroLogCount} entries</span>
        </div>
        <div class="log-table-wrap" bind:this={gyroRef}>
          {#if gyroLogPreview.length === 0}
            <div class="log-empty">No data captured. Press Start.</div>
          {:else}
            <table class="log-table">
              <thead>
                <tr>
                  <th>Timestamp</th>
                  <th class="col-x">X °</th>
                  <th class="col-y">Y °</th>
                  <th class="col-z">Z °</th>
                </tr>
              </thead>
              <tbody>
                {#each gyroLogPreview as row, i (i)}
                  <tr>
                    <td class="ts-cell">{row.ts}</td>
                    <td class="col-x">{fix(row.x)}</td>
                    <td class="col-y">{fix(row.y)}</td>
                    <td class="col-z">{row.z !== null ? fix(row.z) : '—'}</td>
                  </tr>
                {/each}
              </tbody>
            </table>
          {/if}
        </div>
      </section>

      <!-- Accelerometer log -->
      <section class="log-panel" use:reveal>
        <div class="log-panel__head">
          <h3 class="log-panel__title">Accelerometer Log</h3>
          <span class="log-panel__count">{accelLogCount} entries</span>
        </div>
        <div class="log-table-wrap" bind:this={accelRef}>
          {#if accelLogPreview.length === 0}
            <div class="log-empty">No data captured. Press Start.</div>
          {:else}
            <table class="log-table">
              <thead>
                <tr>
                  <th>Timestamp</th>
                  <th class="col-x">X</th>
                  <th class="col-y">Y</th>
                  <th class="col-z">Z</th>
                  <th class="col-abs">|A|</th>
                </tr>
              </thead>
              <tbody>
                {#each accelLogPreview as row, i (i)}
                  <tr>
                    <td class="ts-cell">{row.ts}</td>
                    <td class="col-x">{fix(row.x)}</td>
                    <td class="col-y">{fix(row.y)}</td>
                    <td class="col-z">{fix(row.z)}</td>
                    <td class="col-abs">{fix(row.abs ?? 0)}</td>
                  </tr>
                {/each}
              </tbody>
            </table>
          {/if}
        </div>
      </section>

    </div><!-- /log-row -->

  </main><!-- /page -->
  {:else}
    <VectroScope {darkMode} />
  {/if}

  <!-- Footer -->
  <footer class="footer">
    <span>Wayify Sensor Suite</span>
    <span class="footer__sep">·</span>
    <span>Timestamp: DDMMYYYY – HH:MM:SS:mmm (24h)</span>
  </footer>

</div><!-- /shell -->


<!-- ── Styles ──────────────────────────────────────────────────────── -->
<style>
  /* Shell */
  .shell {
    display: flex;
    flex-direction: column;
    min-height: 100dvh;
    background: var(--color-bg);
  }

  /* ── Top Bar ─────────────────────────────── */
  .topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 20px;
    height: 56px;
    background: var(--color-brand-dark);
    color: var(--color-text-on-dark);
    box-shadow: var(--shadow-md);
    position: sticky;
    top: 0;
    z-index: 100;
  }

  .topbar__brand {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .topbar__logo {
    width: 26px;
    height: 26px;
    color: var(--color-brand-accent);
    flex-shrink: 0;
  }

  .topbar__name {
    font-size: 18px;
    font-weight: 700;
    letter-spacing: 0.04em;
    color: #fff;
  }

  .topbar__status {
    display: flex;
    align-items: center;
    gap: 7px;
  }

  .topbar__right {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  /* ── Nav toggle ────────────────────── */
  .nav-toggle {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 34px;
    padding: 0 14px;
    border-radius: var(--radius-pill);
    background: rgba(255,255,255,0.10);
    border: 1px solid rgba(255,255,255,0.18);
    color: rgba(255,255,255,0.9);
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
    transition: background var(--transition-fast), transform var(--transition-fast), box-shadow var(--transition-fast);
    position: relative;
    overflow: hidden;
  }
  .nav-toggle:hover {
    background: rgba(255,255,255,0.18);
    transform: translateY(-1px);
    box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  }

  /* ── Dark mode toggle ────────────────────── */
  .dark-toggle {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    border-radius: 50%;
    background: rgba(255,255,255,0.10);
    border: 1px solid rgba(255,255,255,0.18);
    color: rgba(255,255,255,0.85);
    cursor: pointer;
    transition: background var(--transition-fast), transform var(--transition-fast), box-shadow var(--transition-fast);
    padding: 0;
    flex-shrink: 0;
  }

  .dark-toggle svg {
    width: 16px;
    height: 16px;
    transition: transform var(--transition-normal);
  }

  .dark-toggle:hover {
    background: rgba(255,255,255,0.20);
    box-shadow: 0 0 0 3px rgba(0,200,215,0.25);
    transform: rotate(15deg);
  }

  .dark-toggle:active { transform: scale(0.92); }

  .status-dot {
    width: 9px;
    height: 9px;
    border-radius: 50%;
    background: var(--color-text-muted);
    transition: background var(--transition-normal);
  }

  .status-dot.active {
    background: var(--color-live);
    box-shadow: 0 0 0 3px rgba(0,200,83,0.30);
    animation: pulse-dot 1.5s infinite;
  }

  @keyframes pulse-dot {
    0%,100% { box-shadow: 0 0 0 3px rgba(0,200,83,0.30); }
    50%      { box-shadow: 0 0 0 6px rgba(0,200,83,0.10); }
  }

  .status-label {
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.1em;
    color: rgba(255,255,255,0.75);
  }

  /* ── Page ────────────────────────────────── */
  .page {
    flex: 1;
    padding: 20px 16px 32px;
    max-width: 860px;
    margin: 0 auto;
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  /* ── Section ─────────────────────────────── */
  .section__title {
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: var(--color-text-muted);
    margin-bottom: 10px;
  }

  /* ── Sensor Cards ────────────────────────── */
  .card-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
  }

  @media (max-width: 520px) {
    .card-row { grid-template-columns: 1fr; }
  }

  .sensor-card {
    background: var(--color-surface);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-md);
    padding: 18px;
    border-top: 3px solid var(--color-brand-secondary);
    transition: box-shadow var(--transition-normal), transform var(--transition-normal);
  }

  .sensor-card:hover {
    box-shadow: var(--shadow-lg);
    transform: translateY(-2px);
  }

  .sensor-card__header {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;
  }

  .sensor-card__icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 38px;
    height: 38px;
    border-radius: var(--radius-md);
    font-size: 20px;
    flex-shrink: 0;
  }

  .gyro-icon {
    background: rgba(0,160,175,0.12);
    color: var(--color-brand-secondary);
    font-style: normal;
  }

  .accel-icon {
    background: rgba(58,123,213,0.12);
    color: var(--color-z);
  }

  .sensor-card__title {
    font-size: 14px;
    font-weight: 600;
    color: var(--color-text-primary);
  }

  .sensor-card__subtitle {
    font-size: 11px;
    color: var(--color-text-muted);
    margin-top: 2px;
  }

  /* Axis grid */
  .axis-grid {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
    gap: 8px;
  }

  .span-full { grid-column: 1 / -1; }

  .axis-item {
    background: var(--color-bg);
    border-radius: var(--radius-md);
    padding: 10px 8px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    border: 1px solid var(--color-border-subtle);
  }

  .axis-label {
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    padding: 2px 6px;
    border-radius: var(--radius-sm);
  }

  .axis-label.x   { background: rgba(230,53,89,0.12);  color: var(--color-x); }
  .axis-label.y   { background: rgba(0,168,120,0.12);  color: var(--color-y); }
  .axis-label.z   { background: rgba(58,123,213,0.12); color: var(--color-z); }
  .axis-label.abs { background: rgba(245,166,35,0.12); color: var(--color-abs); }

  .axis-value {
    font-family: var(--font-mono);
    font-size: 13px;
    font-weight: 500;
    color: var(--color-text-primary);
  }

  .abs-value { font-size: 14px; font-weight: 600; }

  /* ── Controls layout ─────────────────────── */
  .controls.card {
    background: var(--color-surface);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-sm);
    border: 1px solid var(--color-border-subtle);
    padding: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
  }

  .controls-row {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
    justify-content: center;
    width: 100%;
  }

  .btn-group {
    display: flex;
    gap: 12px;
    flex: 1;
    min-width: 240px;
  }

  .btn-group .btn {
    flex: 1;
  }

  /* ── Generic Button ── */
  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    padding: 0 20px;
    height: 42px;
    border-radius: var(--radius-pill);
    font-family: var(--font-family);
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    border: 1px solid transparent;
    transition: all var(--transition-normal);
    white-space: nowrap;
    position: relative;
    overflow: hidden;
  }

  .btn:disabled { opacity: 0.4; cursor: not-allowed; }

  .btn--primary {
    background: var(--color-brand-primary);
    color: #fff;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }
  .btn--primary:hover:not(:disabled) {
    box-shadow: 0 3px 8px rgba(0,0,0,0.15);
    background: var(--color-brand-secondary);
  }

  .btn--outline {
    background: var(--color-surface);
    border-color: var(--color-border);
    color: var(--color-text-primary);
    box-shadow: 0 1px 2px rgba(0,0,0,0.04);
  }
  .btn--outline:hover:not(:disabled) {
    background: var(--color-bg);
  }

  .btn--ghost {
    background: transparent;
    color: var(--color-text-secondary);
    border-color: transparent;
  }
  .btn--ghost:hover:not(:disabled) { 
    background: rgba(128,128,128,0.1); 
  }

  /* ── Hero Capture Button ─────────────────── */
  .btn-capture {
    flex: 1;
    min-width: 140px;
    position: relative;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 0 24px;
    height: 42px;
    border-radius: var(--radius-pill);
    font-family: var(--font-family);
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    border: none;
    outline: none;
    background: var(--color-brand-primary);
    color: #ffffff;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    transition: all var(--transition-normal);
    user-select: none;
    -webkit-tap-highlight-color: transparent;
  }

  .btn-capture:hover {
    box-shadow: 0 3px 8px rgba(0,0,0,0.15);
  }

  .btn-capture:active { 
    transform: scale(0.97); 
  }

  /* Recording state: Apple matte red */
  .btn-capture.is-recording {
    background: #ff3b30; 
    box-shadow: 0 1px 3px rgba(255,59,48,0.2);
  }
  .btn-capture.is-recording:hover {
    box-shadow: 0 3px 8px rgba(255,59,48,0.3);
  }

  /* Animated pulse ring on record */
  .btn-capture__ring {
    position: absolute;
    inset: 0;
    border-radius: inherit;
    border: 2px solid rgba(255,255,255,0.30);
    opacity: 0;
    pointer-events: none;
  }
  .btn-capture.is-recording .btn-capture__ring {
    animation: capture-ring 1.8s cubic-bezier(0.4,0,0.6,1) infinite;
  }

  @keyframes capture-ring {
    0%   { opacity: 0.8; transform: scale(1);    }
    100% { opacity: 0;   transform: scale(1.18); }
  }

  .btn-capture__icon {
    font-size: 18px;
    line-height: 1;
    transition: transform var(--transition-normal);
  }
  .btn-capture.is-recording .btn-capture__icon { transform: scale(1.1); }

  .btn-capture__label {
    font-size: 15px;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  /* Session Times */
  .session-times {
    display: flex;
    gap: 12px;
    margin-top: 16px;
    width: 100%;
    justify-content: center;
    flex-wrap: wrap;
  }
  .time-pill {
    background: var(--color-surface);
    border: 1px solid var(--color-border-subtle);
    padding: 6px 12px;
    border-radius: var(--radius-pill);
    font-size: 11px;
    display: flex;
    align-items: center;
    gap: 6px;
    box-shadow: var(--shadow-sm);
  }
  .time-label {
    color: var(--color-text-muted);
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }
  .time-val {
    font-family: var(--font-mono);
    color: var(--color-brand-primary);
    font-weight: 700;
  }

  /* ── Error & Info Banners ──────────────────────────── */
  .error-banner,
  .info-banner {
    padding: 12px 16px;
    border-radius: var(--radius-md);
    font-size: 13px;
    font-weight: 500;
  }

  .error-banner {
    background: rgba(220,53,69,0.08);
    border: 1px solid rgba(220,53,69,0.3);
    color: var(--color-danger);
  }

  .info-banner {
    background: rgba(0,160,175,0.07);
    border: 1px solid rgba(0,160,175,0.25);
    color: var(--color-brand-primary);
  }

  /* ── Live Charts ─────────────────────────── */
  .charts-section { }

  .chart-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
  }

  @media (max-width: 600px) {
    .chart-row { grid-template-columns: 1fr; }
  }

  .chart-card {
    background: var(--color-surface);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-md);
    border: 1px solid var(--color-border-subtle);
    overflow: hidden;
    transition: box-shadow var(--transition-normal), transform var(--transition-normal);
  }

  .chart-card:hover {
    transform: translateY(-2px);
    box-shadow: var(--shadow-lg);
  }

  .chart-card__head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 12px 14px 8px;
    border-bottom: 1px solid var(--color-border-subtle);
  }

  .chart-card__label {
    display: flex;
    align-items: center;
    gap: 7px;
    font-size: 12px;
    font-weight: 600;
    color: var(--color-text-secondary);
    letter-spacing: 0.02em;
  }

  .chart-card__dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .gyro-dot  { background: var(--color-brand-accent); box-shadow: 0 0 6px var(--color-brand-accent); }
  .accel-dot { background: var(--color-z); box-shadow: 0 0 6px var(--color-z); }

  .chart-legend {
    display: flex;
    gap: 6px;
  }

  .leg-item {
    font-size: 10px;
    font-weight: 700;
    padding: 2px 7px;
    border-radius: var(--radius-pill);
    letter-spacing: 0.05em;
  }
  .leg-item.x   { background: rgba(255,82,82,0.14);  color: var(--color-x); }
  .leg-item.y   { background: rgba(0,230,118,0.14);  color: var(--color-y); }
  .leg-item.z   { background: rgba(68,138,255,0.14); color: var(--color-z); }
  .leg-item.abs { background: rgba(255,171,64,0.14); color: var(--color-abs); }

  .chart-canvas-wrap {
    height: 180px;
    padding: 6px 6px 4px;
  }


  @media (max-width: 600px) {
    .chart-canvas-wrap { height: 160px; }
  }



  /* ── Log Row ─────────────────────────────── */
  .log-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
    flex: 1;
  }

  @media (max-width: 620px) {
    .log-row { grid-template-columns: 1fr; }
  }

  .log-panel {
    background: var(--color-surface);
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-sm);
    display: flex;
    flex-direction: column;
    overflow: hidden;
    border: 1px solid var(--color-border-subtle);
  }

  .log-panel__head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 16px;
    border-bottom: 1px solid var(--color-border-subtle);
    background: linear-gradient(to right, rgba(0,98,114,0.04), transparent);
  }

  .log-panel__title {
    font-size: 13px;
    font-weight: 600;
    color: var(--color-brand-primary);
  }

  .log-panel__count {
    font-size: 11px;
    font-weight: 500;
    color: var(--color-text-muted);
    background: var(--color-bg);
    padding: 2px 8px;
    border-radius: 20px;
    border: 1px solid var(--color-border);
  }

  .log-table-wrap {
    overflow-y: auto;
    max-height: 320px;
    min-height: 80px;
  }

  .log-empty {
    padding: 32px 16px;
    text-align: center;
    color: var(--color-text-muted);
    font-size: 13px;
  }

  /* ── Log Table ───────────────────────────── */
  .log-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 12px;
  }

  .log-table thead {
    position: sticky;
    top: 0;
    z-index: 2;
  }

  .log-table th {
    background: var(--color-bg);
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: var(--color-text-muted);
    padding: 8px 10px;
    text-align: right;
    border-bottom: 1px solid var(--color-border);
    white-space: nowrap;
  }

  .log-table th:first-child { text-align: left; }

  .log-table td {
    padding: 7px 10px;
    text-align: right;
    border-bottom: 1px solid var(--color-border-subtle);
    font-family: var(--font-mono);
    color: var(--color-text-secondary);
  }

  .log-table tbody tr:last-child td { border-bottom: none; }

  .log-table tbody tr:hover td {
    background: rgba(0,98,114,0.03);
  }

  .ts-cell {
    text-align: left !important;
    color: var(--color-text-muted) !important;
    font-size: 11px !important;
    white-space: nowrap;
  }

  .col-x { color: var(--color-x) !important; }
  .col-y { color: var(--color-y) !important; }
  .col-z { color: var(--color-z) !important; }
  .col-abs { color: var(--color-abs) !important; }

  /* ── Footer ──────────────────────────────── */
  .footer {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 14px 16px;
    font-size: 11px;
    color: var(--color-text-muted);
    border-top: 1px solid var(--color-border-subtle);
    background: var(--color-surface);
    flex-wrap: wrap;
    text-align: center;
  }

  .footer__sep { opacity: 0.4; }

  /* ── Dark mode overrides ─────────────────── */

  /* Table row hover */
  :global(.dark) .log-table tbody tr:hover td {
    background: rgba(0,200,215,0.04);
  }

  /* Axis item background */
  :global(.dark) .axis-item {
    background: var(--color-border-subtle);
    border-color: var(--color-border);
  }

  /* Log panel header gradient */
  :global(.dark) .log-panel__head {
    background: linear-gradient(to right, rgba(0,200,215,0.06), transparent);
  }

  /* Log panel count badge */
  :global(.dark) .log-panel__count {
    background: var(--color-border);
  }

  /* Sensor card top border */
  :global(.dark) .sensor-card {
    border-top-color: var(--color-brand-secondary);
  }

  /* Ghost button */
  :global(.dark) .btn--ghost:hover {
    background: var(--color-border);
  }

  /* Outline button */
  :global(.dark) .btn--outline {
    border-color: var(--color-brand-accent);
    color: var(--color-brand-accent);
  }
  :global(.dark) .btn--outline:hover:not(:disabled) {
    background: rgba(0,200,215,0.08);
  }

  /* Error / info banners */
  :global(.dark) .error-banner {
    background: rgba(220,53,69,0.12);
    border-color: rgba(220,53,69,0.35);
  }
  :global(.dark) .info-banner {
    background: rgba(0,200,215,0.08);
    border-color: rgba(0,200,215,0.25);
    color: var(--color-brand-accent);
  }

  /* Log table sticky header */
  :global(.dark) .log-table thead th {
    background: var(--color-surface);
  }

</style>
