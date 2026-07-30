---
name: la-ruche-qui-dit-oui-authenticate-and-read-membership
description: >-
  Obtain an OAuth 2 access token for the La Ruche qui dit Oui! / Food Assembly
  API and read which assemblies (hives) the member belongs to. Covers the
  refresh-token rotation and the credential-handling rules an agent must follow.
api: la-ruche-qui-dit-oui:food-assembly-api
generated: '2026-07-19'
method: generated
source: openapi/la-ruche-qui-dit-oui-api-openapi.yml
operations:
  - createToken
  - getMemberships
---

# Authenticate and read membership

## Credential-handling rule — read this first

The only documented grant is the **OAuth 2 resource-owner password credentials
grant**. That means token acquisition requires the member's actual username and
password. RFC 9700 recommends against this grant and OAuth 2.1 removes it.

**An agent must never collect, store, or replay end-user passwords.** Have a
human broker the token out of band and hand you the resulting access token. If
you only hold a `refresh_token`, the rotation step below is safe to perform
autonomously.

## Steps

1. **Acquire a token** with `createToken` — `POST /oauth/v2/token/`, JSON body.

   Password grant (human-brokered only):
   `grant_type=password` plus `client_id`, `client_secret`, `username`,
   `password`.

   Refresh grant (agent-safe):
   `grant_type=refresh_token` plus `refresh_token`, `client_id`,
   `client_secret`.

   A success returns `access_token`, `expires_in` (documented example `3600`),
   `token_type` (`bearer`), `refresh_token`, and `scope` — which is `null`. No
   scope registry is published, so you cannot request or verify least privilege;
   the token is all-or-nothing.

2. **Store the new `refresh_token`.** The response returns one on every
   exchange. Persist it and discard the old one.

3. **Read membership** with `getMemberships` — `GET /me/` with
   `Authorization: Bearer <access_token>`.

   The response is `{"isMember": bool, "hivesAsMember": [...]}`. Each hive
   carries `id`, `name`, `status` (for example `open`), and `joinedAt` as an
   ISO 8601 timestamp with offset. If `isMember` is `false`, `hivesAsMember`
   should be treated as empty regardless of what it contains.

4. **Refresh before expiry.** Track `expires_in` from the moment of issue and
   re-run step 1 with the refresh grant. There is no token-introspection
   endpoint and no `/.well-known/oauth-authorization-server` metadata (both
   404), so expiry tracking is entirely client-side.

## Errors

- `400` with `{"error": "invalid_grant"}` — credentials or refresh token
  rejected. Do not retry with the same input; re-authenticate through a human.
- `401` with `WWW-Authenticate: Bearer` — missing, malformed, or expired access
  token. Refresh once, then stop.

## Known drift

On 2026-07-19 the documented token endpoint returned `404` on the live host,
while `GET /me/` still returned a correct `401` challenge. The resource routes
are gated and alive; the documented token route is not currently reachable at
the published path. Rediscover the current token endpoint before integrating.
See `lifecycle/la-ruche-qui-dit-oui-lifecycle.yml`.
