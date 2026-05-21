---
name: datacenter-facility-simulation
description: Build interactive web simulations of Data Center Facility equipment and systems. Use this skill whenever the user asks to simulate, visualize, or build a web-based demo of any Data Center equipment including Transformer, Generator, ATS, MDB, ACB, UPS, Rectifier, HVAC/CRAC/CRAH, Chilled Water Cooled (Chiller Plant/CPMS), Chilled Air Cooled (DX/CRAC precision cooling), Grounding System, Lightning Protection System, Fire Alarm System, or Fire Suppression System. Trigger on keywords like "simulate", "web sim", "facility", "data center", "HVAC", "UPS", "generator", "single-line diagram", "SLD", "power system", "cooling", "chiller", "CRAC", "CRAH", "chilled water", "chilled air", "cooling tower", "CPMS", "precision cooling", "DX cooling", "electrical equipment", "fire alarm", "fire suppression", "FM200", "VESDA", "smoke detector", "sprinkler", or any Thai equivalents (หม้อแปลง, เครื่องกำเนิดไฟฟ้า, ระบบปรับอากาศ, ระบบน้ำเย็น, ระบบระบายความร้อน, ชิลเลอร์, คูลลิ่งทาวเวอร์, ระบบกราวด์, ระบบแจ้งเตือนอัคคีภัย, ระบบดับเพลิง ฯลฯ). Always apply this skill even for partial requests like "show me how ATS switching works", "build a UPS status panel", "simulate fire detection zones", "show FM200 discharge sequence", "build a chiller plant CPMS", or "show CRAC cooling operation".
---

# Data Center Facility Web Simulation Skill

Build production-quality, interactive web simulations of Data Center Facility equipment. Each simulation must look like a real HMI/SCADA panel — industrial aesthetic, live animated values, user-controllable states, and a central alarm system.

---

## Design System

### Visual Theme — Industrial HMI / Control Room
```css
:root {
  --bg-primary:    #0a0e14;   /* deep navy-black */
  --bg-panel:      #111827;   /* panel background */
  --bg-card:       #1a2332;   /* equipment card */
  --border:        #2d3748;   /* subtle border */
  --accent-green:  #00d26a;   /* ON / Normal */
  --accent-amber:  #f59e0b;   /* Warning / Standby */
  --accent-red:    #ef4444;   /* Fault / Trip / Alarm */
  --accent-blue:   #38bdf8;   /* Info / Utility */
  --accent-gray:   #6b7280;   /* OFF / De-energized */
  --text-primary:  #e2e8f0;
  --text-muted:    #94a3b8;
  --font-mono:     'JetBrains Mono', 'Courier New', monospace;
  --font-ui:       'Barlow', 'DIN Alternate', sans-serif;
}
```

### Status Indicator Colors
| State | Color Variable | Hex |
|-------|---------------|-----|
| ON / Energized / Normal | `--accent-green` | `#00d26a` |
| Warning / Standby / Degraded | `--accent-amber` | `#f59e0b` |
| FAULT / TRIP / ALARM | `--accent-red` | `#ef4444` |
| OFF / De-energized | `--accent-gray` | `#6b7280` |
| Utility / Grid | `--accent-blue` | `#38bdf8` |

---

## Equipment Simulation Templates

### 1. Generator (GEN)
**State machine:**
```
STOPPED → STARTING (cranking ~10s) → WARMING_UP (5s) → RUNNING → LOADED
LOADED → UNLOADING → COOLING_DOWN (30s) → STOPPED
Any state → FAULT (on trip condition)
```

**Key parameters to display:**
- Status (badge), Speed (RPM), Voltage (V), Frequency (Hz)
- Active Power (kW), Power Factor (PF), Fuel Level (%)
- Engine Temp (°C), Oil Pressure (bar), Run Hours

**Controls:** START / STOP / TRIP (fault injection) / RESET

