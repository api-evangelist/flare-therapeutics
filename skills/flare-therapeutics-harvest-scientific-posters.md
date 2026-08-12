---
name: Harvest Flare Therapeutics scientific posters and presentations
description: >-
  Enumerate the 419-item media library to pull the AACR, ASCO GU, SITC, ENA and ATS poster PDFs and
  data presentations that back the Presentations & Publications page, filtering by media type, MIME
  type and modification date, and selecting the right generated size for images.
api: openapi/flare-therapeutics-content-openapi.yml
operations: [listMedia, getMediaItem, listPages]
generated: '2026-08-12'
method: generated
source: openapi/flare-therapeutics-content-openapi.yml
---

# Harvest Flare Therapeutics scientific posters and presentations

Flare Therapeutics publishes its conference posters and data presentations on the
[Presentations & Publications](https://www.flaretx.com/publications/) page as HTML links. The same
documents are enumerable — with dates, filenames, sizes and MIME types — through the media library
collection on the content API, which is the only place they exist as structured data.

**Base URL:** `https://www.flaretx.com/wp-json`
**Auth:** none.

## 1. Pull the document set

Call `listMedia` filtered to non-image attachments:

```
GET /wp/v2/media?media_type=application&per_page=100&_fields=id,date,modified,slug,title,mime_type,source_url,filename,filesize
```

At harvest time (2026-08-12) the library held **419** attachments — 400 with `media_type=image`,
**19** with `media_type=application`, and none with `media_type=video`. The 19 application items are
the scientific record: AACR, ASCO GU, SITC, ENA and ATS posters as `application/pdf`, plus an ASCO
GU 2026 Phase 1A data presentation as
`application/vnd.openxmlformats-officedocument.presentationml.presentation`.

Narrow further with `mime_type=application/pdf` when you only want posters.

## 2. Pull the imagery

For images, call `listMedia` with `media_type=image` and read `media_details.sizes` on each item.
Fetch the named size you actually need — `medium`, `medium_large`, `large` — instead of
`source_url`, which is the full-resolution original. Some poster originals are print-resolution and
run to tens of megabytes.

## 3. Tie an asset back to its context

- `post` on a media item is the id of the post or page it was uploaded to; `null` means it was
  uploaded straight to the library. Resolve it with `getPost` or `getPage`.
- `listPages` with `slug=publications` gives you the rendered Presentations & Publications page, so
  you can reconcile the ordered human list against the media collection and catch documents that
  exist in the library but are not linked from the page (and vice versa).
- Note that **`featured_media` is 0 on every post** on this deployment, so you cannot walk from a
  press release to its assets that way — match on date and title instead.

## 4. Sync incrementally

`modified_after` works on the media collection exactly as it does on posts:

```
GET /wp/v2/media?media_type=application&modified_after=2026-07-16T00:00:00&per_page=100
```

The newest attachment at harvest time was dated `2026-07-16`.

## Things that will bite you

- `per_page` maxes at 100 and returns `400 rest_invalid_param` above it, not a clamped page.
- `404 rest_post_invalid_id` is the error for a missing attachment — media shares the post id space.
- `filesize` is in bytes and is present on the collection representation, so you can budget a
  download before making it.
- Send `_fields`; the default media representation includes rendered caption and description bodies
  you almost certainly do not need.
- Honour `Crawl-delay: 10` from `robots.txt` when walking 419 items. No rate-limit headers are
  returned and no 429 was observed, which means there is no signal to back off on — pace yourself.
