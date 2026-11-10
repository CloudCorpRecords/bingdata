# bingdata

bingdata is a small local bridge to the hosted HasData
Bing MCP server, which returns Bing search results as
JSON. Give it a query, and it fetches that query's Bing
SERP: the organic results with their titles, links,
snippets and positions, the related searches, and —
as returned upstream — the ad block and the Copilot
answer. Searches can be localised by free-text place
or by latitude/longitude, and shaped by market,
country, interface language, safesearch level, time
filters, device type and page offset.

The bridge stores no results of its own. It serves the
upstream server's tool list locally, exactly as
captured from the live endpoint, and forwards each
tool call upstream, returning the result untouched.
The captured surface is a single read-only fetch:
there is no write, messaging, or payment tool in it.

## Tools

- `hasdata_bing_serp_getSearchResults` — fetch one
  Bing SERP. Required: `q` (the search term). Optional:
  `location` (free-text origin, resolved to coordinates;
  ignored when both are given, `lat` and `lon` take
  precedence), `mkt` (market), `cc` (country),
  `setLang` (interface language), `safeSearch`
  (`off` / `moderate` / `strict`), `filters` (Bing
  filter expressions, including the past-day/week/month
  date filters), `deviceType` (`desktop` / `mobile` /
  `tablet`), and `first` (result offset for paging:
  1 is the first page, 11 the second, 21 the third).
  Returns organic results (title, url, snippet,
  displayed url, position) and related searches.

## Configuration

- `HASDATA_BING_MCP_URL` (optional) — override the
  upstream endpoint. Defaults to
  `https://mcp.hasdata.com/mcp?apis=bing`. (The
  override hook is this rebuild's own labelled
  reconstruction.)
- `HASDATA_API_KEY` (optional) — when set, every
  upstream request carries an `x-api-key: <key>`
  header, for the caller's own HasData API key. When
  it is not set, no key header is sent. (The header
  name and the env name are pinned by the original's
  registry record; forwarding the key from this env
  var is a labelled reconstruction — the original
  surface was captured keyless.)

### Forwarding mechanics

On the first tool call the client opens a streamable
HTTP session with the upstream: a JSON-RPC
`initialize` request (protocolVersion `2025-03-26`,
clientInfo name `bingdata`), then a
`notifications/initialized` notification, then the
`tools/call` request carrying the caller's arguments
unchanged. When the upstream answers `initialize`
with an `Mcp-Session-Id` header, that value rides on
every later request of the session, which is reused
for the life of the process. Replies may arrive as
plain JSON or as server-sent events; both are read.

Successful upstream results pass through verbatim,
including any `isError` flag the upstream sets.
Failures — a JSON-RPC error, a non-2xx status, an
unreadable response, or a request that cannot be
completed — come back as an `isError` result whose
text starts with `bingdata error:` (this rendering
is this rebuild's own labelled reconstruction).

## CLI

The tool is also a subcommand taking one JSON
argument string:

```bash
npx @reneturcios/bingdata hasdata_bing_serp_getSearchResults '{"q":"best coffee in san francisco","location":"San Francisco, California","mkt":"en-us","safeSearch":"moderate"}'
npx @reneturcios/bingdata --version
```

Exit codes: `0` on success, `1` on an `isError`
result, `2` on a usage error.

## MCP server

```bash
npx -y @reneturcios/bingdata mcp
```

MCP client config:

```json
{
  "mcpServers": {
    "bingdata": {
      "command": "npx",
      "args": ["-y", "@reneturcios/bingdata", "mcp"]
    }
  }
}
```

## Docker

```bash
docker run -i ghcr.io/cloudcorprecords/bingdata
docker run --rm ghcr.io/cloudcorprecords/bingdata --version
```

The container's default command is `mcp`, so the first
form is a stdio MCP server; any other arguments reach
the CLI normally.

## Development

```bash
npm install
npm run build
npm test
npm run dev
```

The test suite runs entirely against a stubbed
JSON-RPC upstream — no live network calls.

## License

MIT — see `LICENSE`.