**Simulation logic:**
```javascript
const GEN_STATES = { STOPPED:0, STARTING:1, WARMING:2, RUNNING:3, LOADED:4, FAULT:5 };
// Speed ramps 0 → 1500 RPM over 10s during STARTING
// Voltage ramps 0 → 415V once speed > 80% rated
// Frequency tracks speed linearly
```

---

### 2. ATS (Automatic Transfer Switch)
**State machine:**
```
UTILITY_NORMAL → UTILITY_LOSS_DETECTED (< 90% Vnom, delay 2s)
UTILITY_LOSS_DETECTED → GEN_START_SIGNAL → WAITING_GEN_READY
WAITING_GEN_READY → TRANSFERRING_TO_GEN (100ms) → ON_GENERATOR
UTILITY_RESTORED → RETRANSFER_DELAY (60s) → TRANSFERRING_TO_UTILITY → UTILITY_NORMAL
```

**Display:** Source A status, Source B status, Transfer position, Transfer time (ms), Transfer count

**Controls:** Force transfer, Simulate utility loss, Auto/Manual mode

---

### 3. UPS (Uninterruptible Power Supply)
**Operating modes:**
```
ONLINE (double-conversion) → BATTERY (utility loss, < 20ms transition)
BATTERY → LOW_BATTERY (< 20% SoC) → CRITICAL (< 5% SoC)
ONLINE / BATTERY → BYPASS (maintenance or overload)
```

**Parameters:** Input V/Hz, Output V/Hz, Load % (kVA/kW), Battery SoC %, Battery Voltage, Estimated runtime (min), Efficiency %

**Controls:** Simulate utility loss, Force bypass, Charge/discharge rate

---

### 4. Transformer (TX)
**Parameters:** HV side (kV, A), LV side (V, A), Load (kVA, %), Temperature (°C), Tap position, Vector group

**Alarms:** Overload > 100%, High temp > 85°C (ONAN), Buchholz relay, Winding temp trip > 105°C

**Display:** Nameplate data, real-time load bar, thermal model, cooling fan status (ONAF)

---

### 5. HVAC / Precision Cooling (CRAC/CRAH)
**Control loop (simplified PID):**
```javascript
// Target inlet temp: 21°C (adjustable 18–27°C)
const cooling_output = kP * error + kI * integral;
// Server load generates heat, cooling removes it
// Display: Supply temp, Return temp, Setpoint, Cooling %, Humidity %
```

**Parameters:** Supply Air Temp (°C), Return Air Temp (°C), Humidity %RH, Cooling Capacity (kW), COP, Filter DP (Pa), Compressor status

**Alarms:** High temp > 27°C, High humidity > 60%RH, Filter clog, Compressor fault

---

### 6. MDB + ACB Panel
**Display:**  
- Bus bar voltage (3-phase: L1, L2, L3)  
- Per-feeder: Name, Rated A, Actual A, Load %, breaker status (CLOSED/OPEN/TRIP)

**ACB Controls:** Close / Open / Trip / Reset per breaker  
**Interlock logic:** Normally-open (NO) and Normally-closed (NC) interlocks between feeders

---

### 7. UPS Rectifier
**Parameters:** AC Input (V, A, Hz), DC Output (V, A, kW), Efficiency %, Temperature (°C)  
**States:** NORMAL, DERATED, FAULT  
**Display:** Power flow bar (AC→DC), ripple voltage indicator

---

### 8. Grounding System
**Display:** Ground resistance (Ω), Leakage current (mA), Bonding status (continuity check per zone)  
**Alarm thresholds:** Resistance > 1Ω (warning), > 5Ω (critical), Ground fault current > 30mA (RCD trip)  
**Visualization:** Zone map with color-coded resistance values

---

### 9. Lightning Protection System (LPS)
**Components:** Air terminals (Franklin rods), Down conductors, Earth electrodes, Surge Protective Devices (SPD)  
**Display:** Protection zone coverage (Faraday cage / rolling sphere method), SPD status (OK/FAILED), Last lightning event log  
**Simulation:** Trigger lightning strike → show current path through down conductors → SPD clamping → ground dissipation animation

