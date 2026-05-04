# 🔐 OAuth for MCP Servers & the Provider Landscape

An interactive Reveal.js presentation covering **OAuth for the Model Context Protocol** — how remote MCP servers do delegated authorisation, the role of the **Docker MCP Gateway** as a centralised OAuth broker, and a tour of the commercial and open-source identity providers you can plug in.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/OAuth_for_MCP/)

## 📄 [Markdown Version](presentation.md)

## 📚 [Companion deck — Introduction to OAuth (Part 1)](https://brendanjameslynskey.github.io/Introduction_to_OAuth/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | LLM Host → MCP Server → Auth Server → Token → Tool |
| 02 | Topics | MCP foundations · the OAuth flow · Docker Gateway · provider landscape |
| 03 | OAuth Refresher | One-slide recap of Part 1 |
| 04 | What Is MCP? | Hosts, clients, servers, tools, resources, prompts |
| 05 | Local vs Remote | Why HTTP MCP servers must use OAuth |
| 06 | MCP Auth Profile | The 2025-06 normative spec — MUSTs and SHOULDs |
| 07 | Discovery | RFC 9728 protected-resource + RFC 8414 AS metadata |
| 08 | Dynamic Client Registration | RFC 7591, Initial Access Tokens, software statements |
| 09 | End-to-End Sequence | Host → MCP → AS → Resource → Upstream API |
| 10 | Resource Indicators | RFC 8707 audience binding for MCP |
| 11 | DPoP for MCP | Sender-constrained tokens; what the RS must validate |
| 12 | Anti-Patterns | Token passthrough, confused deputy, static keys, log leaks |
| 13 | Docker MCP Gateway | What it is, what problem it solves |
| 14 | Gateway Architecture | Hosts → Gateway → containers → AS |
| 15 | Gateway OAuth Interceptor | Catalogs, secrets, audit, CLI flow |
| 16 | Provider Landscape | At-a-glance matrix of 17+ providers |
| 17 | Commercial Providers | Auth0, Entra, AWS Cognito, Stytch, WorkOS, Clerk, Descope, FusionAuth |
| 18 | OSS Providers | Keycloak, ZITADEL, Authentik, Ory, Authelia, Logto, SuperTokens, Janssen |
| 19 | SaaS vs Self-Host | Trade-offs, when to switch |
| 20 | Choosing a Provider | Decision matrix by use case |
| 21 | Worked Example 1 | claude.ai → public remote MCP → Auth0 |
| 22 | Worked Example 2 | Claude Desktop → Docker Gateway → Keycloak |
| 23 | Production Checklist | RS, host, AS, operational |
| 24 | Summary | Take-aways and references |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono · inline SVG diagrams.

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

MCP specification — modelcontextprotocol.io/specification · Anthropic MCP docs — docs.anthropic.com/mcp · Docker MCP Gateway — github.com/docker/mcp-gateway · Docker MCP Catalog — hub.docker.com/u/mcp · Auth0 MCP guide · ZITADEL MCP guide · Stytch MCP guide · WorkOS MCP · RFC 6749 (OAuth 2.0) · RFC 6750 (Bearer) · RFC 7591 (DCR) · RFC 7636 (PKCE) · RFC 8252 (Native Apps) · RFC 8414 (AS Metadata) · RFC 8628 (Device) · RFC 8693 (Token Exchange) · RFC 8705 (mTLS) · RFC 8707 (Resource Indicators) · RFC 9068 (JWT AT) · RFC 9126 (PAR) · RFC 9396 (RAR) · RFC 9449 (DPoP) · RFC 9700 (BCP 240 — Security BCP) · RFC 9728 (Protected Resource Metadata) · OpenID Connect Core 1.0

## License

Educational use. Code examples provided as-is.
