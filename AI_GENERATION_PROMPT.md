# Rail Travel — Complete Project Specification

> This document is a self-contained spec. Any developer or AI assistant can build the entire project from scratch using only this file.

## 1. Product Overview

A weekend-trip planning tool for Chinese high-speed rail travelers. Users pick a departure city, set travel dates/time windows, then "radar scan" dozens of nearby cities simultaneously to see real-time ticket availability on an interactive map. Clicking a city shows full train lists with seat counts, weather, and direct 12306 booking links.

### Core User Flow

```
Open app → Auto-detect origin city (geolocation)
         → Set go/return dates & departure time windows
         → Tap "Scan" → Map fills with color-coded availability markers
         → Tap a city marker → Side panel shows train list + weather + city info
         → Tap "Buy" → Opens 12306 with route pre-filled (copies train code on mobile)
```

## 2. Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend framework | React | 19 |
| Language | TypeScript | 6 |
| Build tool | Vite | 8 |
| CSS | Tailwind CSS (Vite plugin) | 4 |
| Map | AMap (Gaode) JS API via `@amap/amap-jsapi-loader` | 2.0 |
| Date util | dayjs | 1.x |
| Backend runtime | Node.js + tsx | — |
| Backend framework | Express | 5 |
| Deployment | Fly.io (Docker) | — |
| Weather API | QWeather (devapi.qweather.com) | v7 |
| IP geolocation | ip-api.com (free, HTTP) | — |
| Train data source | 12306.cn (Chinese Railway) | — |

### Environment Variables

```
VITE_AMAP_KEY=<AMap JS API key>
VITE_API_BASE=<empty for same-origin, or http://localhost:3002 in dev>
QWEATHER_KEY=<QWeather API key>
QWEATHER_HOST=https://devapi.qweather.com   # optional override
PORT=3001                                     # backend port
```

## 3. Project Structure

```
rail-travel/
├── package.json              # Frontend: React 19 + Vite 8 + Tailwind 4
├── vite.config.ts            # Proxy /api → localhost:3002
├── src/
│   ├── index.css             # CSS variables (design tokens) + animations
│   ├── App.tsx               # Root component, all app state, scan logic
│   ├── types/index.ts        # Shared TypeScript interfaces
│   ├── config/constants.ts   # Default origin, travel plan defaults
│   ├── hooks/useAMap.ts      # AMap loader hook
│   ├── services/
│   │   ├── api.ts            # Frontend API client (trains, stations, batch, locate)
│   │   └── weather.ts        # Frontend weather client with 30min cache
│   └── components/
│       ├── ControlBar.tsx    # Top floating bar: origin, search, dates, filters, scan button
│       ├── MapView.tsx       # Full-screen AMap with dynamic markers
│       ├── TrainListPanel.tsx # Right slide-in panel: train list with tabs
│       ├── TrainCard.tsx     # Single train row: times, duration bar, seat dots, buy link
│       ├── CityInfoCard.tsx  # Collapsible city intro + weather + clothing advice
│       └── WaterBottle.tsx   # SVG "water bottle" seat availability visualization
├── public/
│   └── data/
│       ├── stations.json       # ~2000 stations: {name, city, code, lat, lng, province, level, pinyin}
│       ├── search-stations.json # Lightweight: {n, c, p, cn} for search index
│       └── city-profiles.json   # {[city]: {intro, food[], tags[]}}
└── api/
    ├── package.json          # Backend: Express 5 + tsx + dotenv
    ├── Dockerfile
    ├── fly.toml
    ├── data/
    │   ├── stations.json     # Same station data (backend copy)
    │   └── destinations.json # {cities: string[]} — curated destination list
    └── src/
        ├── server.ts         # Express app, 7 routes, SPA fallback
        ├── types.ts          # Backend type definitions
        ├── routes/
        │   ├── health.ts     # GET /api/health
        │   ├── cities.ts     # GET /api/cities?from=&date= (batch city scan)
        │   ├── trains.ts     # GET /api/trains?from=&to=&date=&nocache=
        │   ├── trainsBatch.ts # POST /api/trains/batch (NDJSON streaming)
        │   ├── trainStops.ts # GET /api/train-stops?trainNo=&from=&to=&date=
        │   ├── weather.ts    # GET /api/weather?lat=&lng=
        │   └── locate.ts     # GET /api/locate (IP geolocation → nearest station)
        ├── services/
        │   ├── railway.ts    # 12306 API client: tickets, prices, stops, station map
        │   └── weather.ts    # QWeather client
        └── utils/
            ├── cache.ts      # In-memory Map cache with TTL
            └── parser.ts     # 12306 pipe-separated response parser

```

