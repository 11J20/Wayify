# Wayify – Mobile Sensor Data Logger

Wayify is a high-performance web application designed for mobile devices to capture raw Gyroscope and Accelerometer data at maximum hardware frequencies, visualize it in real-time, and export it for post-session analysis.

Built with Svelte, it utilizes an optimized, throttled Reactivity loop to maintain a perfectly smooth 60fps UI while capturing hundreds of data points per second in the background without draining your device's CPU.

## Key Features

- **Live Hardware Telemetry**: Read exact XYZ coordinates of Gyroscope (orientation) and Accelerometer (motion including gravity) hardware sensors.
- **Throttled Live Graphs**: Real-time canvas charts synchronized perfectly at 10 frames per second for smooth, readable visualizations.
- **Memory-safe Logging**: Appends raw data directly to JS Arrays outside Svelte's reactivity engine, allowing for hours of uninterrupted logging.
- **Export to `.wag`**: Export your captured sessions to the proprietary `.wag` (Wayify Acceleration Gyroscope) format, complete with strict timestamp metadata.
- **VectroScope Analyzer**: A built-in secondary tool to upload and parse `.wag` files for post-session scrubbing and data analysis.

## Usage Instructions

### 1. Recording Data
1. Open the application on a mobile device (requires a physical gyroscope/accelerometer).
2. Tap **"Enable Sensors"**. Your browser may ask for permission to access motion and orientation data—tap **Allow**.
3. The live dashboard will immediately display real-time readouts and scrolling line charts of your phone's physical movements.
4. Tap the **"Start"** button to begin actively recording a session to memory. The button will pulse red indicating a live session.
5. Tap **"Stop"** when you are finished recording.
6. Tap **"Export"** to download your session data as a `.wag` file directly to your phone's storage.

### 2. Analyzing Data (VectroScope)
1. Use the top navigation bar to switch from the "Telemetry" page to the **"VectroScope"** page.
2. Tap the upload area (or the "Upload .wag File" button).
3. Select the `.wag` file you just downloaded from the Wayify exporter.
4. The VectroScope will parse the file, extract the true Start/End timestamps, and render the complete history of both the Gyroscope and Accelerometer data.
5. Use the graph interface to scrub through and analyze your session in high detail.

## Development

To run the project locally:

```bash
# Install dependencies
npm install

# Start the Vite development server
npm run dev
```

### Architecture Notes
- The UI relies heavily on a decoupled `requestAnimationFrame` (`tick`) loop in Svelte to prevent massive DOM thrashing. 
- The raw data is appended by hardware interrupts (`onGyro`, `onAccel`) at ~100Hz, while the Svelte `$state` and DOM updates are strictly capped at 4-10Hz.
