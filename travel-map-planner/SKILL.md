---
name: travel-map-planner
description: Use when the user provides a rough China road-trip itinerary, travel days, origin/destination stops, self-driving route outline, or asks to turn route duration into a daily schedule.
---

# Travel Map Planner

## Overview

Use this skill to convert a rough China self-driving itinerary into a route map preview and a detailed day-by-day schedule. This is a workflow skill: rely on `amap-jsapi-skill` for AMap JSAPI details instead of copying its API docs.

## When To Use

Use when the user provides or asks for:
- A rough route with days, origin, destination, and intermediate stops.
- A self-driving itinerary that needs map visualization.
- A daily schedule based on real AMap driving duration.
- Departure/arrival time planning from route durations.

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

## AMap Preview Workflow

When a map preview is needed:
- Read `amap-jsapi-skill` and relevant references such as routing, geocoder, marker, and security.
- Before generating website code, follow `amap-jsapi-skill` telemetry requirements.
- Use a reusable temporary directory under the workspace, normally `tmp-demo`; overwrite old generated files.
- Do not create a new demo directory per attempt unless the user explicitly asks for separate versions.
- Use an available local port at runtime. Do not require a fixed port.
- Inject `AMAP_JSAPI_KEY` and `AMAP_SECURITY_JSCODE` from environment variables; never hardcode secrets in generated files.
- Stop the related local server and remove the temporary directory when the user says the preview is no longer needed.

Generated preview should:
- Initialize AMap JSAPI v2.0 and set `AMap.getConfig().appname = 'amap-jsapi-skill';` before creating `new AMap.Map()`.
- Use `AMap.Driving` for self-driving route legs.
- Support per-leg `waypoints` for intermediate stops. With coordinates, call `Driving.search(origin, destination, { waypoints }, callback)`; with keyword points, pass `[origin, ...waypoints, destination]` so the first item is the start and the last item is the end.
- Display each leg's planning status, AMap distance, AMap duration, and any failed leg.
- Display each leg's waypoints in the side panel, for example `经：途经点A -> 途经点B`, and mark waypoint pins distinctly from overnight stops.
- Show pure-play days in the side panel but do not route-plan them.
- Distinguish stop sequence markers from day markers. For example, label stops as stop order and label route segments as `D1`, `D3`, etc.; label stay days at their base as `D2 纯玩` rather than implying a driving segment.
- Prefer user-provided precise place names; if geocoding is unstable, ask for more specific place names or coordinates.

Robustness requirements:
- Do not make the whole preview depend on a batch `Geocoder.getLocation()` phase. If geocoding is used, each lookup must have a timeout and a clear failed state.
- Prefer precise user-provided coordinates for route planning. For ambiguous county/town scenic stops, use explicit approximate center coordinates only when necessary and label them as approximate in the preview.
- Plan each route leg independently with its own timeout and status update. A failed or hanging leg must not block later legs.
- Show visible stage text such as "loading AMap JSAPI", "planning D3", or "D3 failed: <reason>" so the user is never left at a generic loading state.
- For long mountain or plateau legs, continue rendering successful legs even if one `AMap.Driving` request returns `error`, `no_data`, or never calls back before timeout.

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
  1. China Meteorological Administration / China Weather 7-day and 8-15-day forecasts.
  2. Caiyun Weather (`https://h5.caiyunapp.com/`) for near-term hourly/minutely forecast checks.
  3. Date-pattern pages such as IP.cn `YYYYMM` daily forecast pages as supplemental cross-checks.
  4. Historical same-date records and monthly climate averages as reference only.
- For date-pattern weather URLs, verify the URL/title year and month match the itinerary year and month before treating the data as current forecast; for example prefer a `YYYYMM` page matching the trip month when available.
- Distinguish source type explicitly: current forecast, hourly forecast, 7/15-day forecast, historical same-date record, or monthly climate average.
- If sources disagree, show each channel separately instead of merging them, for example `China Weather: rain; Caiyun: cloudy to showers; IP.cn YYYYMM page: rain`.
- Do not call historical same-date records or climate averages a forecast. Label them as reference only.
- If the user wants copyable text, output raw source URLs instead of Markdown links.

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
- Do not batch-geocode every stop before route planning; one silent geocoder callback can freeze the whole preview.
- Do not let one failed `AMap.Driving` leg prevent later legs from planning.
- Do not use bare numbers on the map when they can be confused with travel days; make stop order and day labels visually distinct.
- Do not turn same-day waypoints into separate days or overnight stops unless the user explicitly says to stay there.
- Do not ignore waypoint order; AMap should plan through waypoints in the user-provided order.
- Do not persist one-off route data inside this skill.
- Do not reuse a fixed port as a requirement.
- Do not let temporary demo directories accumulate.
- Do not claim route durations are final without AMap planning evidence or clear fallback assumptions.