---

### 10. Fire Alarm System (FAS)

**Standard:** NFPA 72 / BS 5839 / TIS 1586 (มอก. 1586)

**Components & Detection Devices:**
| Device | Type | Location in DC |
|--------|------|----------------|
| Smoke Detector (Optical) | Point detector | Above floor / ceiling |
| Smoke Detector (Ionization) | Point detector | Under raised floor |
| VESDA (Very Early Smoke Detection) | Aspirating / ASD | Server room, telecom room |
| Heat Detector (Fixed/ROR) | Point detector | Battery room, generator room |
| Flame Detector (UV/IR) | Point detector | Fuel storage, transformer area |
| Manual Call Point (MCP) | Break-glass | Near exits, corridors |
| Sounder / Strobe | Output device | All zones |
| Fire Alarm Control Panel (FACP) | Central controller | Main security/ops room |

**Zone Map Structure:**
```javascript
const zones = [
  { id: 'Z1', name: 'Server Room A', detectors: ['SD-01','SD-02','VESDA-01'], status: 'NORMAL' },
  { id: 'Z2', name: 'Server Room B', detectors: ['SD-03','SD-04','VESDA-02'], status: 'NORMAL' },
  { id: 'Z3', name: 'UPS Room',      detectors: ['SD-05','HD-01'],           status: 'NORMAL' },
  { id: 'Z4', name: 'Generator Room',detectors: ['HD-02','FD-01'],           status: 'NORMAL' },
  { id: 'Z5', name: 'Battery Room',  detectors: ['HD-03','GD-01'],           status: 'NORMAL' },
  { id: 'Z6', name: 'Corridor',      detectors: ['SD-06','MCP-01'],          status: 'NORMAL' },
];
// Zone status: 'NORMAL' | 'PRE_ALARM' | 'ALARM' | 'FAULT' | 'ISOLATED'
```

**State Machine (per zone):**
```
NORMAL
  └─▶ PRE_ALARM     (1 detector triggered — investigate, 30s countdown)
        └─▶ ALARM   (2nd detector OR MCP OR timeout — confirmed fire)
              └─▶ SUPPRESSION_RELEASED (if suppression linked)
Any state → FAULT   (detector offline, loop fault, power fail)
Any state → ISOLATED (manual isolation by operator)
```

**FACP Display Panel:**
- Floor plan / zone grid map — color per zone status
- Device list with individual status per detector
- Event log: timestamp | zone | device | event type
- Alarm silence / reset controls
- System fault indicators (power, battery, communication)

**Alarm Escalation Logic:**
```javascript
function onDetectorTrigger(zoneId, deviceId) {
  const zone = zones.find(z => z.id === zoneId);
  const triggered = zone.detectors.filter(d => deviceState[d] === 'ALARM');
  
  if (triggered.length === 1) {
    zone.status = 'PRE_ALARM';
    startCountdown(zoneId, 30); // 30s to confirm or investigate
    raiseAlarm('FACP', 'warning', `Pre-Alarm: ${zone.name} — ${deviceId} activated`);
  } else if (triggered.length >= 2) {
    zone.status = 'ALARM';
    triggerOutputDevices(zoneId);  // sounders, strobes
    raiseAlarm('FACP', 'critical', `FIRE CONFIRMED: ${zone.name}`);
    notifySuppression(zoneId);    // signal suppression system
  }
}
```

**Key Alarms:**
- `FIRE ALARM — Zone X confirmed` (critical, flashing red)
- `PRE-ALARM — Zone X investigate` (warning, amber)
- `DETECTOR FAULT — Device offline` (warning)
- `LOOP FAULT — Communication failure` (critical)
- `BATTERY LOW — FACP backup power` (warning)
- `MAINS FAILURE — FACP on battery` (warning)

