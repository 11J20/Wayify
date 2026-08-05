<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  // ── Types ─────────────────────────────────────────────────────────
  interface SensorEntry {
    ts: string;   // DDMMYYYY - HH:MM
    x:  number;
    y:  number;
    z:  number;
    abs?: number; // only for accelerometer
  }

  // ── State ─────────────────────────────────────────────────────────
  let gyroLog:  SensorEntry[] = $state([]);
  let accelLog: SensorEntry[] = $state([]);

  let gyroLive  = $state({ x: 0, y: 0, z: 0 });
  let accelLive = $state({ x: 0, y: 0, z: 0, abs: 0 });

  let isCapturing  = $state(false);
  let permGranted  = $state(false);
  let permPending  = $state(false);
  let errorMsg     = $state('');
  let sensorSupported = $state(true);

  let gyroRef:  HTMLDivElement;
  let accelRef: HTMLDivElement;

  // ── Helpers ───────────────────────────────────────────────────────
  function formatTs(): string {
    const now = new Date();
    const dd   = String(now.getDate()).padStart(2, '0');
    const mm   = String(now.getMonth() + 1).padStart(2, '0');
    const yyyy = now.getFullYear();
    const hh   = String(now.getHours()).padStart(2, '0');
    const min  = String(now.getMinutes()).padStart(2, '0');
    return `${dd}${mm}${yyyy} - ${hh}:${min}`;
  }

  function fix(n: number) { return n.toFixed(4); }

  function scrollBottom(el?: HTMLDivElement) {
    if (el) el.scrollTop = el.scrollHeight;
  }

  // ── Sensor handlers ───────────────────────────────────────────────
  function onGyro(e: DeviceOrientationEvent) {
    gyroLive = {
      x: e.beta  ?? 0,
      y: e.gamma ?? 0,
      z: e.alpha ?? 0,
    };
    if (isCapturing) {
      gyroLog = [...gyroLog, {
        ts: formatTs(),
        x: gyroLive.x,
        y: gyroLive.y,
        z: gyroLive.z,
      }];
      // keep latest 200 entries
      if (gyroLog.length > 200) gyroLog = gyroLog.slice(-200);
      scrollBottom(gyroRef);
    }
  }

  function onAccel(e: DeviceMotionEvent) {
    const a = e.acceleration ?? e.accelerationIncludingGravity;
    const x = a?.x ?? 0;
    const y = a?.y ?? 0;
    const z = a?.z ?? 0;
    const abs = Math.sqrt(x*x + y*y + z*z);
    accelLive = { x, y, z, abs };
    if (isCapturing) {
      accelLog = [...accelLog, {
        ts: formatTs(),
        x, y, z, abs,
      }];
      if (accelLog.length > 200) accelLog = accelLog.slice(-200);
      scrollBottom(accelRef);
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
  }

  function clearAll() {
    gyroLog  = [];
    accelLog = [];
  }

  function exportCSV() {
    const gyroLines  = ['Timestamp,Gyro_X(deg),Gyro_Y(deg),Gyro_Z(deg)',
      ...gyroLog.map(r => `${r.ts},${r.x},${r.y},${r.z}`)];
    const accelLines = ['Timestamp,Accel_X(m/s²),Accel_Y(m/s²),Accel_Z(m/s²),Abs(m/s²)',
      ...accelLog.map(r => `${r.ts},${r.x},${r.y},${r.z},${r.abs}`)];
    const csv = [...gyroLines, '', ...accelLines].join('\n');
    const blob = new Blob([csv], { type: 'text/csv' });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href     = url;
    a.download = `wayify_${formatTs().replace(/\s|:/g, '')}.csv`;
    a.click();
    URL.revokeObjectURL(url);
  }

  // ── Lifecycle ─────────────────────────────────────────────────────
  onMount(() => {
    if (typeof DeviceOrientationEvent === 'undefined' &&
        typeof DeviceMotionEvent === 'undefined') {
      sensorSupported = false;
      errorMsg = 'Sensors not available on this device / browser.';
    } else {
      // On Android / desktop no permission call needed — try attaching directly
      const DOE = DeviceOrientationEvent as unknown as {
        requestPermission?: () => Promise<string>;
      };
      if (typeof DOE.requestPermission !== 'function') {
        // non-iOS: just attach listeners
        window.addEventListener('deviceorientation', onGyro, true);
        window.addEventListener('devicemotion', onAccel, true);
        permGranted = true;
      }
    }
  });

  onDestroy(() => {
    window.removeEventListener('deviceorientation', onGyro, true);
    window.removeEventListener('devicemotion', onAccel, true);
  });
</script>

<!-- ── Markup ──────────────────────────────────────────────────────── -->

<div class="shell">

  <!-- Top Bar -->
  <header class="topbar">
    <div class="topbar__brand">
      <svg class="topbar__logo" viewBox="0 0 28 28" fill="none" aria-hidden="true">
        <circle cx="14" cy="14" r="13" stroke="currentColor" stroke-width="2"/>
        <path d="M8 14 L12 10 L16 18 L20 14" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
      <span class="topbar__name">Wayify</span>
    </div>
    <div class="topbar__status">
      <span class="status-dot" class:active={isCapturing}></span>
      <span class="status-label">{isCapturing ? 'LIVE' : 'IDLE'}</span>
    </div>
  </header>

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
              <span class="axis-value">{fix(gyroLive.z)}</span>
            </div>
          </div>
        </div>

        <!-- Accelerometer card -->
        <div class="sensor-card">
          <div class="sensor-card__header">
            <span class="sensor-card__icon accel-icon">↗</span>
            <div>
              <div class="sensor-card__title">Accelerometer</div>
              <div class="sensor-card__subtitle">Motion · m/s²</div>
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
    <section class="controls">
      {#if !permGranted}
        <button
          id="btn-permission"
          class="btn btn--primary"
          disabled={permPending}
          onclick={requestPermission}
        >
          {permPending ? 'Requesting…' : '🔓  Enable Sensors'}
        </button>
      {:else}
        <button
          id="btn-toggle"
          class="btn"
          class:btn--record={!isCapturing}
          class:btn--stop={isCapturing}
          onclick={toggleCapture}
        >
          {isCapturing ? '⏹  Stop Capture' : '⏺  Start Capture'}
        </button>
        <button id="btn-export" class="btn btn--outline" onclick={exportCSV}
          disabled={gyroLog.length === 0 && accelLog.length === 0}>
          ⬇  Export CSV
        </button>
        <button id="btn-clear" class="btn btn--ghost" onclick={clearAll}>
          🗑  Clear
        </button>
      {/if}
    </section>

    <!-- Error banner -->
    {#if errorMsg}
      <div class="error-banner" role="alert">
        ⚠ {errorMsg}
      </div>
    {/if}

    <!-- Sensor not supported notice -->
    {#if !sensorSupported}
      <div class="info-banner">
        ℹ  This browser or device does not expose gyroscope / accelerometer APIs.
        Try opening this page in a mobile browser.
      </div>
    {/if}

    <!-- ── Log Tables ─────────────────────────── -->
    <div class="log-row">

      <!-- Gyroscope log -->
      <section class="log-panel">
        <div class="log-panel__head">
          <h3 class="log-panel__title">Gyroscope Log</h3>
          <span class="log-panel__count">{gyroLog.length} entries</span>
        </div>
        <div class="log-table-wrap" bind:this={gyroRef}>
          {#if gyroLog.length === 0}
            <div class="log-empty">No data yet. Start capturing to record.</div>
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
                {#each gyroLog as row, i (i)}
                  <tr>
                    <td class="ts-cell">{row.ts}</td>
                    <td class="col-x">{fix(row.x)}</td>
                    <td class="col-y">{fix(row.y)}</td>
                    <td class="col-z">{fix(row.z)}</td>
                  </tr>
                {/each}
              </tbody>
            </table>
          {/if}
        </div>
      </section>

      <!-- Accelerometer log -->
      <section class="log-panel">
        <div class="log-panel__head">
          <h3 class="log-panel__title">Accelerometer Log</h3>
          <span class="log-panel__count">{accelLog.length} entries</span>
        </div>
        <div class="log-table-wrap" bind:this={accelRef}>
          {#if accelLog.length === 0}
            <div class="log-empty">No data yet. Start capturing to record.</div>
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
                {#each accelLog as row, i (i)}
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

  <!-- Footer -->
  <footer class="footer">
    <span>Wayify Sensor Suite</span>
    <span class="footer__sep">·</span>
    <span>Timestamp format: DDMMYYYY – HH:MM (24h)</span>
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

  /* ── Controls ────────────────────────────── */
  .controls {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    align-items: center;
  }

  .btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 10px 20px;
    border-radius: var(--radius-md);
    font-family: var(--font-family);
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    border: 2px solid transparent;
    transition: all var(--transition-fast);
    white-space: nowrap;
  }

  .btn:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }

  .btn--primary {
    background: var(--color-brand-primary);
    color: #fff;
  }
  .btn--primary:hover:not(:disabled) { background: var(--color-brand-secondary); }

  .btn--record {
    background: var(--color-brand-primary);
    color: #fff;
  }
  .btn--record:hover:not(:disabled) { background: var(--color-brand-secondary); }

  .btn--stop {
    background: var(--color-danger);
    color: #fff;
    animation: record-glow 2s infinite;
  }

  @keyframes record-glow {
    0%,100% { box-shadow: 0 0 0 0 rgba(220,53,69,0.0); }
    50%      { box-shadow: 0 0 0 6px rgba(220,53,69,0.2); }
  }

  .btn--outline {
    background: transparent;
    border-color: var(--color-brand-primary);
    color: var(--color-brand-primary);
  }
  .btn--outline:hover:not(:disabled) {
    background: rgba(0,98,114,0.06);
  }

  .btn--ghost {
    background: transparent;
    color: var(--color-text-secondary);
    border-color: var(--color-border);
  }
  .btn--ghost:hover { background: var(--color-border-subtle); }

  /* ── Banners ─────────────────────────────── */
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
</style>
