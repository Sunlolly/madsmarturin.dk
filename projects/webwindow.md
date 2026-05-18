# WebWindow — Android Website Screenshot Widgets

## Overview

I built **WebWindow**, a native Android application that displays live website screenshots as home screen widgets. The app renders websites in an off-screen WebView, captures high-quality bitmaps, and delivers them to Android's RemoteViews framework with automatic periodic updates scheduled via WorkManager.

---

## Technical Challenges & Solutions

### 1. WebView Cannot Live in Widgets

**Challenge**: Android's RemoteViews API only supports a limited set of views (ImageView, TextView, basic layouts). I couldn't embed a WebView directly in the widget.

**Solution**: Two-step capture pipeline:
- Render the webpage in a hidden/off-screen WebView
- Capture it as a Bitmap using `webView.draw(canvas)`
- Display the resulting image via ImageView in RemoteViews

---

### 2. Android 12+ Blocks Foreground Services from Background Contexts

**Challenge**: On API 31+, `startForegroundService()` and even WorkManager's `setForeground()` are denied when called from background contexts — a security feature to prevent battery abuse.

**Solution**: Two distinct capture paths:
- **Interactive path** (foreground): User configures widget → Activity broadcasts → `ScreenshotService` starts as foreground service (allowed because Activity is in foreground)
- **Background path** (periodic): WorkManager triggers `WidgetUpdateWorker` → captures screenshot **directly inline** using off-screen WebView (no foreground service needed; Workers get 10-minute execution window)

---

### 3. RemoteViews Memory Limits (~1-2MB per widget update)

**Challenge**: High-resolution screenshots would crash the system when delivering updates to the home screen.

**Solution**: `BitmapUtil` implements adaptive compression:
- Target size: ~800KB (half of RemoteViews limit for safety margin)
- JPEG quality starts at 90%, progressively reduces to 20% if needed
- Downscaling as fallback if compression alone is insufficient
- Automatic bitmap recycling after delivery

---

### 4. Rendering When Screen is Off

**Challenge**: When the phone's screen is off (typical for widget updates), hardware-accelerated rendering may fail because the GPU compositor is unavailable.

**Solution**: `webView.setLayerType(LAYER_TYPE_SOFTWARE, null)` forces CPU-only rendering, ensuring consistent output regardless of display state.

---

### 5. Timing JavaScript Execution

**Challenge**: Web pages load asynchronously; capturing too early shows blank pages or loading spinners.

**Solution**:
- 15-second page load timeout with `CompletableDeferred<LoadResult>` for awaitable control flow
- 2-second JavaScript delay after `onPageFinished` to let scripts stabilize
- Blank bitmap detection (samples 20x20 pixel grid; rejects if all pixels match background)

---

## Architecture Highlights

| Component | Purpose |
|-----------|---------|
| `AppWidgetProvider` | Receives update broadcasts, manages widget lifecycle |
| `WorkManager` | Schedules periodic updates (15min minimum interval enforced by OS) |
| `DataStore` | Persists user configs: URL, refresh interval, scale factor |
| `CompletableDeferred` | Bridges WebView's callback-based API to Kotlin coroutines for clean timeouts |
| `BroadcastReceiver` | IPC mechanism delivering captured bitmaps back to the widget UI |

**Key Design Decision**: The broadcast-based result delivery pattern (`ScreenshotService` → broadcast → `ScreenshotResultReceiver`) is standard Android IPC, but I had to implement it because the service and provider live in separate contexts.

---

## Features Delivered (MVP)

- Multiple independent widget instances (each with its own URL)
- Full-page screenshots at widget scale (100%/200%/300%)
- Configurable refresh interval: 15 min to 24 hours
- Tap widget to open URL in browser
- Automatic cookie/session persistence across captures
- Silent background updates (no loading/error flash on failed captures)

---

## Development Context: GenAI-Enabled Workflow

This project was built using **Claude Code**, a Generative AI assistant that handles the heavy lifting of code generation, debugging, and documentation. I define requirements in plain English; Claude:
- Generates production-ready Kotlin code with Android best practices
- Researches architectural patterns (RemoteViews, WorkManager, foreground services)
- Writes detailed architecture docs (ARCHITECTURE.md, WIDGET_UPDATE_FLOW.md)
- Implements complex edge cases (blank bitmap detection, software rendering mode)
- Maintains consistent project structure and documentation

This workflow lets me focus on **problem decomposition** and **architectural decisions** while the AI handles implementation details — a practical demonstration of how GenAI augments modern mobile development.

---

## Code Quality & Tooling

- **Language**: Kotlin 1.9.23
- **Min SDK**: Android 8.0 (API 26) — covers ~95% of active devices
- **Build System**: Gradle with Kotlin DSL
- **CI/CD**: GitHub Actions for builds and releases (Codeberg hosting)
- **Debugging**: `adb logcat` filtering, comprehensive logging in background paths

---

## Learnings

1. **Android's security model is strict but necessary** — foreground service restrictions protect battery life; WorkManager provides a safe alternative with its own constraints.
2. **RemoteViews is intentionally limited** — complex widgets require custom services (ListView-based scrolling) or Glance APIs (API 33+).
3. **Error handling in background tasks is subtle** — silent failures preserve existing screenshots, avoiding jarring "Failed to load" flash on the home screen.

---

## Links

- **Code**: [Codeberg](https://codeberg.org/Marturin/webwindow)
