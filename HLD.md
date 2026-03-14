# GreenCorridor — High-Level Design (HLD)

**Version:** 1.0.0
**Platform:** Single-page web application (`code.html`)
**Domain:** Emergency Vehicle Routing & Traffic Preemption
**City Context:** Bangalore, India

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Module Breakdown](#4-module-breakdown)
5. [Data Flow](#5-data-flow)
6. [API Integrations](#6-api-integrations)
7. [Database Design](#7-database-design)
8. [Geofencing System](#8-geofencing-system)
9. [Multi-Ambulance Concurrency Engine](#9-multi-ambulance-concurrency-engine)
10. [Voice & Notification System](#10-voice--notification-system)
11. [Routing Engine](#11-routing-engine)
12. [UI/UX Design](#12-uiux-design)
13. [Security Considerations](#13-security-considerations)
14. [Limitations & Future Scope](#14-limitations--future-scope)
15. [Project Output](#15-project-output)

---

## 1. Project Overview

**GreenCorridor** is a real-time emergency ambulance navigation and traffic signal preemption system. It simulates how an ambulance dispatched from any location in Bangalore can have all traffic signals on its route turned green automatically, minimizing delays.

### Core Problem Solved

In Indian cities, ambulances lose critical minutes at red signals. GreenCorridor addresses this by:
- Computing the optimal driving route from a user-selected origin to a hospital/destination
- Activating a "green corridor" — preempting signals to green before the ambulance arrives
- Supporting multiple simultaneous ambulances with conflict resolution when two units approach the same signal

### Key Capabilities

| Capability | Description |
|---|---|
| Route Selection | OSRM fetches up to 3 driving alternatives; user picks their preferred path |
| Signal Preemption | Three-ring geofence (2 km / 1 km / 500 m) triggers green light preemption |
| Multi-Ambulance Fleet | Up to 4 concurrent ambulances with independent corridors |
| Conflict Arbitration | Priority-score formula resolves signal conflicts between fleet units |
| Dynamic Rerouting | Detects road blocks and recalculates route in real time |
| Voice Guidance | Multilingual speech alerts in English, Hindi, Kannada, Tamil, Telugu |
| Live Database | Firebase Realtime Database + in-browser DB simulation panel |

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        BROWSER (Client)                     │
│                                                             │
│  ┌───────────────┐   ┌──────────────┐   ┌───────────────┐  │
│  │  Splash Screen│   │   Map View   │   │  Sidebar HUD  │  │
│  │  (Input Page) │   │  (Leaflet)   │   │  (Controls)   │  │
│  └───────┬───────┘   └──────┬───────┘   └───────┬───────┘  │
│          │                  │                   │           │
│  ┌───────▼──────────────────▼───────────────────▼───────┐  │
│  │                    Core Engine                        │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │  │
│  │  │  Animation  │  │  Geofence   │  │  Conflict    │  │  │
│  │  │  Loop (rAF) │  │  Engine     │  │  Arbitration │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────────┘  │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │  │
│  │  │  Routing    │  │  Voice      │  │  DB Sync     │  │  │
│  │  │  Engine     │  │  Engine     │  │  (Firebase)  │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────┬──────────────────┬────────────────┬────────┘
                 │                  │                │
         ┌───────▼──────┐  ┌────────▼──────┐ ┌──────▼──────────┐
         │  OSRM Server  │  │  Nominatim    │ │  Firebase       │
         │  (Routing)    │  │  (Geocoding)  │ │  Realtime DB    │
         └──────────────┘  └───────────────┘ └─────────────────┘
```

### Architecture Pattern

The entire application is a **single-file SPA (Single Page Application)** built with vanilla HTML, CSS, and JavaScript — no build tools, no frameworks, no backend server required. External services are consumed directly from the browser via REST APIs.

---

## 3. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **UI Framework** | Vanilla HTML5 + CSS3 | No framework overhead; full control |
| **Map Rendering** | Leaflet.js v1.9.4 | Interactive map, markers, polylines |
| **Map Tiles** | CartoDB Dark Matter (free) | Dark-theme map tiles, no API key |
| **Routing** | OSRM (Open Source Routing Machine) | Road-network route calculation |
| **Geocoding** | Nominatim / OpenStreetMap | Address search → lat/lng |
| **Geospatial Math** | Turf.js v6 | Geofence distance calculations |
| **Database** | Firebase Realtime Database | Live state sync across sessions |
| **Speech** | Web Speech API (browser native) | Multilingual voice alerts |
| **Fonts** | Google Fonts (Space Mono, Syne) | Monospace HUD + display font |
| **Hosting** | Any static server (e.g., `npx serve`) | No backend required |

---

## 4. Module Breakdown

### M1 — Splash / Input Screen
- Animated neon grid background with floating orbs, corner brackets, and a scan-line effect
- Origin and destination inputs with live Nominatim-powered autocomplete suggestions
- Microphone input (Web Speech API) for voice-based location entry
- GPS "Use My Location" button (Geolocation API)
- On submit: OSRM fetches up to 3 route alternatives → **Route Selection Popup** appears

### M2 — Route Selection Popup
- Shown after OSRM returns alternatives, before the map loads
- Displays each route as a card: badge (FASTEST / ALTERNATIVE 1 / ALTERNATIVE 2), distance (km), estimated duration (min)
- User clicks to select; chosen route is applied and map launches
- "← Back to route setup" to cancel and re-enter locations

### M3 — Map View
- Full-screen Leaflet map with CartoDB dark tiles
- **Ambulance marker** — animated SVG icon with rotation
- **Origin/Destination pins** — Google Maps-style teardrop SVG pins (green = origin, red = destination)
- **Route polyline** — blue line for planned route; green overlay for cleared portion
- **Signal markers** — miniature 3-light traffic signal icons; colour changes based on state
- **Geofence rings** — translucent circles (2 km outer, 1 km mid, 500 m inner) drawn around upcoming signals
- **Traffic vehicles** — 8 simulated vehicles (cars, buses, bikes) moving on side roads

### M4 — Animation Loop
- Runs on `requestAnimationFrame` (~60 fps)
- Updates ambulance position along the sampled route using `ambProgress` (0.0 → 1.0)
- Speed: 5 km/h base with ±0.5 km/h sinusoidal variation
- Speed multipliers: 0.35× when approaching a signal (braking zone), 0.10× at signal stop
- Calls `evaluateGeofences()` and `triggerSignalsByProgress()` every frame
- Pauses for 1.8 s at each cleared signal for visual realism

### M5 — Signal State Machine
Each signal cycles through these states:

```
normal → warning → green → recovery → normal
```

| State | Trigger | Visual |
|---|---|---|
| `normal` | Default / post-recovery | Red + Amber + Green (equal) |
| `warning` | Ambulance enters 1 km ring | Amber highlighted |
| `green` | Ambulance enters 500 m ring | Green highlighted + glow |
| `recovery` | Ambulance 4%+ past signal | Blue highlighted (handover) |
| `red` | Cross-traffic held (other vehicle) | Red highlighted + glow |

### M6 — Geofence Engine (`evaluateGeofences`)
- Runs every animation frame using Turf.js GPS distance
- Three concentric rings per signal: 2 km (outer awareness), 1 km (mid / approach), 500 m (inner / preemption)
- All voice alerts and signal state changes are driven exclusively by this engine
- Detects "passed through" by comparing `ambProgress` vs signal's route index

### M7 — Voice Engine
- Web Speech API `SpeechSynthesisUtterance`
- 5 supported languages: English (en-IN), Hindi (hi-IN), Kannada (kn-IN), Tamil (ta-IN), Telugu (te-IN)
- Deduplication by `key + sigId` prevents the same alert firing twice for the same signal
- Always calls `speechSynthesis.cancel()` before speaking to flush stale queue
- Alert types: `start`, `corridor`, `hospital`, `approaching`, `cleared`, `preemption`, `arrived`, `gps_lost`, `reroute`, `block_detected`, `conflict`

### M8 — Dynamic Rerouting
- On BLOCK ROAD event (manual or auto-detected): extracts ambulance's current GPS position, calls `fetchOSRMRouteAvoidingBlock()` with block coordinates
- OSRM fetches 3 alternatives; algorithm picks the one whose waypoints are furthest from the blocked coordinate
- Applies new route: old route becomes a ghost polyline, new route drawn in dashed blue
- Signals regenerated along new route; geofence state reset

### M9 — Multi-Ambulance Fleet
- Up to 4 simultaneous ambulances (1 primary + 3 secondary)
- Each secondary unit has its own: route, signal markers, geofence state, animation loop
- Load-balanced route selection: secondary unit's route picked to overlap least with active corridors
- Conflict arbitration runs every 1 second on all active units

### M10 — Live DB Panel
- Right-side slide-in drawer with 5 tabs: MISSIONS, FLEET_REGISTRY, SIGNAL_EVENTS, CONFLICT_LOG, GEOFENCE_EVENTS
- Renders every 500 ms; newly inserted rows flash green for 900 ms
- SQL-like query log shows INSERT INTO / UPDATE statements
- All writes mirrored to Firebase Realtime Database (throttled to prevent write storms)

---

## 5. Data Flow

### Emergency Start Flow

```
User selects origin + destination
         │
         ▼
OSRM API → up to 3 route alternatives
         │
         ▼
Route Selection Popup → user picks a route
         │
         ▼
sampleRoute() → 120-point polyline array
generateSignalsAlongRoute() → 3–8 signal objects
         │
         ▼
Map initialises → markers placed
startEmergency() → animateAmbulance() begins
         │
         ▼
  Every animation frame (~16 ms):
  ┌─────────────────────────────────────────┐
  │  ambProgress += (speed × dt) / totalDist │
  │  pos = getPositionOnRoute(ambProgress)   │
  │  ambMarker.setLatLng(pos)                │
  │  evaluateGeofences(pos) → voice + lights │
  │  triggerSignalsByProgress() → visuals    │
  │  updateHUD() → ETA, speed, cleared count │
  │  syncPrimaryUnit() → Firebase write      │
  └─────────────────────────────────────────┘
         │
         ▼
ambProgress >= 1 → triggerArrival()
         → speak("Approaching <dest>. Corridor complete.")
```

### Signal Preemption Flow

```
ambulance position updated
         │
         ▼
turf.distance(ambulance, signal) = distKm
         │
    distKm <= 2 km ?  →  draw geofence rings (awareness)
    distKm <= 1 km ?  →  speak('approaching') + warning state + cross-traffic countdown
    distKm <= 0.5 km? →  speak('preemption') + green state + stop cross-traffic
    ambProgress past signal? → speak('cleared') + 1.8 s pause + recovery state
```

### Firebase Write Flow

```
dbUpsert(table, key, row)         ← live state (MISSIONS, FLEET_REGISTRY)
    │
    ├── throttle check (5 s window per key)
    │       yes → fbSet(path, data)  → Firebase .set()
    │       no  → skip write
    └── update in-memory DB + flash animation

dbAppend(table, row, cap)         ← events (SIGNAL_EVENTS, CONFLICT_LOG, GEOFENCE_EVENTS)
    │
    └── fbPush(path, data) → Firebase .push() (no throttle — events are one-shot)
```

---

## 6. API Integrations

### OSRM (Open Source Routing Machine)

| Property | Value |
|---|---|
| Endpoint | `https://router.project-osrm.org/route/v1/driving/` |
| Auth | None (public demo server) |
| Parameters | `overview=full&geometries=geojson&alternatives=3` |
| Response | GeoJSON polyline, distance (m), duration (s) per route |
| Usage | Route calculation, alternative routes, block avoidance, rerouting, fleet load balancing |
| Rate Limit | Fair-use (demo server); self-host OSRM for production |

### Nominatim (OpenStreetMap Geocoding)

| Property | Value |
|---|---|
| Endpoint | `https://nominatim.openstreetmap.org/search` |
| Auth | None |
| Parameters | `q`, `format=json`, `limit=8`, `viewbox` (Bangalore bbox), `countrycodes=in` |
| Usage | Autocomplete suggestions for origin/destination text input |
| Rate Limit | 1 request/second (enforced in code via debounce) |

### Firebase Realtime Database

| Property | Value |
|---|---|
| SDK | Firebase JS SDK v10.7.1 (compat mode) |
| Auth | Restricted by security rules (read: authenticated; write: authenticated) |
| Operations | `.set()` for live state, `.push()` for events |
| Throttle | 5 000 ms for live-state tables; immediate for event tables |

---

## 7. Database Design

### In-Memory Schema (JavaScript `DB` object)

#### MISSIONS (Map)
| Field | Type | Description |
|---|---|---|
| `missionId` | string | e.g., `MISSION-AMB-01` |
| `ambId` | string | Ambulance unit ID |
| `origin` | string | Origin location name |
| `dest` | string | Destination location name |
| `distKm` | number | Total route distance |
| `signals` | number | Number of signals on route |
| `state` | string | `active` / `arrived` |
| `_ts` | number | Last updated timestamp |

#### FLEET_REGISTRY (Map)
| Field | Type | Description |
|---|---|---|
| `id` | string | Unit ID (e.g., `AMB-01`) |
| `severity` | number | 1–3 (low / medium / critical) |
| `progress` | number | 0.0–1.0 route completion |
| `distKm` | number | Total route distance |
| `state` | string | `standby` / `active` / `arrived` |
| `priorityScore` | number | `severity × 1000 − etaSeconds` |

#### SIGNAL_EVENTS (Array, capped at 60)
| Field | Type | Description |
|---|---|---|
| `amb` | string | Ambulance ID |
| `signal` | string | Signal name |
| `event` | string | `approaching` / `cleared` / `preemption` |
| `ts` | string | ISO 8601 timestamp |

#### CONFLICT_LOG (Array, capped at 40)
| Field | Type | Description |
|---|---|---|
| `winner` | string | Winning unit ID |
| `loser` | string | Losing unit ID |
| `signal` | string | Contested signal name |
| `winScore` | number | Winner's priority score |
| `loseScore` | number | Loser's priority score |
| `ts` | string | ISO 8601 timestamp |

#### GEOFENCE_EVENTS (Array, capped at 40)
| Field | Type | Description |
|---|---|---|
| `amb` | string | Ambulance ID |
| `signal` | string | Signal name |
| `ring` | string | `outer` / `mid` / `inner` |
| `distKm` | number | Distance at trigger |
| `ts` | string | ISO 8601 timestamp |

### Firebase Realtime Database Structure

```
greencorridor-bf773/
├── missions/
│   └── {missionId}/          ← .set() — live state, throttled 5 s
├── fleet_registry/
│   └── {unitId}/             ← .set() — live state, throttled 5 s
├── signal_events/
│   └── {pushId}/             ← .push() — immediate
├── conflict_log/
│   └── {pushId}/             ← .push() — immediate
└── geofence_events/
    └── {pushId}/             ← .push() — immediate
```

---

## 8. Geofencing System

### Three-Ring Model

```
         ┌─────────────────────────────────┐  2.0 km — Outer Ring
         │   ┌─────────────────────────┐   │  (Awareness: draw rings, no action)
         │   │   ┌─────────────────┐   │   │  1.0 km — Mid Ring
         │   │   │   ┌─────────┐   │   │   │  (Approach: warning state, cross-traffic countdown)
         │   │   │   │  🚦 SIG │   │   │   │  0.5 km — Inner Ring
         │   │   │   └─────────┘   │   │   │  (Preemption: force green, stop cross-traffic)
         │   │   └─────────────────┘   │   │
         │   └─────────────────────────┘   │
         └─────────────────────────────────┘
                        🚑 →
```

### Ring Entry Actions

| Ring | Distance | Action |
|---|---|---|
| Outer | ≤ 2.0 km | Draw geofence circles on map |
| Mid | ≤ 1.0 km | Signal → warning state; start 28 s cross-traffic countdown; speak "Approaching" |
| Inner | ≤ 0.5 km | Signal → green state; stop countdown; speak "Signal preempted. Green light activated." |
| Passed | `ambProgress > sigProg + 0.018` | Signal → recovery → normal; speak "Intersection cleared"; 1.8 s pause |

### Implementation Detail

- Distance computed using `turf.distance()` (Haversine formula on real GPS coordinates)
- Direction detection (approaching vs. receding) uses route progress comparison, not raw distance change
- `_geoPassedSignals` Set prevents re-firing "cleared" if ambulance slows near a signal
- Geofence state is reset on every `startEmergency()` and reroute call

---

## 9. Multi-Ambulance Concurrency Engine

### Fleet Architecture

```
Primary (AMB-01)         Secondary Units (AMB-02 to AMB-04)
    │                           │
    ├── own route                ├── own route (load-balanced away from AMB-01)
    ├── own animation loop       ├── own animation loop (requestAnimationFrame)
    ├── own signal markers       ├── own signal markers (scoped per unit)
    └── own geofence state       └── own geofence state
```

### Priority Scoring

```
priorityScore = (severity × 1000) − etaToConflictPoint (seconds)
```

- `severity`: 1 (low) / 2 (medium) / 3 (critical) — set when unit is added
- `etaToConflict`: estimated seconds until unit reaches the contested signal (based on 5 km/h base speed)
- Higher score = higher priority = gets green; lower score unit is held

### Conflict Resolution Flow

```
Every 1 000 ms (setInterval):
  for each pair of active units (A, B):
    for each pair of signals (one from A's route, one from B's route):
      if same physical signal (within 80 m) AND both units within 0.6 km:
        calculate etaA, etaB
        calculate scoreA, scoreB
        winner = higher score → keeps green
        loser  → heldByConflict = true → animation loop pauses
  display conflict banner with scores
  log to CONFLICT_LOG + Firebase
```

### Load Balancing

When a secondary ambulance is added:
1. OSRM fetches 3 route alternatives for the new unit
2. Each alternative's waypoints are compared against all existing active corridors
3. The route with minimum overlap (fewest waypoints within 100 m of existing routes) is selected

---

## 10. Voice & Notification System

### Voice Alerts

| Key | Trigger | Text (en-IN) |
|---|---|---|
| `start` | Emergency begins | Route confirmed. N intersections. ETA X minutes. |
| `corridor` | Corridor activated | Green corridor active. All signals coordinated. |
| `hospital` | Hospital notified | [Dest] notified. ER team is preparing. |
| `approaching` | Ambulance enters 1 km ring | Approaching signal. Corridor active. |
| `preemption` | Ambulance enters 500 m ring | 500 metres. Signal preempted. Green light activated. |
| `cleared` | Ambulance passes signal | Intersection cleared. Proceed straight. |
| `arrived` | Route complete | Approaching [dest]. Corridor complete. Thank you. |
| `gps_lost` | GPS signal lost | GPS lost. Maintaining corridor 30 seconds. |
| `reroute` | Road block detected | Road blocked. Recalculating route. |
| `conflict` | Multi-unit conflict | Priority conflict detected. Higher-priority unit has right of way. |

### Deduplication Logic

```
dedupKey = key + "-" + sigId    (e.g., "approaching-SIG-03")
if dedupKey === _lastSpokenKey → skip
else → cancel current speech → speak after 60 ms
```

### Map Popups (non-intrusive)

| Popup | Position | Trigger | Duration |
|---|---|---|---|
| Start popup | Bottom-right corner | Emergency start | 3.5 s |
| Signal popup | Bottom-left (above legend) | Approaching / cleared | 3.5 s |

---

## 11. Routing Engine

### Route Computation Pipeline

```
1. Fetch from OSRM (up to 3 alternatives, GeoJSON)
2. For each candidate:
   a. Extract [lat,lng] polyline from GeoJSON
   b. Compute road distance (meters from OSRM)
   c. Compute A* deviation score (straight-line efficiency)
3. Sort: primary = road distance; tie-break (within 5%) = A* deviation
4. Return all candidates with distance + duration to user for selection
5. After user selects: sampleRoute() → 120 evenly-spaced waypoints
6. generateSignalsAlongRoute() → 3–8 signals placed at even intervals
7. Cross-validate: road distance ≥ Haversine straight-line (sanity check)
```

### A* Deviation Score

Measures how directly a route travels between origin and destination:
```
deviation = Σ (distance of each waypoint from the origin→dest straight line)
```
Lower deviation = more direct path. Used as tie-breaker when two routes are within 5% of each other in distance.

### Offline Fallback

If OSRM is unreachable:
- Route = straight line between origin and destination
- Distance = Haversine formula
- Signals generated along the straight-line segment
- Warning shown to user

---

## 12. UI/UX Design

### Screen Flow

```
Splash / Input Screen
        │
        ▼ (after OSRM fetch)
Route Selection Popup (new modal overlay)
        │
        ▼ (user selects a route)
Map View + Sidebar HUD  ←──────────────────┐
        │                                   │
        ├── START EMERGENCY → animation     │
        ├── GPS LOST / REROUTE / CONFLICT   │
        ├── BLOCK ROAD → reroute modal      │
        ├── ADD AMBULANCE → fleet modal     │
        └── FLEET VIEW / LIVE DB panel      │
        │                                   │
        ▼ (ambProgress = 1.0)               │
Arrival State → reset → ──────────────────┘
```

### Colour System

| Token | Hex | Usage |
|---|---|---|
| `--green` | `#3fb950` | Active corridor, cleared signals, ambulance marker |
| `--amber` | `#e3b341` | Warning state, distance metric |
| `--red` | `#f85149` | Stopped signals, invalid input |
| `--blue` | `#58a6ff` | ETA, secondary routes, recovery state |
| `--purple` | `#bc8cff` | Fleet unit 3, alternative route badge |
| `--bg` | `#0d1117` | Page background |
| `--panel` | `#161b22` | Sidebar background |

### Splash Screen Design Features
- Animated neon grid (`background-image` CSS animation)
- Floating gradient orbs (`blur(70px)`)
- Corner targeting brackets
- Horizontal scan-line animation
- Stats strip (signal count, coverage radius, route engine, conflict resolution)

### Sidebar HUD
- ETA (min), Speed (km/h), Signals Cleared, Distance Left
- Route display: origin → destination with road distance
- Signal list: live-updating table of all signals with state and distance
- Voice EQ animation (speaking indicator)
- Language selector (5 languages)
- Simulation buttons: GPS LOST, REROUTE, CONFLICT, BLOCK ROAD
- Fleet controls: ADD AMBULANCE, FLEET VIEW
- LIVE DB panel toggle

---

## 13. Security Considerations

### Firebase Security Rules (Recommended)

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```

- Firebase API key is client-side (unavoidable for browser apps); restrict via Firebase console using HTTP referrer restrictions
- No user PII is stored — only mission state, signal events, and geofence logs
- Rate limiting on Nominatim (1 req/s) prevents API abuse
- OSRM public demo server should be replaced with self-hosted OSRM for production to avoid rate limits and data leakage

### Write Throttling

Live-state tables (MISSIONS, FLEET_REGISTRY) are throttled to one write per 5 seconds per key, preventing Firebase write storms from the 60 fps animation loop.

---

## 14. Limitations & Future Scope

### Current Limitations

| Limitation | Description |
|---|---|
| Single-file app | All code in one HTML file — maintainability degrades at scale |
| No real GPS | Ambulance movement is simulated; no actual device GPS feed |
| No real signal hardware | Signal preemption is visual simulation only; no physical signal controller integration |
| Demo OSRM server | Public demo server has rate limits and may be slow; not suitable for production |
| No authentication | Anyone with the URL can view/write to Firebase |
| No offline map tiles | Map tiles require internet; no offline caching |
| Bangalore-only locations | 20 hardcoded Bangalore locations; full Nominatim search is available but Bangalore-biased |

### Future Scope

| Feature | Description |
|---|---|
| Real GPS integration | Connect to device GPS / fleet management system for live ambulance position |
| Signal hardware API | Integrate with physical traffic controller APIs (SCATS, SCOOT) for real preemption |
| Backend server | Node.js / Python backend for authentication, route caching, audit logs |
| Mobile app | React Native / Flutter ambulance driver app with real-time alerts |
| Hospital ER integration | Push notifications to hospital ER systems on ambulance departure and ETA |
| Traffic density layer | Overlay real-time traffic data (Google Maps API / HERE) to improve routing |
| Analytics dashboard | Firebase → BigQuery → Looker Studio for corridor performance metrics |
| Multi-city support | Dynamic city bounding box from user's GPS location |
| Pedestrian warnings | Alert pedestrians via smart crosswalk signals in the green corridor |

---

## 15. Project Output

This section documents the full visual output produced by the running application (`code.html`) — every screen, panel, and state the user encounters.

---

### Screen 1 — Splash / Dispatch Screen

The first thing the user sees when opening the app. A full-screen dark panel with animated neon-green visual effects.

```
┌──────────────────────────────────────────────────┐
│  ● SYSTEM ONLINE                                 │
│  // DYNAMIC GREEN CORRIDOR SYSTEM                │
│                                                  │
│   GREEN                                          │
│   CORRIDOR                                       │
│                                                  │
│   ● EMERGENCY DISPATCH                           │
│  ┌──────────────────────────────────────────┐    │
│  │  8ms Signal Latency | 99.8% Uptime       │    │
│  │  3-RING Geofence                         │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ● AMBULANCE ORIGIN                              │
│  [ Type pickup location…          ] [🎤] [📍]   │
│                                                  │
│  ● DESTINATION / HOSPITAL                        │
│  [ Type destination or hospital…  ] [🎤]        │
│                                                  │
│  ▶  START CORRIDOR                               │
│  ─────────────────────────────────────────────  │
│  🛣️ OSRM + A*   🚦 3-Ring Geofence              │
│  🚑 Multi-Fleet  ⚡ Priority Arbitration         │
└──────────────────────────────────────────────────┘
```

**Visual effects active on this screen:**
- Animated scrolling neon-green dot-grid background
- Three blurred gradient orbs (green, cyan, purple) floating in the background
- Green glowing top-edge neon line on the card
- Corner targeting brackets (top-left, top-right, bottom-left, bottom-right)
- Horizontal cyan scan-line sweeping top to bottom every 5 s
- Pulsing green "SYSTEM ONLINE" indicator dot
- Stats strip values pulsing at 50% opacity in a 3 s cycle

**Autocomplete behaviour:**
- Typing ≥ 1 character → instant local suggestions dropdown (up to 4 results)
- Typing ≥ 3 characters → Nominatim API results merged below local results
- Selected location → input border turns green

---

### Screen 2 — Route Picker Overlay

Shown immediately after OSRM returns alternatives. Overlays the splash screen (blur backdrop).

```
┌─────────────────────────────────────────────────┐
│  // SELECT ROUTE                                │
│  Choose your corridor path                      │
│  Indiranagar 100ft Road → St. John's Hospital   │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ [FASTEST]   3.82 km                     │   │
│  │             ~9 min estimated travel     │ SELECT → │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │ [ALTERNATIVE 1]  4.10 km               │   │
│  │             ~11 min estimated travel    │ SELECT → │
│  └─────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────┐   │
│  │ [ALTERNATIVE 2]  4.55 km               │   │
│  │             ~12 min estimated travel    │ SELECT → │
│  └─────────────────────────────────────────┘   │
│                  ← Back to route setup          │
└─────────────────────────────────────────────────┘
```

Badge colours: FASTEST = green, ALTERNATIVE 1 = blue, ALTERNATIVE 2 = purple.
Hovering a row: slight upward translate + green border glow.

---

### Screen 3 — Main Map View (Pre-Emergency / Standby)

After route selection, the splash fades out (scale + opacity transition) and the full map layout appears.

```
┌──────────────┬──────────────────────────────────────────┐
│   SIDEBAR    │                  MAP                     │
│              │                                          │
│ // DYNAMIC   │   [CartoDB Dark Matter tiles]            │
│ GREEN        │                                          │
│ CORRIDOR     │   🟢──────────────────────────🔴        │
│ ● LIVE NAV   │   (origin pin)   route  (dest pin)      │
│              │                                          │
│ ETA    Speed │   🚦 SIG-01  🚦 SIG-02  🚦 SIG-03 ...  │
│  —      0   │                                          │
│ Cleared Dist │   🚗  🚌  🏍️  (background vehicles)    │
│  0      —   │                                          │
│              │              [NEXT SIGNAL]  —            │
│ ORIGIN       │              [CORRIDOR]  STANDBY         │
│  ——          │                                          │
│ ——km road    │                                          │
│ DESTINATION  │                                          │
│  ——          │   ┌ MAP LEGEND ─────────────────────┐   │
│              │   │ 🟢 Ambulance / Green Signal      │   │
│ SIGNALS (8)  │   │ 🟡 Warning Signal                │   │
│ ⚫ SIG-01    │   │ 🔴 Held Red Signal               │   │
│ ⚫ SIG-02    │   │ 🔵 Recovery Signal               │   │
│ ...          │   │ 🟣 Other Vehicles                │   │
│              │   │ 🟢 Cleared Corridor (faded)      │   │
│ VOICE NAV    │   └──────────────────────────────────┘   │
│ [EQ bars] EN │                                          │
│ Tap START... │                              [zoom ±]    │
│              │                                          │
│ ▶ START EMER │                                          │
│ GPS | REROUTE│                                          │
│ CONFLICT|🚧  │                                          │
│ ➕ ADD AMB   │                                          │
│ 🗺 FLEET VIEW│                                          │
│ 🗄 LIVE DB   │                                          │
└──────────────┴──────────────────────────────────────────┘
```

**Map elements visible at start:**
- Dashed green-tinted polyline = planned route
- Green teardrop pin = ambulance origin (with location name tooltip)
- Red teardrop pin = destination (with location name tooltip)
- 🚑 ambulance marker at origin (with two pulsing concentric ring animations)
- 3–8 miniature 3-light traffic signal icons along the route (all in `normal` state — red+amber+green equally lit)
- 8 background vehicle emoji icons on side roads

---

### Screen 4 — Active Emergency (Corridor Running)

After pressing **START EMERGENCY**, the animation loop begins.

```
SIDEBAR changes:
  ▶ START EMERGENCY  →  ■ END EMERGENCY  (red, pulsing)
  ETA: 7 min  Speed: 22 km/h  Cleared: 0  Dist: 3.8 km
  CORRIDOR: STANDBY → ACTIVE  (green text)
  Voice text: "Route to St. John's Hospital confirmed. 5 signals."

MAP changes:
  🚑 ambulance begins moving along route
  Solid green line grows behind ambulance (cleared corridor trail)
  Map auto-pans to keep ambulance centred
```

**Signal state progression visible on map as ambulance approaches:**

```
Distance   Signal icon       Sidebar row          Voice
─────────────────────────────────────────────────────────
2.0 km     (rings appear)    [2km AWARE] badge    (silent)
1.0 km     amber middle lit  [1km REDUCING] badge "Approaching signal"
                             ⏱ Cross-traffic: 28s countdown
0.5 km     green bottom lit  [500m PREEMPTED]     "500 metres. Signal
           green glow/pulse  badge                 preempted. Green
                                                   light activated."
passed     blue middle lit   RECOVERY badge        "Intersection cleared."
           → normal 5s later normal badge          (1.8s pause in movement)
```

**HUD (top-right map overlay):**
```
┌─────────────┐
│ NEXT SIGNAL │
│    142      │  ← metres away
│    metres   │
└─────────────┘
┌─────────────┐
│  CORRIDOR   │
│   ACTIVE    │  ← green
└─────────────┘
```

**Voice EQ bars:** 4 vertical bars animate up/down while speech is playing.

---

### Screen 5 — Geofence Rings on Map

When the ambulance enters the 2 km awareness zone of a signal:

```
Signal location on map:

     ┌─────────────────────────────────┐  ← 2000m blue dashed circle
     │   ┌─────────────────────────┐   │
     │   │   ┌─────────────────┐   │   │  ← 1000m amber dashed circle
     │   │   │   ┌─────────┐   │   │   │
     │   │   │   │  🚦 SIG │   │   │   │  ← 500m green dashed circle
     │   │   │   └─────────┘   │   │   │
     │   │   └─────────────────┘   │   │
     │   └─────────────────────────┘   │
     └─────────────────────────────────┘
                   🚑 →
```

All three rings drawn simultaneously when outer zone is first entered. Rings remain until ambulance passes through the signal.

---

### Screen 6 — Road Block & Reroute

Triggered by the **🚧 BLOCK ROAD** button or auto-detected after 20–35 s.

```
MAP output:
  🚧 marker placed on route (with tooltip "Road Blocked")
  Short red dashed cross-hatch line across the road

  Old route remainder: red dashed polyline (faded)
  Traveled path:       faded green polyline (frozen in place)
  New route:           bright blue dashed polyline

  New signal markers appear along the new route

TOP-LEFT alert banner (red):
  🚧 Driver reported block — recalculating route to St. John's Hospital…
  (auto-dismisses after 4 s)
  → then: ✓ Route recalculated — 4.2 km to St. John's Hospital. Signal corridor re-synced.

VOICE: "Road blocked. Recalculating route. Stay on current road."
CORRIDOR state: RECALCULATING (red) → ACTIVE (green)
```

---

### Screen 7 — Multi-Ambulance Fleet View

After pressing **➕ ADD AMBULANCE** and deploying a second unit:

```
MAP output:
  Second 🚑 marker (blue colour, slightly smaller)
  Second dashed route line in unit colour
  Second set of signal markers (prefixed AMB-02-SIG-NN)
  Second set of geofence rings in unit colour

FLEET PANEL (bottom-centre map overlay):
┌────────────────────────────────────────────────────┐
│  // COMMAND CENTER — ACTIVE FLEET                  │
│  ┌───────────────┐  ┌───────────────┐              │
│  │ 🚑 AMB-01    │  │ 🚑 AMB-02    │              │
│  │ CRITICAL      │  │ HIGH          │              │
│  │ ████████░░   │  │ ████░░░░░░   │              │
│  │ ACTIVE 2.1km  │  │ ACTIVE 3.8km  │              │
│  │ ▲2850         │  │ ▲1750         │              │
│  └───────────────┘  └───────────────┘              │
└────────────────────────────────────────────────────┘
  Progress bar colour = unit colour
  ▲ score: green if high, amber if mid, red if held
```

**FLEET VIEW button** → map auto-fits bounds to show all active ambulances simultaneously.

---

### Screen 8 — Conflict Detection

When two ambulances approach the same signal within 0.6 km:

```
FLEET PANEL cards:
  AMB-01 card: normal green border
  AMB-02 card: RED pulsing border  ← "🔴 HELD"

CONFLICT BANNER (below fleet cards):
  ⚡ AMB-01 (score 2706) > AMB-02 (score 1490) at Domlur Flyover Signal

TOP RIGHT header tag:
  ⚡ CONFLICT ACTIVE  (red text)

VOICE: "Another ambulance has priority. 30 second delay expected."

AMB-02 animation loop:
  Paused (heldByConflict = true) until AMB-01 clears the contested signal
```

---

### Screen 9 — Live DB Simulation Panel

Opened via the **🗄 LIVE DB SIMULATION** button — slides in from the right edge.

```
┌────────────────────────────────────────┐
│  // LIVE DATABASE SIMULATION        ✕  │
│ [MISSIONS][FLEET][SIGNALS][CONFLICTS]  │
│          [GEOFENCE]                    │
├────────────────────────────────────────┤
│  MISSION_ID   AMB  ORIGIN  DEST  STATE │
│  MISSION-…01  AMB-01  Ind…  St.J  active│  ← green flash on insert
│                                        │
│  (rows update every 500 ms)            │
│  3 rows                                │
├────────────────────────────────────────┤
│  // QUERY LOG — last 15 statements     │
│  INSERT INTO  SIGNAL_EVENTS  time,unit │  "{'event':'green'…}"  12:34:01
│  UPDATE       MISSIONS       MISSION…  │  "{'state':'activ…}"  12:33:58
│  INSERT INTO  GEOFENCE_EVENTS  time,… │  "{'zone':'inner'…}"   12:33:55
└────────────────────────────────────────┘
```

- Active tab has green underline border
- New rows flash bright green for 850 ms on insert
- Query log: verb in blue, table in amber, preview in dim grey

---

### Screen 10 — Arrival State

When `ambProgress` reaches 1.0:

```
SIDEBAR:
  ■ END EMERGENCY → ▶ START EMERGENCY
  CORRIDOR: COMPLETE (blue text)

MAP:
  🚑 stops at destination marker
  Solid green cleared trail spans full route

VOICE: "Approaching St. John's Hospital. Corridor complete. Thank you."

3 seconds later:
  All signal markers reset to normal (red+amber+green equal)
  Route line remains visible
```

---

### Full Application State Summary

| State | Corridor Label | Button Label | Ambulance | Signals |
|---|---|---|---|---|
| Standby | STANDBY (grey) | ▶ START EMERGENCY | At origin, static | All normal |
| Active | ACTIVE (green) | ■ END EMERGENCY (red, pulse) | Moving along route | Cycling through states |
| Rerouting | REROUTING (amber) | ■ END EMERGENCY | Paused briefly | Reset on new route |
| Recalculating | RECALCULATING (red) | ■ END EMERGENCY | Paused briefly | Reset on new route |
| Conflict (loser) | ACTIVE (green) | ■ END EMERGENCY | Held/paused | Normal until released |
| Complete | COMPLETE (blue) | ▶ START EMERGENCY | At destination | All normal (3 s delay) |

---

### Colour Legend (Runtime)

| Colour | Meaning on Map |
|---|---|
| `#3fb950` solid line | Cleared corridor (traveled path) |
| `#3fb950` dashed line | Planned route (not yet traveled) |
| `#58a6ff` dashed line | Rerouted new path |
| `#f85149` dashed line | Blocked old route remainder |
| `#3fb950` circle | 500 m inner geofence ring |
| `#e3b341` circle | 1 km mid geofence ring |
| `#58a6ff` circle | 2 km outer geofence ring |
| `#3fb950` teardrop pin | Origin |
| `#f85149` teardrop pin | Destination |
| `#3fb950` ambulance ring | Primary ambulance (AMB-01) |
| `#58a6ff` / `#bc8cff` / `#e3b341` ambulance ring | Secondary fleet units |
| `rgba(188,140,255,0.4)` vehicle box | Background traffic vehicles |
