---
name: Track the Flare Therapeutics news archive
description: >-
  Read the 42-item corporate news archive — press releases, news coverage and scientific
  presentations, 2021-05-13 to 2026-06-30 — filtered by category and date window, with correct
  pagination and incremental sync via modified_after.
api: openapi/flare-therapeutics-content-openapi.yml
operations: [listCategories, listPosts, getPost]
generated: '2026-08-12'
method: generated
source: openapi/flare-therapeutics-content-openapi.yml
---

# Track the Flare Therapeutics news archive

Flare Therapeutics runs no developer program and publishes no API documentation. This skill uses
the anonymously readable WordPress REST content API behind `https://www.flaretx.com/wp-json`, which
is the only machine-readable surface the company exposes.

**Base URL:** `https://www.flaretx.com/wp-json`
**Auth:** none. Send no credentials — the read surface is open and no key exists to obtain.
**Courtesy:** `robots.txt` advertises `Crawl-delay: 10`. No rate-limit headers are returned, so
honor the crawl delay yourself and back off on any non-2xx.

## 1. Resolve the categories first

Call `listCategories` — `GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`.

At harvest time (2026-08-12) four categories were registered:

| id | slug | name | posts |
|----|------|------|-------|
| 3 | `news` | News | 36 |
| 8 | `press-release` | Press Release | 8 |
| 9 | `flaretx` | FlareTx | 3 |
| 1 | `uncategorized` | Uncategorized | 3 |

Resolve ids at runtime rather than hard-coding them — this is a CMS taxonomy and the company can
add a term at any time. The counts sum to more than 42 because a post may carry several categories.

## 2. List the archive

Call `listPosts`:

```
GET /wp/v2/posts?per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,categories
```

- `per_page` maxes at **100**. Asking for more returns `400 rest_invalid_param` with
  `data.details.per_page.code = rest_out_of_bounds` — it does not clamp.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the walk, and follow the
  RFC 8288 `Link: <...>; rel="next"` header rather than incrementing `page` blindly.
- Always send `_fields`. The default post representation carries the full rendered HTML plus roughly
  20KB of `yoast_head` markup per item.

Filter by category with `categories=8`, by date window with `after=`/`before=` (ISO 8601).

## 3. Sync incrementally

Store the highest `modified` you have seen, then on the next run:

```
GET /wp/v2/posts?modified_after=2026-06-30T00:00:00&orderby=modified&order=asc&per_page=100
```

`modified_after` is the correct sync key — `after` filters on publication date and will miss edits
to older releases.

## 4. Fetch one item

Call `getPost` — `GET /wp/v2/posts/{id}` — for the full `content.rendered` body.
`404 rest_post_invalid_id` means the id does not exist or is not published; resolve it through the
collection or `searchContent` first.

## Things that will bite you

- **`author` is a dead end.** Posts carry an integer `author` (20 on the newest release), but
  `/wp/v2/users` returns `401 rest_cannot_access` and this deployment adds no author object to the
  post body. There is no way to resolve an author name anonymously. Do not report one.
- **`featured_media` is 0** on every post sampled, and `tags` is always empty — the `post_tag`
  taxonomy is registered but has no terms. Do not build logic that depends on either.
- **`class_list` carries the category** (for example `category-press-release`), so you can classify
  without a second request.
- **Errors are not RFC 9457.** The envelope is `{code, message, data:{status}}` as
  `application/json`. Branch on `code`, not on message text. See
  `errors/flare-therapeutics-problem-types.yml`.
- **Every response carries `x-robots-tag: noindex`.** The content is open to read; the provider is
  asking not to have the JSON representation indexed. Respect that in anything you publish.
