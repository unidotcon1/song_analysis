# Music Trends Dashboard: design (DRAFT, not yet approved)

Status: decisions log while grilling. Nothing here is approved for building.

## Intent and why
A Vite web dashboard to find **trends in what people like**, based on **listens** and **time period**,
per genre ("industry" = genre), per artist, and per instrument.
Owner's words: "find trends in what people like based on listen and the time period".

## Owner decisions (their words)
- Vite + React + TypeScript (same stack as `what_we_eat`).
- Colour theme: **Buttermilk `#FFF2BA`** and **Midnight Blue `#0F3C65`** (serif display type like the reference image).
- Data sources: Spotify, SoundCloud, plus Last.fm.
- "Industry" means **genre**.
- Time period: **all three** views: release eras, our own saved history, and Last.fm weekly charts.
- Instruments to start with: Bass, Drums, Guitar, Piano, Trumpet, Vocals, Brazilian drums, bossa nova instruments.
- Use the design agents from `unidotcon1/what_we_eat` (`.github/agents/design-*.md`).
- Owner has a Spotify developer app and Premium. Last.fm key still to create.
- Approach 2 chosen: Vite app + Cloudflare Worker + D1 + scheduled collector.

## Approaches considered
1. Vite + Worker, no database: simple, no history, slow instrument counts.
2. **Chosen:** Vite + Worker + D1 (SQLite) + scheduled job. Same host as `what_we_eat`; history builds over time.
3. Vite + Supabase: familiar but owner pastes SQL and runs a second vendor.

## Constraints found (verified unless marked)
- Spotify Dev Mode (Feb 2026): `popularity` and `followers` removed; `GET /artists/{id}/top-tracks` removed;
  search limit 10; Premium owner needed; 5 users. So Spotify = catalogue, release dates, artwork, not rankings.
  [migration guide](https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide)
- SoundCloud: track `playback_count` exists per third-party guides; new-app registration availability **unverified**.
- Last.fm: free key; methods for tag/chart/artist top lists exist; returned counts and storage terms **unverified**.
- MusicBrainz: instruments exist for all requested names (drums is "drums (drum set)"; vocals is a credit type, not an instrument).
  Brazilian set found: surdo, pandeiro, tamborim, cuica, agogo, reco-reco, zabumba, caixa, berimbau, cavaquinho; violao ~ classical guitar.
  Credit coverage is patchy (2 of 8 sampled bossa nova recordings had credits; tiny sample).
- Cloudflare **Free** plan (checked 2026-11-18): Workers 100k requests/day, 10 ms CPU per invocation;
  D1 5M rows read/day, 100k rows written/day, 5 GB, 10 databases; over-limit = errors, not charges;
  **50 subrequests per invocation, 5 cron triggers per account**. Cost: free.
- Listen counts from different platforms are not comparable: show side by side, never summed.

## Claude's calls (not yet agreed)
- Secrets: local `.env` (git-ignored) + Cloudflare secrets; `.env.example` holds names only.
- Instruments panel shows "based on N credited recordings", never implies full coverage.
- The 50-subrequest cap bounds how much one scheduled run can fetch; see open questions.

## Scope change (owner, 2026-11-18)
Start with **the owner's own artists** (Spotify user "L.Quekster"), not a genre-wide crawl.
Genre-wide coverage (Last.fm top artists per tag) becomes a later phase.
The profile URL cannot be read directly: Dev Mode removed `GET /users/{id}`; the app must log the owner in (OAuth) and use `/me/*`.
Unverified until tested with the owner's app: `GET /me/following?type=artist`, `GET /me/top/artists`, and whether artist `genres` still returns.

Owner confirmed: "my artists" = the artists in the owner's own Spotify playlist **"All"** (created by the owner, **1,306 songs**).
Genres come from those artists. Readable in Dev Mode because the owner owns the playlist.
Batch `GET /artists` was removed, so genres need one call per artist (est. several hundred calls):
the first import runs as a local script (owner starts it), not inside one Worker run (50-subrequest cap).
Playlist items carry `added_at` (unverified in Dev Mode): a free personal time series ("when I added each artist").
Owner should request Spotify's extended streaming history now (arrives in days): true per-play timestamps.

Owner (2026-11-18): trends must also come from **other people**, not only the owner. The world view is required.
Sources for other people: **ListenBrainz sitewide stats** (top artists/recordings with listen counts for this_week, this_month, this_year,
week, month, quarter, year, half_yearly, all_time; also listening/era activity; top 1000 artists per period; auth need and terms unverified),
Last.fm (current charts only, so daily snapshots), SoundCloud (lifetime `playback_count`, so daily snapshots).
Caveat (Claude's call, awaiting owner): these audiences are not "everyone". Label every world chart with its source; never claim all listeners.

## Open questions (grilling)
1. ~~Which genres, and how many artists per genre?~~ Answered: artists and genres from the "All" playlist.
2. Where does the daily collector run (Worker crons vs GitHub Actions)?
3. What does "popular period" mean per artist (peak year of release? peak listening window?).
4. Dashboard views and layout.
5. Anti-goals and success test.
