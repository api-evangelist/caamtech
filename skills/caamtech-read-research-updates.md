---
name: Read CaaMTech research updates
description: >-
  Retrieve CaaMTech's published research announcements and company updates as structured
  JSON from the public WordPress REST API on caam.tech, with pagination, field selection
  and full-text search. No credentials required.
api: openapi/caamtech-wordpress-rest-openapi.yml
base_url: https://caam.tech/wp-json
auth: none
operations:
  - get_wp_v2_posts
  - get_wp_v2_posts_id
  - get_wp_v2_categories
  - get_wp_v2_tags
  - get_wp_v2_search
generated: '2026-08-08'
method: generated
source: >-
  Grounded in openapi/caamtech-wordpress-rest-openapi.yml, which is derived from the route
  index published at https://caam.tech/wp-json/. Every operationId below exists in that spec.
---

# Read CaaMTech research updates

CaaMTech is a pre-clinical psychedelics drug-discovery company. It runs **no developer
programme** — but its web site is WordPress with the REST API left open, so its published
research updates are machine-readable without a credential. That is the surface this skill
uses. Treat it as a **read-only content API**, not a product API.

## Before you start

- Base URL: `https://caam.tech/wp-json`
- Authentication: **none** for the operations below. Do not send credentials.
- Reads are safe and idempotent by HTTP semantics; there is **no** idempotency-key contract
  on this API (see `conventions/caamtech-conventions.yml`). Never attempt a write.
- The host is behind Cloudflare. Send a normal browser-like `User-Agent`; a non-2xx may come
  back as **HTML**, not the JSON error envelope. Always check `Content-Type` before parsing.

## Steps

### 1. List the updates — `get_wp_v2_posts`

```
GET /wp/v2/posts?per_page=20&page=1&_fields=id,slug,link,date,title,excerpt,categories,tags
```

- `per_page` is capped at **100**; asking for more returns `400 rest_invalid_param`.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the walk, and
  follow the RFC 8288 `Link: <...>; rel="next"` header rather than incrementing `page`
  blindly. At the last probe there were 47 posts across 24 pages at `per_page=2`.
- Use `_fields` to keep payloads small; use `_embed=1` when you also want the author, the
  featured image and the terms inlined under `_embedded` instead of making extra calls.

### 2. Search rather than scan — `get_wp_v2_posts` or `get_wp_v2_search`

```
GET /wp/v2/posts?search=psilocin&orderby=date&order=desc
GET /wp/v2/search?search=tryptamine&per_page=20
```

`get_wp_v2_search` spans every public content type (posts *and* pages) and returns a light
`{id, title, url, type, subtype}` shape — use it to locate a resource, then fetch the full
record. Use `get_wp_v2_posts` with `search` when you only want research updates.

### 3. Fetch one update in full — `get_wp_v2_posts_id`

```
GET /wp/v2/posts/{id}
```

- `content.rendered` and `excerpt.rendered` are **HTML**. Strip tags and unescape entities
  before handing the text to a model.
- `date`/`date_gmt` and `modified`/`modified_gmt` let you detect what changed since a prior
  run — prefer `modified_gmt` for incremental syncs.
- An unknown id returns `404 rest_post_invalid_id`.
- Do **not** add `context=edit`; it is refused anonymously with `401 rest_forbidden_context`.

### 4. Resolve the taxonomy — `get_wp_v2_categories`, `get_wp_v2_tags`

Posts carry `categories[]` and `tags[]` as **numeric ids**. Fetch both collections once and
cache the id→name map rather than resolving per post.

## Error handling

| Status | `code` | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | Read `data.details.<param>.message` — it states the real constraint. Fix and retry. |
| 401 | `rest_forbidden` | You touched an administrative route. Back off; do not retry with guessed credentials. |
| 401 | `rest_forbidden_context` | Drop `context=edit`. |
| 404 | `rest_post_invalid_id` | Resolve a live id from the collection route first. |
| 403 + `text/html` | — | Cloudflare edge block, not WordPress. Slow down; do not attempt to evade the challenge. |

Full catalogue: `errors/caamtech-error-codes.yml`.

## Boundaries

- **Read only.** Every write route on this host requires a WordPress Application Password.
  An agent must never attempt one.
- **Attribute the source.** Content retrieved here is CaaMTech's published material; cite
  the `link` field, which is the canonical public URL of the update.
- **Do not treat this as a scientific data API.** It returns web pages and announcements.
  CaaMTech's compound, crystallographic and pharmacological data is published in
  peer-reviewed journals and patents, not through this API.
