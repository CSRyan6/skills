---
name: travel-map-planner
description: Use when the user provides a rough China road-trip itinerary, travel days, origin/destination stops, self-driving route outline, or asks to turn route duration into a daily schedule.
---

# Travel Map Planner

## Overview

Use this skill to convert a rough China self-driving itinerary into an interactive route map preview with real AMap driving paths, distance/duration per leg, per-day schedule with departure/arrival times, altitude labels, an elevation‑colored route mode (~500 m sampling), a weather‑colored route mode (~40 km sampling) with day‑focused rain‑intensity blue gradient, and per‑leg weather displayed in the panel. This is a workflow skill: rely on `amap-jsapi-skill` for AMap JSAPI details instead of copying its API docs.

## When To Use

Use when the user provides or asks for:
- A rough route with days, origin, destination, and intermediate stops.
- A self-driving itinerary that needs map visualization.
- A daily schedule based on real AMap driving duration.
- Departure/arrival time planning from route durations.
- A map preview with elevation‑colored or weather‑colored route.
- Weather conditions shown on the route and in the trip panel.

Do not use for:
- Generic ECharts schematic route maps.
- International routes outside AMap coverage.
- Final production deployment security design; use `amap-jsapi-skill` security guidance for that.

## Required Inputs

Extract from context first; ask only for missing critical fields:
- Ordered stops or daily route legs.
- Travel mode. Default: self-driving.
- Pure-play/rest days and their base location.
- Hard constraints: sunrise/sunset, event time, latest arrival, earliest departure, elderly/children, no night driving.
- Desired output: map preview, Markdown itinerary, or both.
- **Altitudes** for each destination (for elevation fallback interpolation; ask or estimate from general knowledge).
- **Weather data**: preferably weather.com.cn 7‑day forecast URLs per city. Each city needs a weather.com.cn code (e.g. `101271804`). Fetch the 7‑day page (`/weather/{code}.shtml`) and extract day‑by‑day strings (condition + temp range). Also note: a few cities use a 5‑digit suffix format like `10127190204A` (四姑娘山).

## Route Parsing

Normalize user text into this shape:

```json
{
  "days": [
    { "day": 1, "from": "出发地", "to": "目的地", "waypoints": ["途经点A"], "note": "可选备注" },
    { "day": 2, "stay": "驻留地", "note": "纯玩" }
  ]
}
```

Rules:
- A line like `A-B` becomes one route leg.
- A line like `A-经B-C`, `A-B-C`, or user text saying "D3 途经 B" becomes one route leg with `waypoints: ["B"]`; preserve waypoint order.
- A line like `纯玩` becomes a stay day at the previous arrival location.
- Preserve user-provided distance, duration, and notes as original estimates.
- Store intermediate driving stops on that day in `waypoints`, not as separate travel days unless the user explicitly wants to overnight there.
- Generate route legs dynamically from parsed input; never hardcode example destinations into workflow logic.

## Generated HTML Structure

The generated single‑file HTML must embed all data and logic inline. It has three independent display modes toggled by buttons:

| Mode | Button | What it shows |
|------|--------|---------------|
| **day** (default) | — | AMap driving polylines + markers + altitude labels |
| **elevation** | 🌄 | Route segments colored by elevation (~500 m samples) + legend |
| **weather** | 🌤️ | Route segments colored by daytime weather (~40 km samples) + legend |

The generated file must embed per‑leg driving logic, elevation sampling, weather sampling, and a side panel all in one `<script>` block.

## AMap Preview Workflow

When a map preview is needed:
- Read `amap-jsapi-skill` and relevant references such as routing, geocoder, marker, security, and the Weather service reference (services-other.md).
- Before generating website code, follow `amap-jsapi-skill` telemetry requirements.
- Use a reusable temporary directory under the workspace, normally `tmp-demo`; overwrite old generated files.
- Do not create a new demo directory per attempt unless the user explicitly asks for separate versions.
- Use an available local port at runtime. Do not require a fixed port.
- Inject `AMAP_JSAPI_KEY` and `AMAP_SECURITY_JSCODE` from environment variables; never hardcode secrets in generated files.
- Stop the related local server and remove the temporary directory when the user says the preview is no longer needed.

### Generated preview should:

