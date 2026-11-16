# @pipeworx/ca-fqhc-wrap-payments

California Federally Qualified Health Center (FQHC) wrap payment COUNTS and managed care encounter match figures, by health center and calendar year (2021-2023). These are counts and a match percentage. They are not dollar amounts and they are not payment rates.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2006+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ca_fqhc_wrap_lookup(name?, npi?, calendar_year?, limit?)` -- per-center figures across years for a health center found by name (substring) or NPI (10-digit string). An unmatched name returns near matches with their NPIs.
- `ca_fqhc_wrap_year_summary(calendar_year?, rank_by?, top_n?, min_wrap_payments?)` -- the publisher's statewide total row, a recomputed sum over centers, counts of suppressed and unreported centers, and the top centers by wrap payment count or by lowest match percentage.

Fields, in the publisher's words:

- Wrap Payment: "The count of Wrap Payments submitted by the FQHC for the Calendar Year."
- Wrap Encounters: "The count of Managed Care encounters submitted by the Managed Care Plan."
- Matches: "The total count of Wrap Payments and Wrap Encounters that were found to match."
- Match Percentage: "Divides the Matches column by the total Wrap Payments submitted for the FQHC for the Calendar year."
- Annotation: 1 = suppressed due to small number, 2 = suppressed due to complementary small numbers, 3 = no data available. A published "*" is a suppressed cell and is returned as null.

Context from the publisher: a wrap payment is a supplemental payment to a Federally Qualified Health Center covering the difference between its per-visit PPS rate and the amount received from managed care plans for services to Medi-Cal managed care members. The dataset counts wrap payments and the encounters they match; it does not publish what they were worth in dollars.

## Auth

Keyless.

## Data sources

- California Department of Health Care Services (DHCS), Federally Qualified Health Center Wrap Payment and Managed Care Encounter dataset, calendar years 2021-2023, published on the California Health and Human Services open data portal.

This pack answers from a stored copy of the whole table (3,354 rows, about 780 KiB), not a per-call request. The publisher's portal refuses some non-US network locations, so a live call would fail only for callers landing there; a stored copy answers the same everywhere. A missing copy fails loudly with a server-side error; it never answers "not found".

`data_as_of` in every response is the date the publisher's dataset record was last modified, read when the copy was made. `captured_at` is when we read it. Neither is today's date.

Refresh: `node mcps/ca-fqhc-wrap-payments/scripts/bake-index.mjs` (node and network only; run it from a US machine) rewrites `src/ca-fqhc-wrap-data.ts` in place, then `node scripts/check-baked-indexes.mjs` and commit. It refuses to write on fewer than 1,000 rows, a changed column set, a count that is not a number, or a calendar year without exactly one statewide Total row. The publisher adds a year about annually (400-day freshness budget).

Things worth knowing: the publisher truncates long names ("Family Health Centers Of San") and one organisation can hold several NPIs, so a name can return several centers. The dataset includes a publisher-computed statewide "Total" row (NPI "All NPIs"), which the summary tool uses and the lookup tool excludes. Source counts are text with thousands separators and "*" for suppressed cells; the stored copy holds numbers, with suppressed cells as null and the publisher's annotation code kept.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ca-fqhc-wrap-payments": {
      "url": "https://gateway.pipeworx.io/ca-fqhc-wrap-payments/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ca-fqhc-wrap-payments/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2006+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ca_fqhc_wrap_lookup \
  -H 'Content-Type: application/json' \
  -d '{"name":"Asian Health","calendar_year":"2022"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ca_fqhc_wrap_lookup`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-ca-fqhc-wrap-payments` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/ca-fqhc-wrap-payments/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "ca-fqhc-wrap-payments": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ca-fqhc-wrap-payments"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ca Fqhc Wrap Payments data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