**Controls (Simulation):**
- Trigger individual detector (smoke/heat/flame/MCP)
- Silence alarm / Reset system
- Isolate zone
- Simulate device fault
- Test sounder/strobe output

---

### 11. Fire Suppression System (FSS)

**Types supported in Data Center:**
| System | Agent | Typical Use |
|--------|-------|-------------|
| FM-200 (HFC-227ea) | Clean agent gas | Server rooms, UPS rooms |
| Novec 1230 | Clean agent gas | IT rooms, archive rooms |
| CO₂ System | Carbon dioxide | Generator room, cable basement |
| Inert Gas (IG-541/Inergen) | N₂/Ar/CO₂ mix | Telecom rooms |
| Pre-action Sprinkler | Water (2-step) | Non-critical areas |

**FM-200 / Clean Agent State Machine:**
```
STANDBY
  └─▶ PRE-DISCHARGE WARNING  (FACP sends signal → abort opportunity 30s)
        ├─▶ ABORT             (operator presses abort within 30s)
        └─▶ DISCHARGE         (solenoid opens → agent releases ~10s)
              └─▶ HOLD        (maintain concentration, do NOT ventilate ~10 min)
                    └─▶ RELEASED / VENTED (after safe entry confirmed)

Any state → FAULT (cylinder pressure low, solenoid fault, abort switch fault)
```

**Display Panel — Suppression Zone:**
```javascript
const suppressionZones = [
  {
    id: 'SZ1', name: 'Server Room A',
    agent: 'FM-200', cylinderQty: 2,
    cylinderPressure: 42,   // bar (full = 42 bar @ 21°C)
    agentWeight: 68,         // kg
    designConc: 7.9,         // % volume (NFPA 2001)
    status: 'STANDBY',       // STANDBY | PRE_DISCHARGE | DISCHARGING | DISCHARGED | FAULT
    abortActive: false,
    countdown: 0
  }
];
```

**Key Parameters to Display:**
- Zone name & protected area (m²)
- Agent type & design concentration (%)
- Cylinder count & pressure (bar) — with low-pressure alarm < 34 bar
- Agent weight remaining (kg)
- Discharge countdown (s) during pre-discharge phase
- Room integrity status (door/damper closed: ✅/❌)
- Last discharge date & agent replenishment status

**Discharge Sequence Animation:**
```
Step 1: FACP sends FIRE signal to Suppression Panel
Step 2: Suppression panel activates:
        - Close HVAC dampers (prevent agent escape)
        - Close room doors (magnetic hold-open release)
        - Activate pre-discharge alarm (horn + strobe)
        - Start 30s countdown (abort window)
Step 3: Countdown reaches 0 → Solenoid valve opens
Step 4: Agent discharges (animate gas fill in room diagram)
Step 5: Concentration reaches design level (~10s)
Step 6: Hold period begins (10 min)
Step 7: All-clear → ventilation allowed
```

**SVG Room Diagram for Discharge:**
```svg
<!-- Room with gas fill animation -->
<rect id="room-sz1" class="room" .../>
<!-- Gas fill overlay — animate opacity 0 → 0.4 on discharge -->
<rect id="gas-sz1" class="gas-agent" opacity="0" .../>
<!-- CSS -->
.gas-agent { fill: #7dd3fc; }
.gas-agent.discharging { animation: fillRoom 10s forwards; }
@keyframes fillRoom { from { opacity: 0; } to { opacity: 0.4; } }
```

**Interlock with HVAC & Access Control:**
```javascript
function onSuppressionPreDischarge(zoneId) {
  // Close HVAC dampers for this zone
  hvac.closeDamper(zoneId);
  // Release magnetic door holders
  accessControl.releaseDoorHolders(zoneId);
  // Disable HVAC fans in zone
  hvac.stopFans(zoneId);
  raiseAlarm('FSS', 'critical', `PRE-DISCHARGE: ${zoneId} — Evacuate immediately!`);
}
```