1. Initialize AMap JSAPI v2.0 with `AMapLoader.load()` and set `AMap.getConfig().appname = 'amap-jsapi-skill';` before creating `new AMap.Map()`.
2. Use `AMap.Driving` for self‑driving route legs. Each leg must be planned independently with its own 35 s timeout. Serialise requests at 350 ms QPS delay to stay under the AMap free‑tier QPS cap.
3. On success, extract the actual driving path from `steps[].path` and draw with `AMap.Polyline`. On failure (or 35 s timeout), draw a dashed gray straight‑line fallback.
4. Display each leg's planning status, AMap distance, AMap duration, and any failed leg in the side panel.
5. Show waypoints distinctly from overnight markers. Label overnight stops with altitude (e.g. `▲3200 m`).
6. Show pure‑play days in the side panel but do not route‑plan them.
7. Prefer user‑provided precise place names and coordinates. Use explicit approximate center coordinates for ambiguous stops and label them as approximate.
8. After all legs are planned, build altitude labels (`▲{alt} m`) below each destination marker. These persist in all three display modes.
9. After all legs are planned, build the elevation route data and the weather route data in the background (they are computed once and cached).

### Per‑leg data structure each leg must carry:

```javascript
{
  day: 1, date: '5/30', from: '重庆', to: '四姑娘山',
  start: [lng, lat], end: [lng, lat],
  waypoints: [{ name: '理塘', coord: [lng, lat] }],
  status: 'pending' | 'ok' | 'fail' | 'stay',
  distM: null | number, sec: null | number,
  path: null | [[lng, lat], ...],
  failMsg: null | string
}
```

Stay days use `{ day: 7, date: '6/5', stay: '香格里拉镇', note: '游玩', coord: [lng, lat] }`.

## Three Display Mode Architecture

The generated HTML must implement three non‑overlapping display modes controlled by a `displayMode` variable (`'day' | 'elevation' | 'weather'`). Each mode manages its own set of `AMap.Polyline` objects and shows/hides them when toggled. Altitude labels and panel are always visible.

```javascript
var displayMode = 'day'; // default

function switchDisplayMode(mode) {
  displayMode = mode;
  // hide all mode-specific polylines
  // show the selected mode's polylines
  // show/hide legends
}
```

Toggle buttons in the panel:
- 🌄 "海拔高度" toggles `day ↔ elevation`.
- 🌤️ "天气预报" toggles `day ↔ weather`.

## Elevation Mode

### Data collection

After all legs are planned, collect the full combined path (concatenate all leg paths, deduplicating connecting points). Compute cumulative distances. Sample at **~500 m (0.5 km)** intervals using `samplePathIndices(cumDist, intervalKm)`.

### Primary source: Open‑Elevation API

`https://api.open-elevation.com/api/v1/lookup?locations={lat},{lng}|{lat},{lng}|...`

- Batch: **1500 points per request** (the API accepts up to ~2000).
- Dispatch batches in parallel (total samples = route length / 0.5 km).
- Validate response `results.length === batch.length`.
- If any batch fails (fetch error, wrong count, network timeout), fall back to interpolation.

### Fallback: linear interpolation along route

Use known altitude anchor points in route order. For each anchor, find the nearest sample index with a **monotonic constraint**: each subsequent anchor search starts from the previous anchor's match index. This prevents a later city (e.g. 成都, 500 m) from matching a sample point early on the route where altitudes are still high.

Interpolate linearly between consecutive anchor pairs for all sample indices:

```
elev[i] = left.alt + (i - left.si) / (right.si - left.si) * (right.alt - left.alt)
```

### Rendering

Merge adjacent segments with the same elevation color to reduce polyline count. Use a distinct color scale (green → yellow → red → purple for increasing altitude). Draw with `zIndex: 55` so it sits above the base map but below markers.

Show a legend with the elevation range and color stops.

## Weather Mode

### Data collection

After all legs are planned, sample the combined path at **~40 km** intervals. Also add the 10 key city points (origin + destinations + waypoints) to ensure every city appears at least once.

Each sample point is assigned an **adcode** by finding the nearest adcode anchor point (using haversine distance). Known adcodes must cover all trip cities.

### Primary source: AMap.Weather Plugin

