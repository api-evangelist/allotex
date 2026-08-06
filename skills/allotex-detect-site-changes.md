---
name: Detect changes on the Allotex site
description: >-
  Monitor Allotex Inc. for updates without a changelog, a status page or an article feed, by
  polling content modification timestamps and the route index. Read-only, no credentials.
api: openapi/allotex-content-openapi.yml
base_url: https://us.allotex.com/wp-json
operations:
  - listPages
  - listPosts
  - getRouteIndex
  - listTypes
generated: '2026-08-06'
method: generated
---

# Detect changes on the Allotex site

## The problem this solves

Allotex publishes **no changelog, no status page, no RSS feed with entries, and no dated press
releases**. Its Press page carries a single line directing readers to the company's LinkedIn page.
The `/wp/v2/posts` collection is registered but empty (`X-WP-Total: 0`, verified 2026-08-06). So
the usual watch targets do not exist.

What does exist is `modified` on every content record. That is the only change signal available
over the API, and this skill is how to use it.

## Step 1 — the cheap poll (`listPages`)

```
GET /wp/v2/pages?orderby=modified&order=desc&per_page=1&_fields=id,title,link,modified
```

One record, a few hundred bytes. Store `modified`. If it has advanced since your last run,
something on the site changed.

## Step 2 — find what moved (`listPages`)

```
GET /wp/v2/pages?orderby=modified&order=desc&per_page=100&_fields=id,title,link,modified
```

Diff the `modified` timestamps against your stored snapshot. Fetch `content.rendered` only for the
ids that moved — see `skills/allotex-read-corporate-content.md`.

## Step 3 — check whether a news feed ever appears (`listPosts`)

```
GET /wp/v2/posts?per_page=1
```

Read `X-WP-Total`. Today it is `0`. If it ever becomes non-zero, Allotex has started publishing
on-site news and you should switch to polling posts by `date` instead of pages by `modified`.

## Step 4 — watch the contract itself (`getRouteIndex`, `listTypes`)

```
GET /wp-json/
GET /wp/v2/types
```

This surface has no versioning policy. Its shape is governed by which plugins are active on the
Allotex install. Two things worth alerting on:

- **Namespace churn.** Observed 2026-08-06 on us.allotex.com: `oembed/1.0`, `fluent-smtp`,
  `contact-form-7/v1`, `sliderrevolution`, `rankmath/v1` (+ 5 children), `elementor-one/v1`,
  `elementor/v1`, `elementor/v1/documents`, `elementor-ai/v1`, `elementor/v1/feedback`, `mcp`,
  `wp/v2`, `wp-site-health/v1`, `wp-block-editor/v1`, `wp-abilities/v1` — 270 routes across 20
  namespaces. A namespace disappearing means a route you depend on may have gone with it.
- **New post types.** `GET /wp/v2/types` currently registers only the WordPress defaults. If a
  custom type ever appears (a product, trial or publication type), that is the single most
  meaningful change this company could make to its machine-readable surface, and it would mean
  structured device data finally exists.

## Step 5 — the MCP endpoint, if you are tracking agent readiness

```
POST /wp-json/mcp/mcp-adapter-default-server
Accept: application/json, text/event-stream
{"jsonrpc":"2.0","id":1,"method":"tools/list"}
```

Returns `401 rest_forbidden` today. If it ever returns 200, the WordPress MCP Adapter on this
install has been opened up and Allotex has an actual agent surface. See `mcp/allotex-mcp.yml` for
why no MCP claim is made in the meantime.

## Pacing

Step 1 is cheap enough to run daily. Steps 2 and 4 are worth a weekly cadence. No rate-limit
headers are returned on any response, so there is no budget signal — be conservative, and never
poll the full page bodies when the `_fields` projection will tell you whether you need them.

## What you cannot detect this way

Anything Allotex announces on LinkedIn, in trade press, or through a regulatory filing. The 2025
FDA IDE submission and the January 2026 IDE approval are both absent from the site's own content.
For regulatory milestones, watch ClinicalTrials.gov and FDA databases, not this API.

## Error handling

| Status | code | What to do |
|---|---|---|
| 404 | `rest_no_route` | The route was removed. Re-fetch `GET /wp-json/` and re-derive your targets. |
| 400 | `rest_invalid_param` | `per_page` must be 1..100. |
| 401 | `rest_forbidden` | Permanently gated. Do not retry with credentials — none exist for third parties. |
