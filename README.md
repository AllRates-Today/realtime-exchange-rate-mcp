# Realtime Exchange Rate MCP Server — @allratestoday/mcp-server

A Model Context Protocol server that lets Claude Code, Cursor, Claude Desktop, Windsurf, and any other MCP-compatible client fetch real-time currency rates, historical series, and multi-currency lookups from the [AllRatesToday](https://allratestoday.com) API. Rates are mid-market, with no retail spread.

[![Powered by AllRatesToday](https://img.shields.io/badge/Powered%20by-AllRatesToday-orange.svg)](https://allratestoday.com)
[![npm version](https://img.shields.io/npm/v/@allratestoday/mcp-server.svg)](https://www.npmjs.com/package/@allratestoday/mcp-server)
[![npm downloads](https://img.shields.io/npm/dm/@allratestoday/mcp-server.svg)](https://www.npmjs.com/package/@allratestoday/mcp-server)
[![CI](https://github.com/AllRates-Today/realtime-exchange-rate-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/AllRates-Today/realtime-exchange-rate-mcp/actions/workflows/ci.yml)
[![MCP](https://img.shields.io/badge/Model%20Context%20Protocol-1.x-blue.svg)](https://modelcontextprotocol.io)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6.svg)](https://www.typescriptlang.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)

English | [简体中文](./README-zh-CN.md)

After installation, your assistant can answer questions like:

- *"What's the current USD to EUR rate?"*
- *"Show me how GBP/JPY moved over the last 30 days."*
- *"Convert 250 USD into CAD at a real rate."*
- *"Compare USD against EUR, GBP, and JPY simultaneously."*
- *"List every supported currency."*
- *"What is the official ECB rate for USD today?"*

This README documents the npm package [`@allratestoday/mcp-server`](https://www.npmjs.com/package/@allratestoday/mcp-server) (current version 0.7.6). Its source lives in [AllRates-Today/mcp-server](https://github.com/AllRates-Today/mcp-server).

## 🚀 Features

- 💱 **Live mid-market rates** — current rate for any supported ISO 4217 pair
- 📈 **Historical series built in** — `1d` (hourly), `7d` (daily), `30d` (daily), `1y` (weekly)
- 🧰 **Eight focused tools** — `get_exchange_rate`, `get_historical_rates`, `get_rates_authenticated`, `list_currencies`, `list_central_banks`, `get_official_rates`, `get_reference_rate`, `request_trial_key`; a small surface the model uses correctly (the hosted endpoint `https://allratestoday.com/api/mcp` lists 11: these plus `get_official_rate_history`, `compare_official_rates`, `get_publication_calendar`)
- 🏛️ **Official rates** — the latest table from 121 central banks + 3 tax authorities, keyless
- 🔌 **Works everywhere MCP does** — stdio transport, MCP SDK 1.x; Claude Code, Cursor, Claude Desktop, Windsurf, or any generic stdio host
- 🔓 **Works with no API key** — starts in keyless mode and never exits for a missing key; keyless answers say which official table they came from and its `rate_date`
- 🛡️ **Honest errors** — relays upstream API errors instead of guessing, and tools that need a key say how to get one
- 🔒 **Nothing leaks** — only the request parameters and your API key ever reach allratestoday.com; never conversation context
- 📦 **Two runtime dependencies** — `@modelcontextprotocol/sdk` and `zod`; Node.js ≥ 18

With a key, `get_exchange_rate`, `get_historical_rates` and `get_rates_authenticated` return a **mid-market rate** — the right number for price display and conversion. It is not the official rate a tax authority or auditor may require; for those use `get_official_rates` (latest published table of any of 121 central banks + 3 tax authorities, keyless), or see the [AllRatesToday docs](https://allratestoday.com/docs).

## 🔑 Get your API key

The key is **optional**. Without `ALLRATES_API_KEY` the server starts in **keyless mode**:

| Tool | Keyless | What you get |
|---|---|---|
| `get_exchange_rate` | ✅ | An official central-bank rate: the ECB table first, else the central bank that issues the target or source currency (e.g. USD/GBP → Bank of England). Only pairs one bank publishes, or their inverses (flagged `derived: true`), work keyless |
| `list_currencies` | ✅ | All 160+ supported codes |
| `list_central_banks` | ✅ | All 121 central banks + 3 tax authorities with codes and latest publication date |
| `get_official_rates` | ✅ | Latest official table (or one pair); a dated table needs a paid-plan key |
| `get_reference_rate` | ✅ | Latest Daily Reference Rate table (16:00 London); a past date needs a paid-plan key |
| `request_trial_key` | ✅ | Self-issues a 100-call, 7-day key with no account and adopts it for the session |
| `get_historical_rates` | 🔑 | Paid-plan key (free and trial keys get 403) |
| `get_rates_authenticated` | 🔑 | Free key for latest rates; `time` (a past datetime) needs a paid plan |

A free key unlocks real-time mid-market rates for 160+ currencies and multi-target lookups. The free tier is **300 requests, lifetime** — it never resets. All history and dated rates — mid-market history, point-in-time lookups, dated official and reference-rate tables — need a paid plan; free and trial keys get `403 historical_requires_paid_plan` ([pricing](https://allratestoday.com/pricing)).

1. Register at [allratestoday.com/register](https://allratestoday.com/register)
2. Verify your email
3. Copy your key from the dashboard (format: `art_live_xxxxx`)
4. Use it as `ALLRATES_API_KEY` in the configs below

If the key is missing, the server prints a short keyless-mode summary on stderr and keeps running — it never exits for a missing key.

## 📦 Installation

The simplest install is **zero-install via `npx`**, which is what every config below uses:

```bash
# Run without installing (recommended)
npx -y @allratestoday/mcp-server
```

```bash
# Or install globally
npm install -g @allratestoday/mcp-server
allratestoday-mcp
```

Both commands launch the stdio MCP server and wait for a client to connect — they are not meant to be run interactively from your shell; your MCP client launches them as a subprocess.

## 🏁 Quick start

Each client reads MCP servers from a different config file. Pick yours below.

### Claude Code

The fastest path uses the built-in CLI:

```bash
claude mcp add allratestoday -- npx -y @allratestoday/mcp-server
claude mcp env allratestoday ALLRATES_API_KEY=art_live_xxxxx
```

Restart Claude Code, then ask: *"What's the current USD to EUR rate?"*

### Cursor

Edit `~/.cursor/mcp.json` (or `.cursor/mcp.json` inside your project for a project-scoped server):

```json
{
  "mcpServers": {
    "allratestoday": {
      "command": "npx",
      "args": ["-y", "@allratestoday/mcp-server"],
      "env": {
        "ALLRATES_API_KEY": "art_live_xxxxx"
      }
    }
  }
}
```

Restart Cursor. The eight tools should appear in the MCP tool picker.

### Claude Desktop

Edit the config file (path depends on OS):

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

```json
{
  "mcpServers": {
    "allratestoday": {
      "command": "npx",
      "args": ["-y", "@allratestoday/mcp-server"],
      "env": {
        "ALLRATES_API_KEY": "art_live_xxxxx"
      }
    }
  }
}
```

**Fully quit and reopen Claude Desktop** (Cmd+Q on macOS, right-click tray icon → Exit on Windows). Closing the window alone keeps the old config loaded.

### Windsurf

Edit `~/.codeium/windsurf/mcp_config.json` with the same `mcpServers` block as above, then restart Windsurf.

### Generic stdio MCP client

Any MCP host that supports stdio transport works. The launch command is:

```bash
npx -y @allratestoday/mcp-server
```

…optionally with `ALLRATES_API_KEY` set in the subprocess environment. The same block, ready to copy, also lives in [`.mcp.json`](./.mcp.json) in this repo; [`smithery.yaml`](./smithery.yaml) describes the same stdio launch for Smithery, and [`server.json`](./server.json) is the MCP registry manifest.

### Verify it works

1. **Server starts** — open the client. It starts with or without a key, so a red dot or "failed to connect" means the client could not launch it — usually `npx` or Node ≥ 18 (see [Troubleshooting](#-troubleshooting)).
2. **Tools are listed** — most clients have a "tools" or "MCP" panel showing all eight tools.
3. **A live call returns a number** — ask *"What's the current USD to EUR rate?"* The assistant should call `get_exchange_rate(source: "USD", target: "EUR")` and reply with a real rate. If it produces a number without a tool call, the server is not connected.

## 📚 API reference

| Tool | Purpose | Required input |
|---|---|---|
| [`get_exchange_rate`](#get_exchange_rate) | Current rate for one pair | `source`, `target` |
| [`get_historical_rates`](#get_historical_rates) | Time series over a preset period | `source`, `target` |
| [`get_rates_authenticated`](#get_rates_authenticated) | Multiple targets in one call, optional point-in-time | `source`, `target` |
| [`list_currencies`](#list_currencies) | All supported codes, names, symbols | — |
| [`list_central_banks`](#list_central_banks) | Every covered central bank / tax authority with its code | — |
| [`get_official_rates`](#get_official_rates) | Official table (or one pair) a named institution published | `bank` |
| `get_reference_rate` | Daily Reference Rate, one fixed mid-market table per business day (16:00 London) | — |
| `request_trial_key` | Self-issue a 100-call, 7-day trial key, no account | — |

Only `get_historical_rates` and `get_rates_authenticated` require `ALLRATES_API_KEY`: a free key covers latest multi-target rates; `get_historical_rates` and `time` need a paid plan. Input schemas set `additionalProperties: false` — unknown fields are rejected. Full per-tool reference: [mcp-server README](https://github.com/AllRates-Today/mcp-server#-tools-reference).

### `get_exchange_rate`

Current mid-market rate between two currencies. Calls `GET /rate` with a key; keyless it calls `GET /open/central-bank/{bank}` (ECB first, then the bank issuing the target, then the source currency).

| Field | Type | Required | Description |
|---|---|---|---|
| `source` | string (exactly 3 chars) | yes | ISO 4217 code, e.g. `USD` |
| `target` | string (exactly 3 chars) | yes | ISO 4217 code, e.g. `EUR` |

```json
{ "source": "USD", "target": "EUR" }
```

Response with a key:

```json
{ "rate": 0.888889, "source": "mid-market" }
```

Response keyless (`USD` → `GBP` is not in the ECB table, so the Bank of England answers; `derived: true` because it publishes the inverse):

```json
{
  "rate": 0.7598784195,
  "source": "Bank of England official rate (keyless mode)",
  "bank": "boe",
  "rate_date": "2026-10-14",
  "derived": true,
  "note": "Keyless mode: this is the Bank of England official rate published on 2026-10-14, not a live mid-market quote. For real-time rates across 160+ currencies set ALLRATES_API_KEY — free tier at https://allratestoday.com/register."
}
```

### `get_historical_rates`

Time-series data points for a currency pair over a fixed period. Calls `GET /historical-rates`. Needs an API key on a paid plan (free and trial keys get a 403).

| Field | Type | Required | Description |
|---|---|---|---|
| `source` | string (exactly 3 chars) | yes | Source currency code |
| `target` | string (exactly 3 chars) | yes | Target currency code |
| `period` | string | no (default `7d`) | One of `1d`, `7d`, `30d`, `1y` |

Granularity per period:

| `period` | Granularity |
|---|---|
| `1d` | Hourly |
| `7d` | Daily |
| `30d` | Daily |
| `1y` | Weekly |

```json
{ "source": "USD", "target": "INR", "period": "30d" }
```

Response (truncated):

```json
{
  "source": "USD",
  "target": "INR",
  "period": "30d",
  "data": [
    { "date": "2026-09-17", "rate": 96.1006, "timestamp": 1789603200000 },
    { "date": "2026-09-18", "rate": 95.8142, "timestamp": 1789689600000 },
    "...",
    { "date": "2026-10-16", "rate": 94.9643, "timestamp": 1792108800000 }
  ]
}
```

### `get_rates_authenticated`

Multiple targets in one call, with an optional historical timestamp or grouping window. Calls `GET /v1/rates`.

| Field | Type | Required | Description |
|---|---|---|---|
| `source` | string (exactly 3 chars) | yes | Source currency code |
| `target` | string | yes | One or more codes, comma-separated (`EUR,GBP,JPY`) |
| `time` | string (ISO 8601 date-time) | no | Historical point in time (paid plans) |
| `group` | string | no | One of `hour`, `day`, `week`, `month` |

```json
{ "source": "USD", "target": "EUR,GBP,JPY" }
```

Response — `{ rates: [{ rate, source, target, time }] }`:

```json
{
  "rates": [
    { "rate": 0.888889, "source": "USD", "target": "EUR", "time": "2026-10-16T12:16:01+0000" },
    { "rate": 0.749766, "source": "USD", "target": "GBP", "time": "2026-10-16T12:16:01+0000" },
    { "rate": 157.985, "source": "USD", "target": "JPY", "time": "2026-10-16T12:16:01+0000" }
  ]
}
```

### `list_currencies`

All supported currencies with codes, names, and symbols. Calls `GET /v1/symbols`, cached 24 h upstream — cheap to call for validating user input before the other tools.

**Input** — none.

Response (truncated):

```json
{
  "currencies": [
    { "code": "USD", "name": "US Dollar", "symbol": "$" },
    { "code": "EUR", "name": "Euro", "symbol": "€" },
    { "code": "GBP", "name": "British Pound", "symbol": "£" },
    "..."
  ],
  "count": 163
}
```

### `list_central_banks`

Every official source — 121 central banks plus HMRC, the US Treasury and Swiss BAZG — with the `code` that `get_official_rates` takes as `bank`. Keyless. Calls `GET /open/central-banks` (or `/v1/central-banks` with a key).

```json
{
  "sources": [
    { "code": "ecb", "name": "European Central Bank", "latest": "2026-10-15", "age_days": 1, "stale": false, "cadence": "daily" },
    { "code": "fed", "name": "US Federal Reserve (H.10)", "latest": "2026-10-09", "age_days": 7, "stale": false, "cadence": "weekly" },
    "..."
  ],
  "stale_count": 3,
  "checked_at": "2026-10-16T12:16:46.232Z"
}
```

### `get_official_rates`

The fixed, citable rate an institution published. Input: `bank` (required), optional `date` (`YYYY-MM-DD`, **paid plan** — a free key gets `403 Historical central bank rates require a paid plan`), optional `source` + `target` for one pair. Keyless for the latest table.

```json
{
  "bank": "ecb",
  "rate_date": "2026-10-15",
  "source": "EUR",
  "target": "USD",
  "rate": 1.1208,
  "rate_type": "reference",
  "derived": false,
  "method": "published",
  "age_days": 1,
  "stale": false,
  "cadence": "daily",
  "attribution": { "source": "Official rates published by ECB, served by AllRatesToday", "url": "https://allratestoday.com/central-bank-rates-api/ecb/", "terms": "Free to use with visible attribution linking to allratestoday.com" }
}
```

## 🗺️ Currencies covered

Every currency the AllRatesToday API serves is available through these tools — call `list_currencies` for the authoritative live list. Commonly used codes include:

🇺🇸 `USD` · 🇪🇺 `EUR` · 🇬🇧 `GBP` · 🇯🇵 `JPY` · 🇨🇦 `CAD` · 🇮🇳 `INR`

The `source` and `target` fields of `get_exchange_rate` and `get_historical_rates` are validated as exactly three characters, so pass ISO 4217 codes, not currency names.

## ⚙️ Environment variables

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `ALLRATES_API_KEY` | — | no | Your API key, sent as a `Bearer` token in the `Authorization` header. Unset ⇒ keyless mode; the server never exits for a missing key. |
| `ALLRATES_BASE_URL` | `https://allratestoday.com/api` | no | Override for a self-hosted or staging deployment. Trailing slashes are stripped. |

Set these in your MCP client's config (in the `env` block), not in your shell — MCP servers are launched as subprocesses with isolated environments.

## 🛡️ Error handling

Tool failures come back as an MCP tool result with `isError: true`. The text is `AllRatesToday error (<status>): <message>`.

| HTTP status | Message |
|---|---|
| 400 | The API's `error` field, else `Bad request — possibly an unknown currency code` |
| 401 | `Invalid AllRatesToday API key` |
| 403 | `HTTP 403 — <upstream message>`, e.g. `Historical rates require a paid plan…` for `get_historical_rates` or `time` on a free or trial key, or `Historical central bank rates require a paid plan…` for a dated table |
| 404 (keyless) | `No keyless official source covers X/Y. Real-time rates for 160+ currencies need an API key …` |
| 429 | `AllRatesToday API quota exceeded` |
| 5xx | `HTTP <status> — <upstream message>` |

Errors raised locally, before any HTTP call:

- Keyless call to `get_rates_authenticated` (no `time`) → `… needs an AllRatesToday API key. The free tier covers it — sign up at https://allratestoday.com/register …`
- Keyless call to `get_historical_rates`, or `get_rates_authenticated` with `time` → `… needs an AllRatesToday API key on a paid plan …`
- Keyless call with a `date` to `get_official_rates` / `get_reference_rate` → `… needs an AllRatesToday API key on a paid plan …`
- Unrecognised tool name → `MCP error -32602: Tool <name> not found`

Because these arrive as text, the assistant relays them to the user — a 429 surfaces as *"the API quota has been exceeded."*

## 🛠️ Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Client shows "MCP server failed to start" or a red dot | Not a missing key (0.5.0+ starts without one) — usually `npx` cannot reach the registry, or Node is older than 18 | Run `npx -y @allratestoday/mcp-server` in a shell and read stderr |
| Every call returns a 401 error | Key malformed, truncated, or revoked | Copy a fresh key from the dashboard |
| Calls return a 429 error | Quota used up — the free tier is 300 requests lifetime and never resets | Upgrade at [allratestoday.com/pricing](https://allratestoday.com/pricing) |
| Calls return a 403 error | `get_historical_rates`, `time`, or `date` on `get_official_rates` / `get_reference_rate` with a free or trial key | History is paid-only: omit `date`/`time` for the latest rates, or upgrade |
| `get_historical_rates` returns a 400 error | Invalid period or unknown currency code | `period` must be `1d`/`7d`/`30d`/`1y`; codes must be exactly 3 letters |
| Server starts but tools never appear | Client did not reload after the config change | Fully quit (not just close) and reopen the client |
| `npx` runs but hangs forever | Normal — the server is waiting for an MCP client on stdio | Let your MCP client launch it |

To inspect what the server is doing, run it manually (with or without the key):

```bash
ALLRATES_API_KEY=art_live_xxxxx npx -y @allratestoday/mcp-server
```

No output means healthy — stdout is reserved for the MCP protocol; errors print to stderr.

## 💡 Notes

**Do you store my conversation or query data?** No. Only your API key and the request parameters (`source`, `target`, `period`, `time`, `group`) are sent to allratestoday.com — never the model's conversation context.

**What happens to my API key?** It is only sent as a `Bearer` token in the `Authorization` header on requests to the AllRatesToday API. The server does not log it.

**Why is the first call slow?** Cold start of `npx` (the first run downloads the package) plus an upstream cache miss.

**Can I run this without npm/Node?** Not currently — `engines` requires Node ≥ 18. If a standalone binary matters to you, open an issue.

**Is there a self-hosted option?** Point `ALLRATES_BASE_URL` at your own instance.

**Does it work with other clients?** Any MCP-compatible host with stdio transport works, and remote-MCP clients can use the hosted endpoint `https://allratestoday.com/api/mcp`; the four clients above are simply the ones with documented config paths here.

## 👩‍💻 Development

The `src/` in this repo is an older 0.3.x snapshot. The published package is built from [AllRates-Today/mcp-server](https://github.com/AllRates-Today/mcp-server) — develop there:

```bash
git clone https://github.com/AllRates-Today/mcp-server.git
cd mcp-server
npm install
npm run build
ALLRATES_API_KEY=art_live_xxxxx node dist/index.js
```

`npm run build` runs `tsc`; `npm run dev` watches and rebuilds; `npm start` runs the compiled server. CI ([`.github/workflows/ci.yml`](./.github/workflows/ci.yml)) runs `npm ci && npm run build` on Node 22 for every push to `main` and every pull request.

To test against a local AllRatesToday instance:

```bash
ALLRATES_BASE_URL=http://localhost:8080/api ALLRATES_API_KEY=test_key node dist/index.js
```

Project structure:

```text
src/
├── index.ts      # MCP server, tool definitions, request handlers
├── client.ts     # HTTP client for the AllRatesToday API + error mapping
└── home-banks.ts # currency → issuing central bank (keyless fallback; mcp-server only)
dist/             # Compiled JS (gitignored)
server.json       # MCP registry manifest
smithery.yaml     # Smithery launch config
.mcp.json         # Ready-to-copy client config
```

Issues and PRs are welcome. Before opening a PR: `npm run build` must succeed, exercise the change against a real API key, and update both the tool descriptions in `src/index.ts` and the API reference above if tool behaviour changes.

## 📝 Changelog

Full changelog: [mcp-server README → Changelog](https://github.com/AllRates-Today/mcp-server#-changelog). Recent highlights:

- **0.7.6** — Keyless `get_rates_authenticated` with `time` says a paid plan is needed; history documented as paid-only
- **0.7.4** — Keyless `get_exchange_rate` falls back from the ECB table to the central bank issuing either currency (USD/GBP, GBP/JPY, USD/INR answer keyless), like the hosted endpoint; dated-table errors say a paid plan is needed
- **0.7.0** — `get_reference_rate` (Daily Reference Rate) and `request_trial_key` (self-issued 100-call trial key)
- **0.6.0** — Keyless `list_central_banks` and `get_official_rates` (121 central banks + 3 tax authorities)
- **0.5.0** — Keyless mode: the server starts and answers without an API key instead of exiting
- **0.3.x** — API key required for all tools; fail-fast at startup with clear instructions
- **0.2.x** — Removed the news tool; required auth on `get_historical_rates`
- **0.1.x** — Initial release with 5 tools

## 🔗 Links

- **Website:** [allratestoday.com](https://allratestoday.com)
- **API docs:** [allratestoday.com/docs](https://allratestoday.com/docs)
- **Free API key:** [allratestoday.com/register](https://allratestoday.com/register)
- **Status:** [allratestoday.com/status](https://allratestoday.com/status)
- **Support:** [allratestoday.com/contact](https://allratestoday.com/contact)
- **npm:** [@allratestoday/mcp-server](https://www.npmjs.com/package/@allratestoday/mcp-server)
- **MCP protocol:** [modelcontextprotocol.io](https://modelcontextprotocol.io)
- **Bug reports:** [github.com/AllRates-Today/mcp-server/issues](https://github.com/AllRates-Today/mcp-server/issues)
- **AI agents:** Claude Code plugin `/plugin marketplace add AllRates-Today/claude-code-plugin` · hosted MCP endpoint `https://allratestoday.com/api/mcp` (keyless)
- **New in Oct 2026:** agents can self-issue a trial key — `POST https://allratestoday.com/api/agent/keys` (JSON) → 100 calls / 7 days — and read the keyless [Daily Reference Rate](https://allratestoday.com/reference-rate/) (16:00 London fix) at `https://allratestoday.com/api/open/reference-rate`
- **Dashboard:** [allratestoday.com/profile](https://allratestoday.com/profile)


## 📜 License

MIT — see [LICENSE](./LICENSE).