**Controls (Simulation):**
- Manual discharge trigger (per zone)
- Abort discharge (within countdown window)
- Reset after discharge
- Simulate cylinder pressure low
- Simulate damper/door fault
- Test alarm devices

**Key Alarms:**
- `FIRE SUPPRESSION DISCHARGED — Zone X` (critical)
- `PRE-DISCHARGE COUNTDOWN — Zone X [Xs remaining]` (critical, flashing)
- `ABORT ACTIVATED — Zone X` (warning)
- `CYLINDER PRESSURE LOW — Zone X` (warning, < 34 bar)
- `DAMPER FAULT — Zone X` (critical, pre-discharge interlock failed)
- `SUPPRESSION FAULT — Solenoid circuit open` (critical)

---

### 12. Chilled Water Cooled System (Chiller Plant / CPMS)

**System topology:**
```text
Cooling Tower (CT) ← condenser water loop ← Condenser Water Pump (CDP) ← Chiller (CH)
                                                                             ↓
                                                             Chilled Water Pump (CHP)
                                                                             ↓
                                                                   IT Load (Server Rooms)
```

**Equipment state objects:**
```javascript
const chiller = {
  id: 'CH-01', run: true, fault: false,
  load: 74,          // % of rated capacity
  kw: 452,           // compressor input power kW
  cop: 5.22,         // coefficient of performance
  chws: 7.1,         // chilled water supply temp °C
  chwr: 12.3,        // chilled water return temp °C
  cws: 32.0,         // condenser water supply temp °C
  cwr: 37.1          // condenser water return temp °C
};
const pump = { id: 'CDP-01', run: true, fault: false, flow: 480, kw: 18.6 };  // L/s, kW
const ctCell = { id: 'CT-1/1', run: true, fault: false, kw: 4500 };           // fan power W
const tower  = { ewt: 37.2, lwt: 32.0 };  // entering/leaving water temp °C
```

**Chiller state machine:**
```text
STANDBY → STARTING (pre-lube, purge ~30s) → LOADING (ramp load 0→rated) → RUNNING
RUNNING → UNLOADING → COOLDOWN → STANDBY
Any state → FAULT (high discharge pressure, low suction pressure, motor overcurrent, oil pressure low)
```

**Key parameters to display:**
- Per chiller: Load %, kW, COP, CHWS/CHWR °C, CWS/CWR °C, compressor status
- Per pump: Flow (L/s), Differential pressure (kPa), Power (kW), VFD speed %
- Per CT cell: Fan status (RUN/STBY), Fan kW, cell ID
- Tower summary: EWT/LWT °C, running cells count
- Plant KPIs: Total plant kW, System COP, ΔCHW temperature, ΔCW temperature

**Typical operating config (N+1):** 3 chillers running / 1 standby, 3 CDP / 1 standby, 3 CHP / 1 standby, 6 CT cells / 2 standby

**Canvas drawing style (3D CPMS):**
- Isometric `box3D()` for chiller bodies — color = run: `#0a3a55`, standby: `#1a3040`, fault: `#4a0a0a`
- `cylinder3D()` for pump casings — CDW orange `#ff7700`, CHW cyan `#00ccff`
- `pipe3D()` gradient tubes — CDR=orange, CDS=yellow, CHS=cyan, CHR=blue
- Cooling tower: large isometric shell, 4×2 fan cell grid, spray + fill + basin animation
- Flow dots animated along pipe paths when `run: true`

**Alarms:**
- `HIGH DISCHARGE PRESSURE — CH-0X` (critical)
- `LOW SUCTION PRESSURE — CH-0X` (critical)
- `CHILLED WATER HIGH TEMP — CHWR > 14°C` (warning)
- `COOLING TOWER FAN FAULT — CT-X/X` (warning)
- `PUMP FLOW LOW — CDP/CHP < 400 L/s` (warning)
- `PLANT COP LOW — System COP < 3.5` (warning)

