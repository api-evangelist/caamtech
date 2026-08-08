---
name: Discover the CaaMTech machine-readable surface
description: >-
  Enumerate everything caam.tech exposes to a machine — the WordPress REST namespaces and
  routes, the registered content types and taxonomies, the media library, and the deployed
  (authenticated) MCP adapter — starting from the site's own discovery document.
api: openapi/caamtech-wordpress-rest-openapi.yml
base_url: https://caam.tech/wp-json
auth: none for discovery; WordPress credential required for the MCP route
operations:
  - get_root
  - get_wp_v2_types
  - get_wp_v2_taxonomies
  - get_wp_v2_media
  - get_wp_abilities_v1_abilities
  - get_mcp
  - post_mcp_mcp_adapter_default_server
generated: '2026-08-08'
method: generated
source: >-
  Grounded in openapi/caamtech-wordpress-rest-openapi.yml, derived from
  https://caam.tech/wp-json/. Every operationId listed exists in that spec.
---

# Discover the CaaMTech machine-readable surface

Use this when you need to know **what is actually callable** on caam.tech before writing an
integration. The answer is: the WordPress REST API, and nothing else. There is no product
API, no developer portal, no `/.well-known/` document, and no OpenAPI published by the
company itself.

## Steps

### 1. Start at the discovery document — `get_root`

```
GET /wp-json/
```

Returns 204 KB of JSON describing **155 routes across 10 namespaces**, plus the site name,
description, and the `authentication` block that names the credential type. This is the one
document worth caching; everything else is reachable from it.

Namespaces present at the last probe: `wp/v2` (226 ops), `akismet/v1`, `wp-abilities/v1`,
`objectcache/v1`, `regenerate-thumbnails/v1`, `wpforms/v1`, `wp-site-health/v1`, `mcp`,
`wp-block-editor/v1`, `oembed/1.0`.

### 2. Learn the content model — `get_wp_v2_types`, `get_wp_v2_taxonomies`

```
GET /wp/v2/types
GET /wp/v2/taxonomies
```

Each entry gives the `rest_base` you append to `/wp/v2/`, and which taxonomies apply. This is
how you avoid hard-coding route names. The pre-resolved graph is in
`data-model/caamtech-data-model.yml`.

### 3. Enumerate published assets — `get_wp_v2_media`

```
GET /wp/v2/media?per_page=100&_fields=id,source_url,mime_type,title,date
```

Returns images **and PDFs** attached to the site. Respect the site's terms; this is
CaaMTech's material.

### 4. Check the agent surface — `get_wp_abilities_v1_abilities`, `get_mcp`

```
GET /wp-abilities/v1/abilities
GET /wp-json/mcp
```

The `mcp` namespace advertises a deployed **WordPress MCP Adapter** at
`/wp-json/mcp/mcp-adapter-default-server` accepting `POST`, `GET` and `DELETE`.

### 5. Expect the MCP gate — `post_mcp_mcp_adapter_default_server`

```
POST /wp-json/mcp/mcp-adapter-default-server
Accept: application/json, text/event-stream
{"jsonrpc":"2.0","id":1,"method":"tools/list"}
```

This returns **`401 rest_forbidden`** anonymously. That is the expected, correct outcome —
the tool set is not publicly introspectable. Do not retry, do not guess credentials, and do
not infer a tool list. There is no OAuth metadata to fall back to:
`/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource` both
return 404.

## What is NOT here

Confirmed absent by direct probe on 2026-08-08 (see `well-known/caamtech-well-known.yml`):
`/llms.txt`, `/.well-known/security.txt`, `/.well-known/agent-card.json`,
`/.well-known/agent.json`, `/.well-known/api-catalog`, `/.well-known/openid-configuration`,
`/openapi.json`, `/swagger.json`, `/graphql`, `/api-docs`, `/docs`, `/developers` — all 404.
`api.caam.tech`, `docs.caam.tech`, `developers.caam.tech` and `status.caam.tech` do not
resolve in DNS. There is no npm, PyPI or GitHub presence for CaaMTech.

Do not re-derive these as "probably exists". They were checked.
