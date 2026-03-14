# GreenCorridor — Low-Level Design (LLD)

**Version:** 1.0.0
**File:** `code.html` (single-file SPA)
**Domain:** Emergency Ambulance Routing & Traffic Signal Preemption
**City:** Bangalore, India

---

## Table of Contents

1. [Global State & Constants](#1-global-state--constants)
2. [Data Structures](#2-data-structures)
3. [Module: Map Initialisation (`initMap`)](#3-module-map-initialisation-initmap)
4. [Module: Animation Loop (`animateAmbulance`)](#4-module-animation-loop-animateambulance)
5. [Module: Signal State Machine](#5-module-signal-state-machine)
6. [Module: Geofence Engine (`evaluateGeofences`)](#6-module-geofence-engine-evaluategeofences)
7. [Module: Routing Engine](#7-module-routing-engine)
8. [Module: Dynamic Rerouting](#8-module-dynamic-rerouting)
9. [Module: Voice Engine (`speak`)](#9-module-voice-engine-speak)
10. [Module: Autocomplete & Input Handling](#10-module-autocomplete--input-handling)
11. [Module: GPS & Microphone Input](#11-module-gps--microphone-input)
12. [Module: Multi-Ambulance Fleet Engine](#12-module-multi-ambulance-fleet-engine)
13. [Module: Conflict Arbitration (`detectAndResolveConflicts`)](#13-module-conflict-arbitration-detectandresolveconflicts)
14. [Module: Load Balancing (`fetchLoadBalancedRoute`)](#14-module-load-balancing-fetchloadbalancedroute)
15. [Module: Live DB Simulation & Firebase Sync](#15-module-live-db-simulation--firebase-sync)
16. [Module: Splash Screen & Route Picker](#16-module-splash-screen--route-picker)
17. [Module: Simulation Controls](#17-module-simulation-controls)
18. [UI Component Inventory](#18-ui-component-inventory)
19. [Function Reference Table](#19-function-reference-table)
20. [Error Handling & Fallbacks](#20-error-handling--fallbacks)
21. [Timing & Interval Summary](#21-timing--interval-summary)

---

## 1. Global State & Constants

### Route & Position State

| Variable | Type | Initial Value | Description |
|---|---|---|---|
| `userRoute` | `Array<[lat,lng]>` | `null` | 120-point sampled polyline from OSRM |
| `userSignals` | `Array<Signal>` | `null` | Generated signals along user route |
| `userRouteTotalDist` | `number` | `4.2` | Road distance in km |
| `activeRoute` | `Array<[lat,lng]>` | `ROUTE` (default) | Currently active route for animation |
| `activeSignals` | `Array<Signal>` | `SIGNALS` (default) | Signals on currently active route |
| `ambProgress` | `number` | `0` | Ambulance position (0.0 → 1.0) on route |
| `isRunning` | `boolean` | `false` | Emergency active flag |
| `clearedCount` | `number` | `0` | Number of signals cleared |
| `currentSigIdx` | `number` | `0` | Index of next upcoming signal |
| `selectedOrigin` | `Object\|null` | `null` | `{name, lat, lng}` of origin |
| `selectedDest` | `Object\|null` | `null` | `{name, lat, lng}` of destination |
| `selectedLang` | `string` | `'en-IN'` | Active voice language |

### Animation State

| Variable | Type | Description |
|---|---|---|
| `animFrame` | `number` | `requestAnimationFrame` handle for primary ambulance |
| `lastFrame` | `number` | Previous rAF timestamp (ms) for delta-time calculation |
| `waitUntil` | `number` | rAF timestamp until which ambulance pauses at signal |
| `lastPausedSig` | `number` | Index of last signal the ambulance paused at |
| `lastClearedIdx` | `number` | Index of last cleared signal (prevents double-fire) |
| `lastApproachIdx` | `number` | Index of last signal approach event |

### Geofence State

| Variable | Type | Description |
|---|---|---|
| `sigGeoState` | `Object<sigId, string>` | Current ring zone per signal: `idle\|outer\|mid\|inner` |
| `geoRingLayers` | `Object<sigId, Object>` | Leaflet circle layer references per signal |
| `crossTimers` | `Object<sigId, Object>` | Cross-traffic countdown per signal `{remaining, intervalId}` |
| `_geoPassedSignals` | `Set<sigId>` | Signals the ambulance has physically passed (prevents re-fire) |

### Geofence Radii (km)

```js
const GEO_RADII = { outer: 2.0, mid: 1.0, inner: 0.5 };
```

### Road Block State

| Variable | Type | Description |
|---|---|---|
| `roadBlocks` | `Array<Object>` | `[{lat, lng, marker, line}]` — active block markers |
| `autoBlockTimer` | `number\|null` | setTimeout handle for auto block detection |

### Leaflet Map References

| Variable | Description |
|---|---|
| `map` | Leaflet map instance |
| `ambMarker` | Primary ambulance marker |
| `routeLine` | Dashed route polyline |
| `clearedLine` | Solid green cleared-path polyline |
| `altRouteLine` | Blue dashed rerouted path |
| `oldRouteGhostLine` | Red dashed blocked remainder |
| `oldRouteTraveledLine` | Faded green traveled old path |
| `originMarker` / `destMarker` | Teardrop pin markers |
| `signalMarkers` | `Object<sigId, L.Marker>` |
| `vehicleMarkers` | `Object<vehicleId, L.Marker>` |
| `vehicleProgress` | `Object<vehicleId, number>` — 0.0→1.0 loop progress |

---

## 2. Data Structures

### Signal Object

```js
{
  id:    'SIG-01',              // string — unique identifier
  name:  'CMH Road Junction',  // string — display name
  lat:   12.9761,              // number — WGS84 latitude
  lng:   77.6371,              // number — WGS84 longitude
  state: 'normal'              // 'normal'|'warning'|'green'|'red'|'recovery'
}
```

### Vehicle Object (OTHER_VEHICLES)

```js
{
  id:    'V01',
  emoji: '🚗',
  lat:   12.9760,
  lng:   77.6395,
  route: [[lat,lng], [lat,lng], [lat,lng]],  // 3-point looping route
  speed: 0.00008                              // progress units per frame
}
```

### Location Object (LOCATIONS)

```js
{
  name: 'Indiranagar 100ft Road',
  lat:  12.9784,
  lng:  77.6408,
  _gps: true   // only present for GPS-sourced entries
}
```

### Fleet Unit Object

```js
{
  id:              'AMB-02',
  severity:        3,            // 1=low, 2=high, 3=critical
  color:           '#58a6ff',
  origin:          { name, lat, lng },
  dest:            { name, lat, lng },
  route:           [[lat,lng], ...],  // 120-point sampled
  signals:         [Signal, ...],
  distKm:          6.4,
  progress:        0.0,
  state:           'standby',    // 'standby'|'active'|'arrived'
  marker:          L.Marker,
  routeLine:       L.Polyline,
  clearedLine:     L.Polyline,
  animFrame:       null,
  priorityScore:   0,
  etaToConflict:   Infinity,
  clearedCount:    0,
  lastFrame:       0,
  waitUntil:       0,
  lastPausedSig:   -1,
  lastClearedIdx:  -1,
  lastApproachIdx: -1,
  heldByConflict:  false,
  signalMarkers:   {},           // scoped Leaflet markers
  sigGeoState:     {},
  geoRingLayers:   {},
  crossTimers:     {},
}
```

### In-Memory DB Schema

```js
const DB = {
  MISSIONS:        Map<missionId, MissionRow>,
  FLEET_REGISTRY:  Map<unitId,    FleetRow>,
  SIGNAL_EVENTS:   Array<SignalEventRow>,   // cap 60
  CONFLICT_LOG:    Array<ConflictRow>,      // cap 40
  GEOFENCE_EVENTS: Array<GeofenceRow>,      // cap 40
}
```

**MissionRow fields:** `missionId, ambId, origin, dest, distKm, signals, state, _ts`
**FleetRow fields:** `id, severity, progress, distKm, state, priorityScore`
**SignalEventRow fields:** `time, unit, signal, event, lat, lng`
**ConflictRow fields:** `time, unit_a, unit_b, winner, loser, score_a, score_b, signal`
**GeofenceRow fields:** `time, unit, signal, zone, dist_m`

---

## 3. Module: Map Initialisation (`initMap`)

**Trigger:** Called once from `_launchMap()` after the splash screen exits.

### Steps

1. Remove `#mapLoader` spinner element.
2. Create `L.map` centred at `[12.9672, 77.6258]`, zoom 15, `preferCanvas: true`.
3. Add CartoDB Dark Matter tile layer (`https://{s}.basemaps.cartocdn.com/dark_all/...`).
4. Add zoom control (`bottomright`).
5. Add `routeLine` — dashed green-tinted polyline over `activeRoute`.
6. Add `clearedLine` — solid green polyline, initially empty `[]`.
7. Create **origin marker** — SVG teardrop (green, white circle), anchored at base.
8. Create **destination marker** — SVG teardrop (red, white circle).
9. Create **ambulance marker** — 32×32 div with animated `ambring` pulse rings + 🚑 emoji.
10. For each signal in `SIGNALS`: call `createSignalMarkerHTML('normal', sig.id)` → `L.divIcon` → `L.marker`.
11. For each vehicle in `OTHER_VEHICLES`: emoji icon in dark rounded box → `L.marker`.
12. Call `renderSignalList()`.

### `createSignalMarkerHTML(state, id)`

Returns inline HTML for a 3-light traffic signal icon. Color arrays per state:

| State | Red light | Amber light | Green light |
|---|---|---|---|
| `normal` | `#f85149` | `#e3b341` | `#3fb950` |
| `warning` | `#f85149` | `#e3b341ff` | `#3fb95022` |
| `green` | `#f8514922` | `#e3b34122` | `#3fb950ff` |
| `red` | `#f85149ff` | `#e3b34122` | `#3fb95022` |
| `recovery` | `#f8514922` | `#58a6ffff` | `#3fb95022` |

Box glow added via inline `box-shadow` for `green` and `red` states.

---

## 4. Module: Animation Loop (`animateAmbulance`)

**Trigger:** `requestAnimationFrame(animateAmbulance)` — called by `startEmergency()`.

### Per-Frame Execution (≈60 fps)

```
ts (DOMHighResTimeStamp)
│
├── Guard 1: if (!isRunning) → return
├── Guard 2: if (ts < waitUntil || _primaryHeldByConflict) → re-queue, return
│
├── dt = min(ts - lastFrame, 100)   // cap at 100ms to avoid jump on tab switch
│
├── Compute speedMult:
│     for each signal i:
│       sigProg = i / signals.length
│       dist    = sigProg - ambProgress
│       dist ∈ (0.005, 0.04)  → speedMult = min(speedMult, 0.35)  // braking
│       dist ∈ [0, 0.005]     → speedMult = min(speedMult, 0.10)  // near-stop
│
├── speedKmh = (5 + sin(ambProgress*8) * 0.5) * speedMult
├── ambProgress += (speedKmh/3600 * dt) / userRouteTotalDist
│
├── if ambProgress >= 1 → triggerArrival(); return
│
├── pos = getPositionOnRoute(ambProgress)
├── ambMarker.setLatLng(pos)
│
├── Rebuild clearedLine: sample every 0.02 progress steps
│
├── evaluateGeofences(pos[0], pos[1])   // GPS geofence check
├── triggerSignalsByProgress()          // visual signal state machine
├── moveOtherVehicles()                 // animate 8 side-road vehicles
├── updateHUD()                         // ETA, speed, dist, next signal
├── syncPrimaryUnit(ts)                 // keep fleet registry in sync
├── renderSignalList()                  // sidebar signal list re-render
│
├── map.panTo(pos, {animate:false})
└── animFrame = requestAnimationFrame(animateAmbulance)
```

### `getPositionOnRoute(t)`

```
totalSegs = route.length - 1
seg = min(t * totalSegs, totalSegs - 0.001)
i   = floor(seg)
frac = seg - i
return lerpLatLng(route[i], route[i+1], frac)
```

### `lerpLatLng(a, b, t)`

```
return [a[0] + (b[0]-a[0])*t, a[1] + (b[1]-a[1])*t]
```

### `updateHUD()`

| HUD Element | Formula |
|---|---|
| `speedVal` | `round(20 + sin(ambProgress*10)*4)` km/h (visual display speed) |
| `distVal` | `(1 - ambProgress) * userRouteTotalDist` km |
| `etaVal` | `max(1, round(distLeft / speed * 60))` min |
| `nextSigVal` | `max(0, (sigProg - ambProgress) * totalKm * 1000)` m |

### `moveOtherVehicles()`

- Each vehicle loops its 3-point route using `vehicleProgress[id] += speed`.
- `vehicleProgress % 1` gives current position.
- Vehicle halts (marker not updated) if within 150 m of any `green` or `red` signal.

---

## 5. Module: Signal State Machine

### `triggerSignalsByProgress()`

Runs every animation frame. Transitions signals based on ambulance's route progress:

| Condition (`dist = sigProg - ambProgress`) | Transition |
|---|---|
| `dist < -0.04` and state ≠ `normal` | → `recovery`, then `normal` after 5 s |
| `-0.01 ≤ dist < 0.01` and state ≠ `green` | → `green`; advance `currentSigIdx` |
| `0.01 ≤ dist < 0.03` and state = `normal` | → `green` (pre-clear) |
| `0.03 ≤ dist < 0.06` and state = `normal` | → `warning` |

### `updateSignalMarker(sigId, state)`

1. Find signal in `activeSignals`.
2. Rebuild icon HTML via `createSignalMarkerHTML`.
3. `signalMarkers[sigId].setIcon(icon)`.
4. Update `sig.state`.
5. Call `renderSignalList()`.
6. Call `dbLogSignalEvent(...)`.

### `renderSignalList()`

Maps `activeSignals` array → sidebar HTML rows. Each row shows:
- State icon/emoji, signal name, geofence zone badge (`geoZoneBadge`)
- Distance ahead (`m ahead`) calculated from `ambProgress`
- Cross-traffic countdown bar (`crossTimerHTML`)
- State badge label

---

## 6. Module: Geofence Engine (`evaluateGeofences`)

**Trigger:** Called every animation frame with current ambulance `[lat, lng]`.

### Algorithm per signal

```
ambPt = turf.point([ambLng, ambLat])

for each signal (sig, i):
  sigProg = i / signals.length
  prev    = sigGeoState[sig.id] || 'idle'

  // --- Passed-through detection ---
  ambPastSignal = ambProgress > sigProg + 0.018
  if ambPastSignal:
    if (prev==='inner'||prev==='mid') AND sig.id not in _geoPassedSignals:
      _geoPassedSignals.add(sig.id)
      clearedCount++
      speak('cleared', false, sig.id)
      showSignalPopup(...)
      waitUntil = lastFrame + 1800   // 1.8 s pause
    sigGeoState[sig.id] = 'idle'
    removeGeoRings(sig.id)
    stopCrossTimer(sig.id)
    continue

  if sig.state === 'recovery': skip

  distKm = turf.distance(ambPt, turf.point([sig.lng, sig.lat]), {units:'kilometers'})

  // --- Inner ring (≤ 0.5 km) ---
  if distKm <= 0.5 AND prev !== 'inner':
    sigGeoState = 'inner'
    drawGeoRings(sig)           // draw all 3 circles on map
    stopCrossTimer(sig.id)
    if state==='normal'||'warning': updateSignalMarker(sig.id, 'green')
    speak('preemption', false, sig.id)
    dbLogGeofenceEvent('inner', distKm)

  // --- Mid ring (≤ 1.0 km) ---
  else if distKm <= 1.0 AND prev not in ['mid','inner']:
    sigGeoState = 'mid'
    drawGeoRings(sig)
    startCrossTimer(sig.id, 28)   // 28-second countdown
    if state==='normal': updateSignalMarker(sig.id, 'warning')
    speak('approaching', false, sig.id)
    showSignalPopup('Approaching ...')
    dbLogGeofenceEvent('mid', distKm)

  // --- Outer ring (≤ 2.0 km) ---
  else if distKm <= 2.0 AND prev === 'idle':
    sigGeoState = 'outer'
    drawGeoRings(sig)
    dbLogGeofenceEvent('outer', distKm)

  // --- Outside all rings ---
  else if prev !== 'idle':
    sigGeoState = 'idle'
    removeGeoRings(sig.id)
    stopCrossTimer(sig.id)
```

### `drawGeoRings(sig)`

Creates 3 `L.circle` layers per signal (if not already drawn):

| Ring | Radius | Colour | Dash |
|---|---|---|---|
| Outer | 2000 m | `#58a6ff` (blue) | `8 10` |
| Mid | 1000 m | `#e3b341` (amber) | `5 7` |
| Inner | 500 m | `#3fb950` (green) | `4 5` |

### `startCrossTimer(sigId, seconds)`

Creates `setInterval` (1 s tick) decrementing `crossTimers[sigId].remaining` until 0, then auto-stops. Timer shown in sidebar as a progress bar.

---

## 7. Module: Routing Engine

### `haversineKm(a, b)`

Great-circle distance between two `[lat,lng]` points in km using the Haversine formula:

```
R    = 6371
dLat = (b[0]-a[0]) * π/180
dLng = (b[1]-a[1]) * π/180
x    = sin²(dLat/2) + cos(a[0]°) * cos(b[0]°) * sin²(dLng/2)
return R * 2 * atan2(√x, √(1-x))
```

### `routeLengthKm(pts)`

Sum of Haversine distances over all consecutive pairs in `pts`.

### `aStarDeviation(pts, origin, dest)`

Measures average perpendicular distance of each waypoint from the `origin→dest` straight-line corridor. Lower = more direct path. Used as tie-breaker.

```
for each point p in pts:
  t = clamp(dot(p-origin, dest-origin) / |dest-origin|², 0, 1)
  closest = origin + t*(dest-origin)
  totalDev += haversineKm(p, closest)
return totalDev / pts.length
```

### `sampleRoute(raw, maxPts=120)`

Down-samples a point array to `maxPts` evenly spaced points preserving endpoints:

```
step = (raw.length-1) / (maxPts-1)
return Array.from({length:maxPts}, (_,i) => raw[round(i*step)])
```

### `fetchOSRMRoute(originLoc, destLoc)`

```
GET https://router.project-osrm.org/route/v1/driving/
    {lng},{lat};{lng},{lat}
    ?overview=full&geometries=geojson&alternatives=3

Response → data.routes[]
  for each route:
    raw      = coordinates.map([lng,lat] → [lat,lng])
    distKm   = distance / 1000
    duration = round(duration / 60) min
    deviation = aStarDeviation(raw, origin, dest)

Sort:
  if |distA - distB| / minDist < 0.05:   → prefer lower deviation (A*)
  else:                                   → prefer shorter distance

best   = candidates[0]
points = sampleRoute(best.raw, 120)
distKm = max(best.distKm, haversineKm(origin, dest))  // sanity: road ≥ straight-line

return { points, distKm, candidateCount, allCandidates }
```

### `generateSignalsAlongRoute(pts)`

```
count = clamp(floor(pts.length / 6), 3, 8)
for i in 0..count-1:
  idx = round((i+1) * (pts.length-1) / (count+1))
  return { id:'SIG-NN', name:'Signal N', lat:pts[idx][0], lng:pts[idx][1], state:'normal' }
```

---

## 8. Module: Dynamic Rerouting

### Manual Reroute (`performReroute`)

```
speak('reroute')
corridorState → 'REROUTING'
currentPos = getPositionOnRoute(ambProgress)
fromLoc    = {lat:currentPos[0], lng:currentPos[1]}
destLoc    = selectedDest || last point of activeRoute

result = fetchOSRMRouteDifferentPath(fromLoc, destLoc)
applyNewRoute(currentPos, result.points, result.distKm)
```

### `fetchOSRMRouteDifferentPath(fromLoc, destLoc)`

Picks the OSRM alternative that diverges most from the remaining active route:

```
remainingPts = activeRoute.filter(i/segs >= ambProgress)

for each candidate route:
  sampled = sampleRoute(raw, 24)
  avgDivergence = mean(min distance from each sampled pt to remainingPts)

valid   = candidates where distKm <= minDist * 1.5
best    = max(avgDivergence) among valid
```

### Block-Avoiding Reroute (`recalculateAroundBlock`)

```
placeBlockMarker(lat, lng)   // 🚧 icon + red cross-hatch line
speak('block_detected', critical=true)
fetchOSRMRouteAvoidingBlock(fromLoc, destLoc, roadBlocks)
applyNewRoute(...)
```

### `fetchOSRMRouteAvoidingBlock(fromLoc, destLoc, blockPts)`

```
AVOID_RADIUS_KM = 0.18   // 180 m bubble

for each candidate:
  passesBlock = any route point within 0.18 km of any block point

Sort: safe routes first, then by distance + A* deviation
```

### `applyNewRoute(currentPos, newRoutePts, newDistKm)`

1. Freeze traveled path as faded green `oldRouteTraveledLine`.
2. Draw remaining old route as red dashed `oldRouteGhostLine`.
3. Draw new route as blue dashed `altRouteLine`.
4. Remove old signal markers, generate new signals with `generateSignalsAlongRoute`.
5. Reset: `ambProgress=0`, `currentSigIdx=0`, all counters, `clearAllGeofences()`.
6. Update `userRouteTotalDist`, HUD label.

### Auto Block Detection (`scheduleAutoBlockDetection`)

Fires once per emergency, 20–35 s after start:

```
delay = 20000 + random() * 15000   // 20–35 s
aheadProg = min(ambProgress + 0.35, 0.85)
pt = getPositionOnRoute(aheadProg)
reason = random pick from ['Accident detected', 'Road construction', 'Crowd blockage', 'Vehicle breakdown']
recalculateAroundBlock(pt[0], pt[1], reason)
```

---

## 9. Module: Voice Engine (`speak`)

### `VOICE_ALERTS` Dictionary

11 alert keys × 5 languages (`en-IN`, `hi-IN`, `kn-IN`, `ta-IN`, `te-IN`):

| Key | Trigger |
|---|---|
| `start` | Emergency begins |
| `corridor` | Green corridor activated |
| `hospital` | Hospital notified |
| `approaching` | Ambulance enters 1 km ring |
| `preemption` | Ambulance enters 500 m ring |
| `cleared` | Ambulance passes signal |
| `arrived` | Route complete |
| `gps_lost` | GPS signal lost |
| `reroute` | Road block detected |
| `block_detected` | Auto/manual block detected |
| `conflict` | Multi-unit conflict detected |

### `speak(key, critical, sigTag)`

```
text     = VOICE_ALERTS[key][selectedLang] || VOICE_ALERTS[key]['en-IN']
dedupKey = sigTag ? `${key}-${sigTag}` : key

if dedupKey === _lastSpokenKey → return   // deduplication
_lastSpokenKey = dedupKey

voiceText.textContent = text
eq.classList.add('speaking')

if window.speechSynthesis:
  speechSynthesis.cancel()               // flush stale queue
  utt = new SpeechSynthesisUtterance(text)
  utt.lang = selectedLang; utt.rate = 1.05; utt.volume = 1
  utt.onend → eq.classList.remove('speaking')
  setTimeout(() => speechSynthesis.speak(utt), 60)   // 60 ms flush gap
```

---

## 10. Module: Autocomplete & Input Handling

### Two-Stage Search

**Stage 1 — Instant local match:**
```
LOCATIONS.map(loc → {name, sub:'Bangalore', lat, lng, score:scoreTextMatch(query, loc.name)})
         .filter(score > 0)
         .sort(desc by score)
         .slice(0, 4)
→ renderSuggestions immediately
```

**Stage 2 — Debounced Nominatim (350 ms):**
```
GET https://nominatim.openstreetmap.org/search
    ?q={query} Bangalore&format=json&limit=8
    &viewbox=77.4,13.2,77.85,12.78&bounded=1&addressdetails=1&accept-language=en

apiItems = data.map(r → {name:shortDisplay, sub:suburb+city, lat, lng})
merged   = [...localItems, ...apiItems not in localItems].slice(0, 10)
→ renderSuggestions
```

### `scoreTextMatch(query, name)`

| Match type | Score |
|---|---|
| Exact match | 100 |
| Name starts with query | 80 |
| Name contains query | 60 |
| Word-level partial matches | `20 + word.length` per word |

### Keyboard Navigation (`onInputKeydown`)

- `ArrowDown` / `ArrowUp` → cycle `sugKbdIdx`, apply `.kbd-focus` class
- `Enter` → `pickSuggestion(field, sugKbdIdx >= 0 ? sugKbdIdx : 0)`
- `Escape` → `closeSuggestions(field)`

---

## 11. Module: GPS & Microphone Input

### GPS Input (`useCurrentLocation` / `splashUseCurrentLocation`)

```
navigator.geolocation.getCurrentPosition(
  { enableHighAccuracy:true, timeout:15000, maximumAge:0 }
)
→ success:
    Nominatim reverse geocode: GET /reverse?lat=&lon=&format=json&zoom=16
    displayName = road + suburb (or coordinate fallback)
    LOCATIONS.unshift({ name:displayName, lat, lng, _gps:true })
    selectItem('origin', displayName, lat, lng)
→ error: show mapped error string
```

Error code mapping: `1=Permission denied`, `2=GPS unavailable`, `3=Timeout`.

### Microphone Input (`startMicInput` / `splashMicInput`)

```
SR = window.SpeechRecognition || window.webkitSpeechRecognition
recognition.lang = 'en-IN'
recognition.maxAlternatives = 8
recognition.interimResults = false

onresult:
  transcripts = all alternative transcripts
  idx = matchLocationFromSpeech(transcripts)
  if idx >= 0: selectItem(field, LOCATIONS[idx])
  else: fill input with heard text, trigger autocomplete suggestions
```

### `matchLocationFromSpeech(transcripts)`

Multi-transcript fuzzy scorer against `LOCATIONS`:

| Match type | Score |
|---|---|
| Exact transcript match | 200 |
| Transcript contains location name | 150 |
| Location name contains transcript (≥4 chars) | 120 |
| Word overlap scoring | `30 + word.length*3` per match |
| Common prefix ≥ 4 chars | `prefix * 4` |

Returns index of best match if `score >= 20`, else `-1`.

---

## 12. Module: Multi-Ambulance Fleet Engine

### Fleet Registry

```js
const fleetRegistry = new Map();  // unitId → FleetUnit
let   fleetIdCounter = 1;         // AMB-01=primary; secondaries start at 02
const FLEET_COLORS = ['#3fb950', '#58a6ff', '#bc8cff', '#e3b341'];
```

### Primary Unit Lifecycle

| Event | Action |
|---|---|
| `startEmergency()` | `registerPrimaryUnit()` — creates AMB-01 entry in `fleetRegistry` |
| Each animation frame | `syncPrimaryUnit(ts)` — copies `ambProgress`, `activeRoute`, `activeSignals` into AMB-01 entry |
| `endEmergency()` / arrival | `unregisterPrimaryUnit()` — removes AMB-01, clears `_primaryHeldByConflict` |

### `confirmAddUnit()` — Deploy Secondary Unit

```
Validate origin ≠ dest, both selected
fleetIdCounter++
unitId = `AMB-${padded counter}`
color  = FLEET_COLORS[min(counter-1, 3)]
unit   = makeFleetUnit(unitId, severity, color, origin, dest)

result = fetchLoadBalancedRoute(origin, dest)
unit.route   = result.points
unit.distKm  = result.distKm
unit.signals = generateSignalsAlongRoute(result.points)
unit.etaToConflict = (distKm/5)*3600
unit.priorityScore = calcPriorityScore(unit)

fleetRegistry.set(unitId, unit)
deployFleetUnit(unit)
```

### `deployFleetUnit(unit)`

```
prefixUnitSignals(unit)           // prefix sig IDs with unitId
unit.routeLine  = L.polyline(unit.route, {color, dashed})
unit.clearedLine = L.polyline([])
deployUnitSignalMarkers(unit)     // add Leaflet markers for each signal
createUnitMarker(unit)            // add ambulance icon marker
unit.state = 'active'
unit.animFrame = requestAnimationFrame(ts => animateUnit(unit.id, ts))
```

### `animateUnit(unitId, ts)` — Per-Unit Loop

Exact mirror of `animateAmbulance` but scoped to one fleet unit:

```
guard: unit.state !== 'active' → return
guard: ts < unit.waitUntil OR unit.heldByConflict → re-queue, return

speedMult from per-unit signal proximity (same ±0.35/0.10 logic)
speedKmh = (5 + sin(unit.progress*8)*0.5) * speedMult
unit.progress += (speedKmh/3600 * dt) / unit.distKm

if progress >= 1: unit.state='arrived'; fade routeLine; return

pos = getFleetPos(unit)
unit.marker.setLatLng(pos)
rebuild unit.clearedLine

triggerUnitSignals(unit, ts)
evaluateUnitGeofences(unit, pos[0], pos[1])

unit.etaToConflict = Infinity    // reset; arbitration recalculates per second
unit.animFrame = rAF(animateUnit)
```

### Signal System (Per-Unit)

Mirrors primary signal system but namespaced:

| Primary Function | Per-Unit Equivalent |
|---|---|
| `updateSignalMarker(sigId, state)` | `updateUnitSignalMarker(unit, sigId, state)` |
| `drawGeoRings(sig)` | `drawUnitGeoRings(unit, sig)` — outer ring uses `unit.color` |
| `removeGeoRings(sigId)` | `removeUnitGeoRings(unit, sigId)` |
| `startCrossTimer(sigId, s)` | `startUnitCrossTimer(unit, sigId, s)` |
| `evaluateGeofences(lat, lng)` | `evaluateUnitGeofences(unit, lat, lng)` |
| `triggerSignalsByProgress()` | `triggerUnitSignals(unit, ts)` |
| `clearAllGeofences()` | `clearUnitGeofences(unit)` |

---

## 13. Module: Conflict Arbitration (`detectAndResolveConflicts`)

**Trigger:** `setInterval(detectAndResolveConflicts, 1000)` — runs every second.

### Algorithm

```
active = [AMB-01 if isRunning, ...all fleet units with state='active']
if active.length < 2:
  clear all heldByConflict flags
  signalGranted.clear()
  return

conflicts = []

for each pair (uA, uB) in active:
  posA = getFleetPos(uA)
  posB = getFleetPos(uB)

  for each (sA in uA.signals, sB in uB.signals) not yet passed:
    if haversineKm(sA, sB) > 0.08: continue    // not the same signal
    distA = haversineKm(posA, sA)
    distB = haversineKm(posB, sB)
    if distA > 0.6 AND distB > 0.6: continue   // neither close enough

    etaA = (distA / 5) * 3600   // seconds at 5 km/h
    etaB = (distB / 5) * 3600
    uA.etaToConflict = min(uA.etaToConflict, etaA)
    uB.etaToConflict = min(uB.etaToConflict, etaB)

    scoreA = uA.severity*1000 - etaA
    scoreB = uB.severity*1000 - etaB
    winner = scoreA >= scoreB ? uA : uB
    loser  = other unit

    key = `${sA.lat.toFixed(4)},${sA.lng.toFixed(4)}`
    signalGranted.set(key, winner.id)
    conflicts.push({ winner, loser, sigName: sA.name })

loserIds = new Set(conflicts.map(c => c.loser.id))
fleetRegistry.forEach(u => u.heldByConflict = loserIds.has(u.id))
window._primaryHeldByConflict = loserIds.has('AMB-01')

if conflicts:
  updateConflictBanner(formatted message)
  dbLogConflict for each new unique conflict pair
else:
  _notifiedConflicts.clear()
  hide conflictBanner

renderFleetPanel()
```

### Priority Score Formula

```
priorityScore = (severity × 1000) − etaToConflictPoint_seconds
```

Higher score → gets the green signal. Lower score unit: `heldByConflict = true` → its `animateUnit` loop skips movement until conflict resolves.

---

## 14. Module: Load Balancing (`fetchLoadBalancedRoute`)

Used when deploying a secondary fleet unit to avoid route overlap with existing corridors.

```
GET OSRM alternatives (up to 3)

occupied = all route waypoints from every active unit in fleetRegistry + primary

for each candidate route:
  sampled = sampleRoute(raw, 24)
  for each sampled point:
    minD = min distance to any occupied corridor point (capped at 0.5 km)
  loadScore = mean(minD) over all sampled points
  // Higher loadScore = less overlap = preferred

valid = candidates where distKm <= minDist * 1.45
best  = max(loadScore) among valid
return sampleRoute(best.raw, 120)
```

---

## 15. Module: Live DB Simulation & Firebase Sync

### Write Operations

| Function | DB Operation | Firebase | Throttle |
|---|---|---|---|
| `dbUpsert(table, key, row)` | `Map.set(key, row)` | `firebaseDB.ref(path).set(...)` | 5 000 ms per key |
| `dbAppend(table, row, cap)` | `Array.unshift(row)`, trim to cap | `firebaseDB.ref(path).push(...)` | None (immediate) |

### Throttle Logic

```js
throttleKey = `${table}/${key}`
now = Date.now()
if !_fbLastWrite[throttleKey] OR now - _fbLastWrite[throttleKey] >= 5000:
  _fbLastWrite[throttleKey] = now
  fbSet(path, row)
```

### Event Logging Helpers

| Helper | Table | Trigger |
|---|---|---|
| `dbLogSignalEvent(unitId, sigId, name, event, lat, lng)` | `SIGNAL_EVENTS` | `updateSignalMarker` / `updateUnitSignalMarker` |
| `dbLogGeofenceEvent(unitId, sigName, zone, distKm)` | `GEOFENCE_EVENTS` | Each geofence ring entry |
| `dbLogConflict(winner, loser, sigName)` | `CONFLICT_LOG` | New conflict pair detected |

### DB Panel Render

- Refreshes every 500 ms via `setInterval(renderDBPanel, 500)`.
- Newly inserted rows get `.db-row-flash` CSS class (green flash 850 ms).
- Query log shows last 20 SQL-like statements (`INSERT INTO` / `UPDATE`).
- 5 tabs: `MISSIONS`, `FLEET_REGISTRY`, `SIGNAL_EVENTS`, `CONFLICT_LOG`, `GEOFENCE_EVENTS`.

### Firebase Configuration

```js
{
  apiKey:            "AIzaSyAODSwadIPqPqoZn4C_yb6Xo_fByIhAkv8",
  authDomain:        "greencorridor-bf773.firebaseapp.com",
  databaseURL:       "https://greencorridor-bf773-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId:         "greencorridor-bf773",
  storageBucket:     "greencorridor-bf773.firebasestorage.app",
  messagingSenderId: "129832118137",
  appId:             "1:129832118137:web:b6a1ee7888d420a3bfb1a8",
}
```

SDK: Firebase JS v10.7.1 (compat mode). Operations: `.set()` for live state, `.push()` for events.

---

## 16. Module: Splash Screen & Route Picker

### Splash Screen Flow

```
User types origin/dest
  → onSplashInput(field)
      → instant LOCATIONS filter → showSplashSug()
      → debounced 280ms Nominatim → merge → showSplashSug()

User clicks START CORRIDOR → startFromSplash()
  → validate origin ≠ dest, both selected
  → fetchOSRMRoute(origin, dest)
      success → showRoutePicker(allCandidates, origin, dest)
      failure → offline fallback: straight-line → _launchMap()
```

### Route Picker

Displayed as a card overlay on the splash screen. Shows up to 3 routes:

| Badge | Route | Label |
|---|---|---|
| `b0` (green) | candidates[0] | FASTEST |
| `b1` (blue) | candidates[1] | ALTERNATIVE 1 |
| `b2` (purple) | candidates[2] | ALTERNATIVE 2 |

Each card shows: distance (km), estimated duration (min), SELECT button.

### `selectRoute(idx)`

```
chosen = _rpCandidates[idx]
userRoute          = sampleRoute(chosen.raw, 120)
userRouteTotalDist = max(chosen.distKm, haversineKm(origin, dest))
userSignals        = generateSignalsAlongRoute(userRoute)
selectedOrigin = origin; selectedDest = dest
_launchMap(origin, dest)
```

### `_launchMap(origin, dest)`

```
splashScreen.classList.add('splash-exit')   // fade + scale out (450ms)
setTimeout(420ms):
  splashScreen.display = 'none'
  layout.display       = 'flex'
  update sidebar route labels
  initMap()
  setTimeout(280ms):
    map.fitBounds([origin, dest], {padding:60, maxZoom:15})
    startEmergency()
```

---

## 17. Module: Simulation Controls

### `simFire(type)`

| Button | `type` | Action |
|---|---|---|
| GPS LOST | `'gps_lost'` | `speak('gps_lost', critical=true)` |
| REROUTE | `'reroute'` | `performReroute()` |
| CONFLICT | `'conflict'` | `speak('conflict')` |
| BLOCK ROAD | `'block'` | Block point 25% ahead → `recalculateAroundBlock()` |

### `toggleEmergency()`

- If not running → `showRouteModal()` (legacy in-map modal path)
- If running → `endEmergency()`

### `endEmergency()`

```
isRunning = false
cancelAnimationFrame(animFrame)
ambMarker → activeRoute[0]
all signals → 'normal'
HUD → reset (—, 0, —)
corridorState → 'STANDBY'
unregisterPrimaryUnit()
```

### `triggerArrival()`

```
isRunning = false
cancelAnimationFrame(animFrame)
VOICE_ALERTS.arrived dynamically updated with selectedDest.name
speak('arrived')
unregisterPrimaryUnit()
corridorState → 'COMPLETE' (blue)
3s delay → all signals reset to 'normal'
```

---

## 18. UI Component Inventory

### Splash Screen (`#splashScreen`)

| Element | Purpose |
|---|---|
| `.splash-orb-1/2/3` | Animated blurred gradient orbs (atmosphere) |
| `.splash-card` | Main input card with neon border |
| `.splash-scanline` | Animated horizontal scan line |
| `.splash-corner-tl/tr/bl/br` | Targeting bracket corners |
| `#splashOriginInput` / `#splashDestInput` | Location text inputs |
| `#splashOriginSugs` / `#splashDestSugs` | Autocomplete dropdowns |
| `#splashMicOriginBtn` / `#splashMicDestBtn` | Microphone buttons |
| `#splashGpsBtn` | GPS current location button |
| `#splashStatus` | Status/error message line |
| `#splashBtn` | START CORRIDOR button |
| `#routePickerOverlay` | Route selection overlay |
| `#rpRouteList` | Route option cards |

### Map Area (`.map-area`)

| Element | Purpose |
|---|---|
| `#map` | Leaflet map container |
| `.map-hud` | Top-right HUD (NEXT SIGNAL, CORRIDOR state) |
| `#blockAlert` | Road block alert banner (top-left) |
| `#startPopup` | Bottom-right popup (emergency start) |
| `#signalPopup` | Bottom-left popup (signal approaching/cleared) |
| `.map-legend` | Bottom-left colour legend |
| `#dbPanel` | Right slide-in Live DB panel |
| `#fleetPanel` | Bottom fleet command center panel |

### Sidebar (`.sidebar`)

| Element | Purpose |
|---|---|
| `.sb-header` | Logo + LIVE badge |
| `.status-grid` | ETA / Speed / Signals Cleared / Dist Left cards |
| `.route-info` | Origin → destination display |
| `#signalList` | Scrollable signal list |
| `.voice-panel` | Voice text display + EQ bars + language selector |
| `.controls-panel` | START EMERGENCY + simulation buttons + fleet controls + LIVE DB button |

### Modals

| Element | Purpose |
|---|---|
| `#routeModal` | In-map route setup modal (legacy path) |
| `#addUnitModal` | Deploy new fleet unit |

---

## 19. Function Reference Table

| Function | Lines | Description |
|---|---|---|
| `initMap()` | 1021 | Initialise Leaflet map, all markers, signal icons |
| `createSignalMarkerHTML(state, id)` | 1121 | Build traffic-light SVG HTML string |
| `updateSignalMarker(sigId, state)` | 1142 | Update signal icon + sidebar + DB log |
| `renderSignalList()` | 1175 | Rebuild sidebar signal list HTML |
| `focusSignal(i)` | 1205 | Pan map to signal `i` |
| `lerpLatLng(a, b, t)` | 1209 | Linear interpolation between two lat/lng points |
| `getPositionOnRoute(t)` | 1213 | Interpolate position on active route at progress `t` |
| `updateHUD()` | 1221 | Refresh sidebar stats (ETA, speed, dist, next signal) |
| `animateAmbulance(ts)` | 1242 | Primary animation loop (rAF) |
| `triggerSignalsByProgress()` | 1303 | Visual signal state transitions per frame |
| `moveOtherVehicles()` | 1324 | Animate 8 background traffic vehicles |
| `toggleEmergency()` | 1347 | Toggle start/end emergency |
| `showRouteModal()` / `hideRouteModal()` | 1359 | In-map route modal open/close |
| `onLocationInput(field)` | 1394 | Autocomplete trigger (modal) |
| `searchNominatim(field, q, locals)` | 1417 | Async Nominatim API search |
| `scoreTextMatch(q, name)` | 1441 | Fuzzy text scoring |
| `renderSuggestions(field, items, loading)` | 1453 | Render suggestion dropdown |
| `pickSuggestion(field, idx)` | 1478 | Select suggestion from dropdown |
| `selectItem(field, name, lat, lng)` | 1488 | Confirm selection, mark input valid |
| `onInputKeydown(e, field)` | 1498 | Keyboard nav for suggestion list |
| `drawGeoRings(sig)` | 1529 | Draw 3 concentric geofence circles |
| `removeGeoRings(sigId)` | 1547 | Remove geofence circles from map |
| `startCrossTimer(sigId, s)` | 1554 | Start cross-traffic countdown |
| `stopCrossTimer(sigId)` | 1564 | Stop cross-traffic countdown |
| `clearAllGeofences()` | 1570 | Reset all geofence state |
| `evaluateGeofences(lat, lng)` | 1581 | Geofence ring evaluation per GPS ping |
| `haversineKm(a, b)` | 1664 | Great-circle distance in km |
| `routeLengthKm(pts)` | 1674 | Sum of Haversine segments |
| `aStarDeviation(pts, o, d)` | 1682 | A* perpendicular deviation score |
| `sampleRoute(raw, maxPts)` | 1703 | Downsample polyline to N points |
| `fetchOSRMRoute(o, d)` | 1714 | Fetch + rank OSRM alternatives |
| `generateSignalsAlongRoute(pts)` | 1750 | Place signals evenly along route |
| `confirmRoute()` | 1762 | Modal confirm → fetch OSRM → `startEmergency` |
| `useCurrentLocation()` | 1858 | GPS locate → Nominatim reverse geocode |
| `startMicInput(field)` | 1953 | Start Web Speech API recognition |
| `matchLocationFromSpeech(transcripts)` | 1920 | Fuzzy location match against LOCATIONS |
| `resetToOriginalSignalMarkers()` | 2018 | Rebuild signal markers from userRoute |
| `startEmergency()` | 2038 | Activate corridor, begin animation |
| `endEmergency()` | 2081 | Deactivate corridor, reset state |
| `triggerArrival()` | 2100 | Handle route completion |
| `speak(key, critical, sigTag)` | 2137 | Voice alert with deduplication |
| `setLang(lang)` | 2164 | Set `selectedLang` |
| `showBlockAlert(msg, ms)` | 2171 | Show block overlay alert |
| `placeBlockMarker(lat, lng)` | 2190 | Add 🚧 icon + red line to map |
| `fetchOSRMRouteAvoidingBlock(f,d,blocks)` | 2211 | Block-avoiding OSRM fetch |
| `applyNewRoute(pos, pts, distKm)` | 2248 | Apply rerouted path to map + state |
| `fetchOSRMRouteDifferentPath(f, d)` | 2308 | Most-divergent OSRM alternative |
| `performReroute()` | 2352 | Manual reroute (REROUTE button) |
| `recalculateAroundBlock(lat, lng, reason)` | 2383 | Block-triggered reroute |
| `scheduleAutoBlockDetection()` | 2420 | Schedule random auto block 20–35 s |
| `simFire(type)` | 2435 | Simulation button dispatcher |
| `showStartPopup(msg)` | 2454 | Show bottom-right start popup |
| `showSignalPopup(msg)` | 2463 | Show bottom-left signal popup |
| `makeFleetUnit(...)` | 2485 | Create fleet unit object |
| `calcPriorityScore(unit)` | 2506 | `severity*1000 - etaToConflict` |
| `detectAndResolveConflicts()` | 2518 | Conflict arbitration (runs every 1s) |
| `fetchLoadBalancedRoute(o, d)` | 2619 | Load-balanced OSRM route for new unit |
| `getFleetPos(unit)` | 2661 | Interpolate fleet unit position |
| `prefixUnitSignals(unit)` | 2672 | Namespace signal IDs per unit |
| `deployUnitSignalMarkers(unit)` | 2677 | Add unit signal markers to map |
| `clearUnitSignalMarkers(unit)` | 2687 | Remove unit signal markers |
| `updateUnitSignalMarker(unit, sigId, state)` | 2693 | Update unit signal icon |
| `drawUnitGeoRings(unit, sig)` | 2707 | Draw unit geofence rings |
| `evaluateUnitGeofences(unit, lat, lng)` | 2761 | Per-unit geofence evaluation |
| `triggerUnitSignals(unit, ts)` | 2815 | Per-unit signal state machine |
| `createUnitMarker(unit)` | 2843 | Create unit ambulance Leaflet marker |
| `animateUnit(unitId, ts)` | 2857 | Per-unit animation loop |
| `deployFleetUnit(unit)` | 2913 | Deploy unit on map, start loop |
| `removeFleetUnit(unitId)` | 2926 | Remove unit from map + registry |
| `toggleFleetView()` | 2942 | Fit map bounds to all active units |
| `renderFleetPanel()` | 2959 | Render command center fleet panel |
| `registerPrimaryUnit()` | 3009 | Register AMB-01 in fleet registry |
| `unregisterPrimaryUnit()` | 3021 | Remove AMB-01 from registry |
| `syncPrimaryUnit(ts)` | 3028 | Sync primary globals into registry |
| `confirmAddUnit()` | 3129 | Deploy new secondary unit |
| `onSplashInput(field)` | 3165 | Splash autocomplete trigger |
| `startFromSplash()` | 3347 | Validate splash inputs → fetch routes |
| `showRoutePicker(candidates, o, d)` | 3396 | Render route picker overlay |
| `selectRoute(idx)` | 3429 | Select route from picker, launch map |
| `_launchMap(origin, dest)` | 3456 | Transition splash → map |
| `dbUpsert(table, key, row)` | 3535 | Upsert in-memory DB + Firebase |
| `dbAppend(table, row, cap)` | 3550 | Append event to DB table + Firebase |
| `dbLogSignalEvent(...)` | 3568 | Log signal state change to DB |
| `dbLogGeofenceEvent(...)` | 3579 | Log geofence entry to DB |
| `dbLogConflict(winner, loser, sig)` | 3589 | Log conflict to DB |

---

## 20. Error Handling & Fallbacks

| Scenario | Fallback |
|---|---|
| OSRM unreachable | Straight-line route (Haversine); warning shown to user |
| Nominatim unreachable | Local LOCATIONS matches only (Stage 1 results remain) |
| `window.turf` not loaded | `evaluateGeofences` / `evaluateUnitGeofences` → early return |
| `window.speechSynthesis` absent | Voice text shown in sidebar only; EQ animation times out after 2.5 s |
| `SpeechRecognition` absent | Status: "Speech recognition not supported. Use Chrome or Edge." |
| `navigator.geolocation` absent | Status: "Geolocation not supported." |
| GPS permission denied | Error code 1 → mapped message shown |
| GPS timeout | Error code 3 → mapped message shown |
| Firebase `.set()` / `.push()` fails | `console.warn` only; in-memory DB continues working |
| Road block avoidance — no clear route | Best available route used (may still pass block) |
| Fleet load-balancing — OSRM fails | `setUnitStatus('Route fetch failed.', 'err')` shown in modal |
| `ambProgress >= 1` | `triggerArrival()` called; animation stops cleanly |

---

## 21. Timing & Interval Summary

| Timer | Interval | Purpose |
|---|---|---|
| `requestAnimationFrame` (primary) | ~16 ms (60 fps) | `animateAmbulance` — position + geofence + HUD |
| `requestAnimationFrame` (per unit) | ~16 ms | `animateUnit` — per secondary ambulance |
| `setInterval(detectAndResolveConflicts, 1000)` | 1 000 ms | Conflict arbitration loop |
| `setInterval(renderDBPanel, 500)` | 500 ms | Live DB panel re-render |
| `setInterval(renderFleetPanel, 2000)` | 2 000 ms | Fleet panel re-render |
| `setInterval` (cross-traffic timer) | 1 000 ms | Per-signal cross-traffic countdown |
| `setTimeout` (signal recovery) | 5 000 ms | `recovery → normal` after ambulance passes |
| `setTimeout` (start popup dismiss) | 3 500 ms | Auto-hide start popup |
| `setTimeout` (signal popup dismiss) | 3 500 ms | Auto-hide signal popup |
| `setTimeout` (row flash) | 900 ms | DB row green flash animation |
| `setTimeout` (splash exit) | 420 ms | Splash fade completes before map shows |
| `setTimeout` (map fit after launch) | 280 ms | Allow map to initialise before fitBounds |
| `setTimeout` (speech cancel gap) | 60 ms | Allow `speechSynthesis.cancel()` to flush |
| `setTimeout` (voice after start) | 2 000 ms | `speak('corridor')` 2 s after start announcement |
| `setTimeout` (signal reset after arrival) | 3 000 ms | All signals → normal 3 s after arrival |
| `scheduleAutoBlockDetection` | 20 000–35 000 ms | One-shot auto block event per emergency |
| Nominatim debounce | 350 ms | Route modal search debounce |
| Splash/Unit modal debounce | 280 ms | Splash and unit modal search debounce |
| Firebase throttle | 5 000 ms | Live-state table write throttle per key |

---

*LLD authored for GreenCorridor v1.0.0 — single-file SPA (`code.html`)*