**Controls:** Start/Stop per chiller, pump, CT cell | Fault inject | All ON / All OFF | Mode: Auto / Manual

---

### 13. Chilled Air Cooled System (CRAC / DX Precision Cooling)

**System topology:**
```text
Air-Cooled Condenser Unit (outdoor)
        ↕  refrigerant circuit (DX)
Computer Room Air Conditioner (CRAC) — indoor
        ↓  supply air (cold)
  [ Server Rack Rows ]
        ↑  return air (hot)
```

**Equipment state objects:**
```javascript
const crac = {
  id: 'CRAC-01', run: true, fault: false,
  mode: 'COOLING',     // COOLING | DEHUMIDIFY | FREECOOLING | STANDBY | FAULT
  supplyTemp: 18.2,    // °C — cold air delivered to room
  returnTemp: 26.8,    // °C — hot air from server rows
  setpoint: 21.0,      // °C — target supply air temperature
  humidity: 47,        // %RH
  humiditySetpoint: 50,// %RH
  coolingCapacity: 48, // kW
  compressorKw: 14.2,  // kW input
  fanSpeedPct: 78,     // % VFD speed
  filterDp: 62,        // Pa — differential pressure across filter
  suctionPressure: 4.8,// bar (refrigerant suction side)
  dischargePressure: 18.4, // bar (refrigerant discharge side)
  refrigerant: 'R410A'
};
```

**CRAC state machine:**
```text
OFF → STARTING (fan + compressor pre-start checks ~5s) → COOLING
COOLING → DEHUMIDIFY  (humidity > setpoint + 5%RH)
COOLING → FREECOOLING (outdoor temp < 18°C — economizer active, compressor off)
COOLING → STANDBY     (lead-lag rotation or low load)
Any state → FAULT     (high discharge pressure, low suction pressure, high supply temp, filter clog, fan motor fault)
```

**Key parameters to display:**
- Supply Air Temp (°C) vs setpoint — trend bar
- Return Air Temp (°C) — server inlet proxy
- Humidity %RH vs setpoint
- Cooling Capacity (kW), Compressor Input (kW), EER (kW/kW)
- Fan Speed % (VFD), Filter DP (Pa)
- Refrigerant: suction pressure (bar), discharge pressure (bar), superheat (°C)
- Run hours, last maintenance date

**Room airflow visualization (Canvas):**
- Hot aisle / Cold aisle layout with airflow arrows
- Color gradient: blue (cold supply) → red (hot return)
- Animate airflow particles from CRAC → server rows → CRAC return

**Alarm thresholds:**
| Parameter | Warning | Critical |
|-----------|---------|----------|
| Supply Air Temp | > 22°C | > 25°C |
| Return Air Temp | > 30°C | > 35°C |
| Humidity | > 60%RH or < 30%RH | > 70%RH |
| Filter DP | > 150 Pa | > 250 Pa |
| Discharge Pressure (R410A) | > 25 bar | > 28 bar |
| Suction Pressure (R410A) | < 3.5 bar | < 2.5 bar |

**Controls:** Start/Stop, setpoint adjust, force dehumidify, force freecooling, fault inject (high discharge pressure, filter clog, fan fault), Reset

**Lead-lag logic (multiple units):**
```javascript
// Rotate lead unit every 8h to equalize run hours
function rotateLead(units) {
  const sorted = [...units].sort((a, b) => a.runHours - b.runHours);
  sorted[0].mode = 'COOLING';   // lowest run hours → lead
  sorted.slice(1).forEach(u => u.mode = u.run ? 'STANDBY' : 'OFF');
}
```

---

## Alarm System (Required in Every Simulation)

Every simulation **must** include a central alarm panel. Implement this pattern:

```javascript
// Alarm bus
const alarms = [];

function raiseAlarm(equipId, severity, message) {
  // severity: 'critical' | 'warning' | 'info'
  alarms.unshift({
    id: `${equipId}-${Date.now()}`,
    equipment: equipId,
    severity,
    message,
    timestamp: new Date().toLocaleString('th-TH'),
    acknowledged: false
  });
  renderAlarmPanel();
}

function acknowledgeAlarm(id) {
  const alarm = alarms.find(a => a.id === id);
  if (alarm) alarm.acknowledged = true;
  renderAlarmPanel();
}
```

**Alarm panel UI:**
- Fixed panel at bottom or right side
- Unacknowledged alarms flash / blink (CSS animation)
- Color-coded rows: red (critical), amber (warning), blue (info)
- Columns: Time | Equipment | Severity | Message | [ACK button]
- Alarm count badge on panel header

---

## Standard UI Layout

```
┌─────────────────────────────────────────────────────────┐
│  HEADER: Project name, system name, date/time clock     │
├──────────────┬──────────────────────────────────────────┤
│              │                                          │
│  EQUIPMENT   │   MAIN VISUALIZATION                     │
│  NAVIGATOR   │   (SLD / Animated diagram / Gauges)      │
│  (sidebar)   │                                          │
│              ├──────────────────────────────────────────┤
│              │   PARAMETERS / STATUS PANEL              │
└──────────────┴──────────────────────────────────────────┤
│  ALARM PANEL (collapsible, fixed height)                │
└─────────────────────────────────────────────────────────┘
```

---

## Single-Line Diagram (SLD) Guidelines

When building SLD as SVG:
- Grid/Utility enters from top
- Power flows top → bottom
- Equipment symbols must follow IEC 60617 / ANSI standards (simplified)
- Buses shown as thick horizontal lines
- Breakers as diagonal cross symbols (ANSI) or box symbols
- Animate power flow with dashed-line path animation in energized state
- Color the entire path segment: energized (green), de-energized (gray), fault (red, blinking)

```svg
<!-- Example: energized bus line -->
<line class="bus energized" x1="100" y1="150" x2="400" y2="150"/>
<!-- CSS -->
.bus.energized { stroke: var(--accent-green); stroke-width: 4; }
.bus.fault     { stroke: var(--accent-red); animation: blink 0.5s infinite; }
```

---

## Deliverable Checklist

For every simulation request, ensure:
- [ ] Equipment state panel with live animated values
- [ ] Control buttons (Start / Stop / Trip / Reset / Fault Inject)
- [ ] Status indicators (color + icon + text — not color alone)
- [ ] Central alarm panel with ACK functionality
- [ ] Single-line diagram or equipment diagram (SVG)
- [ ] All numeric values show units (kV, kW, A, Hz, °C, %RH, Ω)
- [ ] Responsive at 1920×1080 and 1366×768
- [ ] No external API calls (offline-capable)
- [ ] Comments on all state machine transitions

---

## Reference: Equipment Relationships

```
GRID (Utility)
  └─▶ TX (HV→LV)
        └─▶ MDB / ACB (Main Bus)
              ├─▶ UPS ──▶ RECT ──▶ Battery
              │     └─▶ Critical Load
              ├─▶ HVAC (Cooling Load)
              ├─▶ Normal Load
              └─▶ [ATS] ←── GEN (backup source)

Safety:
  GND ←── All equipment chassis bonded
  LPS ←── Air terminals → down conductors → GND
  SPD ←── Installed at TX LV side, MDB, and critical panels

Fire Safety:
  FAS (FACP) ←── Detectors (smoke/heat/flame/MCP) per zone
  FAS ──▶ FSS (suppression trigger signal) on confirmed alarm
  FSS ──▶ HVAC (close dampers on pre-discharge)
  FSS ──▶ Access Control (release door holders on pre-discharge)
  FSS ──▶ Sounder/Strobe (pre-discharge warning)
```

---

## Thai Terminology Reference