Use the built‑in AMap JSAPI Weather plugin (NOT a direct `fetch` to `restapi.amap.com/v3/weather/` — the JSAPI key cannot call the Web服务 API):

```javascript
AMap.plugin(['AMap.Weather'], function() {
  var weather = new AMap.Weather();
  weather.getForecast(adcode, function(err, data) {
    // data.forecast[].date, .dayWeather, .dayTemp, .nightTemp
  });
});
```

- Iterate through unique adcodes **serially** with 350 ms QPS delay.
- Store results in `weatherApiData[adcode] = data.forecast`.
- `getForecast()` returns **4 days** (today + 3). This only covers early trip days.

### Fallback: embedded forecastData

Embed a `forecastData` object with weather.com.cn data covering the full trip. Format:

```javascript
var forecastData = {
  '101271804': { name: '丹巴', days: {
    '5/28': '小雨 15℃',
    '5/29': '小雨转中雨 27℃/14℃',
    // ...
  }},
  // ... more cities
};
```

Each key is the weather.com.cn city code, and each `days` entry has key `'M/D'` (no leading zero) and value `'{condition} {tempRange}'`. Extract condition as the first space‑separated token: `weatherStr.split(' ')[0]`.

### Merge strategy: fallback as base, API as overlay

Always start from **forecastData** fallback (covers all days). Then overlay AMap.Weather API data for matching `(adcode, date)` pairs. Only use `cast.dayWeather` (daytime). This ensures:
- Dates within the 4‑day API window get real‑time AMap data.
- Dates beyond the window still show fallback data.
- If AMap plugin fails entirely, pure fallback is shown.

```javascript
// fallback first
var condition = forecastData[code].days[date].split(' ')[0];
// then overlay
if (apiData[adcode] matches date && cast.dayWeather) {
  condition = cast.dayWeather;
}
```

### Weather colors: rain‑intensity blue gradient

All rain conditions use a monochromatic **blue gradient** (deeper = heavier rain). Non‑rain conditions use distinguishable hue families:

| Category | Examples | Color strategy |
|----------|----------|----------------|
| Sunny | 晴, 少云, 晴间多云 | Warm (orange/amber/yellow) |
| Cloudy | 多云, 阴 | Gray (light to dark) |
| **Rain (blue gradient)** | 毛毛雨/细雨, 小雨, 小雨‑中雨, 阵雨, 中雨, 中雨‑大雨, 大雨, 大雨‑暴雨, 暴雨, 雷阵雨 | Light blue → deep blue by intensity |
| Snow | 雨夹雪, 雪, 小雪 | Purple/pink pastels |
| Fog/haze | 轻雾, 雾, 浓雾, 霾 | Muted gray‑teal, brown |

### Handling "A转B" transition patterns

Weather strings like `'多云转小雨'` or `'小雨转阴'` contain a `转` character. The color logic must:

1. Split by `转`.
2. Check each part for any rain keyword (using a rain priority list ordered by intensity).
3. If **any** part is rain, use the **heaviest** rain's color (e.g., `'多云转小雨'` → blue for 小雨, not gray for 多云).
4. If neither part is rain, use the first part's color.

Implement as a `rainOrder` array and a `getWeatherColor(weatherStr)` function that first checks for `转` patterns.

### Legend

The weather legend shows only the weather conditions that actually appear on the route (no unused entries). Each legend item is a colored swatch + condition name.

## Panel Display

### Per‑leg weather text

In the side panel, after each leg's origin and destination, append the weather condition + temperature range in parentheses:

```
重庆(多云 29℃/22℃) → 四姑娘山(小雨转阴 24℃/16℃)
```

Implement a `cityToWCode` mapping (city name → weather.com.cn code) and a `getCityWeatherText(cityName, date)` helper that looks up `forecastData[code].days[date]` and returns the full raw string (condition + temps).

### Per‑leg distance/duration

Show formatted distance and duration after each driving leg:
```
D1  重庆 → 四姑娘山
     391 km · 5h48min
```

### Schedule section (below all leg statuses)

Show departure/arrival times for each leg with altitude link:
```
D1 08:00 重庆 → 四姑娘山 13:48 📍3200m 预报
```

