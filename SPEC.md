# OpenFit Ultrahuman — Specification

> Forked from [FlavioAdamo/openfit](https://github.com/FlavioAdamo/openfit)
> Provider: **Ultrahuman Ring** via UltraSignal Partner API
> LLM: **Configurable** (MiniMax by default, Codex-compatible interface)

---

## Overview

This is a macOS desktop app that connects to the **Ultrahuman Ring** (via the UltraSignal Partner API) and surfaces health data with an AI assistant — the same architecture as OpenFit, but with a different health provider and a swappable LLM backend.

The core contract (IPC between Electron main and renderer, health context XML injection, assistant event streaming) is unchanged.

---

## Provider: Ultrahuman Ring

**API:** UltraSignal Partner API
**Auth:** Static API key (`Authorization: <key>`) + Partner Code (`Partner-Code: UDUCCTPQ`)
**Credentials:** Stored in Electron `safeStorage` (same as OpenFit's OAuth tokens)

### Endpoints used

| Method | URL | Purpose |
|---|---|---|
| GET | `https://partner.ultrahuman.com/api/v1/metrics?email=...&date=YYYY-MM-DD` | Daily metrics (sleep, HRV, HR, activity, etc.) |

### Data → Legacy Format Mapping

The UltraSignal response is flat. It must be translated into the **Fitbit Legacy response shape** that the rest of the app (normalizer, UI) expects.

Key mappings:

| UltraSignal field | Fitbit Legacy equivalent |
|---|---|
| `sleep_data.sleep_score` | `sleep[0].efficiency` |
| `sleep_data.total_sleep` (minutes) | `sleep[0].minutesAsleep` |
| `sleep_data.deep_sleep_minutes` | `sleep[0].levels.summary.deep.minutes` |
| `sleep_data.rem_sleep_minutes` | `sleep[0].levels.summary.rem.minutes` |
| `sleep_data.light_sleep_minutes` | `sleep[0].levels.summary.light.minutes` |
| `sleep_data.awake_time_minutes` | `sleep[0].minutesAwake` |
| `sleep_data.no_of_complete_cycles` | `sleep[0].levels.summary` |
| `hrv_data.hrv_value` | `hrv[0].value.dailyRmssd` |
| `recovery_index.recovery_index` | `hrv[0]` or `sleep[0]` |
| `heart_rate.resting_hr` | `heartIntraday[0].value.restingHeartRate` |
| `activity.total_steps` | `activity.summary.steps` |
| `activity.total_calories_burned` | `activity.summary.caloriesOut` |
| `activity.active_minutes` | `activity.summary.activeMinutes` |
| `sleep_data.sleep_start_time` | `sleep[0].startTime` |
| `sleep_data.sleep_end_time` | `sleep[0].endTime` |
| `temperature.skin_temp_avg_celsius` | `skinTemperature[0]` |

### What the API does NOT provide (null/missing)
- ECG data
- Blood glucose (no CGM)
- SpO2
- Breathing rate (raw)
- VO2 Max (not from ring alone)
- Food/water logs

---

## LLM: Configurable Service

**Default:** MiniMax HTTP API
**Interface:** Identical to `CodexService` — same constructor options, same `startTurn()` signature, same event callbacks.

### MiniMaxService vs CodexService

| Concern | CodexService | MiniMaxService |
|---|---|---|
| Transport | `codex app-server` subprocess (stdio JSONL) | MiniMax HTTP API (fetch) |
| Auth | `CODEX_API_KEY` env var | `MINIMAX_API_KEY` env var or constructor option |
| Model | `options.model` | `options.model` |
| System prompt | `options.developerInstructions` | Same — `HEALTH_ASSISTANT_DEVELOPER_INSTRUCTIONS` |
| Streaming | Yes (stdio delta) | Yes (SSE via `eventsource` or fetch streams) |
| Thread/conversation | Codex thread ID | MiniMax conversation ID |

### MiniMax API Reference

```javascript
// Base URL
const BASE_URL = 'https://api.minimax.chat/v1'

// Chat completions
POST https://api.minimax.chat/v1/text/chatcompletion_v2
Headers: Authorization: Bearer <key>
Body: { model, messages: [{role, content}], stream: true }

// Stream format: SSE, each event: data: {"choices": [{"delta": {"content": "..."}}]}
```

### Configuration

```javascript
// In electron/main.cjs
const MiniMaxService = require('./minimax-service.cjs')

const llmService = new MiniMaxService({
  apiKey: process.env.MINIMAX_API_KEY,
  model: 'MiniMax-Text-01',       // configurable
  developerInstructions: '...',    // same prompt as Codex
  maxHealthContextChars: 500_000,
})
```

Users can later swap to OpenAI, Anthropic, or any OpenAI-compatible endpoint by passing a different service class — the IPC contract is identical.

---

## Credentials Storage

Same pattern as OpenFit — encrypted JSON via Electron `safeStorage`:

```
~/Library/Application Support/<app-name>/credentials.secure.json
```

Stored fields:
```json
{
  "ultrahuman": {
    "apiKey": "<encrypted>",
    "partnerCode": "UDUCCTPQ",
    "email": "udupi.sachin.acharya@gmail.com"
  },
  "llm": {
    "provider": "minimax",
    "apiKey": "<encrypted>",
    "model": "MiniMax-Text-01"
  }
}
```

---

## What Changes (vs OpenFit)

### Files modified
- `electron/main.cjs` — register `ultrahuman-service` as a provider, use `MiniMaxService` instead of `CodexService`, update CSP for MiniMax API domain
- `package.json` — rename app to `OpenFit Ultrahuman`, drop Codex CLI dep, add MiniMax SDK
- `src/lib/normalizeFitbitData.ts` — add `if (source === 'ultrahuman')` branch in `normalizeDashboardData()`

### Files added
- `electron/ultrahuman-service.cjs`
- `electron/minimax-service.cjs`

### Files NOT changed
- `src/lib/health-assistant.ts` — context building unchanged
- `src/lib/format.ts` — formatting unchanged
- All React UI components in `src/`
- `electron/preload.cjs`, `electron/health-cache.cjs` — unchanged

---

## macOS Build

```bash
npm run dist   # → dist/mac/OpenFit Ultrahuman.dmg
```

App metadata:
- **Bundle ID:** `com.pulseboard.ultrahuman`
- **Category:** `public.app-category.healthcare-fitness`
- **App name:** `OpenFit Ultrahuman`
- **Icon:** reuse `build/icon.png` (no redesign)
