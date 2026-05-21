# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Purpose

Interactive web simulations of Data Center Facility equipment and systems — visualizing and simulating real-world behavior of electrical, mechanical, and safety infrastructure entirely client-side (no backend, offline-capable).

---

## Running & Developing

This is a static frontend project — no build step required.

```bash
# Serve locally (any static server works)
npx serve .
# or
python -m http.server 8080
# or open index.html directly in browser
```

No package manager, no bundler, no test runner unless explicitly added. When a test runner is introduced, document it here.

---

## Architecture

### Project Structure

```text
project-root/
├── index.html              # Main dashboard entry point
├── css/
│   ├── main.css            # Global styles and CSS custom properties
│   └── [equipment].css     # Per-equipment styles
├── js/
│   ├── main.js             # App init and event bus
│   ├── simulate/           # One file per equipment — state machine + logic
│   └── ui/                 # Rendering and DOM updates
├── assets/
│   ├── icons/              # SVG equipment icons
│   └── sounds/             # Optional alarm audio
└── CLAUDE.md
```

### Core Patterns

**Equipment State Objects** — every simulated device owns a state object:

```javascript
const generatorState = {
  status: 'standby',   // 'standby' | 'starting' | 'running' | 'fault'
  voltage: 0,          // V
  frequency: 0,        // Hz
  loadKW: 0,
  fuelLevel: 100,      // %
  alarm: null
};
```

**Event Bus (alarmBus)** — all equipment pushes alarms to a central panel:

```javascript
alarmBus.push({
  id: 'GEN-001',
  equipment: 'Generator',
  severity: 'critical' | 'warning' | 'info',
  message: 'Generator failed to start',
  timestamp: new Date().toISOString()
});
```

**Simulation Loop** — use `setInterval` or `requestAnimationFrame`; never block the main thread. Each equipment module exports an `update(deltaMs)` function called by the main loop.

**Fault Injection** — every equipment module must expose a `injectFault(type)` function so the UI control panel can trigger faults (overcurrent, utility loss, cooling failure, ground fault).

---

## Equipment-Specific Rules

### Power Flow (always follow this hierarchy)

```text
Utility (Grid)
  └─→ Transformer (HV → LV)
        └─→ MDB / ACB
              ├─→ UPS → Critical Load
              ├─→ Normal Load
              └─→ Rectifier → Battery Charger
```

### Generator Start Sequence (must be enforced in `simulate/generator.js`)

1. Utility voltage < 90% nominal → ATS detects loss
2. Generator cranks → 10 s simulated ramp to 80% rated speed
3. Reaches 415 V / 50 Hz rated → ATS transfers load
4. On utility return → ATS transfers back after cooldown delay

### UPS Behavior (`simulate/ups.js`)

- **Online**: double-conversion, battery 100%
- **Utility loss**: switch to battery within < 20 ms simulated
- **Battery < 20%**: alarm + graceful shutdown sequence
- **Bypass**: load on raw utility, UPS out of loop

### HVAC Control Loop (`simulate/hvac.js`)

- Target `serverInletTemp`: 18–27°C (ASHRAE A1)
- PID-like output adjustment each tick
- Warning alarm > 27°C; critical alarm > 35°C

---

## Constants & Naming

**No magic numbers.** All thresholds must be named constants at the top of each module:

```javascript
const RATED_VOLTAGE_V     = 415;
const RATED_FREQUENCY_HZ  = 50;
const GEN_RAMP_TIME_MS    = 10_000;
const UPS_TRANSFER_MS     = 20;
const BATTERY_LOW_PCT     = 20;
const TEMP_WARNING_C      = 27;
const TEMP_CRITICAL_C     = 35;
const UTILITY_LOW_THRESH  = 0.90;   // fraction of nominal
```

All displayed values must include units: `kV`, `kW`, `kVA`, `A`, `Hz`, `°C`, `%RH`, `Ω`.

---

## UI & Visual Conventions

**Theme**: Industrial / Control Room — dark backgrounds, HMI-style accents.

| State | Color | Must also show |
| ----- | ----- | -------------- |
| Normal / Energized / Online | 🟢 Green | text label or icon |
| Warning / Standby / Degraded | 🟡 Amber | text label or icon |
| Fault / Trip / Alarm | 🔴 Red | text label or icon |
| De-energized / Off | ⚪ Gray | text label or icon |

**Status indicators must never rely on color alone** — always pair with an icon or text label for accessibility.

**Fonts**: monospace (`JetBrains Mono`, `Courier New`) for live values/labels; sans-serif (`Barlow`, `DIN`) for UI chrome.

**Visualization**: SVG for single-line diagrams (SLDs); Canvas for real-time animations; Chart.js or D3.js for trend graphs.

**Target resolutions**: 1920×1080 (control room) and 1366×768 (laptop). No external CSS frameworks unless explicitly requested.

---

## Deliverable Checklist

Every simulation feature must include:

- Live **equipment state panel** (real-time readings)
- **Control panel** — start / stop / trip / reset / fault injection buttons
- **Alarm panel** — timestamp, equipment ID, severity badge, message, acknowledge button
- Optional: **SLD overlay** as inline SVG

Single-file bundle (`.html`) is acceptable when requested; otherwise use the multi-file structure above.

---

## Key Terminology

| Thai | English | ID Prefix |
| ---- | ------- | --------- |
| หม้อแปลงไฟฟ้า | Transformer | TX |
| เครื่องกำเนิดไฟฟ้า | Generator | GEN |
| สวิตช์สับเปลี่ยนอัตโนมัติ | Automatic Transfer Switch | ATS |
| ตู้จ่ายไฟหลัก | Main Distribution Board | MDB |
| เซอร์กิตเบรกเกอร์อากาศ | Air Circuit Breaker | ACB |
| เครื่องสำรองไฟ | UPS | UPS |
| เครื่องเรียงกระแส | Rectifier | RECT |
| ระบบปรับอากาศ | HVAC / Precision Cooling | CRAC/CRAH |
| ระบบกราวด์ | Grounding System | GND |
| ระบบป้องกันฟ้าผ่า | Lightning Protection System | LPS |
| ระบบแจ้งเหตุเพลิงไหม้ / ระบบดับเพลิง | Fire Alarm / Fire Suppression | FA / FS |
| ระบบทำความเย็นด้วยอากาศ / น้ำเย็น | Chilled Air Cooled / Chilled Water Cooled | CAC / CWC |

Use these ID prefixes for all alarm IDs, DOM element IDs, and state object keys (e.g., `GEN-001`, `ATS-001`, `UPS-001`).
