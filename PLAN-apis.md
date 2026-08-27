# Plan — which APIs, for real, for free

Research pass on what Cornell/eBird actually offer, what the eBird key you
generated covers, and whether we can drop every undocumented endpoint.

## TL;DR

- **eBird API 2.0** (your key) is the real, sanctioned, free-for-non-commercial
  API. Use it for everything *except* media.
- **eBird has zero photos/audio.** No endpoint returns media. Never has.
- **Macaulay Library has no API.** Official access is: website browsing, an
  embed-tool iframe, or a Help-Desk ticket with a CSV of catalog numbers.
  The `search.macaulaylibrary.org` call we were using is their internal
  website endpoint — undocumented, unsupported, CORS-blocked. That's the hack
  we're cutting.
- **Merlin has no API.** It's an app; its content is Macaulay + eBird.
- So media has to come from a **documented non-Cornell API**. Cornell's own
  media-request page literally points developers to **iNaturalist** for
  programmatic use.

## What the eBird key gets you

Sanctioned terms: free for non-commercial use, attribution expected,
commercial use needs separate permission (ebird@cornell.edu). Community-known
soft limit ~1,000 requests/day; "excessive behaviour" (bulk-scraping all
checklists) gets the account banned. Sends CORS headers, so it works from a
browser. Key goes in the `X-eBirdApiToken` header or a `key=` param.

Endpoints, by group:

**Observations**
- recent observations for a region
- recent observations near a lat/lng (`data/obs/geo/recent`) — *what we use for the species pool*
- recent notable/rare observations (region or near point)
- recent observations of one species
- nearest place a species was seen
- historic observations on a given date

**Checklists**
- recent checklist submissions for a location/region
- full contents of one checklist

**Hotspots**
- all hotspots in a region
- hotspots near a point
- info for one hotspot (name, coords, species count)

**Regions**
- sub-region lists (country → state → county)
- adjacent regions
- a region's bounding box + info

**Taxonomy**
- full eBird taxonomy: species code, common + scientific name, **family
  (common & scientific), order, banding codes**, category (species/hybrid/…)
- forms/subspecies of a species
- taxonomy version history

**Statistics**
- top-100 observers for a region/date
- regional daily totals (checklists, species, contributors)

**No media anywhere in that list.** Confirmed.

### Can we build the game on just the eBird key?

No. eBird gives us the *questions* (which birds are plausible here) and the
*taxonomy* (for good distractors and the reveal), but the game is showing a
photo and playing a call — and eBird has neither. We need one more source.

## Legit media sources (all documented, all free)

| Source | Gives us | Key? | Browser-callable? | Notes |
|---|---|---|---|---|
| **iNaturalist API** | photos + some sounds + Wikipedia summary | none | yes (CORS ok) | 60 req/min; asks for a custom User-Agent. Media default CC BY-NC; can filter to CC0 / CC-BY / CC-BY-NC. **Cornell points devs here.** |
| **xeno-canto API v3** | bird sounds (the standard archive), rich tags (call vs song, quality A–E) | **yes — free**, since Oct 2025 | partial | Key is per-app and **must not be committed to git** → has to live server-side. |
| **Wikimedia / Wikipedia REST** | canonical lead photo + description text | none | yes | Good for the blurb and a fallback image. |
| **GBIF API** | occurrence photos (aggregates iNat + museums) | none | yes | Quality varies; useful as a photo fallback. |
| **Macaulay Library** | — | — | — | No API. Embed-tool iframes only; needs catalog numbers; not a quiz data source. |

## Two ways to build it without hacks

### Option A — static, keyless, zero setup
- Front-end calls eBird directly (your key sits in `config.local.js`, still
  gitignored). eBird keys are read-only, free, rate-limited, revocable — low
  risk client-side, and this is the sanctioned use.
- Media: **iNaturalist only** (photos + sounds), plus Wikipedia for blurbs.
- No server. `open index.html`. This is today's app minus the dead Macaulay path.
- Cost: iNaturalist's bird-*sound* coverage is thinner than xeno-canto — some
  species will have a photo but no call.

### Option B — small Python server  ← recommended
- `server.py` (Flask, ~120 lines). Secrets in `.env` (gitignored):
  `EBIRD_TOKEN`, `XENOCANTO_KEY`.
- Endpoints the browser calls (and *only* these):
  - `GET /api/species?lat&lng` → eBird `geo/recent`, deduped, joined with the
    cached taxonomy call for family/order.
  - `GET /api/media?code&sci` → iNaturalist photo + xeno-canto audio (fall
    back to iNaturalist audio, then GBIF photo).
  - `GET /api/blurbs?codes=…` → Wikipedia summaries, batched + cached.
- Server-side cache (dict + TTL) keeps us far under eBird ~1k/day and
  iNat 60/min, and makes rounds snappy.
- Front-end drops every third-party `fetch` — it only knows about `/api/*`.
  No keys, no external calls, no CORS anywhere in the client.
- Cost: you run `python server.py` before playing. Sets up cleanly for a
  real deploy later (same server behind gunicorn, or port the handlers to a
  serverless function).

### Why B

It's the only option that (1) uses each organisation's API the way they
document, (2) keeps the xeno-canto key out of git as their terms require,
(3) gets proper bird-sound coverage, (4) is a straight line to deployment.
Option A is the fallback if zero-setup matters more than sound quality.

## Attribution (required by all of them)

- eBird: cite eBird / Cornell Lab of Ornithology as the data source.
- iNaturalist: show observer name + license per photo/sound (already in the JSON).
- xeno-canto: show recordist + license + a link back to the recording page.
- Wikipedia: link the article.

Put a small "Sources" line in the footer and per-media credit in the caption
strip (the caption strip already does this).

## Proposed next steps

1. Rip out the Macaulay code paths (search + CDN helpers) from `index.html`.
2. Build Option B: `server.py`, `requirements.txt`, `.env.example`, update
   `.gitignore` (`.env`), point the front-end at `/api/*`.
3. Add the cached eBird taxonomy join → family-matched distractors.
4. Keep iNaturalist as the photo source; xeno-canto as the primary audio
   source with iNaturalist audio as fallback.
5. Update `PLAN.md` / `README` for the new "run the server" flow.
