---
name: Harvest the Allotex media library
description: >-
  Enumerate the 218-item media library behind us.allotex.com — clinical imagery, product
  photography, headshots and logo assets — with their size variants and source URLs. Read-only, no
  credentials.
api: openapi/allotex-content-openapi.yml
base_url: https://us.allotex.com/wp-json
operations:
  - listMedia
  - getMediaItem
  - getOEmbed
generated: '2026-08-06'
method: generated
---

# Harvest the Allotex media library

## Before you start

- No authentication required, and none obtainable.
- **Copyright.** These are Allotex's own assets, including clinical imagery and staff photographs.
  Enumerating the metadata is not a licence to republish. Verify permission before any reuse.
- **Faces.** The library contains headshots of named staff and advisory board members. If you are
  building a dataset, treat these as personal data and exclude them unless you have a reason not
  to.

## Step 1 — enumerate (`listMedia`)

```
GET /wp/v2/media?per_page=100&_fields=id,slug,title,alt_text,media_type,mime_type,source_url,media_details,post,date
```

Expect 218 items (`X-WP-Total: 218` as of 2026-08-06) across 3 pages at `per_page=100`. Read
`X-WP-TotalPages` and stop there — paging past it returns `400 rest_post_invalid_page_number`.

Filter server-side rather than client-side:

```
GET /wp/v2/media?media_type=image&per_page=100
GET /wp/v2/media?mime_type=image/webp&per_page=100
```

## Step 2 — pick the right variant (`getMediaItem`)

```
GET /wp/v2/media/{id}
```

`media_details.sizes` carries WordPress's generated derivatives (`thumbnail`, `medium`, `large`,
`full`, plus theme-registered sizes), each with its own `source_url`, `width` and `height`. Choose
the smallest variant that meets your need — the `full` original is often several megabytes and
there is no CDN transform parameter available.

Use `alt_text` as the caption of record. It is more reliable on this site than `title.rendered`,
which is frequently just the uploaded filename.

## Step 3 — relate assets to pages

`post` is the page id an attachment was uploaded against, or `null` for unattached library items
(most of them). To go the other way, read `featured_media` on a page from
`skills/allotex-read-corporate-content.md`, or expand in one round trip:

```
GET /wp/v2/pages/11285?_embed
```

and read `_embedded['wp:featuredmedia']`.

## Step 4 — embeds instead of raw files (`getOEmbed`)

For an iframe-able representation of any page rather than a bare image:

```
GET /oembed/1.0/embed?url=https%3A%2F%2Fus.allotex.com%2Fprocedure%2F
```

Returns oEmbed 1.0 `{version, provider_name: "Allotex US", provider_url, author_name, title,
thumbnail_url, html}`. `?format=xml` is also accepted here — it is the one endpoint on this API
that is not JSON-only.

## Pacing

No rate-limit headers are returned on any response, so there is no signal to back off on. The
origin sits behind Cloudflare and Kinsta. Three paginated calls plus targeted item fetches is the
whole library; do not parallel-fetch 218 originals.

## Error handling

| Status | code | What to do |
|---|---|---|
| 404 | `rest_post_invalid_id` | The attachment id does not exist. Verified: id 12100 returns this. Re-enumerate. |
| 400 | `rest_invalid_param` | `per_page` must be 1..100. |
| 400 | `rest_post_invalid_page_number` | Stop at `X-WP-TotalPages`. |
| 404 | `rest_no_route` | Route removed by a plugin change; re-fetch `GET /wp-json/`. |

Note that a `source_url` is served from `wp-content/uploads/sites/2/...` — the `sites/2` segment is
the multisite path for us.allotex.com. Assets on the sibling host `allotex.com` use a different
prefix and are demo-theme content, not Allotex material.
