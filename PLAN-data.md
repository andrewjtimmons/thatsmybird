# Plan — Q1: what data do we get about the bird and the guesses?

Inventory of every field the three APIs hand us, what it's good for, and what
it costs to wire in. The current prototype uses maybe 10% of this.

---

## Source 1 — eBird API 2.0 (needs the free token)

### `/v2/data/obs/geo/recent` — the nearby-species feed (we already call this)

Per observation record:

| Field | Example | Use |
|---|---|---|
| `speciesCode` | `norcar` | key for Macaulay media lookup — **essential** |
| `comName` | Northern Cardinal | the answer + the choice labels |
| `sciName` | Cardinalis cardinalis | reveal screen; distractor matching |
| `locName` | Sapsucker Woods | "last seen at…" on the reveal |
| `obsDt` | 2026-08-24 07:11 | "reported near you 2 days ago" |
| `howMany` | 3 (often null) | flavor only |
| `lat` / `lng` | 42.47 / -76.45 | distance-from-you calc |
| `obsValid` / `obsReviewed` | true / false | filter out sketchy records |
| `subId` | S12345678 | link to the source checklist |

**Thing we currently throw away:** we dedupe by `speciesCode`. If we *count*
duplicates instead, the tally ≈ how often the bird is being reported ≈ how
common/expected it is locally. That single number unlocks difficulty tuning.

### `/v2/ref/taxonomy/ebird` — taxonomy reference (one call, cache forever)

Per species:

| Field | Example | Use |
|---|---|---|
| `order` | Passeriformes | coarse grouping |
| `familyComName` | Cardinals and Allies | **plausible distractors** — pick wrong answers from the same family |
| `familySciName` | Cardinalidae | reveal screen |
| `taxonOrder` | 30631 | taxonomic adjacency — nearby numbers = similar birds |
| `bandingCodes` / `comNameCodes` | NOCA | 4-letter alpha code, birder catnip |
| `category` | species / slash / spuh / hybrid | drop non-species junk from the pool |

### Other eBird endpoints worth knowing

- `/v2/product/spplist/{regionCode}` — everything ever recorded in a region (codes only). Bigger pool for a "hard mode" that isn't limited to the last 30 days.
- `/v2/ref/hotspot/geo` — birding hotspots near a point. Could let the user pick "quiz me on the birds of \<hotspot\>".

### What eBird does **not** give

No images, no audio, no range maps, no size/description text, no "cool facts."
All of that is in All About Birds / Birds of the World, which have **no open API**.

---

## Source 2 — Macaulay Library search (unofficial, no auth)

`search.macaulaylibrary.org/api/v2/search?taxonCode=…&mediaType=photo|audio`

Per asset (field names observed, not documented — parse defensively):

| Field | Use |
|---|---|
| `assetId` / `catalogId` | builds the CDN URL for image or audio |
| `mediaType` | photo / audio / video |
| `age` / `sex` | reveal caption: "Adult male" |
| `behaviors` | "Singing", "In flight", "Feeding" |
| `location` / `latitude` / `longitude` | "photographed in Ontario" |
| `obsDttm` | when the shot/recording was made |
| `userDisplayName` | **photographer/recorder credit — we should display this** (Macaulay terms expect attribution) |
| `rating` / `ratingCount` | quality sort (we already sort by this) |
| `width` / `height` | layout |
| `assetLength` | audio clip duration |
| `ebirdChecklistId` | link back to the checklist |

CDN URL patterns:
- photo: `cdn.download.ams.birds.cornell.edu/api/v2/asset/{id}/1200`
- audio: `cdn.download.ams.birds.cornell.edu/api/v2/asset/{id}/audio`

---

## Source 3 — iNaturalist (public, no auth, CORS-friendly) — the fallback

`api.inaturalist.org/v1/observations?taxon_name=…&photos=true|sounds=true`

Per observation:

| Field | Use |
|---|---|
| `taxon.preferred_common_name` / `taxon.name` | names |
| `taxon.ancestors[]` | full lineage kingdom→species — family/order without a second call |
| `taxon.wikipedia_url` | "read more" link |
| `taxon.conservation_status` | Least Concern … Endangered badge |
| `taxon.default_photo` | a guaranteed thumbnail |
| `photos[]`: `url`, `attribution`, `license_code` | image + required credit |
| `sounds[]`: `file_url`, `attribution`, `license_code` | audio + required credit |
| `place_guess` / `location` / `observed_on` | where/when this obs happened |
| `user.login` | observer credit |
| `quality_grade` | keep only `research` |