Use these schedule defaults:
- Normal start: 08:00
- 7+ hour driving day: 07:30
- Pure‑play: 09:00–17:30
- Buffer: 15 min per 2 h + 30 min if crossing lunch (11:30–13:00) or dinner (18:00–19:30).

## Schedule Drafting Rules

Base the schedule on AMap route duration when available; keep user-provided duration as a comparison note.

Defaults:
- Normal driving day starts at 08:00.
- Very long driving day (7+ hours) starts at 07:30 and avoids heavy sightseeing.
- Pure-play day uses 09:00-17:30 as the default activity window.
- Add 15-20 minutes buffer per 2 hours driving, plus a meal stop if crossing lunch or dinner.
- Avoid planning mountain/plateau night driving unless the user explicitly accepts it.
- For hard constraints like sunset or sunrise viewing, work backward from the target time and clearly show the assumption.

## Weather Source Rules

When adding weather to the itinerary:
- Use this weather source priority:
  1. China Meteorological Administration / China Weather 7-day and 8-15-day forecasts. Fetch from `https://www.weather.com.cn/weather/{code}.shtml`.
  2. Caiyun Weather (`https://h5.caiyunapp.com/`) for near-term hourly/minutely forecast checks.
  3. Date-pattern pages such as IP.cn `YYYYMM` daily forecast pages as supplemental cross-checks.
  4. Historical same-date records and monthly climate averages as reference only.
- For date-pattern weather URLs, verify the URL/title year and month match the itinerary year and month before treating the data as current forecast; for example prefer a `YYYYMM` page matching the trip month when available.
- Distinguish source type explicitly: current forecast, hourly forecast, 7/15-day forecast, historical same-date record, or monthly climate average.
- If sources disagree, show each channel separately instead of merging them, for example `China Weather: rain; Caiyun: cloudy to showers; IP.cn YYYYMM page: rain`.
- Do not call historical same-date records or climate averages a forecast. Label them as reference only.
- If the user wants copyable text, output raw source URLs instead of Markdown links.

**Important**: Some scenic-area city codes use a 5‑digit suffix format (`10127190204A` for 四姑娘山). The 7‑day weather page URL for these codes still works: `https://www.weather.com.cn/weather/10127190204A.shtml`. The `weather2d` page (`/weather2d/{code}.shtml`) only returns 2‑day data and may serve cached old pages for less‑common codes; prefer the 7‑day page.

Output each day as:

```markdown
### D1 出发地 -> 目的地
- 出发：
- 预计到达：
- 路线依据：（含途经点，如有）
- 当天安排：
- 风险/注意：
```

## Final Response

Return:
- The local map preview URL if a preview was generated.
- A concise Markdown itinerary with daily departure/arrival times.
- A note that AMap duration is planning data, not a guarantee; actual travel depends on road conditions, weather, altitude, and closures.
- Any missing or failed route legs that need user clarification.

## Common Mistakes

- Do not modify `amap-jsapi-skill`; treat it as the lower-level AMap reference.
- Do not fix route-preview workflow failures only in the generated demo; update this skill when the generator behavior needs a new guardrail.
- Do not batch‑geocode every stop before route planning; one silent geocoder callback can freeze the whole preview.
- Do not let one failed `AMap.Driving` leg prevent later legs from planning.
- Do not use bare numbers on the map when they can be confused with travel days; make stop order and day labels visually distinct.
- Do not turn same-day waypoints into separate days or overnight stops unless the user explicitly says to stay there.
- Do not ignore waypoint order; AMap should plan through waypoints in the user-provided order.
- Do not persist one-off route data inside this skill.
- Do not reuse a fixed port as a requirement.
- Do not let temporary demo directories accumulate.
- Do not claim route durations are final without AMap planning evidence or clear fallback assumptions.
- **Do not fetch `restapi.amap.com/v3/weather/` directly** — the JSAPI key cannot call the Web服务 API. Use `AMap.Weather.getForecast()` plugin instead.
- **Do not rely on AMap Weather API alone** — it only returns 4 days. Always embed watermarked forecastData for complete trip coverage and overlay AMap data on top.
- **Do not let "转" patterns slip** — `多云转小雨` must color as rain (blue), not as cloudy (gray). Check both sides of `转` for rain keywords.
- **Do not use a fixed port** when serving the preview; pick an available port at runtime.