## 4. Design System (CSS Variables)

All styling uses CSS custom properties. No hardcoded colors in components.

```css
:root {
  /* Primary palette */
  --c-primary: #0284C7;         /* Sky blue — main actions, scan button */
  --c-primary-light: #E0F2FE;   /* Hover/active backgrounds */
  --c-primary-hover: #0369A1;
  --c-accent: #059669;          /* Green — return trips */
  --c-accent-light: #D1FAE5;
  --c-warning: #F59E0B;         /* Amber — limited tickets */
  --c-warning-light: #FEF3C7;
  --c-danger: #DC2626;          /* Red — very few tickets */
  --c-danger-light: #FEE2E2;

  /* Surfaces */
  --c-bg: #F8FAFC;
  --c-card: #FFFFFF;
  --c-border: #E2E8F0;
  --c-border-light: #F1F5F9;

  /* Text */
  --c-text: #0F172A;
  --c-text-secondary: #64748B;
  --c-text-muted: #94A3B8;

  /* Semantic */
  --c-origin: #E85D4A;          /* Origin city marker */
  --c-g-train: #3B82F6;         /* G-series (high-speed) */
  --c-d-train: #06B6D4;         /* D-series (express) */
  --c-c-train: #8B5CF6;         /* C-series (intercity) */
  --c-k-train: #94A3B8;         /* K/T/Z (conventional) */

  /* Elevation & shape */
  --shadow-sm: 0 1px 3px rgba(0,0,0,0.08);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.1);
  --shadow-lg: 0 8px 24px rgba(0,0,0,0.12);
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 16px;
  --radius-full: 9999px;
  --font-family: -apple-system, "PingFang SC", "Helvetica Neue", "Microsoft YaHei", sans-serif;
}
```

### Animations

| Name | Purpose | Duration |
|------|---------|----------|
| `slide-in-right` | TrainListPanel entrance | 0.28s cubic-bezier |
| `pulse-ring` | Scanning marker ripple | 1.5s infinite |
| `spin` | Loading spinners | 0.8s linear infinite |
| `scan-bar` | ControlBar progress shimmer | 1.5s infinite |

## 5. Data Models

### Frontend Types (`src/types/index.ts`)

```typescript
interface Station {
  name: string;           // "杭州东"
  city: string;           // "杭州"
  code: string;           // "HGH"
  lat: number;
  lng: number;
  province?: string;      // "浙江省"
  level?: 'city' | 'county' | 'station';
  pinyin?: string;
}

interface City {
  name: string;   code: string;   lat: number;   lng: number;
  province?: string;   level?: 'city' | 'county' | 'station';   pinyin?: string;
}

interface Train {
  trainNo: string;        // Internal 12306 ID (e.g., "5l000G130501")
  trainCode: string;      // Display code (e.g., "G1305")
  fromStation: string;    toStation: string;
  departTime: string;     // "08:30"
  arriveTime: string;     // "11:45"
  duration: string;       // "03:15"
  secondClassSeats: string;  // "有" | "15" | "--" | "无"
  firstClassSeats: string;
  businessSeats: string;
  noSeatTickets: string;
}

interface TravelPlan {
  goDate: string;                      // "2026-05-02"
  returnDate: string;
  goDepartRange: [string, string];     // ["07:00", "10:00"]
  returnDepartRange: [string, string]; // ["17:00", "21:00"]
  trainTypes: 'all' | 'G/D';
}

interface CityTicketSummary {
  goPercent: number;       // 0-100, best seat availability across all go trains
  goColor: string;         // Hex color from seatLevel()
  goLabel: string;
  returnPercent: number;
  returnColor: string;
  returnLabel: string;
  distanceKm?: number;     // Haversine from origin
  minDurationGo?: string;  // "2h15m"
  weatherText?: string;    // "18~25° Sunny"
}

interface OriginInfo {
  city: string;   code: string;   lat: number;   lng: number;   province?: string;
}

interface SearchStation {
  n: string;   // Station name
  c: string;   // Station code
  p: string;   // Pinyin
  cn: string;  // Parent city name
}
```

