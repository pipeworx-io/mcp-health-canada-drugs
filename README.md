# Health Canada Drugs — the Canadian Drug Product Database

Every drug authorised for sale in **Canada**: DIN, brand name, active ingredients with strengths, dosage form, route, market status and the company behind it.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1573+ live data sources.

## Scope — Canada, and only Canada

This is Canadian marketing authorisation. US approvals and labels are `openfda` and `dailymed`; the two are separate regulatory acts and a drug can be authorised in one and not the other. Tools are prefixed `health_canada_*` so they never win a generic drug question.

## Tools

| Tool | Answers |
|---|---|
| `health_canada_search_drugs` | *Is Tylenol approved in Canada?* — brand-name search, returns DINs |
| `health_canada_drug_detail` | *What is DIN 01933655?* — ingredients, strengths, form, route, status |
| `health_canada_drugs_by_ingredient` | *Which Canadian drugs contain acetaminophen?* |

## Auth

Keyless. No registration.

## Data source

- <https://health-products.canada.ca/api/drug/> — Health Canada Drug Product Database.

### Things the next person would otherwise rediscover

- **An unrecognised filter is silently ignored and returns the ENTIRE 15 MB database**, with HTTP 200 and the drug you asked about somewhere inside it. The query parameter is `brandname`; the JSON field is `brand_name`. Measured: `?brandname=ACTIFED` → 3,576 bytes; `?brand_name=ACTIFED` → 15,040,036 bytes; `?search=ACTIFED` → 15,040,036 bytes. Matching the response field name is the natural guess and it is the wrong one. This pack whitelists the working parameters and throws rather than send an unknown one.
- **`id=` means different things on different endpoints.** On `/drugproduct/`, `/activeingredient/`, `/status/`, `/form/` and `/route/` it is the **drug** code. On `/company/` it is the **company** code. Drug `11685` is ACTIFED PLUS CAPLET (Glaxo Wellcome); company `11685` is DERMAL DEFENSE, INC. of Michigan. Passing a drug code to `/company/` returns a real record for an unrelated firm with nothing to signal the switch — so the company is read from the drug's own record and `/company/` is never called with a drug code here.
- **The upstream returns a bare object for a single hit and an array otherwise.** Code that assumes an array drops every single-result lookup.
- **DIN leading zeros are significant** — `01933655` is not `1933655`.
- **No working company-name filter.** Both `companyname` and `company_name` return the full 1.4 MB company list, so company search is deliberately not offered rather than shipped as a 1.4 MB-per-call tool.
- **A "Cancelled Post Market" status is not a safety withdrawal** — Health Canada retains records for products no longer marketed, for any reason.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "health-canada-drugs": {
      "url": "https://gateway.pipeworx.io/health-canada-drugs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/health-canada-drugs/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1573+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "health-canada-drugs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-health-canada-drugs"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-health-canada-drugs
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Health Canada Drugs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
