# Realtime Exchange Rate MCP Server — @allratestoday/mcp-server

A Model Context Protocol server that lets Claude Code, Cursor, Claude Desktop, Windsurf, and any other MCP-compatible client fetch real-time currency rates, historical series, and multi-currency lookups from the [AllRatesToday](https://allratestoday.com) API. Rates are mid-market, with no retail spread.

[![Powered by AllRatesToday](https://img.shields.io/badge/Powered%20by-AllRatesToday-orange.svg)](https://allratestoday.com)
[![npm version](https://img.shields.io/npm/v/@allratestoday/mcp-server.svg)](https://www.npmjs.com/package/@allratestoday/mcp-server)
[![npm downloads](https://img.shields.io/npm/dm/@allratestoday/mcp-server.svg)](https://www.npmjs.com/package/@allratestoday/mcp-server)
[![CI](https://github.com/cahthuranag/realtime-exchange-rate-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/cahthuranag/realtime-exchange-rate-mcp/actions/workflows/ci.yml)
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

## 🚀 Features

- 💱 **Live mid-market rates** — current rate for any supported ISO 4217 pair
- 📈 **Historical series built in** — `1d` (hourly), `7d` (daily), `30d` (daily), `1y` (weekly)
- 🧰 **Four focused tools** — `get_exchange_rate`, `get_historical_rates`, `get_rates_authenticated`, `list_currencies`; a small surface the model uses correctly
- 🔌 **Works everywhere MCP does** — stdio transport, MCP SDK 1.x; Claude Code, Cursor, Claude Desktop, Windsurf, or any generic stdio host
- 🛡️ **Fail-fast and honest** — refuses to start without an API key and relays upstream API errors verbatim instead of guessing
- 🔒 **Nothing leaks** — only the request parameters and your API key ever reach allratestoday.com; never conversation context
- 📦 **Two runtime dependencies** — `@modelcontextprotocol/sdk` and `zod`; Node.js ≥ 18

Everything these tools return is a **mid-market rate** — the right number for price display and conversion. It is not the official rate a tax authority or auditor may require; for published central-bank and tax-authority rates, see the [AllRatesToday docs](https://allratestoday.com/docs).

## 🔑 Get your API key

The server **will not start** without a valid `ALLRATES_API_KEY`, and all four tools require it. A free key is enough for development and personal use.

1. Register at [allratestoday.com/register](https://allratestoday.com/register)
2. Verify your email
3. Copy your key from the dashboard (format: `art_live_xxxxx`)
4. Use it as `ALLRATES_API_KEY` in the configs below

If the key is missing, the server prints registration instructions on stderr and exits with code 1.

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

Restart Cursor. The four tools should appear in the MCP tool picker.

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

…with `ALLRATES_API_KEY` set in the subprocess environment. The same block, ready to copy, also lives in [`.mcp.json`](./.mcp.json) in this repo; [`smithery.yaml`](./smithery.yaml) describes the same stdio launch for Smithery, and [`server.json`](./server.json) is the MCP registry manifest.

### Verify it works

1. **Server starts** — open the client. A red dot or "failed to connect" means the API key is missing or wrong (see [Troubleshooting](#-troubleshooting)).
2. **Tools are listed** — most clients have a "tools" or "MCP" panel showing all four tools.
3. **A live call returns a number** — ask *"What's the current USD to EUR rate?"* The assistant should call `get_exchange_rate(source: "USD", target: "EUR")` and reply with a real rate. If it produces a number without a tool call, the server is not connected.

## 📚 API reference

| Tool | Purpose | Required input |
|---|---|---|
| [`get_exchange_rate`](#get_exchange_rate) | Current rate for one pair | `source`, `target` |
| [`get_historical_rates`](#get_historical_rates) | Time series over a preset period | `source`, `target` |
| [`get_rates_authenticated`](#get_rates_authenticated) | Multiple targets in one call, optional point-in-time | `source`, `target` |
| [`list_currencies`](#list_currencies) | All supported codes, names, symbols | — |

All four tools require `ALLRATES_API_KEY`, and every input schema sets `additionalProperties: false` — unknown fields are rejected.

### `get_exchange_rate`

Current mid-market rate between two currencies. Calls `GET /rate`.

| Field | Type | Required | Description |
|---|---|---|---|
| `source` | string (exactly 3 chars) | yes | ISO 4217 code, e.g. `USD` |
| `target` | string (exactly 3 chars) | yes | ISO 4217 code, e.g. `EUR` |

```json
{ "source": "USD", "target": "EUR" }
```

Response shape — `rate` (number) and `source` (string, the upstream data source identifier):

```json
{ "rate": 0.92145, "source": "..." }
```

### `get_historical_rates`

Time-series data points for a currency pair over a fixed period. Calls `GET /historical-rates`.

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
    { "date": "2026-03-27T00:00:00Z", "rate": 83.42, "timestamp": 1743033600000 },
    { "date": "2026-03-28T00:00:00Z", "rate": 83.51, "timestamp": 1743120000000 },
    "..."
  ]
}
```

### `get_rates_authenticated`

Multiple targets in one call, with an optional historical timestamp or grouping window. Calls `GET /v1/rates`.

| Field | Type | Required | Description |
|---|---|---|---|
| `source` | string (exactly 3 chars) | yes | Source currency code |
| `target` | string | yes | One or more codes, comma-separated (`EUR,GBP,JPY`) |
| `time` | string (ISO 8601 date-time) | no | Historical point in time |
| `group` | string | no | One of `hour`, `day`, `week`, `month` |

```json
{ "source": "USD", "target": "EUR,GBP,JPY" }
```

Response — an array of `{ rate, source, target, time }`:

```json
[
  { "rate": 0.9214, "source": "USD", "target": "EUR", "time": "2026-04-26T11:00:00Z" },
  { "rate": 0.7891, "source": "USD", "target": "GBP", "time": "2026-04-26T11:00:00Z" },
  { "rate": 151.34, "source": "USD", "target": "JPY", "time": "2026-04-26T11:00:00Z" }
]
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
  "count": 162
}
```

## 🗺️ Currencies covered

Every currency the AllRatesToday API serves is available through these tools — call `list_currencies` for the authoritative live list. Commonly used codes include:

🇺🇸 `USD` · 🇪🇺 `EUR` · 🇬🇧 `GBP` · 🇯🇵 `JPY` · 🇨🇦 `CAD` · 🇮🇳 `INR`

The `source` and `target` fields of `get_exchange_rate` and `get_historical_rates` are validated as exactly three characters, so pass ISO 4217 codes, not currency names.

## ⚙️ Environment variables

| Variable | Default | Required | Purpose |
|---|---|---|---|
| `ALLRATES_API_KEY` | — | **yes** | Your API key. The server exits with code 1 at startup if unset; sent as a `Bearer` token in the `Authorization` header. |
| `ALLRATES_BASE_URL` | `https://allratestoday.com/api` | no | Override for a self-hosted or staging deployment. Trailing slashes are stripped. |

Set these in your MCP client's config (in the `env` block), not in your shell — MCP servers are launched as subprocesses with isolated environments.

## 🛡️ Error handling

Tool failures come back as an MCP tool result with `isError: true`. The text is `AllRatesToday error (<status>): <message>`, where `<message>` is the `error` field from the API response body when present, and `HTTP <status>` otherwise.

| HTTP status | Meaning |
|---|---|
| 400 | Bad request — usually an unknown or malformed currency code |
| 401 | Invalid or missing API key |
| 429 | Rate limit or quota exceeded |
| 5xx | Server-side issue upstream |

Two errors are raised locally, before any HTTP call:

- No API key at request time → `API key is required. Get one at https://allratestoday.com/register, then set ALLRATES_API_KEY in your MCP config.`
- Unrecognised tool name → `Unknown tool: <name>`

Because these arrive as text, the assistant relays them to the user — a 429 surfaces as *"the API quota has been exceeded."*

## 🛠️ Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Client shows "MCP server failed to start" or a red dot | `ALLRATES_API_KEY` not set | Add the key to the `env` block in your client config |
| Every call returns a 401 error | Key malformed, truncated, or revoked | Copy a fresh key from the dashboard |
| Calls return a 429 error | Plan request limit hit | Wait for the quota to reset or upgrade the plan |
| `get_historical_rates` returns a 400 error | Invalid period or unknown currency code | `period` must be `1d`/`7d`/`30d`/`1y`; codes must be exactly 3 letters |
| Server starts but tools never appear | Client did not reload after the config change | Fully quit (not just close) and reopen the client |
| `npx` runs but hangs forever | Normal — the server is waiting for an MCP client on stdio | Let your MCP client launch it |

To inspect what the server is doing, run it manually with the key set:

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

**Does it work with other clients?** Any MCP-compatible host with stdio transport works; the four clients above are simply the ones with documented config paths here.

## 👩‍💻 Development

```bash
git clone https://github.com/cahthuranag/realtime-exchange-rate-mcp.git
cd realtime-exchange-rate-mcp
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
└── client.ts     # HTTP client for the AllRatesToday API + error mapping
dist/             # Compiled JS (gitignored)
server.json       # MCP registry manifest
smithery.yaml     # Smithery launch config
.mcp.json         # Ready-to-copy client config
```

Issues and PRs are welcome. Before opening a PR: `npm run build` must succeed, exercise the change against a real API key, and update both the tool descriptions in `src/index.ts` and the API reference above if tool behaviour changes.

## 📝 Changelog

See [GitHub Releases](https://github.com/cahthuranag/realtime-exchange-rate-mcp/releases) for the full list. Recent highlights:

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
- **Bug reports:** [github.com/cahthuranag/realtime-exchange-rate-mcp/issues](https://github.com/cahthuranag/realtime-exchange-rate-mcp/issues)

## 📜 License

MIT — see [LICENSE](./LICENSE).