### Backend Types (`api/src/types.ts`)

Backend `Train` extends frontend's with: `fromStationCode`, `toStationCode`, `fromStationNo`, `toStationNo`, `seatTypeCodes`.

Additional backend types: `TrainWithPrice`, `TicketPrice` (`{ business?, firstClass?, secondClass?, noSeat? }`), `TrainStop`, `CityResult`.

## 6. Backend API Specification

### `GET /api/trains?from=HGH&to=BJP&date=2026-05-02&nocache=1`

Query 12306 for trains between two stations on a date. Returns `{ from, to, date, trainCount, trains: Train[] }`.

- `nocache=1`: bypass server cache
- Cache TTL: 5 minutes (in-memory Map)

### `POST /api/trains/batch`

Body: `{ queries: [{ from, to, date, key }] }` (max 40 queries)

Response: **NDJSON stream** (newline-delimited JSON). Each line: `{ key, trainCount, trains }`. Server processes sequentially (due to 12306 throttle) and flushes each result immediately so the frontend can update progressively.

### `GET /api/cities?from=HGH&date=2026-05-02`

Scan all destination cities from `destinations.json`. Returns cities with train availability. Slower (sequential due to throttle).

### `GET /api/train-stops?trainNo=&from=&to=&date=`

Returns intermediate stops for a train: `{ trainNo, stops: TrainStop[] }`.

### `GET /api/weather?lat=30.27&lng=120.15`

Proxies QWeather 3-day forecast. Returns `{ days: WeatherDay[] }`. Cache: 2 hours.

### `GET /api/locate`

IP-based geolocation → nearest railway station city. Reads `fly-client-ip`, `cf-connecting-ip`, or `x-forwarded-for` headers. Calls ip-api.com. Returns `{ city, code, lat, lng, source: 'ip'|'fallback' }`. Falls back to Hangzhou.

### `GET /api/health`

Returns 200 OK.

## 7. 12306 Integration Details

### Rate Limiting

**1.5 seconds minimum** between any two requests to `kyfw.12306.cn`. Enforced via `throttledFetch()` with timestamp tracking. Going below ~1s triggers CAPTCHAs or IP bans.

### Session Cookie

12306 requires a session cookie. Obtained by visiting `/otn/leftTicket/init`. Cached for 30 minutes.

### Ticket Query

Three fallback endpoints tried in order: `queryZ`, `query`, `queryA`. URL pattern:
```
https://kyfw.12306.cn/otn/leftTicket/{endpoint}?
  leftTicketDTO.train_date=2026-05-02&
  leftTicketDTO.from_station=HGH&
  leftTicketDTO.to_station=BJP&
  purpose_codes=ADULT
```

Response: `{ data: { result: string[] } }` where each result is a pipe-separated string with 36+ fields.

### Parser Field Index

| Index | Field |
|-------|-------|
| 2 | trainNo |
| 3 | trainCode |
| 6 | fromStationCode |
| 7 | toStationCode |
| 8 | departTime |
| 9 | arriveTime |
| 10 | duration |
| 16 | fromStationNo |
| 17 | toStationNo |
| 26 | noSeatTickets |
| 30 | secondClassSeats |
| 31 | firstClassSeats |
| 32 | businessSeats |
| 35 | seatTypeCodes |

### Seat Availability Values

- `"有"` = abundant (100+ seats)
- `"15"` = exact count
- `"--"` or `""` or `"无"` = sold out
- `"*"` = not applicable

### Station Map

Fetched from `https://kyfw.12306.cn/otn/resources/js/framework/station_name.js`. Format: `@abbreviation|name|code|pinyin|initial|index`. Parsed into `Map<code, name>`. Cache: 24 hours.

### Price Query (implemented but not used in MVP)

```
/otn/leftTicket/queryTicketPrice?train_no=&from_station_no=&to_station_no=&seat_types=&train_date=
```
Returns `{ data: { A9: "¥1200", M: "¥650", O: "¥350", WZ: "¥350" } }`. Price codes: A9=business, M=first, O=second, WZ=no-seat.