Plus `GET /v1/taxa/{id}` → `wikipedia_summary` — a ready-made paragraph of
description text for the reveal screen ("did you know").

---

## Rolled up — what's available for THE BIRD

| Category | Fields | Where from |
|---|---|---|
| Names | common, scientific, 4-letter code | eBird |
| Taxonomy | order, family (common + sci) | eBird taxonomy / iNat ancestors |
| Local context | last reported date, place, distance from you | eBird geo/recent |
| Abundance | count of recent nearby reports | eBird geo/recent (stop deduping) |
| Photo | URL, dimensions, age/sex, behavior, capture place + date, photographer | Macaulay / iNat |
| Sound | URL, duration, recordist, recording place + date | Macaulay / iNat |
| Blurb | Wikipedia summary paragraph | iNat `/taxa` |
| Conservation | status badge | iNat |

## What's available for THE GUESSES (the 4 choices)

Today: 4 random common names from the deduped pool. On the table:

- **Same-family distractors** — `familyComName` match → "Cardinal vs. Grosbeak vs. Bunting vs. Tanager" instead of "Cardinal vs. Mallard vs. Owl vs. Hummingbird"
- **Taxonomic adjacency** — pick distractors with nearby `taxonOrder`
- **Abundance-based traps** — pair the rare target with a common look-alike, or vice versa
- **Per-choice metadata** — each choice already carries `speciesCode` + `sciName`; the reveal can show family/scientific name of *what you picked* next to the answer
- **Choice thumbnails** — a mini photo per option (costs one extra Macaulay call per choice — the expensive option)

## What no open API gives us

- Merlin's real ID model or confidence scores
- Range-map polygons (Birds of the World, paywalled)
- Curated cool-facts, measurement tables, "similar species" lists (All About Birds, no API)
- Field-mark annotations on photos

---

## Recommendation for v2 (cheap wins first)

1. **Stop deduping** the geo feed — keep the per-species report count. Free.
2. **One taxonomy call**, cached — adds family/order to every species. ~1 request.
3. **Family-matched distractors** using that data. No extra requests.
4. **Attribution line** under the photo/audio (`userDisplayName` / iNat `attribution`). Data already in hand.
5. **Reveal screen** showing: family, alpha code, "last seen \<place\>, \<date\>", photographer. All from data already fetched.
6. Later / optional: Wikipedia blurb (1 call per reveal), conservation badge, choice thumbnails (4 calls per round — only if it feels worth it).

---

## Decided features (build these)

### A. Caption strip under the photo — no inference, no LLM

Read straight from the Macaulay asset JSON we already fetch and print, one
line each, skipping any field that comes back blank:

- **Age / sex** — `age` + `sex` → "Adult male"
- **Behavior** — `behaviors` → "Singing", "In flight", "Foraging"
- **Where** — `location` → "Tompkins County, New York"
- **When** — `obsDttm` → "June 2023" (month + year, not full date)
- **Photo by** — `userDisplayName`

Rules:
- Nothing here runs on load beyond the media fetch that already happens.
- Do **not** read `commonName` / `sciName` from this JSON until after the
  player answers — they're in the same object and would give it away.
- Coverage is partial: ~half of assets have no age/sex or behavior tag. Show
  whatever is present; a round might show all five lines or just where/when/by.
- iNaturalist-fallback media has no clean behavior field — expect
  where / when / by only, usually no age/sex/behavior.

### B. Wikipedia blurb for all 4 choices, shown on answer

On answer click, fetch a plain-English description for **each of the 4
choices** and stack them, correct one highlighted, so the player can learn
what all four birds are.

- Source: iNaturalist `GET /v1/taxa?q={sciName}` → first result →
  `wikipedia_summary`.
- 4 requests per round, fired in parallel, **cached by `speciesCode`** so
  repeat species are instant.
- Strip HTML tags from the summary; trim to ~2 sentences with a "more" toggle.
- Fallbacks: empty summary → Wikipedia REST summary-by-name
  (`en.wikipedia.org/api/rest_v1/page/summary/{name}`, CORS-ok) → if still
  empty, "No description available."
