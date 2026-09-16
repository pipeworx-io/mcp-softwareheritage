# @pipeworx/softwareheritage

The Software Heritage archive of public source code: which repositories have
been captured, when each was last captured successfully, and the commits inside
them — addressed by content, so they keep resolving after a repository is
renamed, moved or deleted.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `origin_search(query, limit?, with_visit?)` — find archived origins whose URL
  matches a substring. Answers "is this project in the archive, and under which
  URL", which you need before anything else because the archive keys everything
  on the origin URL.
- `origin_status(url)` — the most recent capture of one repository that actually
  produced a snapshot: date, snapshot id, visit type. Answers "is this code
  safe, and as of when".
- `origin_visits(url, limit?)` — the capture history for one repository, newest
  first, with each visit's outcome. Answers "how continuously has this been
  archived", and separates never-archived from archived-for-years-and-unreachable-today.
- `revision(id)` — one archived commit by its 40-hex id or SWHID: message,
  author, committer, dates, tree id, parents. Answers "what was in this commit"
  for a repository that may no longer exist anywhere else.

## Auth

Keyless. Anonymous callers get a lower rate-limit bucket than token holders, and
the limit is per-IP, so a Worker sharing an egress address with other traffic
can be throttled by calls that were not ours. The archive asks callers to
identify themselves; this pack sends its own User-Agent.

## Data sources

- <https://archive.softwareheritage.org/api/1/origin/search/{query}/> — origin
  search. Returns a bare JSON array, not an object.
- <https://archive.softwareheritage.org/api/1/origin/{url}/visit/latest/> —
  latest visit for one origin.
- <https://archive.softwareheritage.org/api/1/origin/{url}/visits/> — visit
  history, paginated with `per_page`.
- <https://archive.softwareheritage.org/api/1/revision/{sha}/> — one commit.

Things worth knowing before you touch this:

- **`visit/latest` without `require_snapshot=true` is a trap.** It returns the
  latest visit *whatever its outcome*, so a repository archived continuously
  since 2015 answers `status: "not_found"` because today's crawl happened to
  fail — the exact opposite of the question "is this archived". Reproduced on
  `https://github.com/torvalds/linux` on 2026-09-03: visit 479 was `not_found`
  while visit 478, the day before, was `full`. `origin_status` always sets the
  flag; if you add an endpoint here, set it too.
- **The origin URL is the key, and it is matched exactly.** A trailing slash, a
  `.git` suffix or `http` vs `https` makes it a different origin with its own
  visit history. Resolve with `origin_search` rather than constructing the URL.
- **The origin URL is embedded in the path, unescaped.** `/origin/https://…/get/`
  is the documented shape — do not URL-encode it, the API will not find it.
- **Origin endpoints return a bare JSON array**, revision endpoints return an
  object. Anything reading these has to handle both.
- **A 404 means "not archived", not "bad request".** This pack turns it into an
  explicit `archived: false` / `error: "revision not archived"` so an absence
  cannot arrive looking like an empty success.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "softwareheritage": {
      "url": "https://gateway.pipeworx.io/softwareheritage/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/softwareheritage/mcp` returns the tools in the table
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

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "softwareheritage": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-softwareheritage"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-softwareheritage
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Softwareheritage data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
