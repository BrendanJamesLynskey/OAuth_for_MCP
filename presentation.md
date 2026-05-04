# OAuth for MCP Servers & the Provider Landscape

**Remote MCP, the Docker MCP Gateway & the identity-provider landscape**

*How remote MCP servers do delegated authorisation; what the Docker MCP Gateway adds; and how to pick a provider — commercial or open-source — that won't paint you into a corner.*

```
LLM Host -> MCP Server -> Auth Server -> Token (RI + DPoP) -> Tool Call
```

Discover  |  Register  |  Authorise  |  Bind  |  Audit

---

## Table of Contents

1. [Topics](#slide-01--topics)
2. [OAuth Refresher (One Slide)](#slide-02--oauth-refresher-one-slide)
3. [What Is the Model Context Protocol?](#slide-03--what-is-the-model-context-protocol)
4. [Local vs Remote MCP — Why Auth Matters](#slide-04--local-vs-remote-mcp--why-auth-matters)
5. [The MCP Authorization Profile (2025-06)](#slide-05--the-mcp-authorization-profile-2025-06)
6. [Discovery — How the Client Finds the AS](#slide-06--discovery--how-the-client-finds-the-as)
7. [Dynamic Client Registration](#slide-07--dynamic-client-registration)
8. [End-to-End Sequence](#slide-08--end-to-end-sequence--host--mcp--as--resource)
9. [Resource Indicators & Audience Binding in MCP](#slide-09--resource-indicators--audience-binding-in-mcp)
10. [DPoP for MCP — Sender-Constrained Tokens](#slide-10--dpop-for-mcp--sender-constrained-tokens)
11. [Anti-Patterns & Confused-Deputy Pitfalls](#slide-11--anti-patterns--confused-deputy-pitfalls)
12. [The Docker MCP Gateway — Intro](#slide-12--the-docker-mcp-gateway--intro)
13. [Docker MCP Gateway — Architecture](#slide-13--docker-mcp-gateway--architecture)
14. [Docker MCP Gateway + OAuth — The Interceptor](#slide-14--docker-mcp-gateway--oauth--the-interceptor)
15. [Provider Landscape — At a Glance](#slide-15--provider-landscape--at-a-glance)
16. [Commercial Providers — Detail](#slide-16--commercial-providers--detail)
17. [Free / Open-Source Providers — Detail](#slide-17--free--open-source-providers--detail)
18. [SaaS vs Self-Hosted — Trade-offs](#slide-18--saas-vs-self-hosted--trade-offs)
19. [Choosing a Provider for an MCP Server](#slide-19--choosing-a-provider-for-an-mcp-server)
20. [Worked Example 1 — claude.ai → Remote MCP → Auth0](#slide-20--worked-example-1--claudeai--remote-mcp--auth0)
21. [Worked Example 2 — Claude Desktop → Docker Gateway → Keycloak](#slide-21--worked-example-2--claude-desktop--docker-gateway--keycloak)
22. [Production Checklist](#slide-22--production-checklist)
23. [Summary & References](#slide-23--summary--references)

---

## Slide 01 — Topics

### MCP foundations

- The Model Context Protocol — one-slide refresher
- Local (stdio) vs Remote (HTTP / SSE / Streamable) servers
- Why remote MCP *has* to use OAuth
- The MCP authorisation profile (2025-06 spec)

### The MCP OAuth flow

- Discovery — RFC 9728 + RFC 8414
- Dynamic Client Registration (RFC 7591)
- End-to-end sequence with Auth Code + PKCE + RIs
- Confused-deputy & token-passthrough — what NOT to do

### Docker MCP Gateway

- What the Gateway is and why it exists
- Architecture — catalogs, secrets, OAuth interceptor
- Centralising auth for many MCP servers behind one front door
- Compose, MCP Toolkit and CLI usage

### Provider landscape

- Commercial — Auth0/Okta, Microsoft Entra, AWS Cognito, Google, Stytch, WorkOS, Clerk, Descope, FusionAuth
- Free / OSS — Keycloak, Authentik, ZITADEL, Ory Hydra, Authelia, Logto, SuperTokens
- Decision matrix & production checklist
- Worked examples: Claude.ai → MCP → Auth0; Claude Desktop → Gateway → Keycloak

---

## Slide 02 — OAuth Refresher (One Slide)

### If you read nothing else from Part 1

- **Authorization Code + PKCE** is the only flow when a user is involved.
- **Access tokens** = call the API, short-lived. **Refresh tokens** = mint new access tokens, rotated. **ID tokens** = identity assertion (OIDC).
- Tokens must be **audience-bound** (RFC 8707) and ideally **sender-constrained** (DPoP / mTLS).
- OAuth 2.1 = OAuth 2.0 + a decade of errata. PKCE everywhere, no Implicit, no ROPC, no wildcards.

### The four roles

- **Resource Owner** = user
- **Client** = the app — here, the MCP host (Claude Desktop, Cursor, claude.ai)
- **Authorization Server** = issues tokens (Auth0, Keycloak, Entra, ...)
- **Resource Server** = the MCP server itself

### Tokens at a glance

| Token | Sent to | Lifetime |
|---|---|---|
| Access | MCP server | 5–60 min |
| Refresh | AS only | hours–days |
| ID (OIDC) | stays in client | 5–15 min |

### RFCs we'll lean on heavily

RFC 6749 (core) · RFC 7591 (DCR) · RFC 7636 (PKCE) · RFC 8414 (AS metadata) · RFC 8707 (Resource Indicators) · RFC 9068 (JWT AT) · RFC 9449 (DPoP) · RFC 9700 (BCP 240) · RFC 9728 (Protected Resource Metadata)

---

## Slide 03 — What Is the Model Context Protocol?

**MCP** is an open protocol from Anthropic (Nov 2024) that standardises how LLM applications talk to **tools**, **resources** and **prompts**. Think of it as *"USB-C for AI integrations"* — one protocol between any host and any data source or tool.

```
        ┌──────────────┐                                  ┌──────────────┐
        │   MCP Host   │  ───stdio JSON-RPC──▶  ┌─────────│  local MCP   │── filesystem · sqlite · git
        │ Claude Dsktp │                       │          │   server     │
        │   Cursor     │                       │          └──────────────┘
        │  claude.ai   │  ───HTTP/SSE─────────▶│          ┌──────────────┐
        └──────────────┘                       └─────────▶│  remote MCP  │── GitHub · Slack · Linear · Stripe
                                                          │   server     │
                                                          └──────────────┘
```

Local servers run as a subprocess. Remote servers are real network services — and that's where OAuth becomes mandatory.

### Three primitives

- **Tools** — functions the LLM can *call* (write a file, send a Slack message, run a SQL query).
- **Resources** — data the LLM can *read* (files, DB rows, doc fragments, identified by URI).
- **Prompts** — reusable templates the host can offer to the user.

---

## Slide 04 — Local vs Remote MCP — Why Auth Matters

### Local server (stdio)

- Spawned as a child process by the host.
- Trust boundary = the OS user. Anything that user can do, the server can do.
- No network — no OAuth, no tokens, no network attacker.
- Good for: filesystem, local DBs, dev-loop tools.
- Risk: *prompt injection* + dangerous tools = local arbitrary code execution.

### Remote server (HTTP / Streamable HTTP / SSE)

- Reached over the public internet (or VPC). Multi-tenant. Survives between hosts.
- Trust boundary = whoever holds a valid token.
- Needs to know *which user* is calling and *what they're allowed to do*.
- Good for: SaaS connectors, enterprise data, anything where the API isn't on this laptop.
- Without OAuth, the only options are bearer-API-keys-in-config-files, which puts you back in 2006.

### Why a global API key is wrong for remote MCP

- **No user identity** — the MCP server can't enforce per-user permissions.
- **No revocation** — leaked once, replayable forever.
- **No audit** — you can't tell *which* human triggered which tool.
- **No consent** — users have no UI to approve or revoke specific scopes.
- **No MFA** — credential strength is whatever the operator chose, not the org's IdP policy.

Every one of these is a thing OAuth gives you for free. That is why the MCP spec mandates it for HTTP transports.

---

## Slide 05 — The MCP Authorization Profile (2025-06)

The MCP spec includes a normative **Authorization** section for HTTP-based servers. It's a profile of OAuth 2.1 — pinning the choices a generic spec leaves open.

### What the MCP profile says — MUST

- The MCP server is an OAuth 2.1 **Resource Server**.
- The MCP client uses **Authorization Code + PKCE (S256)**.
- The client discovers the AS via the server's **RFC 9728 Protected Resource Metadata** document.
- The client uses **RFC 8707 Resource Indicators** to bind the access token's audience to the MCP server URL.
- The MCP server validates `aud` on every token and rejects any token not minted for it.
- The server returns **`401 + WWW-Authenticate`** with a `resource_metadata` URL when no/expired token is presented.

### What it says — SHOULD / MAY

- **Dynamic Client Registration** (RFC 7591) so users don't have to pre-register every host.
- **DPoP** (RFC 9449) for sender-constrained tokens — strongly recommended over plain Bearer.
- Refresh tokens with **rotation**.
- Scopes named after MCP capabilities (e.g. `tools:write`, `resources:read`).
- OIDC if the server needs to know who the user is, not just that they're authorised.

### Anti-patterns the spec calls out by name

- **Token passthrough** — the MCP server forwarding the user's token to a downstream API the client never consented to.
- **Confused-deputy** tokens — accepting any valid token from "your" AS regardless of audience.
- **Static API keys** in HTTP headers in lieu of OAuth.

---

## Slide 06 — Discovery — How the Client Finds the AS

The MCP client doesn't get told upfront which auth server to use. It asks the MCP server.

```
Client                MCP Server                    Auth Server
  │                       │                              │
1.├──POST /tools/list (no token)──▶                      │
2.◀──401 + WWW-Authenticate: Bearer resource_metadata="…"
3.├──GET /.well-known/oauth-protected-resource──▶        │
4.◀──{ resource: "…", authorization_servers: ["https://…"] }
5.├────────GET https://as/.well-known/oauth-authorization-server─▶
6.◀────────────{ authorization_endpoint, token_endpoint, jwks_uri, registration_endpoint, … }
                         → client now knows where to start the OAuth flow
```

### RFC 9728 — Protected Resource Metadata

```json
GET /.well-known/oauth-protected-resource
{
  "resource": "https://mcp.example.com/",
  "authorization_servers": ["https://auth.example.com"],
  "scopes_supported": ["tools:read","tools:write"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://docs..."
}
```

### RFC 8414 — AS Metadata

```json
GET /.well-known/oauth-authorization-server
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": ".../authorize",
  "token_endpoint": ".../token",
  "registration_endpoint": ".../register",
  "jwks_uri": ".../jwks",
  "code_challenge_methods_supported": ["S256"],
  "grant_types_supported": ["authorization_code","refresh_token"],
  "dpop_signing_alg_values_supported": ["ES256"]
}
```

---

## Slide 07 — Dynamic Client Registration

Pre-MCP, every OAuth client was registered by hand in the AS UI. That doesn't scale to "any user, any host". **RFC 7591 Dynamic Client Registration** lets the host register itself the first time it sees a server.

### The DCR call

```
POST /register HTTP/1.1
Host: auth.example.com
Content-Type: application/json

{
  "client_name": "Claude Desktop",
  "redirect_uris": ["http://127.0.0.1:53682/callback"],
  "grant_types": ["authorization_code","refresh_token"],
  "token_endpoint_auth_method": "none",
  "application_type": "native",
  "software_id": "com.anthropic.claude",
  "software_version": "1.4.0"
}

# response
{
  "client_id": "rdM_b1...",
  "client_id_issued_at": 1700000000,
  "redirect_uris": [...],
  "grant_types": [...]
}
```

### Why MCP needs this

- Hosts (Claude Desktop, Cursor, claude.ai, Goose, custom agents) cannot pre-arrange with every MCP server's AS.
- DCR turns "first connection" into a one-RPC handshake.
- Registration produces a per-host, per-AS `client_id` the host stores locally.
- The client uses that `client_id` in every subsequent Authorization Code + PKCE flow.

### DCR with Initial Access Tokens

For tighter control, an AS can require an **Initial Access Token** on `/register` — issued out-of-band by an admin. Stops random clients self-registering against an enterprise AS.

### Watch out

Open DCR is a free abuse vector. Enterprise deployments usually disable it and use software statements (RFC 7591 §2.3) or pre-issued client IDs instead.

---

## Slide 08 — End-to-End Sequence — Host → MCP → AS → Resource

```
User    Host (Client)    MCP Server (RS)    Auth Server          Upstream API
 │           │                 │                  │                   │
 1.add server│                 │                  │                   │
   ───────▶  │                 │                  │                   │
 2.          │ tools/list (no token)              │                   │
             ──────────────▶  │                   │                   │
 3.          │ ◀── 401 + resource_metadata="…"   │                   │
 4.          │ GET /.well-known/oauth-protected-resource              │
             ──────────────▶  │                   │                   │
 5.          │ ◀── { authorization_servers, … }   │                   │
 6.          │ POST /register (DCR, RFC 7591)                         │
             ──────────────────────────────────▶  │                   │
 7.          │ ◀── { client_id }                                      │
 8.          │ open browser to /authorize+PKCE+resource=…             │
   ◀──────── │                                    │                   │
 9. user logs in & consents                       │                   │
   ────────────────────────────────────────────▶  │                   │
 10.         │ ◀── 302 redirect → loopback?code=… │                   │
 11.         │ POST /token (code, code_verifier, resource=…)          │
             ──────────────────────────────────▶  │                   │
 12.         │ ◀── { access_token (aud=https://mcp/…), refresh_token }│
 13.         │ tools/call  Authorization: Bearer/DPoP <at>            │
             ──────────────▶  │                   │                   │
 14.                          │ server's own creds → upstream API     │
                              ──────────────────────────────────────▶ │
 15.                          │ ◀── result → back to host             │
```

Step 14 is the critical separation: the MCP server uses its *own* credentials upstream — it doesn't replay the user's token. That's how the spec avoids the confused-deputy class of bugs.

---

## Slide 09 — Resource Indicators & Audience Binding in MCP

### Why RIs are non-negotiable for MCP

- One AS often serves many MCP servers — and other unrelated APIs.
- Without `resource=`, an access token issued for an innocent MCP could be replayed against any other RS that trusts the same AS.
- RFC 8707 lets the client name the intended RS so the AS can pin `aud`.
- The MCP server's job: validate `aud` = *its own URL*.

### If you skip RIs

You've built a ready-made *token launderer*. A compromised MCP server gets bearer tokens it can spend at every RS in the org.

### What it looks like on the wire

```
# /authorize
GET /authorize?
    response_type=code&
    client_id=rdM_b1&
    redirect_uri=http://127.0.0.1:53682/callback&
    scope=tools:write+resources:read&
    code_challenge=XYZ&
    code_challenge_method=S256&
    state=...&
    resource=https://mcp.example.com/

# /token
POST /token
grant_type=authorization_code&
code=...&
code_verifier=...&
resource=https://mcp.example.com/

# resulting access token (JWT, decoded)
{
  "iss": "https://auth.example.com",
  "sub": "user_42",
  "aud": "https://mcp.example.com/",
  "scope": "tools:write resources:read",
  "client_id": "rdM_b1",
  "exp": 1700003600
}
```

---

## Slide 10 — DPoP for MCP — Sender-Constrained Tokens

### Why MCP cares about DPoP specifically

- MCP tokens travel from a host → over the public internet → to an MCP server.
- Hosts often run on dev laptops with browser extensions, proxies, MITM dev tools — token leak surface is large.
- A leaked plain Bearer = the attacker can act as the user against the MCP server until expiry.
- DPoP binds the token to a private key the host generated locally and never sends.

### The exchange

```
# 1. host generates a key pair, includes JWK on /token
POST /token  ...
DPoP: eyJ...   # proof for the token endpoint

# 2. AS issues an access token bound to that key
{ "access_token": "eyJ...",
  "token_type": "DPoP",
  "expires_in": 600 }

# AT payload includes
"cnf": { "jkt": "sha256(jwk)" }

# 3. host calls MCP server
POST /mcp/tools/call
Authorization: DPoP eyJ...
DPoP: eyJ...   # fresh proof per request, signs htm+htu+iat+jti
```

### What the MCP server must check

1. Validate the access token (signature, `iss`, `aud`, `exp`, `scope`).
2. Parse the `DPoP` header JWS.
3. Verify the proof is signed by the JWK whose hash matches `cnf.jkt`.
4. Check `htm` = request method, `htu` = request URL.
5. Check `iat` within a small window (default 60 s).
6. Check `jti` hasn't been seen before (replay cache).

### In code (pseudocode)

```python
def authorize(req):
    at = parse_bearer_or_dpop(req)
    claims = jwt.verify(at, jwks)
    require(claims.aud == MY_RS_URL)
    require("tools:write" in claims.scope)
    if claims.token_type == "DPoP":
        proof = req.headers["DPoP"]
        verify_dpop(proof, claims["cnf"]["jkt"], req)
    return claims["sub"]
```

---

## Slide 11 — Anti-Patterns & Confused-Deputy Pitfalls

### 1. Token passthrough

The MCP server takes the user's access token and calls a downstream API *with it*.

- The downstream API was never named in the user's consent.
- The token's `aud` doesn't match — but downstream APIs that don't validate `aud` won't notice.
- This is the OAuth "confused deputy" attack with a thin wrapper.

**Fix:** the MCP server obtains its own credentials for the downstream API (Client Credentials, or Token Exchange RFC 8693 if user identity must be preserved). Never re-send the user's bearer.

### 2. Skipping `aud` checks

"Any token from our AS that signs is fine." → token launderer. Always pin `aud == my_rs_url`.

### 3. Accepting tokens from any issuer

If the server lists multiple AS in its protected-resource metadata, every one is implicitly trusted. Pin to a specific issuer or maintain an explicit allow-list.

### 4. Per-server static API keys ("OAuth on the wire only")

Some early MCP servers fronted a static upstream API key with OAuth login. The AS issues real access tokens, but the server doesn't actually use the user identity downstream — every user shares the same upstream credentials. Result: zero per-user audit, no scope enforcement, single-secret breach radius.

**Fix:** per-user upstream creds via Token Exchange or per-user OAuth-on-behalf-of.

### 5. Running without DPoP/mTLS over the public internet

Bearer tokens through dev-laptop network paths get leaked. Ship DPoP from day one for any non-trivial scope.

### 6. Open DCR with no rate-limiting

Anyone on the internet can spin up infinite client_ids on your AS, blowing through quotas and audit storage. Rate-limit, require Initial Access Tokens, or use software statements.

### 7. Logging the `Authorization` header

Tokens end up in CloudWatch / Loki / Datadog. Scrub aggressively.

---

## Slide 12 — The Docker MCP Gateway — Intro

The **Docker MCP Gateway** (Docker Inc., 2025) is an open-source binary that fronts *many* MCP servers (each running in its own container) behind a single endpoint. It pairs with the **Docker MCP Catalog** and the **Docker MCP Toolkit** in Docker Desktop.

### What problem it solves

- Each MCP server has a different install story (npm? Python? Docker?). The Gateway runs them all uniformly as containers from a curated catalog.
- Each server has its own auth model (env-var API key here, OAuth there). The Gateway centralises secret handling and OAuth.
- Hosts (Claude Desktop, Cursor, claude.ai, VS Code) want one HTTP endpoint, not *n*.
- Operators want one place to log, audit, sign and policy-gate every tool call.

### Components

- **Gateway** — the binary. Speaks MCP to hosts upstream and MCP to servers downstream.
- **Catalog** — JSON/YAML index of vetted servers, container images, default scopes, OAuth metadata.
- **Secrets** — local secret store; never written into container env vars.
- **OAuth Interceptor** — handles discovery, registration, code/PKCE, token cache, refresh.
- **Toolkit** — Docker Desktop UI + CLI for browsing the catalog and enabling servers.

### Why Docker, specifically?

- Server images can be pinned, signed (Sigstore / Notation) and scanned like any other.
- Per-container resource limits, seccomp, no-new-privileges by default — sandboxing for prompt-injected tools.
- Compose makes "stand up a stack of servers + gateway + AS" a one-liner.
- Docker Hub already operates as a registry the org's policy engine knows.

### Where to find it

`github.com/docker/mcp-gateway` · Docker MCP Catalog at `hub.docker.com/u/mcp` · ships inside Docker Desktop's MCP Toolkit pane.

---

## Slide 13 — Docker MCP Gateway — Architecture

```
         hosts                           Docker MCP Gateway                     servers
   ┌────────────────┐               ┌────────────────────────────┐        ┌──────────────────┐
   │ Claude Desktop │ ── stdio ─┐   │  ┌────────────┐ ┌────────┐ │  ┌──── │ github MCP (cnt) │
   │ Cursor / VSC   │ ── stdio ─┼─▶ │  │MCP front-  │ │catalog │ │  │     │ slack MCP (cnt)  │
   │ claude.ai      │ ── HTTP ──┘   │  │   end      │ │loader  │ │  │     │ linear MCP       │
   └────────────────┘               │  └────────────┘ └────────┘ │  ├───▶ │ notion MCP       │
                                    │  ┌────────────┐ ┌────────┐ │  │     │ filesystem MCP   │
                                    │  │ OAuth      │ │ token  │ │  └──── │ ...              │
                                    │  │ interceptor│ │ cache  │ │        └──────────────────┘
                                    │  └────────────┘ └────────┘ │
                                    │  ┌────────────┐ ┌────────┐ │              │
                                    │  │ secrets    │ │ audit  │ │              │
                                    │  └────────────┘ └────────┘ │              ▼
                                    │  policy / scope / allow    │     ┌──────────────────┐
                                    └────────────────────────────┘     │  Auth Server     │
                                                ▲                      │  (Auth0,         │
                                                └──── discovery + DCR  │   Keycloak,      │
                                                      + /token (per    │   Entra,         │
                                                      server)          │   ZITADEL …)     │
                                                                       └──────────────────┘
```

One MCP endpoint to the host. The Gateway fans out to many sandboxed servers, brokers OAuth per-server, and centralises audit.

---

## Slide 14 — Docker MCP Gateway + OAuth — The Interceptor

The Gateway's **OAuth interceptor** moves the entire OAuth dance out of every host and into one trusted process. Hosts only see "tools that work"; the Gateway holds the tokens.

### What it does

- For each enabled server, reads OAuth metadata from the catalog (issuer, scopes, audience, DCR endpoint).
- On first use, opens the user's system browser to complete Auth Code + PKCE.
- Stores access + refresh tokens in the local secrets store, scoped per-server, per-user.
- Refreshes silently when an access token expires; rotates refresh tokens.
- Injects the right token on each downstream call to the right MCP server.
- Logs every tool invocation with user, scope, server, and request hash for audit.

### A catalog entry (sketch)

```yaml
name: github
image: mcp/github:1.4
oauth:
  issuer: https://github.com
  scopes: [repo, read:user]
  audience: https://api.github.com
secrets:
  - name: GITHUB_PAT
    optional: true
```

### Why this is better than each host doing its own OAuth

- **One consent**, all hosts on the machine benefit.
- **Tokens never leave a single trust boundary** — Cursor doesn't see your Slack token, claude.ai doesn't see your GitHub one unless explicitly allowed.
- **Centralised revocation** — kill the Gateway secret, every server is logged out.
- **Per-server scope enforcement** at one chokepoint.
- **Audit** — one log file vs *n* hosts × *n* servers.

### CLI flow

```
# list catalog
docker mcp catalog ls

# enable a server
docker mcp server enable github
# → Gateway opens browser for OAuth
# → tokens stored in local secret store
# → 'github' now appears as a tool source to every host

# revoke
docker mcp server logout github
```

### Trust note

You are now trusting the Gateway with every connected service's tokens. Run it locally; pin its image; restrict who can talk to its socket. For team deployments, treat it like a privileged credential broker.

---

## Slide 15 — Provider Landscape — At a Glance

| Provider | Model | OAuth 2.1 ready | DCR | DPoP | Free tier | Sweet spot |
|---|---|---|---|---|---|---|
| **Auth0 (Okta)** | SaaS | yes | yes | preview | 7,500 MAU | Fast time-to-first-token; deep MCP samples. |
| **Okta Workforce / Customer** | SaaS | yes | limited | n/a | — | Enterprise IdP with SCIM, governance, advanced policies. |
| **Microsoft Entra ID** | SaaS | yes | limited | via FAPI profile | 50k MAU (External ID) | M365 / Azure shops; CIAM via External ID. |
| **Google Identity Platform** | SaaS | yes | via Firebase | n/a | generous | Consumer apps already on GCP / Firebase. |
| **AWS Cognito** | SaaS | 2.0 + most BCP 240 | limited | n/a | 50k MAU | AWS-native CIAM; cheap at scale. |
| **Stytch** | SaaS | yes | yes | yes | 10k MAU | Modern API-first; strong MCP & agent story. |
| **WorkOS** | SaaS | yes | yes | via AuthKit | 1M users (AuthKit) | "SSO / SCIM / MCP for B2B SaaS" turn-key. |
| **Clerk** | SaaS | yes | yes | preview | 10k MAU | React/Next.js DX-first; simple MCP integration. |
| **Descope** | SaaS | yes | yes | preview | 7k MAU | Drag-and-drop flows; agentic identity features. |
| **FusionAuth** | SaaS or self-host | yes | partial | n/a | Community edition free | Same product on-prem & in cloud; flexible licensing. |
| **Keycloak** | OSS, self-host | yes | yes | yes | free | Most flexible OSS IdP; FAPI 2.0 conformant. |
| **ZITADEL** | OSS or SaaS | yes | yes | yes | 25k MAU (cloud) | Modern multi-tenant CIAM; first-class MCP guides. |
| **Authentik** | OSS, self-host | yes | partial | n/a | free | Easy OIDC + SAML for homelabs & small teams. |
| **Ory Hydra (+ Kratos)** | OSS or SaaS (Ory Network) | yes | yes | yes | free | OAuth/OIDC server you compose with your own login UI. |
| **Authelia** | OSS, self-host | OIDC 1.0 | limited | n/a | free | Forward-auth for reverse-proxies; great for homelabs. |
| **Logto** | OSS or SaaS | yes | partial | n/a | 5k MAU (cloud) | Lightweight CIAM for small SaaS / indie hackers. |
| **SuperTokens** | OSS or SaaS | core OAuth | partial | n/a | 5k MAU | Self-host with managed dashboard option. |

---

## Slide 16 — Commercial Providers — Detail

### Auth0 (Okta)

- Most-cited SaaS IdP; published official "OAuth for MCP" guides and samples in 2025.
- Full OIDC, OAuth 2.1, DCR, fine-grained scopes, Actions for custom logic.
- Free dev tenant: 7,500 MAU (essentials), social connections, basic MFA. Paid plans add SAML, attack protection, large MAU.

### Microsoft Entra ID (formerly Azure AD)

- Default for any org on Microsoft 365.
- Entra **External ID** = the CIAM brand (replaces Azure AD B2C). Free tier ~50k MAU.
- App-registration UI, conditional access, FAPI 2.0 profile available.

### AWS Cognito

- User pools + identity pools; very cheap at scale (free up to 50k MAU on Lite).
- OAuth 2.0 + OIDC. DCR limited; some BCP 240 items lag behind Auth0.
- Best when the rest of the stack is on AWS and you want "good enough" without an extra vendor.

### Google Identity Platform / Firebase Auth

- Same engine that powers "Sign in with Google".
- Firebase Auth = the friendly DX layer; Identity Platform = the enterprise SKU.
- Strong consumer presence; weaker for B2B SSO.

### Stytch

- API-first, opinionated SDKs, explicit MCP & agent identity story (2025).
- OAuth 2.1, DCR, DPoP, passkeys, magic links, B2B orgs out of the box.
- Free tier: 10k MAU; metered pricing above.

### WorkOS (AuthKit)

- "Make my B2B SaaS enterprise-ready" — SSO, SCIM, audit logs, AuthKit OIDC.
- Pivoted hard into MCP server hosting + agent auth in 2025.
- AuthKit free up to ~1M MAU; SSO connections charged per active connection.

### Clerk

- React/Next.js-first DX; pre-built UI components.
- Added OAuth-server features (so your app can issue tokens to MCP clients) in 2025.
- Free up to 10k MAU.

### Descope · FusionAuth · Frontegg

- **Descope** — drag-and-drop flows; explicit agentic identity features.
- **FusionAuth** — same product self-host or cloud; permissive licence.
- **Frontegg** — embedded multi-tenant SaaS auth.

---

## Slide 17 — Free / Open-Source Providers — Detail

### Keycloak

- Red Hat / community OSS IdP — Apache 2.0.
- Full OAuth 2.1 + OIDC + SAML, FAPI 2.0 conformant, DCR, DPoP, mTLS.
- Realms = multi-tenant; clients, scopes, mappers, brokers, fine-grained admin.
- Heavy (Java/Quarkus); needs Postgres + JVM tuning.
- De-facto standard when "we'll self-host an IdP" is the answer.

### ZITADEL

- Go, event-sourced, multi-tenant from day one.
- OAuth 2.1, OIDC, SAML, DPoP, DCR, passkeys, actions.
- Both **self-host** (Apache 2.0) and **ZITADEL Cloud** (free up to 25k MAU).
- Published explicit "OAuth + MCP" guidance and DPoP support; popular pick for greenfield MCP servers.

### Authentik

- Python/Django; LDAP-friendly; great web UI.
- OIDC + SAML + reverse-proxy auth; enterprise plan adds RAC.
- Sweet spot: homelabs, small teams, Proxmox / Unraid stacks.

### Ory Hydra (+ Kratos + Keto + Oathkeeper)

- **Hydra** = pure OAuth 2.1 / OIDC server; you bring your own login UI (often Kratos).
- **Kratos** = identity + login flows; **Keto** = Zanzibar-style permissions; **Oathkeeper** = identity-aware proxy.
- Apache 2.0 self-host; Ory Network is the SaaS form.
- Best for teams that want OAuth without dragging in a full IdP UI.

### Authelia

- Forward-auth companion for nginx / Traefik / Caddy.
- OIDC 1.0 server, MFA, single sign-on for self-hosted services.
- Light footprint; ideal for homelab stacks pairing with Caddy.

### Logto · SuperTokens · Janssen

- **Logto** — modern OSS CIAM with managed cloud option (free 5k MAU).
- **SuperTokens** — OSS auth with optional managed dashboard.
- **Janssen Project** — Linux Foundation OAuth/OIDC stack (Gluu lineage); enterprise-grade and FIDO2.

---

## Slide 18 — SaaS vs Self-Hosted — Trade-offs

| Concern | SaaS (Auth0, Stytch, Entra...) | Self-host (Keycloak, ZITADEL, Hydra...) |
|---|---|---|
| Time to first OAuth flow | minutes | hours – a day |
| Operational burden | vendor's problem | your problem (Postgres, JVM, TLS, certs) |
| Compliance evidence | SOC2 / ISO reports come from vendor | you produce them |
| Data residency / sovereignty | vendor regions; sometimes a constraint | full control |
| Vendor lock-in | moderate to high; export tooling varies | low |
| Cost at 100 users | often free | ~ small VPS |
| Cost at 1M users | £tens of thousands / month | ~ a few VMs + ops time |
| Custom flows / branding | limited to vendor primitives | anything (you wrote it) |
| FedRAMP / sovereign cloud | special SKUs; not all features | you choose the cloud |
| "Auth team" required | no | yes, eventually |

### A reasonable rule

SaaS until either (a) you're paying more than the cost of one auth engineer, or (b) you have a data-residency / sovereignty / compliance constraint that the SaaS doesn't meet. At that point, Keycloak or ZITADEL self-host.

---

## Slide 19 — Choosing a Provider for an MCP Server

| If you... | Strongest pick | Why |
|---|---|---|
| Want to ship a public remote MCP today, minimal ops | **Auth0** or **Stytch** | Both publish current MCP samples, full OAuth 2.1, DCR, DPoP-ready, generous free tier. |
| Build a B2B SaaS that resells MCP to enterprise customers | **WorkOS** | SSO + SCIM + AuthKit + MCP hosting bundled. |
| Are deep in Microsoft 365 / Azure | **Microsoft Entra ID** | Users already exist; conditional access already configured. |
| Are deep in AWS, cost-sensitive at scale | **AWS Cognito** | Cheapest hyperscaler CIAM; native IAM glue. |
| Are deep in Next.js / React DX | **Clerk** | Best-in-class components; OAuth-server features added 2025. |
| Have data residency / air-gap requirements | **Keycloak** or **ZITADEL self-host** | Full control of where tokens and PII live. |
| Want OAuth/OIDC but bring your own login UX | **Ory Hydra (+ Kratos)** | Headless, composable, no opinion on login screens. |
| Run a homelab or 10-user internal stack | **Authentik** or **Authelia** | Light, easy, OIDC for everything self-hosted. |
| Need FAPI 2.0 / banking-grade security | **Keycloak** or **Curity** | Conformant FAPI profiles, PAR, mTLS, JAR. |
| Want a single binary, OSS, multi-tenant out-of-the-box | **ZITADEL** | Go, event-sourced, modern CIAM features baked in. |

---

## Slide 20 — Worked Example 1 — claude.ai → Remote MCP → Auth0

You ship a public remote MCP server (`https://mcp.acme.com`) that wraps Acme's REST API. You want claude.ai users to enable it with one click.

### Server side

1. Create an Auth0 tenant; create an *API* (audience) `https://mcp.acme.com` with scopes `tools:read`, `tools:write`.
2. Enable *Dynamic Application Registration* (Auth0's DCR); turn on *require Initial Access Token* if you want gating.
3. Publish `/.well-known/oauth-protected-resource` on the MCP server pointing to the Auth0 issuer.
4. On every MCP request, validate the JWT: signature via Auth0 JWKS, `iss`, `aud == https://mcp.acme.com`, `exp`, scope.
5. For DPoP-bound tokens, validate the proof header.

### Server's protected-resource document

```json
{
  "resource": "https://mcp.acme.com/",
  "authorization_servers": ["https://acme.eu.auth0.com/"],
  "scopes_supported": ["tools:read","tools:write"],
  "bearer_methods_supported": ["header"],
  "dpop_signing_alg_values_supported": ["ES256"]
}
```

### Client side (claude.ai)

1. User pastes `https://mcp.acme.com` into "Add MCP server".
2. claude.ai gets `401 + WWW-Authenticate` with `resource_metadata`.
3. Fetches the protected-resource doc; fetches Auth0's `/.well-known/oauth-authorization-server`.
4. Calls `POST /oidc/register` on Auth0 (DCR) to mint a per-claude.ai `client_id`.
5. Pops a system browser to `/authorize` with `response_type=code`, PKCE S256, `resource=https://mcp.acme.com/`, `scope=tools:read tools:write`.
6. User logs into Auth0, sees consent for the listed scopes.
7. Auth0 redirects back to claude.ai's loopback / `https://claude.ai/oauth/callback`.
8. claude.ai exchanges the code at `/oauth/token` with PKCE verifier and `resource=`.
9. Stores access + refresh tokens in claude.ai's per-server token cache.
10. Calls `tools/list` on the MCP server; off you go.

---

## Slide 21 — Worked Example 2 — Claude Desktop → Docker Gateway → Keycloak

You're an enterprise. You self-host an internal MCP server (`jira-mcp`) talking to internal Jira. Claude Desktop must auth users via your existing Keycloak realm.

### Topology

- **User** on a laptop with Docker Desktop + MCP Toolkit.
- **Claude Desktop** → speaks stdio MCP → **Docker MCP Gateway** (local).
- **Gateway** → over HTTPS → **`jira-mcp`** container in your VPC.
- **`jira-mcp`** → calls Jira's internal REST API using a *service-account* token (Client Credentials), not the user's token.
- **Keycloak** = the AS, used for both: user → MCP Server, and MCP Server → Jira.

### Keycloak config

- Realm: `acme`; client: `jira-mcp-rs` (resource server, audience `https://jira-mcp.acme.internal`).
- Realm scopes: `jira:issue:read`, `jira:issue:write`, `jira:project:read`.
- DCR enabled for `application_type=native`, restricted to known software statements.
- FAPI 2.0 profile turned on; DPoP required for the `jira-mcp-rs` client.

### Gateway catalog entry

```yaml
name: jira
image: registry.acme.internal/mcp/jira:2.1
oauth:
  issuer: https://sso.acme.com/realms/acme
  audience: https://jira-mcp.acme.internal
  scopes: [jira:issue:read, jira:issue:write]
  dpop: required
  initial_access_token_env: KC_DCR_TOKEN
secrets:
  - name: KC_DCR_TOKEN
    source: vault
audit:
  destination: https://logs.acme.com/mcp
  fields: [user_sub, server, tool, scope, request_hash]
```

### First-run UX

1. User enables `jira` in MCP Toolkit.
2. Gateway reads catalog; calls Keycloak DCR with the Initial Access Token.
3. Opens browser to Keycloak; user does corporate SSO + MFA.
4. Tokens land in Gateway's local secret store, bound to user's keypair (DPoP).
5. Claude Desktop now sees `jira` tools; every call is logged centrally with the user's `sub`.

---

## Slide 22 — Production Checklist

### MCP server (resource server)

- Publish `/.well-known/oauth-protected-resource`.
- Pin a single `iss` (or explicit allow-list).
- Require `aud == my_url` on every token.
- Cache JWKS 10–60 min; refresh on unknown `kid`.
- Enforce per-tool minimum scopes — fail closed.
- Accept DPoP if the AS supports it; reject plain Bearer for write tools.
- Return rich `WWW-Authenticate` with `resource_metadata` on 401.
- Never forward the user's token downstream — use service credentials or RFC 8693 Token Exchange.
- Scrub `Authorization` & `DPoP` from logs.
- Rate-limit DCR; require Initial Access Token in production.

### MCP host (client)

- Always Auth Code + PKCE S256 + `state` + `resource=`.
- Use the system browser, never an embedded webview.
- Generate a fresh DPoP keypair per server (or per session).
- Rotate refresh tokens; treat reuse as compromise.
- Store tokens in OS keychain / DPAPI, not plaintext config.
- Validate the `iss` on the callback (RFC 9207).

### Authorization server config

- RFC 8414 metadata + RFC 9728 protected-resource metadata.
- Refresh token rotation on for public clients.
- Strict `redirect_uri` string match.
- DCR disabled by default; opt-in per realm with software statements / Initial Access Tokens.
- DPoP supported; FAPI 2.0 profile available for high-value APIs.
- Realm-level audit trail of every `/authorize`, `/token`, `/register`.

### Operational

- NTP everywhere — JWT clock skew is the most common 401 in prod.
- Pin AS issuer URL; certificate-pin if you're paranoid.
- Per-environment client IDs (no shared "dev = prod" credentials).
- Threat-model the Gateway / MCP server containers — seccomp, no-new-privileges, signed images.
- Monitor for refresh-token reuse alerts and 401 spikes.

### Re-read before shipping

RFC 9700 (BCP 240) · MCP Authorization spec · RFC 9728 · RFC 8707 · RFC 9449.

---

## Slide 23 — Summary & References

### What we covered

- Why remote MCP needs OAuth (not API keys)
- The 2025-06 MCP authorization profile — Auth Code + PKCE + RFC 9728 + RFC 8707
- Discovery flow and Dynamic Client Registration
- Audience binding and DPoP for sender-constrained tokens
- Confused-deputy and token-passthrough — the things to avoid
- The Docker MCP Gateway as a centralised OAuth broker
- Commercial providers — Auth0, Okta, Microsoft Entra, Google, AWS Cognito, Stytch, WorkOS, Clerk, Descope, FusionAuth
- Free / OSS providers — Keycloak, ZITADEL, Authentik, Ory Hydra, Authelia, Logto, SuperTokens, Janssen
- SaaS vs self-host trade-offs & decision matrix
- Two end-to-end worked examples
- Production checklist

### Companion deck — Part 1

**"Introduction to OAuth"** — purpose, history, OAuth 1.0a → 2.0 → 2.1, all grant types, PKCE, OIDC, JWTs, DPoP/mTLS, attacks & defences.

### Take-aways

1. Remote MCP is just OAuth 2.1 with one specific profile — Auth Code + PKCE + Resource Indicators on every flow.
2. **Audience-bind** every token. **Sender-constrain** with DPoP if the user is on the public internet.
3. Don't pass the user's token downstream — that's the confused-deputy door.
4. The Docker MCP Gateway turns "OAuth in every host" into "OAuth in one trusted local broker".
5. Provider choice: SaaS for speed, self-host for sovereignty. Auth0/Stytch and Keycloak/ZITADEL are the safe defaults at each end.

### One-line takeaway

OAuth for MCP is not a new protocol — it's a small set of pinned choices on top of OAuth 2.1. Pick a conformant provider, validate `aud`, and let the Gateway hold your tokens.

### References

MCP spec — modelcontextprotocol.io/specification · Anthropic MCP docs — docs.anthropic.com/mcp · Docker MCP Gateway — github.com/docker/mcp-gateway · Docker MCP Catalog — hub.docker.com/u/mcp · Auth0 MCP guide — auth0.com/blog/mcp-oauth · ZITADEL MCP guide — zitadel.com/docs · Stytch MCP guide — stytch.com/docs/mcp · WorkOS MCP · RFC 6749, 6750, 7591, 7636, 8414, 8628, 8693, 8705, 8707, 9068, 9126, 9396, 9449, 9700, 9728
