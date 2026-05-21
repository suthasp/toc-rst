# CLAUDE.md — Data Center Facility Web Simulation

## Project Overview

This workspace is for building **interactive web simulations** of Data Center Facility equipment and systems. The goal is to visualize and simulate real-world behavior of electrical, mechanical, and safety infrastructure in a Data Center environment.

---

## Scope of Equipment & Systems

### Electrical Systems
- **Transformer** — Step-up/step-down voltage, load monitoring, tap changer
- **Generator** — Auto-start on utility failure, fuel level, load kW/kVA
- **ATS (Automatic Transfer Switch)** — Utility ↔ Generator switching logic, transfer time
- **MDB (Main Distribution Board)** — Bus bar, circuit breakers, load balancing
- **ACB (Air Circuit Breaker)** — Trip/close, overcurrent protection, interlocking logic
- **UPS (Uninterruptible Power Supply)** — Battery backup, bypass mode, charge state
- **Rectifier** — AC→DC conversion, output voltage/current regulation

### Mechanical / Environmental Systems
- **HVAC** — Cooling units (CRAC/CRAH), temperature/humidity control, setpoints
- **Precision Cooling** — Floor-mount / overhead units, airflow simulation
- **Chilled Water Cooled (Chiller Plant)** — Chiller (CH), Condenser Water Pump (CDP), Chilled Water Pump (CHP), Cooling Tower (CT), CPMS display, COP monitoring
- **Chilled Air Cooled (DX/CRAC)** — DX refrigerant circuit, air-cooled condenser unit, compressor/evaporator/expansion valve, hot-aisle/cold-aisle airflow, lead-lag rotation

### Safety & Protection Systems
- **Grounding System** — Ground resistance, bonding paths, fault current flow
- **Lightning Protection System (LPS)** — Air terminals, down conductors, surge arrestors
- **Fire Alarm System (FAS/FACP)** — Multi-zone detection (smoke/heat/flame/VESDA/MCP), pre-alarm & confirmed alarm escalation, alarm panel
- **Fire Suppression System (FSS)** — FM-200 / Novec 1230 / CO₂ / Inert gas, pre-discharge countdown, abort, discharge animation, HVAC damper interlock

---

## Tech Stack & Conventions

### Frontend
- **Primary**: Vanilla HTML5 + CSS3 + JavaScript (ES6+)
- **Alternative**: React (JSX) when component reuse is needed
- **Visualization**: SVG for single-line diagrams; Canvas for real-time animation
- **Charting**: Chart.js or D3.js for trend graphs, power curves
- **Styling**: CSS custom properties (variables) for theming; no external CSS frameworks unless specified

### Design Aesthetic
- **Industrial / Control Room** theme — dark backgrounds, amber/green HMI-style accents
- Font: Monospace for values/labels (e.g., `JetBrains Mono`, `Courier New`); Sans-serif for UI (e.g., `Barlow`, `DIN`)
- Color coding:
  - 🟢 Green = Normal / Energized / Online
  - 🟡 Amber = Warning / Standby / Degraded
  - 🔴 Red = Fault / Trip / Alarm
  - ⚪ Gray = De-energized / Off
- All values must display units: kV, kW, kVA, A, Hz, °C, %RH, Ω

### Simulation Logic
- Use `setInterval` or `requestAnimationFrame` for real-time updates
- All equipment has **state objects**: `{ status, voltage, current, power, alarm, ... }`
- Equipment state transitions must follow real-world logic (e.g., Generator cannot take load before reaching rated speed)
- Support **fault injection**: user can trigger faults (overcurrent, utility loss, cooling failure, ground fault)

---

## File & Folder Conventions

```
project-root/
├── index.html              # Main entry / dashboard
├── css/
│   ├── main.css            # Global styles, CSS variables
│   └── [equipment].css     # Per-equipment styles
├── js/
│   ├── main.js             # App init, event bus
│   ├── simulate/
│   │   ├── generator.js    # Generator simulation logic
│   │   ├── ats.js
│   │   ├── ups.js
│   │   └── ...
│   └── ui/
│       ├── dashboard.js    # Dashboard rendering
│       └── alarms.js       # Alarm panel
├── assets/
│   ├── icons/              # SVG equipment icons
│   └── sounds/             # Optional alarm sounds
└── CLAUDE.md
```

---

## Coding Standards

1. **Comment every state machine** — each equipment simulation must have clear inline comments for state transitions
2. **No hardcoded magic numbers** — use named constants: `const RATED_VOLTAGE_KV = 11;`
3. **Responsive layout** — simulations must work on 1920×1080 (control room monitor) and 1366×768 (laptop)
4. **Accessibility** — status indicators must not rely on color alone; use icons + text labels
5. **No external API calls** — all simulation is client-side only (offline-capable)
6. **Single-file delivery optional** — when user requests, bundle into one `.html` file

