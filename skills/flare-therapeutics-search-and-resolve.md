---
name: Search Flare Therapeutics content and resolve the hits
description: >-
  Use cross-content search to find material across posts, pages and media without knowing where it
  lives, then map each hit's subtype to a rest_base via the discovery documents and resolve it to
  the full object.
api: openapi/flare-therapeutics-content-openapi.yml
operations: [searchContent, getPost, getPage, getMediaItem, listPostTypes, listTaxonomies]
generated: '2026-08-12'
method: generated
source: openapi/flare-therapeutics-content-openapi.yml
---

# Search Flare Therapeutics content and resolve the hits

The site has no search API of its own and no sitemap of program pages worth crawling. The WordPress
cross-content search endpoint is the right entry point when you know a term — `FX-909`, `PPARG`,
`urothelial`, `Roche` — but not which collection holds it.

**Base URL:** `https://www.flaretx.com/wp-json`
**Auth:** none.

## 1. Search

Call `searchContent`:

```
GET /wp/v2/search?search=FX-909&per_page=100
```

A search for "flare" returned **49** hits at harvest time (2026-08-12). Read `X-WP-Total` for the
real count and page with `page`/`per_page` (max 100).

Each hit is a **pointer**, not an object:

```json
{ "id": 1245, "title": "Flare Therapeutics Presents Updated FX-909 ...",
  "url": "https://www.flaretx.com/...", "type": "post", "subtype": "post" }
```

Constrain with `type` (`post` | `term` | `post-format`) and `subtype` when you already know the
shape you want.

## 2. Map subtype to a collection

Call `listPostTypes` — `GET /wp/v2/types` — and read `rest_base` for the subtype you got back. Do
not hard-code the mapping; read it. At harvest time eleven post types were registered and all of
them were WordPress core:

`post` → `posts`, `page` → `pages`, `attachment` → `media`, plus `nav_menu_item`, `wp_block`,
`wp_template`, `wp_template_part`, `wp_global_styles`, `wp_navigation`, `wp_font_family` and
`wp_font_face`.

## 3. Resolve

Fetch `/wp/v2/{rest_base}/{id}` — `getPost`, `getPage` or `getMediaItem` depending on the subtype.

## 4. Know what search cannot reach

Call `listTaxonomies` — `GET /wp/v2/taxonomies` — and read it for what it discloses. The `post_tag`
taxonomy declares `types: ["post", "team_member"]`, which reveals a **`team_member` post type
registered in the CMS but absent from `/wp/v2/types`**. It is not exposed over REST, so the
leadership roster is structured behind the scenes and unreadable through the API. The same is true
of the drug-program detail on `/pipeline/`, `/fx-909/` and `/pipeline/fx-111/`: there is no custom
post type for programs, so that content exists only as rendered HTML inside the `content.rendered`
field of the corresponding page. Fetch the page and parse the HTML — do not expect a structured
pipeline entity, and do not invent one.

## Things that will bite you

- Search results carry no date, no excerpt and no category — you must resolve to get any of it.
- `type=post` with `subtype=any` (the default) covers posts, pages and media; there is no way to
  search only attachments' file contents.
- Empty collections are real: `tags`, `comments`, `blocks` and `wp_pattern_category` all return
  `X-WP-Total: 0`. A zero here means the site does not use the feature, not that your query failed.
- Errors are the WordPress envelope `{code, message, data:{status}}`, not RFC 9457. `400
  rest_invalid_param` on a bad `type`/`subtype`, `404 rest_no_route` on a bad path.
