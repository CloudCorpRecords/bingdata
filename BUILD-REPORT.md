# BUILD REPORT — bingdata

Rebuild of `com.hasdata/bing` v1.0.3 (remote-only, keyless capture)
as `@reneturcios/bingdata` 0.1.0 / `io.github.CloudCorpRecords/bingdata`.

## Fixture conflicts

**None.** The brief, the spec prose, and the fixture agree on the tool
name and its required field. Required field as served, taken from
the fixture (`harness/fixtures/hasdata-bing-tools-original.json`):

- `hasdata_bing_serp_getSearchResults` — required: `q`

The fixture's tool literal (name, title, description, inputSchema —
including the `title` key, every property description and enum, the
trailing newlines in the `filters` and `first` property descriptions,
`additionalProperties: false`, and the `$schema` key) is served
verbatim from `src/forward.ts`; the literal was generated
programmatically from the fixture JSON (parsed and re-serialised, not
retyped) and verified by the registration-integrity test (served
tools/list deep-equals the fixture's 1 tool object) and again by the
parity harness.

## What was built

All 19 factory files under `builds/bingdata/`:

- `package.json` (`@reneturcios/bingdata` 0.1.0, bin
  `bingdata` → `dist/cli.js`, deps exactly `@modelcontextprotocol/sdk ^1`
  + `zod ^3`; devDeps `@types/node ^20`, `tsx ^4`, `typescript ^5`)
- `server.json` (description exactly the prescribed 98-char string
  "Bing search results as JSON: SERPs with the ad block and Copilot answer, geo and market targeting.";
  npm entry carries `"version": "0.1.0"`; OCI entry
  `ghcr.io/cloudcorprecords/bingdata:0.1.0` carries **no** version field)
- `Dockerfile` (node:20-alpine both stages; server-name label
  `io.modelcontextprotocol.server.name="io.github.CloudCorpRecords/bingdata"`
  matches `server.json`)
- `tsconfig.json` (fleet-identical, 318 bytes), `.gitignore`,
  `package-lock.json`, `README.md` (fresh prose), `LICENSE` (MIT,
  Copyright (c) 2026 Rene Turcios), `CHANGELOG.md` (single 0.1.0 entry,
  2026-10-28)
- `src/index.ts`, `src/cli.ts`, `src/mcp.ts`, `src/types.ts`,
  `src/forward.ts` (core; exports `VERSION = "0.1.0"`)
- `test/forward.test.ts`
- `.github/workflows/{ci.yml, publish.yml, publish-container.yml,
  registry-publish.yml}` (container workflow: IMAGE
  `ghcr.io/cloudcorprecords/bingdata`, smoke assertion on server name
  `bingdata`)

Plus `BUILD-REPORT.md` (this file) and the `parity-out/` harness output.

## Design

- Forwarding client to `https://mcp.hasdata.com/mcp?apis=bing`
  (pinned; the query string is part of the pinned URL), overridable via
  `HASDATA_BING_MCP_URL` (labelled reconstruction). tools/call is
  forwarded as a streamable-HTTP JSON-RPC client: initialize
  (protocolVersion `2025-03-26`, clientInfo `bingdata`) →
  notifications/initialized → tools/call, honouring `Mcp-Session-Id`
  when issued; session established lazily and reused per client.
  Plain-JSON and SSE replies both parsed; the matching-id message
  wins; upstream results pass through verbatim (including `isError`
  and extra fields such as `structuredContent`).
- When `HASDATA_API_KEY` is set, an `x-api-key` header is
  sent on every upstream request (header name pinned by the registry
  record's remote declaration; forwarding it from this env var is a
  labelled reconstruction — the capture was keyless). When unset, no
  key header is sent. No `Authorization` header is sent in either
  case. Both branches are covered by tests.
- Upstream failures (JSON-RPC error, non-2xx status, malformed body,
  throwing/timing-out fetch) surface as `bingdata error:` text
  results (labelled reconstruction).
- Registration via the MCP SDK **low-level** `Server` API
  (`ListToolsRequestSchema` / `CallToolRequestSchema` handlers) serving
  the fixture literal exactly; no `McpServer` + zod registration
  (per the task brief, overriding TEMPLATE.md §2 item 12).
  `src/mcp.ts` also exports `createServer` / `startMcpServer` /
  `handleCallTool` so the MCP layer is testable without a transport;
  the CLI's `mcp` subcommand starts it over stdio.
- Core takes an injectable fetch (`ForwardDeps.fetchFn`), so all tests
  run against a stubbed JSON-RPC upstream. No live network calls in
  tests; no live call claimed anywhere. The captured surface is one
  read-only SERP fetch; no write, messaging, or payment tool exists
  in the surface — none was added.

## Lockfile (Route 3, clean from birth)

Generated in a temp dir (`/tmp/bingdata-lock`) containing only the
new `package.json`, with no `node_modules` present:
`npm install --package-lock-only --ignore-scripts --offline`.
Verified: parses, `name == @reneturcios/bingdata`,
**128 package entries**; placed into the build dir with matching
sha256 (`96997033…aec` on both sides: full value
`969970335f1ae34d279be91ed5a1e5e30befd5cf4eab14df2a5830576d4b7aec`).
Then `npm install --offline --ignore-scripts` in the build dir for
`node_modules`. No write-guard failure occurred; `dist/` was built
in-tree normally.

## Verification (all in the build dir unless noted)

- `npx tsc --noEmit` → exit 0
- `npm run build` → real `dist/` (cli.js 2285 B, forward.js 16192 B,
  mcp.js 2668 B, all non-empty, with `.d.ts` + source maps)
- `node dist/cli.js --version` → `0.1.0`; unknown subcommand → usage on
  stderr, exit 2
- `npm test` → **51 tests, 51 pass, 0 fail** (8 suites: registration
  integrity — served tools deep-equal the fixture's full 1 tool
  object, name in order, exactly the fixture's keys, title key,
  required field, every property key in order, market/safesearch/
  device enums, additionalProperties/$schema, isKnownTool; session
  establishment/reuse/session-id handling, clientInfo/protocol
  version; env URL override, explicit url override, explicit-over-env
  precedence, blank-env fallback; x-api-key header present on every
  request when the key is set / absent when unset / no Authorization
  header in either case; verbatim argument forwarding incl. lat/lon
  geo targeting and a q-only search; verbatim result pass-through
  incl. upstream isError and SSE; JSON-RPC/init/malformed/no-result/
  HTTP/throw/timeout failures → `bingdata error:`; parseRpcBody JSON
  + SSE + matching-id + [DONE] cases; MCP-layer MethodNotFound /
  InvalidParams validation; CLI --version / -v / unknown exit 2 /
  bad-JSON exit 2 / `mcp extra` exit 2)
- Parity: `python3 harness/parity.py --original
  harness/fixtures/hasdata-bing-tools-original.json --rebuild
  "stdio:node builds/bingdata/dist/cli.js mcp" --out
  builds/bingdata/parity-out` (run from `~/workspace/mcp-rebuild-100`)
  → **PASS, parity 100.0, 1/1 identical, no missing/extra/schema
  diffs** (see `parity-out/parity.json`)
- Consistency sweep: package.json version = server.json top-level =
  npm-entry version = OCI tag = core `VERSION` = CHANGELOG heading
  (0.1.0); package.json `mcpName` = server.json `name` = Dockerfile
  label; server.json description measured at 98 chars; OCI entry
  confirmed to have no `version` field.

Clean-room: all prose and code written fresh for bingdata; only the
mechanics were patterned on the sibling staydata build, and only the
fixture's tool literal is served verbatim.
