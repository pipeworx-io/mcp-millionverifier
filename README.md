# @pipeworx/millionverifier

Verify a single email address via MillionVerifier (millionverifier.com).
Part of `enrich-waterfall`'s `enrich_verify_email` ordered verifiers (fleet
#2860).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `millionverifier_verify(email, timeout?, _apiKey)` — result
  (ok/catch_all/disposable/invalid/unknown), explicit `catch_all` boolean,
  quality, free/role flags.

## Auth

BYO only. Pass your MillionVerifier API key as `_apiKey` (sent as the `api`
QUERY PARAM on a GET request). Find your key at
https://app.millionverifier.com/api. No platform key (D4).

## Data sources

- `https://api.millionverifier.com/api/v3/` — single email verify.

## Verification notes (fleet #2860, 2026-10-08, no vendor key held)

- **TRAP, confirmed live by edge probe: MillionVerifier returns HTTP 200 on
  an auth failure.** Invalid key:
  ```
  GET https://api.millionverifier.com/api/v3/?api=invalidkey123&email=test@example.com
  -> HTTP 200
  {"email":"test@example.com","result":"error","resultcode":4,"subresult":"",
   "free":false,"role":false,"didyoumean":"","credits":0,"executiontime":0,
   "error":"Apikey not found","livemode":true}
  ```
  A `res.ok` check alone would not catch this — the pack checks
  `result === 'error'` (or a present `error` string) and throws, rather than
  handing back `result: "error"` as though it were a verdict about the
  email address. This is exactly the silent-200 shape the house rules flag:
  a 200 with a payload that LOOKS like a normal result but means "the call
  itself failed."
- **Response field names and `catch_all` representation confirmed from
  vendor docs** (https://developer.millionverifier.com/, read 2026-10-08):
  `result` includes `catch_all` as a first-class value (not a separate
  flag on top of "ok") — this pack derives `catch_all: result === 'catch_all'`
  so the caller still gets a stable boolean field.
- **Not verified (no key held):** the full enumeration of `result` values
  beyond what the docs and the error probe showed (`ok`, `catch_all`,
  `unverified`, `invalid`, `unknown`, `disposable`, `error` were all that
  surfaced); exact `resultcode` -> `result` mapping.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "millionverifier": {
      "url": "https://gateway.pipeworx.io/millionverifier/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/millionverifier/mcp` returns the tools in the table
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

This pack takes your own API key (`_apiKey`) — we don't front one for it, so there's no curl here that would run without it. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/millionverifier_verify`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-millionverifier` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/millionverifier/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "millionverifier": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-millionverifier"]
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
ask_pipeworx({ question: "your question about Millionverifier data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
