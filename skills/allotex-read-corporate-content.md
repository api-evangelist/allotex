---
name: Read Allotex corporate and clinical content
description: >-
  Retrieve Allotex Inc.'s own account of its science, device history, procedure, advisory board and
  the conditions it treats, from the WordPress REST content API behind us.allotex.com. Read-only,
  no credentials.
api: openapi/allotex-content-openapi.yml
base_url: https://us.allotex.com/wp-json
operations:
  - getRouteIndex
  - listPages
  - getPage
  - searchContent
  - listTypes
generated: '2026-08-06'
method: generated
---

# Read Allotex corporate and clinical content

Allotex Inc. is an ophthalmic biologics and medical device company. It runs **no developer
program**. The only credential-free machine-readable surface it exposes is the WordPress REST API
behind its marketing site, and the pages collection is the part worth reading: unlike many
Elementor-built sites, `content.rendered` is populated, so the full prose of every corporate page
comes back over the API.

## Before you start

- **No authentication.** Every step below works anonymously. Do not send an `Authorization`
  header — the only scheme the site advertises is WordPress application passwords, and Allotex
  operates no public signup, so you cannot obtain one.
- **No rate-limit headers.** Nothing signals a budget in-band. Cloudflare and Kinsta front the
  origin. Space requests out; do not parallel-fetch the whole site.
- **This is marketing content.** No patient data, no device telemetry, no clinical records. Treat
  every clinical or regulatory statement as the company's own claim, and never as medical advice.

## Step 1 — confirm the surface is still there (`getRouteIndex`)

```
GET https://us.allotex.com/wp-json/
```

Check that `namespaces` still contains `wp/v2`. The route table on this install is governed by
plugin activation, not by a versioning policy, so namespaces come and go without notice. If
`wp/v2` is gone, stop — nothing below will work.

## Step 2 — list the pages (`listPages`)

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent,modified
```

Expect 16 records (`X-WP-Total: 16` as of 2026-08-06). Read `X-WP-TotalPages` and stop paging
there; requesting a page beyond it returns `400 rest_post_invalid_page_number`.

**Do not key on `slug`.** The slugs on this site do not match the titles. `Our Team` lives at
`/about-us/leadership/`, `Medical Advisory Board` at `/about-us/clinical-leaders/`,
`Conditions We Treat` at `/about-us/founders/`, `Careers` at `/about-us/our-team/`, and `History`
at `/tissue-processing/eu-footer/`. Match on `title.rendered` or on the ids below.

## Step 3 — fetch the pages that carry substance (`getPage`)

```
GET /wp/v2/pages/{id}
```

The substantive ids, verified 2026-08-06:

| id | Title | What it contains |
|---|---|---|
| 11285 | Science | How the TransForm inlay works, biocompatibility, why tissue addition over ablation |
| 11286 | History | Dated timeline of the procedure from Barraquer 1949 through Allotex 2016 |
| 11292 | Conditions We Treat | Presbyopia and hyperopia, with cited prevalence and clinical outcomes |
| 12145 | Procedure | The step-by-step surgical handling summary |
| 11287 | Refractive Surgeons | The surgeon-facing material |
| 11293 | Medical Advisory Board | Named ophthalmic surgeons and their credentials |
| 12186 | Our Team | Executive leadership and staff, grouped by function |
| 11290 | About Us | The company summary and its two cited references |
| 11284 | Home | The one-paragraph positioning statement |

Read `content.rendered`. It is HTML — strip tags before feeding it to a model.

## Step 4 — or search instead (`searchContent`)

```
GET /wp/v2/search?search=cornea&per_page=20
```

Returns lightweight `{id, title, url, type, subtype}` records across all content types. Verified:
`search=cornea` returns the Medical Advisory Board, Science and Conditions We Treat pages. Follow
the `id` into `getPage` for the body.

## Step 5 — walk the tree, not the list (`listPages` with `parent`)

The pages form a real hierarchy. `11282` (For Europe) parents About Us, Science, Press,
Conferences, Blog and Contact; `11290` (About Us) parents Our Team, Medical Advisory Board,
Conditions We Treat and Careers; `11285` (Science) parents History and Refractive Surgeons.

```
GET /wp/v2/pages?parent=11290&_fields=id,title,link
```

## Error handling

| Status | code | What to do |
|---|---|---|
| 404 | `rest_post_invalid_id` | The id does not exist. Re-list; do not retry. |
| 404 | `rest_no_route` | The route was removed (plugin change). Re-fetch the route index. |
| 400 | `rest_post_invalid_page_number` | You paged past `X-WP-TotalPages`. Stop. |
| 400 | `rest_invalid_param` | Read `data.details[<param>].message`. `per_page` max is 100. |
| 401 | `rest_forbidden_context` | You sent `context=edit`. Use the default `view`. |
| 401 | `rest_forbidden` | Permanently gated. **Do not retry with credentials** — none are obtainable. |

Errors are `{code, message, data.status}` on `application/json`. This is **not** RFC 9457
problem+json; do not look for `type`/`title`/`detail`.

## What you will not find

There are no custom post types. Allotex publishes zero structured records about its own
device — no Product, no ClinicalTrial, no Publication entity. The TransForm allograft exists only
as prose inside `content.rendered`. If you need structured device or trial data, this API cannot
give it to you; go to ClinicalTrials.gov instead.
