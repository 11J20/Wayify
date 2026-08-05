<script lang="ts">
  import ScopeChart from './lib/ScopeChart.svelte';
  import type { ScopePoint } from './lib/ScopeChart.svelte';

  // Props
  interface Props {
    darkMode: boolean;
  }
  let { darkMode = false }: Props = $props();

  // State
  let fileInput: HTMLInputElement;
  let fileName = $state<string>('');
  let gyroData = $state<ScopePoint[]>([]);
  let accelData = $state<ScopePoint[]>([]);
  let errorMsg = $state<string>('');
  let dataKey = $state<number>(0); // bump to reset chart zoom
  let parsedStartTime = $state<string | null>(null);
  let parsedEndTime = $state<string | null>(null);

  // Parser
  function parseCSV(text: string) {
    const lines = text.split(/\r?\n/);
    const newGyro: ScopePoint[] = [];
    const newAccel: ScopePoint[] = [];

    for (const line of lines) {
      if (!line.trim() || line.startsWith('===')) continue;

      if (line.startsWith('Session Start:')) {
        const val = line.substring(14).trim();
        if (val !== 'N/A') parsedStartTime = val;
        continue;
      }
      if (line.startsWith('Session End:')) {
        const val = line.substring(12).trim();
        if (val !== 'N/A') parsedEndTime = val;
        continue;
      }

      if (line.startsWith('Section')) continue;

      const parts = line.split(',');
      if (parts.length < 5) continue;

      const type = parts[0];
      const ts = parts[1];

      if (type === 'GYROSCOPE') {
        const x = parseFloat(parts[2]);
        const y = parseFloat(parts[3]);
        const z = parts[4] ? parseFloat(parts[4]) : null;
        if (!isNaN(x) && !isNaN(y)) {
          newGyro.push({ ts, x, y, z });
        }
      } else if (type === 'ACCELEROMETER') {
        const x = parseFloat(parts[2]);
        const y = parseFloat(parts[3]);
        const z = parseFloat(parts[4]);
        const abs = parseFloat(parts[5]);
        if (!isNaN(x) && !isNaN(y) && !isNaN(z)) {
          newAccel.push({ ts, x, y, z, abs: isNaN(abs) ? undefined : abs });
        }
      }
    }

    if (newGyro.length === 0 && newAccel.length === 0) {
      errorMsg = 'No valid sensor data found in the CSV file.';
      return;
    }

    gyroData = newGyro;
    accelData = newAccel;
    dataKey++;
  }

  function handleFileSelect(e: Event) {
    const target = e.target as HTMLInputElement;
    const file = target.files?.[0];
    if (!file) return;

    fileName = file.name;
    errorMsg = '';
    gyroData = [];
    accelData = [];
    parsedStartTime = null;
    parsedEndTime = null;

    const reader = new FileReader();
    reader.onload = (ev) => {
      const text = ev.target?.result;
      if (typeof text === 'string') {
        parseCSV(text);
      }
    };
    reader.onerror = () => {
      errorMsg = 'Failed to read file.';
    };
    reader.readAsText(file);
  }

  function triggerFileInput() {
    fileInput.click();
  }
</script>

