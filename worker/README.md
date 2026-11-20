# ynab-mcp-connector (worker/)

Cloudflare Worker that hosts the same MCP server as the repo root, remotely at
the configured origin at `/mcp`, with per-user
YNAB OAuth instead of a personal access token. Design blueprint:
[docs/hosted-oauth-connector.md](../docs/hosted-oauth-connector.md).
Not affiliated with YNAB.

This directory contains the Worker source maintained in
[drewmanole/ynab-mcp-server](https://github.com/drewmanole/ynab-mcp-server).
The original project and attribution remain at
[oliverames/ynab-mcp-server](https://github.com/oliverames/ynab-mcp-server).

## How it works

- [`@cloudflare/workers-oauth-provider`](https://github.com/cloudflare/workers-oauth-provider)
  is the OAuth 2.1 server MCP clients talk to (`/authorize`, `/token`, `/register`);
  grants live in KV with props encrypted and tokens stored hash-only.
- [`McpAgent`](https://developers.cloudflare.com/agents/) (SQLite Durable Object)
  serves `/mcp` (streamable HTTP) and `/sse` (legacy). Each session builds the
  full 58-domain-tool surface plus four discovery/executor helpers via
  `createYnabServer()` from [`../index.js`](../index.js),
  tools, prompts, resources, host pinning, and `confirmed:true` gates are the
  exact code the local stdio server runs. Every tool also returns the same
  app-client contract: human-readable title, input/output schemas, matching
  `structuredContent`, and private/bounded impact hints.
- The consent page redirects to YNAB's own OAuth with PKCE S256. Consent and
  callback state use 192-bit opaque values, keyed hashes, one-time KV records,
  and 10-minute TTLs without depending on browser cookies. Access tokens last 2
  hours; refresh happens automatically inside a 60-second safety window;
  refresh tokens rotate and are persisted per YNAB user in KV. Users choose
  read-only vs write access at consent (`read-only` scope vs no scope).
- YNAB tokens and undo journals are encrypted with AES-GCM before they enter KV.
  The encryption key is separate from the cookie-signing key.
- The undo journal is scoped to each YNAB user; `/delete` proves identity with a
  fresh YNAB sign-in, then purges tokens, journal entries, and every paginated
  connector grant.

## Deploy

```bash
cd worker
npm ci --include=optional
npx wrangler login --use-keyring
npx wrangler whoami
npx wrangler kv namespace create OAUTH_KV
```

Replace the example namespace ID in `wrangler.jsonc`, set `CONNECTOR_BASE_URL` to
the exact workers.dev origin, and create the YNAB OAuth application described
below. Save its credentials and two different random 32+ byte encryption keys
in a password manager.

For the initial deployment, put all four values in one temporary JSON or `.env`
file outside the repository, restrict it to the current user, and upload code
and secrets as a single Worker version:

```bash
chmod 600 /absolute/path/to/ynab-worker-secrets.env
npx wrangler deploy --secrets-file /absolute/path/to/ynab-worker-secrets.env --strict
```

The file must define `YNAB_CLIENT_ID`, `YNAB_CLIENT_SECRET`,
`COOKIE_ENCRYPTION_KEY`, and `DATA_ENCRYPTION_KEY`. Delete only that temporary
file after the deployment and production checks succeed. Do not use four
sequential `wrangler secret put` calls for the initial release: every call can
create a separately deployed Worker version with only part of the required
configuration. Losing `DATA_ENCRYPTION_KEY` makes existing encrypted token and
undo records unreadable.

Prerequisites: a YNAB OAuth application (YNAB → Account Settings → Developer
Settings) whose name is `MCP Server for YNAB`, website is the exact workers.dev
origin, privacy URL ends in `/privacy`, and redirect URI is exactly
`https://connector.example/callback`. Keep default
plan selection disabled. Wrangler must be authenticated against the Cloudflare
account whose workers.dev subdomain is used. Local dev: copy `.dev.vars.example`
to `.dev.vars` (redirect URI
`http://localhost:8787/callback` must also be registered on the YNAB app while
testing locally).

New YNAB OAuth apps run in
[Restricted Mode](https://api.ynab.com/#oauth-applications). The app owner is
exempt, but the application can obtain at most 25 access tokens for other users
before YNAB blocks new authorizations. Removing the limit requires YNAB's
review, which its current documentation says takes 2 to 4 weeks. This is a
rollout limit, not a cause of connector consent errors that happen before the
browser reaches YNAB.
Directory review and publication are not part of this personal deployment.

## Connector identity

The Worker serves YNAB's permitted, unmodified
["Works with YNAB" mark](https://api.ynab.com/papi/works_with_ynab.svg) at
`/assets/works-with-ynab.svg` and an equivalent 196x78 PNG rendering at
`/assets/works-with-ynab.png`. Hosted consent, callback, privacy, and deletion
pages use this mark and carry the required non-affiliation and trademark
language.

For connector discovery, MCP `initialize` advertises the 256px Blurple tree
first, followed by `/assets/icon.png`, which is generated from the exact
1024x1024 `codex/assets/icon.png` artwork displayed by the Codex plugin. The
landing page publishes the latter artwork as its Open Graph image. Conventional
ICO, 16px, 32px, 48px, 64px, 96px, 128px, 256px, and Apple touch icon routes
use the Blurple YNAB tree artwork from `assets/ynab-tree-blurple.svg`, downloaded
from YNAB's public Webflow CDN asset. The ICO includes a 256px frame so hosts that
derive custom-connector branding from the origin favicon can use a card-sized
asset. The Claude-facing 256px PNG is forced to 8-bit RGBA and served from the
versioned `/assets/ynab-tree-icon-v1.png` route so a previously rejected image
cannot remain pinned by the immutable asset cache. The generator, dimensions,
MIME types, and
SHA-256 digests are pinned in source and covered by the Worker tests.

YNAB's API Terms expressly permit the integration mark and the "for YNAB"
naming form. The square connector icon is documented as the repository's
existing Codex branding choice rather than as a Terms-permitted integration
asset.

Custom connectors may ignore MCP icon metadata or cached favicon changes. A
Connectors Directory listing has a separate icon field, but public submission is
not part of deployment and must be handled as its own review and publication
decision.

## Verify

```bash
npm test
npx wrangler deploy --dry-run
curl -I "$CONNECTOR_BASE_URL"/
curl -I "$CONNECTOR_BASE_URL"/assets/works-with-ynab.png
curl "$CONNECTOR_BASE_URL"/.well-known/oauth-protected-resource/mcp
curl "$CONNECTOR_BASE_URL"/.well-known/oauth-authorization-server
```

The unit suite is credential-free. Production acceptance passed on July 15,
2026, in ChatGPT, Claude.ai, and Mistral Vibe Work. Each host signed in through
YNAB, invoked the connector, returned the live budget list, and reported
`writes_enabled: true` for the explicitly write-authorized grant. No acceptance
chat performed a financial mutation; an actual write/undo exercise still
requires separate approval for the exact operation.

## Troubleshooting: the client shows read tools but no write tools

Write tools are gated **per connection, at consent time** — not by a server
setting you can change after the fact. If the `writes` box on the connector's
consent screen was left unchecked, YNAB issues a read-only token, the grant
scope is `["read"]`, and the write tools are never registered for that session,
so a correct client (ChatGPT, Claude.ai, etc.) simply does not list them. This
is by design, not a bug.

To confirm which case you are in, ask the client to call **`ynab_auth_status`**
(a read tool, always available). It reports:

- `writes_enabled` — `false` means this connection is read-only.
- `write_enablement` — host-specific instructions for turning writes on.

**To enable writes:** disconnect/remove the YNAB connector in your client, add
it again, and on the YNAB consent screen **check "Allow write access."** New
grants always default to read-only, so the box must be checked on every fresh
authorization. After reconnecting, `ynab_auth_status` should report
`writes_enabled: true` and the update/create tools should appear.

**Client enablement caveat:** some clients require a developer/beta mode, or
workspace-admin approval, before they will expose or invoke custom-connector
write tools — independent of the YNAB grant. (In ChatGPT, custom-MCP write
actions run through developer mode and prompt for per-action approval.) If
`writes_enabled: true` but the client still will not call a write tool, this is
a client-side enablement issue, not a connector defect — verify the server side
independently with the MCP Inspector or a direct `tools/call`.
