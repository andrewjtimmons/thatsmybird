# That's My Bird — plan

A browser game: show a photo + sound of a bird that actually occurs near you,
give 4 multiple-choice names, keep score.

## The API reality (important)

There is **no official public "Merlin API."** Merlin is an app that talks to
Cornell's internal services. What Cornell *does* expose that we can use:

| Need | Service | Auth | Notes |
|------|---------|------|-------|
| "What birds are near me" | **eBird API 2.0** (`api.ebird.org`) | free API token | Sends `Access-Control-Allow-Origin: *`, so it works from a plain HTML file. Get a key in 10 seconds at https://ebird.org/api/keygen |
| Photos + audio for a species | **Macaulay Library search** (`search.macaulaylibrary.org/api/v2/search`) + asset CDN (`cdn.download.ams.birds.cornell.edu`) | none | **Undocumented / unofficial.** Cornell can change or block it anytime. May or may not send CORS headers depending on the day. |
| Fallback media | **iNaturalist API** (`api.inaturalist.org`) | none | Fully public, CORS-enabled, keyless. Photos *and* sounds. Not Cornell, but keeps the demo alive. |

So the prototype is: **eBird for the local species list, Macaulay Library for
media, iNaturalist as an automatic fallback when Macaulay fails.**

## Flow

1. **Setup screen**
   - Paste eBird API token (saved to `localStorage`).
   - "Use my location" button (browser geolocation) → fills lat/lng, or type them.
   - Start.
2. **Load local species**
   - `GET /v2/data/obs/geo/recent?lat=&lng=&back=30&maxResults=200`
   - Dedupe by `speciesCode` → list of `{code, comName, sciName}`.
   - Need ≥ 4 species or bail with a message.
3. **Round**
   - Pick 4 random species from the list; one is the answer.
   - Fetch media for the answer:
     - Try Macaulay: search `taxonCode` for `mediaType=photo` and `=audio`,
       take a top-rated `assetId`, build CDN URLs
       (`/asset/{id}/1200` for image, `/asset/{id}/audio` for sound).
     - On any failure / empty result → iNaturalist by scientific name.
     - If still no media after a couple of retries with new answers, show whatever we have.
   - Render: photo, `<audio controls>`, 4 buttons (shuffled common names).
4. **Answer**
   - Click → highlight right/wrong, bump score, "Next" button.
5. Repeat. Score shown as `correct / total`.

## Non-goals for the prototype

- No build step, no framework, no server. One `index.html`, vanilla JS.
- No accounts, no persistence beyond the token in `localStorage`.
- No difficulty tuning, no spectrograms, no range maps, no offline.
- Ugly on purpose.

## Known ways it breaks

- Macaulay endpoint shape is guessed; parsing is defensive but may miss.
- Macaulay CORS may block the `fetch` (images/audio still load via tags, but we
  can't get asset IDs) → it silently falls back to iNaturalist.
- Sparse-birding locations (ocean, desert in August) → few species → repetitive.
- eBird `geo/recent` is "recently reported," not "definitely present."

## If it should stop being shitty later

- Cache species + asset lists; preload next round's media.
- Weight distractors by taxonomic similarity (harder, fairer).
- Let the user pick a region code (`US-NY`) instead of lat/lng.
- Add a hotspot picker, streaks, per-species stats, a "why" reveal with range map.
- Proxy Macaulay through a tiny serverless function to kill the CORS risk.
- Swap in Xeno-canto for audio (better sound coverage; needs an API key now).
