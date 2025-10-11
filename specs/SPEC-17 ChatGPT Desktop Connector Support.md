---
title: 'SPEC-17: ChatGPT Desktop Connector Support'
type: spec
permalink: specs/spec-17-chatgpt-desktop-connector-support
tags:
- mcp
- chatgpt
- integration
- development
---

# SPEC-17: ChatGPT Desktop Connector Support

## Why
Developers want to use Basic Memory with ChatGPT Desktop via MCP. Running the server locally with `basic-memory mcp --transport streamable-http --host 0.0.0.0 --port 8000 --path /mcp` works, but registering the connector in ChatGPT fails with “Unsafe URL” for `http://localhost:8000/mcp`. ChatGPT’s connector validator strongly prefers HTTPS (trusted cert), public or tunnel-accessible endpoints, and header-based auth (no query-string secrets). We need a clear, repeatable, documented integration path and (optionally) first-class CLI support for TLS so contributors can connect ChatGPT reliably in development and production.

## What
Provide a supported, documented way to connect ChatGPT Desktop to the Basic Memory MCP server:
- A recommended setup that passes ChatGPT safety checks (HTTPS, no query-string auth; optional discovery if/when needed).
- Clear local-dev instructions using a trusted HTTPS tunnel or local TLS reverse proxy.
- Optional CLI improvement: TLS passthrough flags for `basic-memory mcp` to serve HTTPS directly (if supported by FastMCP/Uvicorn).
- Verify our ChatGPT-oriented MCP tools expose the expected names/structures (`search`, `fetch`) and document their usage.
- Update docs so users can connect quickly and avoid “Unsafe URL”.

Scope includes documentation, optional CLI enhancements, and light tests around ChatGPT-specific tool adapters. No changes to core knowledge graph, storage, or sync are required.

## How (High Level)
1. Document two supported connection paths:
   - HTTPS tunnel (Cloudflare Tunnel or ngrok) that forwards `https://<domain>/mcp → http://127.0.0.1:8000/mcp`.
   - Local reverse proxy with trusted certificates (Caddy + mkcert) that serves `https://memory.local/mcp` to local server.
2. Ensure MCP server options and defaults are documented (`streamable-http`, `--project`, host/port/path). Consider adding TLS flags if FastMCP exposes them.
3. Validate ChatGPT adapter tools in `src/basic_memory/mcp/tools/chatgpt_tools.py`:
   - `search(query)` returns a content array with a single `{ "type": "text", "text": "{...JSON...}" }` item and the JSON includes `results` and `total_count`.
   - `fetch(id)` returns a similarly structured content array with document fields (`id`, `title`, `text`, `url`, `metadata`).
4. Add tests for adapters to guard JSON structure and error handling.
5. Update docs (README or docs page) with concise ChatGPT Desktop setup instructions and troubleshooting.

## Implementation Details

### A. Server Startup (existing)
Run the server via the built-in CLI (already implemented):

```
basic-memory mcp --transport streamable-http --host 0.0.0.0 --port 8000 --path /mcp
```

Notes:
- Use `--project <name>` to constrain access during development for safety.
- Windows users can run the same command in PowerShell.

### B. Recommended HTTPS options (development)

Option B1 — Cloudflare Tunnel (preferred for simplicity):
- Install `cloudflared`.
- Run: `cloudflared tunnel --url http://127.0.0.1:8000`
- Cloudflared prints a public `https://<random>.trycloudflare.com` URL. Use `https://.../mcp` in ChatGPT.

Option B2 — ngrok:
- `ngrok http http://127.0.0.1:8000`
- Use the provided `https://.../mcp` URL in ChatGPT.

Option B3 — Local TLS reverse proxy (Caddy + mkcert):
- Install mkcert and trust local CA.
- Configure a hosts entry (e.g., `memory.local`) → `127.0.0.1`.
- Caddyfile example:

```
memory.local {
  tls internal
  reverse_proxy 127.0.0.1:8000 {
    header_up Host {host}
  }
}
```

- Access via `https://memory.local/mcp` in ChatGPT.