## 8. Frontend Architecture

### State Management (App.tsx)

All state lives in `App.tsx` via `useState`. No external state library.

| State | Type | Purpose |
|-------|------|---------|
| `origin` | `OriginInfo` | Current departure city (auto-detected or user-selected) |
| `selectedCity` | `City \| null` | City whose train panel is open |
| `goTrains` / `returnTrains` | `Train[]` | Trains for selected city |
| `loading` | `boolean` | Train query in progress |
| `queriedCities` | `Map<string, CityTicketSummary>` | All scanned cities with availability summary |
| `travelPlan` | `TravelPlan` | Dates, time ranges, train type filter |
| `scanning` | `boolean` | Radar scan in progress |
| `scanProgress` | `string` | "R2 5/12" format |
| `scanningCity` | `string \| null` | Currently querying city (shows pulse animation) |
| `scanRing` | `number` | Current scan ring index (0-3) |
| `showOnlyAvailable` | `boolean` | Hide no-ticket markers |
| `allCities` | `City[]` | Full city list from stations.json |
| `flyToCity` | `City \| null` | Triggers map fly-to animation |
| `searchStations` | `SearchStation[]` | Search index |
| `mapZoom` | `number` | Current map zoom level |

### Radar Scan Algorithm

1. Load all stations, deduplicate by city
2. Filter to `level === 'city'` only
3. Calculate haversine distance from origin to each city
4. Determine max scan radius from current zoom: `zoom ≥ 10 → 200km`, `≥ 8 → 400km`, `≥ 6 → 700km`, else `1200km`
5. Sort cities into concentric rings: `[0-200km, 200-400km, 400-700km, 700-1200km]`
6. Iterate ring-by-ring, city-by-city (closest first)
7. For each city: `Promise.all([fetchTrains(go), fetchTrains(return)])` → compute `CityTicketSummary` → update `queriedCities` Map → marker updates reactively
8. Can be aborted mid-scan via `scanAbortRef`
9. Skip already-queried cities

### Water Bottle Visualization

SVG "bottle" that fills with color based on seat availability:

| Seats | Percent | Color | Label |
|-------|---------|-------|-------|
| "有" (abundant) | 100% | `#22c55e` green | "充足" |
| ≥ 50 | 90% | `#22c55e` green | count |
| 20-49 | 65% | `#3b82f6` blue | count |
| 5-19 | 35% | `#f59e0b` amber | count |
| 1-4 | 15% | `#ef4444` red | count |
| 0 / "--" / "无" | 0% | `#d1d5db` gray | "无票" |

`aggregateStatus()` scans all trains' second/first/business seats and returns the best level found.

### Map Marker Types

1. **Origin marker**: Red pill with pin icon, city name. Non-clickable.
2. **Default marker**: Blue pill. Provincial capitals are darker/larger. Varies by `level` (city > county > station). Hover scales up.
3. **Scanning marker**: Blue pill with spinning border + pulse ripple animation.
4. **Queried marker**: White card with city name + two water bottles (go/return) + distance/duration info + weather. Glow color reflects best availability.

### Zoom-Based Visibility

| Level | Zoom < 5 | 5-6 | 7-8 | ≥ 9 |
|-------|----------|-----|-----|-----|
| city (capital) | hidden | visible | visible | visible |
| city (other) | hidden | hidden | visible | visible |
| county | hidden | hidden | hidden | visible |
| station | hidden | hidden | hidden | visible |

Queried cities override: always visible once scanned (county at zoom ≥ 5, station at zoom ≥ 7).

### Search

ControlBar has two search dropdowns:
- **Origin search**: switch departure city. Searches `searchStations` by name or pinyin.
- **Destination search**: find & fly to a city. Triggers `handleSearchSelect` → `setFlyToCity` + `handleCityClick`.

Both show max 8 results, display station name + province, click to select.

## 9. Component Specifications

### ControlBar
- Position: absolute, top center, z-50, frosted glass (`rgba(255,255,255,0.92)` + `backdrop-filter: blur(16px)`)
- Max width: 520px. Collapsible.
- Row 1 (always visible): Origin button (with dropdown) + search bar + collapse toggle
- Row 2: Go date + departure time range (PillInput components)
- Row 3: Return date + departure time range
- Row 4: Filter chips (高铁 toggle, 有票 toggle, 周末 shortcut) + Scan/Stop button
- Scan button: shows animated progress bar + "R2 5/12" format when scanning