| Thai | English | Symbol/Abbr |
|------|---------|-------------|
| หม้อแปลงไฟฟ้า | Power Transformer | TX |
| เครื่องกำเนิดไฟฟ้า | Generator | GEN |
| สวิตช์สับเปลี่ยนอัตโนมัติ | Automatic Transfer Switch | ATS |
| ตู้จ่ายไฟหลัก | Main Distribution Board | MDB |
| เซอร์กิตเบรกเกอร์อากาศ | Air Circuit Breaker | ACB |
| ระบบปรับอากาศความแม่นยำ | Precision Cooling / CRAC | CRAC |
| เครื่องสำรองไฟ | Uninterruptible Power Supply | UPS |
| เครื่องเรียงกระแส | Rectifier | RECT |
| ระบบกราวด์ | Grounding / Earthing System | GND |
| ระบบป้องกันฟ้าผ่า | Lightning Protection System | LPS |
| อุปกรณ์ป้องกันไฟกระชาก | Surge Protective Device | SPD |
| แผนภาพสายเดี่ยว | Single Line Diagram | SLD |
| ระบบแจ้งเตือนอัคคีภัย | Fire Alarm System | FAS |
| แผงควบคุมสัญญาณเพลิงไหม้ | Fire Alarm Control Panel | FACP |
| เครื่องตรวจจับควัน | Smoke Detector | SD |
| เครื่องตรวจจับความร้อน | Heat Detector | HD |
| เครื่องตรวจจับเปลวไฟ | Flame Detector | FD |
| ระบบตรวจจับควันแบบดูดอากาศ | Very Early Smoke Detection Apparatus | VESDA |
| สัญญาณแจ้งเตือนด้วยมือ | Manual Call Point | MCP |
| ระบบดับเพลิงอัตโนมัติ | Fire Suppression System | FSS |
| ระบบดับเพลิงด้วยก๊าซ FM-200 | FM-200 Clean Agent Suppression | FM-200 |
| ระบบดับเพลิงด้วยก๊าซเฉื่อย | Inert Gas Suppression (IG-541) | IG-541 |
| ระบบสปริงเกลอร์ชนิดพรีแอ็กชั่น | Pre-action Sprinkler System | Pre-action |
| ความเข้มข้นการออกแบบ | Design Concentration | % vol |
| ระบบน้ำเย็น / ระบบชิลเลอร์ | Chilled Water Cooled / Chiller Plant | CHW |
| ชิลเลอร์ | Chiller | CH |
| ปั๊มน้ำเย็น | Chilled Water Pump | CHP |
| ปั๊มน้ำระบายความร้อน | Condenser Water Pump | CDP |
| หอระบายความร้อน | Cooling Tower | CT |
| อุณหภูมิน้ำเข้า/ออกหอระบายความร้อน | Entering/Leaving Water Temp | EWT / LWT |
| อุณหภูมิน้ำจ่าย/น้ำกลับ (เย็น) | Chilled Water Supply/Return Temp | CHWS / CHWR |
| อุณหภูมิน้ำจ่าย/น้ำกลับ (ระบาย) | Condenser Water Supply/Return Temp | CWS / CWR |
| ระบบอัตราส่วนสมรรถนะ | Coefficient of Performance | COP |
| ระบบปรับอากาศแม่นยำแบบ DX | DX Precision Cooling / CRAC | CRAC |
| เครื่องส่งลมเย็น (ใช้น้ำเย็น) | Chilled Air Cooled / CRAH | CRAH |
| แผงควบคุมระบบชิลเลอร์ | Chiller Plant Management System | CPMS |
| ตู้ระบายความร้อนอากาศ (นอกอาคาร) | Air-Cooled Condenser Unit | ACU |
| อุณหภูมิลมจ่าย/ลมกลับ | Supply Air Temp / Return Air Temp | SAT / RAT |
| สารทำความเย็น | Refrigerant | R410A / R32 |