Rationale: ChatGPT often rejects `http://localhost` and plain HTTP as “Unsafe URL”. HTTPS with a trusted certificate works reliably.

### C. Optional: TLS passthrough in CLI
If `FastMCP.run()`/underlying Uvicorn supports TLS, add flags to `basic-memory mcp` and forward them:
- `--ssl-certfile <path>`
- `--ssl-keyfile <path>`
- `--ssl-ca-certs <path>` (optional)

This is optional because tunnels/reverse proxies suffice for most users, but native TLS improves ergonomics.

### D. ChatGPT tool adapters (existing)
File: `src/basic_memory/mcp/tools/chatgpt_tools.py`
- `search(query)` returns a content array with a single `{"type": "text", "text": "{...JSON...}"}` entry, JSON body: `{results, total_count, query}`.
- `fetch(id)` returns a content array with a single `{"type": "text", "text": "{...JSON...}"}` entry, JSON body: `{id, title, text, url, metadata}`.
- Both log requests and handle errors gracefully (empty results or serialized error payloads instead of exceptions).

### E. Documentation updates
Add a “ChatGPT Desktop (MCP) Setup” section covering:
- Server startup (`basic-memory mcp ...`).
- Choosing HTTPS path (tunnel or local TLS proxy) and why plain HTTP/localhost may be rejected.
- Connector URL examples (`https://<domain>/mcp`).
- Auth guidance: Prefer header-based auth (no query string tokens). For now, keep endpoints open locally; document future auth if needed.
- Troubleshooting: “Unsafe URL”, certificate trust, path forwarding, proxy rewrites.

## How to Evaluate

### Success Criteria
- Users can add a Basic Memory MCP connector in ChatGPT Desktop using an HTTPS URL without “Unsafe URL” errors.
- ChatGPT can list tools and successfully invoke `search` and `fetch`.
- Documentation is sufficient for a new contributor to set this up in under 10 minutes.

### Testing Procedure
1. Unit tests
   - Add `tests/mcp/test_chatgpt_tools.py` to validate `search()` and `fetch()` return arrays with a single `{type: "text", text: "..."}` item and a valid JSON body containing expected keys.
   - Error-path tests: malformed query, unknown id.
2. Manual validation with ChatGPT Desktop
   - Start server locally.
   - Create a tunnel (`cloudflared tunnel --url http://127.0.0.1:8000`).
   - Register connector in ChatGPT with the HTTPS URL ending in `/mcp`.
   - Use ChatGPT to run `search` with a query; then `fetch` a returned id.
3. Docs check
   - Follow the new documentation on a clean machine and confirm setup time (< 10 minutes).

### Metrics
- Time-to-setup (median across team).
- Error rate when adding connector (reports of “Unsafe URL”).
- Adapter test pass rate; regression count after changes.

## Living Checklist
- [ ] Document HTTPS tunnel setup (Cloudflare/Ngrok) with step-by-step instructions
- [ ] Document local TLS reverse proxy (Caddy + mkcert) example
- [ ] Add optional CLI TLS flags if FastMCP supports it (certfile/keyfile)
- [ ] Add tests: `tests/mcp/test_chatgpt_tools.py` (search + fetch, success and error cases)
- [ ] Add “ChatGPT Desktop (MCP) Setup” section to README or dedicated doc page
- [ ] Verify Windows-specific guidance (PowerShell, mkcert install, hosts entry)
- [ ] Validate against the latest ChatGPT Desktop build

## Risks and Considerations
- Certificate trust on Windows/macOS: mitigate with mkcert or tunnels.
- Proxy rewrites must not alter `/mcp` path (no extra prefixes); document exact forwarding.
- Auth: keep local dev open; if auth is added later, use headers/Bearer tokens, not query params.
- Availability: tunnels can be rate-limited on free plans; note alternatives.

## Relations
- spec [[SPEC-1 Specification-Driven Development Process]]
- spec [[SPEC-16 MCP Cloud Service Consolidation]]
- docs: `src/basic_memory/cli/commands/mcp.py`, `src/basic_memory/mcp/tools/chatgpt_tools.py`