<main class="page reveal visible">
  <div class="hero">
    <h1 class="hero__title">VectroScope</h1>
    <p class="hero__subtitle">Import and analyze exported Wayify sensor sessions</p>

    <div class="hero__actions">
      <input
        type="file"
        accept=".wag,.csv,text/csv,text/plain,application/csv,application/octet-stream"
        bind:this={fileInput}
        onchange={handleFileSelect}
        style="display: none;"
      />
      <button class="btn btn--primary btn-upload" onclick={triggerFileInput}>
        <span class="btn__icon">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
            <polyline points="17 8 12 3 7 8"></polyline>
            <line x1="12" y1="3" x2="12" y2="15"></line>
          </svg>
        </span>
        Select .wag File
      </button>
      {#if fileName}
        <span class="file-name">{fileName}</span>
      {/if}
    </div>

    {#if parsedStartTime}
      <div class="session-times">
        <div class="time-pill">
          <span class="time-label">Start:</span>
          <span class="time-val">{parsedStartTime.includes(' - ') ? parsedStartTime.split(' - ')[1] : parsedStartTime}</span>
        </div>
        {#if parsedEndTime}
          <div class="time-pill">
            <span class="time-label">End:</span>
            <span class="time-val">{parsedEndTime.includes(' - ') ? parsedEndTime.split(' - ')[1] : parsedEndTime}</span>
          </div>
        {/if}
      </div>
    {/if}
  </div>

  {#if errorMsg}
    <div class="error-banner reveal visible" role="alert">⚠ {errorMsg}</div>
  {/if}

  {#if gyroData.length > 0 || accelData.length > 0}
    <section class="charts-section reveal visible">
      <div class="chart-row">
        <!-- Gyroscope Scope -->
        <div class="chart-card">
          <div class="chart-card__head">
            <div class="chart-card__label">
              <span class="chart-card__dot gyro-dot"></span>
              Gyroscope Telemetry ({gyroData.length} pts)
            </div>
            <div class="chart-legend">
              <span class="leg-item x">X</span>
              <span class="leg-item y">Y</span>
              <span class="leg-item z">Z</span>
            </div>
          </div>
          <div class="chart-canvas-wrap scope-wrap-container">
            <ScopeChart data={gyroData} dark={darkMode} {dataKey} />
          </div>
        </div>

        <!-- Accelerometer Scope -->
        <div class="chart-card">
          <div class="chart-card__head">
            <div class="chart-card__label">
              <span class="chart-card__dot accel-dot"></span>
              Accelerometer Telemetry ({accelData.length} pts)
            </div>
            <div class="chart-legend">
              <span class="leg-item x">X</span>
              <span class="leg-item y">Y</span>
              <span class="leg-item z">Z</span>
              <span class="leg-item abs">|A|</span>
            </div>
          </div>
          <div class="chart-canvas-wrap scope-wrap-container">
            <ScopeChart data={accelData} showAbs={true} dark={darkMode} {dataKey} />
          </div>
        </div>
      </div>
    </section>
  {:else if !fileName && !errorMsg}
    <div class="empty-state reveal visible">
      <div class="empty-icon">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
          <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
          <line x1="3" y1="9" x2="21" y2="9"></line>
          <line x1="9" y1="21" x2="9" y2="9"></line>
          <path d="M13 17l2-2 3 3"></path>
          <path d="M13 13h5"></path>
        </svg>
      </div>
      <p>Load a .wag file exported from Wayify to view its data.</p>
    </div>
  {/if}
</main>

<style>
  .hero {
    background: var(--color-surface);
    padding: 24px 24px;
    border-radius: var(--radius-lg);
    box-shadow: var(--shadow-sm);
    border: 1px solid var(--color-border-subtle);
    margin-bottom: 20px;
    text-align: center;
  }

  .hero__title {
    font-size: 24px;
    font-weight: 700;
    color: var(--color-brand-primary);
    margin-bottom: 8px;
    letter-spacing: -0.02em;
  }

  .hero__subtitle {
    color: var(--color-text-secondary);
    font-size: 15px;
    margin-bottom: 24px;
  }

  .hero__actions {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 16px;
    flex-wrap: wrap;
  }

  .btn__icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 18px;
    height: 18px;
  }
  .btn__icon svg {
    width: 100%;
    height: 100%;
  }

  .btn-upload {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 28px;
    font-size: 15px;
    border-radius: var(--radius-pill);
    box-shadow: 0 4px 14px rgba(0, 160, 175, 0.25);
    border: none;
    cursor: pointer;
    background: linear-gradient(135deg, var(--color-brand-primary), var(--color-brand-secondary));
    color: #fff;
    font-weight: 600;
    transition: transform var(--transition-fast), box-shadow var(--transition-fast);
  }

  .btn-upload:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(0, 160, 175, 0.35);
  }

  .file-name {
    font-family: var(--font-mono);
    font-size: 13px;
    color: var(--color-brand-accent);
    background: rgba(0, 200, 215, 0.1);
    padding: 6px 12px;
    border-radius: var(--radius-sm);
    border: 1px dashed var(--color-brand-accent);
  }

  .empty-state {
    text-align: center;
    padding: 60px 20px;
    color: var(--color-text-muted);
  }

  .empty-icon {
    width: 64px;
    height: 64px;
    margin: 0 auto 16px;
    color: var(--color-brand-primary);
    opacity: 0.6;
  }
  .empty-icon svg {
    width: 100%;
    height: 100%;
  }

  /* Session Times */
  .session-times {
    display: flex;
    gap: 12px;
    margin-top: 24px;
    width: 100%;
    justify-content: center;
    flex-wrap: wrap;
  }
  .time-pill {
    background: var(--color-bg);
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

  .scope-wrap-container {
    height: 350px !important; /* Larger height for analysis */
    padding: 0;
  }

  @media (max-width: 600px) {
    .scope-wrap-container {
      height: 280px !important;
    }
  }

  /* Inherit chart card styles from App.svelte since they are global-ish, but redefine specifically for scoping here if needed, 
     Actually App.svelte has them scoped, so let's copy the needed ones for chart card here to be safe, or make them global.
     Since we have Svelte scoped css, it's safer to copy the specific card styles here.
  */
  .chart-row {
    display: grid;
    grid-template-columns: 1fr;
    gap: 20px;
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

  .error-banner {
    padding: 12px 16px;
    border-radius: var(--radius-md);
    font-size: 13px;
    font-weight: 500;
    background: rgba(220,53,69,0.08);
    border: 1px solid rgba(220,53,69,0.3);
    color: var(--color-danger);
    margin-bottom: 20px;
  }
</style>