---

## Simulation Behavior Rules

### Power Flow Priority
```
Utility (Grid)
  └─→ Transformer (HV → LV)
        └─→ MDB / ACB
              ├─→ UPS → Critical Load
              ├─→ Normal Load
              └─→ Rectifier → Battery Charger
```

### Generator Start Sequence
1. Utility voltage drops below threshold (< 90% nominal)
2. ATS detects loss → starts generator
3. Generator cranks → reaches 80% rated speed (~10s simulated)
4. Generator reaches rated voltage & frequency (415V / 50Hz)
5. ATS transfers load to generator
6. On utility return → ATS transfers back after cooldown

### UPS Behavior
- **Normal**: Online/double-conversion, battery at 100%
- **Utility loss**: Switch to battery within < 20ms
- **Battery low** (< 20%): Alarm + prepare for graceful shutdown
- **Bypass mode**: UPS bypassed, load on raw utility

### HVAC Control Loop
- Monitor `serverInletTemp` target: 18–27°C (ASHRAE A1)
- PID-like cooling output adjustment
- Raise alarm if temp > 27°C; critical alarm > 35°C

### Fire Alarm System (FAS) — Zone State Machine

```text
NORMAL
  └─▶ PRE_ALARM    (1st detector triggered → 30s investigate countdown)
        └─▶ ALARM  (2nd detector OR MCP OR timeout → confirmed fire)
              └─▶  → notify suppression, activate sounders/strobes
Any state → FAULT   (detector offline / loop fault)
Any state → ISOLATED (manual operator isolation)
```

- Zone grid: color per status — green (NORMAL), amber (PRE_ALARM), red (ALARM), gray (ISOLATED)
- Event log: timestamp | zone | device | event
- Controls: trigger detector, silence, reset, isolate zone, test devices

### Fire Suppression System (FSS) — FM-200 Discharge Sequence

```text
STANDBY
  └─▶ PRE-DISCHARGE  (FACP fire signal → close HVAC dampers, release door holders, start 30s abort window)
        ├─▶ ABORT     (operator presses abort within 30s)
        └─▶ DISCHARGE (solenoid opens → agent fills room ~10s animation)
              └─▶ HOLD (maintain concentration ~10 min, do NOT ventilate)
                    └─▶ RELEASED / VENTED (all-clear confirmed)
Any state → FAULT (cylinder pressure < 34 bar, solenoid fault, damper fault)
```

- Display: cylinder pressure (bar), agent weight (kg), design concentration (% vol), abort countdown
- Interlock: on PRE-DISCHARGE → `hvac.closeDamper(zone)`, `accessControl.releaseDoorHolders(zone)`
- SVG room diagram: animate gas fill overlay opacity 0 → 0.4 during DISCHARGE state

---

## Alarm System

All equipment must push alarms to a central **Alarm Panel**:

```javascript
alarmBus.push({
  id: 'GEN-001',
  equipment: 'Generator',
  severity: 'critical' | 'warning' | 'info',
  message: 'Generator failed to start',
  timestamp: new Date().toISOString()
});
```

Display: timestamp, equipment ID, severity badge, message, acknowledge button.

---

## Deliverable Format

For each simulation request, deliver:
- Working HTML/CSS/JS (single file or multi-file as appropriate)
- Equipment state panel (live readings)
- Control panel (user interactions: start/stop/trip/reset/fault injection)
- Alarm panel
- Optional: single-line diagram (SLD) as SVG overlay

---

## Key Terminology (TH/EN)

| Thai | English | Abbreviation |
|------|---------|--------------|
| หม้อแปลงไฟฟ้า | Transformer | TX |
| เครื่องกำเนิดไฟฟ้า | Generator | GEN |
| ระบบแจ้งเหตุและเตือนภัย/ระบบระงับอัคคีภัย |Fire Alarm/Fire Suppression | FA/FS |  
| สวิตช์สับเปลี่ยนอัตโนมัติ | Automatic Transfer Switch | ATS |
| ตู้จ่ายไฟหลัก | Main Distribution Board | MDB |
| เซอร์กิตเบรกเกอร์อากาศ | Air Circuit Breaker | ACB |
| ระบบปรับอากาศ | HVAC / Precision Cooling | CRAC/CRAH/Chlled Air Cooled/Chilled Water Cooled |
| เครื่องสำรองไฟ | Uninterruptible Power Supply | UPS |
| เครื่องเรียงกระแส | Rectifier | RECT |
| ระบบกราวด์ | Grounding System | GND |
| ระบบป้องกันฟ้าผ่า | Lightning Protection System | LPS |
