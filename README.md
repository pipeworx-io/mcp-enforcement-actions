# @pipeworx/enforcement-actions

US federal enforcement actions from primary sources — DOJ prosecutions and
settlements, SEC litigation releases, administrative proceedings and trading
suspensions. The branded entry point for "what has the government brought
against COMPANY".

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `enforcement_search({ query, limit?, since? })` — search DOJ press releases
  (~270,000 back to 2009) by company, person or topic. Newest first, dated,
  linked, with the DOJ components involved.
- `enforcement_recent({ source?, limit? })` — recent actions across DOJ and SEC,
  merged newest-first. `source` = `all` | `doj` | `sec`.
- `sec_enforcement_actions({ type?, limit? })` — SEC by type: `litigation`
  (civil actions in federal court), `administrative` (before an ALJ or the
  Commission), `trading_suspension`.

## Auth

Keyless. Both upstreams are public.

## Data sources

- <https://www.justice.gov/api/v1/press_releases.json> — DOJ press releases.
- <https://www.sec.gov/enforcement-litigation/litigation-releases/rss> — and the
  sibling `administrative-proceedings/rss` and `trading-suspensions/rss` feeds.

### Things worth not rediscovering

**The DOJ API silently ignores most of its plausible parameters.** Measured
2026-08-05 against the live API:

| parameter | behaviour |
|---|---|
| `title=<term>` | **works** — substring, case-insensitive. The only text filter. |
| `sort=date&direction=DESC` | **works**, and is required — default order is oldest-first |
| `q=` `keys=` `search=` | **ignored** — returns the full ~270k set, HTTP 200 |
| `component=` `topic=` `date_gte=` | **ignored** — same |

An ignored filter returning 200 with every record looks like "nothing was
excluded", not like "your filter did nothing". None of the ignored parameters
are exposed by this pack; `since` is applied client-side after fetching, and
says so in its description.

**Search matches the TITLE only.** A company named in the body of a release but
not in the headline will not be found. `enforcement_search` says so in its
`finding` when a query returns nothing, and suggests the short form of the name.

**One action, many announcements.** DOJ routinely publishes the same case from
Public Affairs *and* the relevant US Attorney's office — "Binance and CEO Plead
Guilty" returns twice. Results are collapsed on title+date, with the extra URLs
kept under `also_announced_by`.

**A few DOJ records have malformed dates** (raw HTML in the `date` field), so
date parsing is defensive and returns `null` rather than a garbage timestamp.

**SEC feeds are a recent window, not an archive** (~25 items each) and are **not
searchable**. Any "has the SEC ever sued X" question is out of scope for this
pack; only DOJ has history here.

## Not included in v1

**FTC.** No structured feed survives. Probed live 2026-08-05, all failing:

| endpoint | result |
|---|---|
| `/api/v0/cases`, `/api/v0/press-releases` | 404 |
| `/feeds/press-releases.xml`, `/feeds/enforcement-actions.xml` | 404 |
| `/feeds/press-release-enforcement.xml` | 503 |
| `/rss.xml`, `/news-events/rss`, `/stories/rss.xml` | 404 |
| `/news-events/news/press-releases/feed` | 200 but returns HTML, not RSS |

Including FTC would mean scraping its Drupal listing pages, which is a separate
decision rather than a keyless-first integration.

**State attorneys general.** No common format; each state publishes its own
press page. Would be one pack per jurisdiction under the standing rule, not a
multiplexed `state` argument.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "enforcement-actions": {
      "url": "https://gateway.pipeworx.io/enforcement-actions/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/enforcement-actions/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/enforcement_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"Binance"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/enforcement_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "enforcement-actions": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-enforcement-actions"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-enforcement-actions
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Enforcement Actions data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
