# @pipeworx/oig-self-disclosure-settlements

HHS-OIG Self-Disclosure Protocol settlements — has this provider, grantee or
contractor voluntarily self-disclosed conduct to OIG and settled a Civil
Monetary Penalties Law case over it? Complements `oig-corporate-integrity`:
a Corporate Integrity Agreement is an ongoing compliance obligation, while
most Self-Disclosure Protocol settlements are a one-time payment resolving a
self-reported issue, often with no CIA attached at all.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `oig_sdp_recent(program?, limit?, offset?)` — the most recent settled
  self-disclosure cases, newest first. Returns totals first (matching count,
  totals by program, and the approximate dollar sum of the returned page)
  and then the settlements themselves: provider, date, headline amount,
  allegation and a detail-page link. `program` filters to `"provider"`
  (Provider Self-Disclosure Protocol, the large majority), `"grant"` (Grant
  and Contractor Self-Disclosure Program) or `"all"` (default).
- `oig_sdp_check(name, program?, limit?)` — has a specific provider,
  grantee or contractor settled a self-disclosure case? Matches a name
  against the baked settlement list (exact title, exact/substring provider
  name, or token match, each labelled); for the best match, fetches the
  settlement's own detail page **live** to return the exact settlement
  amount, the full entity description and the complete allegation
  narrative — the headline figure baked into the list is frequently
  **rounded** (e.g. a title reading "$442,000" against a detail page reading
  "$442,713.81"), so this is the call to make when the exact figure matters.
  Returns `found: false` with the list's as-of date and total settlement
  count when no match exists.

## Auth

Keyless.

## Data sources

- HHS Office of Inspector General, Enforcement Actions — Self-Disclosure
  Protocol settlements. Covers both the Provider Self-Disclosure Protocol
  (~1,057 settlements as captured) and the Grant and Contractor
  Self-Disclosure Program (~17 as captured). No JSON/REST endpoint and no
  bulk CSV/Excel export exist for this slice of OIG's enforcement record.
- Each settlement's own enforcement-action detail page — the provider's
  full legal name and location, the exact settlement amount, and the
  complete allegation narrative. Fetched live, per call, never baked.

### Why only the LIST is baked

Root CLAUDE.md's standing rule for this shape: a baked table never sits in
gateway module scope. `scripts/bake-index.mjs` crawls both programs' full
page ranges and bakes only the headline (which carries the provider name,
an approximate dollar figure, and the allegation in one sentence), the
detail-page path, the publish date and the program — into
`src/sdp-index-data.ts`, registered in
`workers/gateway/src/pack-baked-indexes.json`, uploaded to KV at deploy, and
injected as `args._bakedIndex` (fleet #2754 pattern, same as
`oig-corporate-integrity`). A missing or malformed injection throws loudly
rather than answering "no settlement found" — see the `ensureIndex` guard in
`src/index.ts`.

The exact settlement amount, the provider's full legal name/location, and
the full allegation narrative are **deliberately not baked** — they are
fetched live from that one settlement's own detail page on every
`oig_sdp_check` call. Most callers never ask about most of the ~1,074 rows,
so pre-crawling every detail page would be wasted work; fetching live also
means a newly-posted settlement's narrative is visible as soon as the list
is next re-baked, without a second crawl pass.

Re-run `node mcps/oig-self-disclosure-settlements/scripts/bake-index.mjs`
periodically to refresh the list and recommit. OIG's own listing carries no
static "last updated" banner — `data_as_of` on every response is simply the
bake script's own capture date.

### Narrative sentence shapes

Detail-page body paragraphs follow one of two templates, both handled by
`parseDetailPage` in `src/index.ts`:

- "After it self-disclosed conduct to OIG, **ENTITY, LOCATION**, entered
  into a **AMOUNT** settlement agreement with OIG. \[The settlement
  agreement] resolves allegations \[that] **ALLEGATIONS**."
- "After it self-disclosed conduct to OIG, **ENTITY, LOCATION**, agreed
  to pay **AMOUNT** for allegedly violating the Civil Monetary Penalties
  Law\[, including/provisions ...]. \[The] OIG alleged \[that] **ALLEGATIONS**."

Verified against a 60-row spread across the full page range (both
programs): 100% narrative-parse coverage. A settlement phrased outside both
templates (seen on a handful of grant-fraud cases resolved under the False
Claims Act rather than the CMP law) falls back to a loose dollar-amount
extraction so `settlement_amount_exact` still comes back populated even when
`entity_description`/`allegations` cannot be split out.

### Transport/parse failures throw loud

Same contract as `oig-corporate-integrity`: a non-200 from a detail page, or
a 200 whose content does not match the expected detail-page shape, is
reported as a thrown error naming the upstream status — never a silent
`found: false`, which for a compliance-screening tool would be
indistinguishable from "no settlement on file".

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "oig-self-disclosure-settlements": {
      "url": "https://gateway.pipeworx.io/oig-self-disclosure-settlements/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/oig-self-disclosure-settlements/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/oig_sdp_recent \
  -H 'Content-Type: application/json' \
  -d '{"program":"provider","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/oig_sdp_recent`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "oig-self-disclosure-settlements": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-oig-self-disclosure-settlements"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-oig-self-disclosure-settlements
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Oig Self Disclosure Settlements data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