### MapView
- Full-screen AMap container. Style: `amap://styles/normal`.
- Creates markers for all cities on mount. Updates marker content reactively when `queriedCities` changes.
- Renders dashed circle overlay during scan showing current ring radius.
- `flyToCity` triggers `map.setZoomAndCenter(8, coords, false, 500)`.
- Reports zoom changes to App via `onZoomChange`.
- Map click → find nearest city within 50km → open its panel.

### TrainListPanel
- Right-side slide-in panel, 400px wide, full height minus margins.
- Header: origin ↔ destination, dates, train counts.
- CityInfoCard: weather + city intro + food + tags (collapsible).
- Filter chips: All / G/D / Available.
- Tabs: Go / Return (show plan-filtered count / total count).
- Scrollable train list.

### TrainCard
- Train code (colored by type) + duration + buy button
- Timeline: depart time — duration bar — arrive time
- Seat dots: 4 columns (second/first/business/no-seat), each with WaterBottle + label
- Buy button: links to 12306. On mobile: copies train code to clipboard + opens 12306 app via universal link (iOS) or intent scheme (Android).

### CityInfoCard
- Loads city profile + weather in parallel on mount.
- Compact line: intro text + temperature + clothing advice.
- Expandable: food list + tag chips.
- Clothing advice: avg temp → text mapping (30°+ → "短袖短裤", 5-10° → "厚外套").

## 10. Caching Strategy

| Layer | Scope | TTL | Storage |
|-------|-------|-----|---------|
| Backend ticket cache | per route+date | 5 min | In-memory Map |
| Backend weather cache | per lat,lng | 2 hours | In-memory Map |
| Backend station map | global | 24 hours | In-memory Map |
| Frontend train cache | per city name | Session (ref) | `trainCacheRef` |
| Frontend weather cache | per lat,lng | 30 min | Module-level Map |
| Frontend city profiles | global | Session | Module-level variable |
| Frontend search stations | global | Session | Module-level variable |

Changing origin or travel plan clears frontend caches (`trainCacheRef.current.clear()`).


## 11. Key Algorithms

### Haversine Distance

```typescript
function haversineKm(lat1, lng1, lat2, lng2): number {
  const R = 6371;
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLng = ((lng2 - lng1) * Math.PI) / 180;
  const a = sin(dLat/2)**2 + cos(lat1*π/180) * cos(lat2*π/180) * sin(dLng/2)**2;
  return R * 2 * atan2(sqrt(a), sqrt(1-a));
}
```

Used in: origin detection, scan ring assignment, city distance display, map click nearest-city lookup (within 50km).

### Duration Parsing & Display

`"03:15"` → `195` minutes → `"3h15m"`. Used for sorting, min-duration display, timeline bar width (`min(100, max(15, durMin/360*100))%`).

### Train Filtering Pipeline

```
Raw trains from API
  → filterByTimeRange(trains, [start, end])  // depart time window
  → filterByTrainType(trains, 'G/D' | 'all')  // G/D prefix filter
  → filterTrains(trains, 'available')  // optional: has any seat > 0
```

## 12. Mobile Considerations

- ControlBar: `width: calc(100vw - 24px)`, max 520px
- TrainListPanel: 400px fixed (overlaps map on mobile — acceptable for MVP)
- Buy button: copies train code + opens 12306 app
  - iOS: `https://www.12306.cn/index/` (universal link triggers app)
  - Android: `intent://www.12306.cn/#Intent;scheme=https;package=com.MobileTicket;...;end`
- Touch: all interactive elements have adequate tap targets

## 13. Known Limitations & Future Enhancements

- **12306 throttle**: 1.5s/request → 30 cities ≈ 90s scan time. Could test 1s with exponential backoff.
- **No persistent storage**: all cache is in-memory, resets on restart.
- **Price data**: price query endpoint exists but disabled (too slow with throttle). Seat counts from ticket query are sufficient.
- **TrainListPanel**: fixed 400px width, not responsive on narrow mobile.
- **Station data**: static JSON files, not auto-updated from 12306.
