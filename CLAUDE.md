# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 出力言語

すべての返答・説明・コメントは必ず日本語で出力すること。

## Running the app

This is a zero-dependency, single-file web app. No build step is needed.

- Open `index.html` directly in a browser, **or**
- Serve it locally to avoid potential CORS issues:
  ```
  npx serve .
  # or
  python -m http.server 8000
  ```

There are no tests, no linter, and no package manager.

## Architecture

Everything lives in one file: `index.html`. CSS, HTML, and JavaScript are all inline. There is no bundler, framework, or external dependency.

### External APIs

| API | Purpose |
|-----|---------|
| `https://site.api.espn.com/apis/site/v2/sports/soccer/jpn.1` | Schedule, standings, and match summaries (ESPN) |
| `https://api.open-meteo.com/v1/forecast` | Stadium weather forecast |
| `https://r.jina.ai/https://www.reysol.co.jp/ticket/tktscd.php` | Associates presale dates (see below) |

`TEAM_ID = '7476'` is Kashiwa Reysol's ESPN identifier. All ESPN requests are rooted at the `BASE` constant.

### Tab system and data loading

Three tabs — 順位表 (standings), 試合結果 (results), and REYSOL日程 (schedule) — are lazy-loaded: a tab's data is fetched only on first activation, tracked by the `loaded` object. The active tab's content is rendered into `#tab-<name>`. `switchTab` maps tab elements to names positionally, so the order of `.tab` elements and the `TABS` array must stay in sync.

The 試合結果 tab absorbed what used to be separate 全試合 and 次の試合 tabs, so everything fixture-related now lives there.

Switching tabs resets scroll to the top, except that REYSOL日程 restores its previous offset from `scrollPos` — the list is long and it is normal to leave it and come back.

### Season detection

J.League moved to an autumn–spring calendar in 2026-27, and ESPN models it as `season=2026, seasontype=4` ("Regular Season", 2026-07-01 → 2027-07-01) alongside the transitional `seasontype=1` (2026 J1 100 Year Vision League). **Never hard-code the season or season type.** `getSeasonInfo()` calls the standings endpoint with no season parameters — ESPN then returns whatever season/type is currently live — and caches `{ year, type, start, label, teamCount, standings }`. `label` ("2026-27") drives the header and section headings.

### Schedule caching

`fetchTeamSchedule(teamId)` merges two calls to `/teams/<id>/schedule`: one pinned to the current season/type, one with ESPN's default (which may still return the previous season during a transition). If neither yields a current-season match, it falls back to `getLeagueScoreboard()` — the league-wide `/scoreboard?dates=…` covering the whole season, cached once and shared by all callers. ESPN rejects date ranges of 366 days or more, so the window is 364 days from the season start.

`getSchedule()` caches Kashiwa's merged events; `getSeasonSchedule()` filters those to the current season via `isCurrentSeason()`.

### Data flow

1. `getSchedule()` / `getSeasonSchedule()` → raw ESPN events array (ascending by date)
2. `parseEvent(ev)` → normalises each event into a flat object with `ourScore`, `oppScore`, `isHome`, `ourLogo`, `oppLogo`, `oppId`, `completed`, `timeValid`, etc.
3. Tab renderers (`showStandings`, `showResults`, `showSchedule`) consume parsed events and build HTML strings injected via `innerHTML`.

### `mwList` — the shared fixture list

`buildMwList()` produces `mwList`, Kashiwa's fixtures for the season in ascending date order, as `[{ ev, e, round, others }]`. Both the 試合結果 and REYSOL日程 tabs render from it, so it is built exactly once: `ensureMwList()` memoises the *promise* in `mwListPromise` and returns the season info. It also sets `mwLatest`, the index of the last completed (or in-progress) match.

### 試合結果 tab (one match at a time)

`showResults` does not list every result. It consumes `mwList` and shows exactly one entry at a time: a large card for Kashiwa's match, followed by the other matches of the same matchweek (`others`, resolved through `buildRounds`). ◀▶ (`moveMatchweek`) and horizontal swipe (`mwTouchStart` / `mwTouchEnd`) move between fixtures, with a slide-in animation (`.mw-slide.in-left` / `.in-right`). The initial position `mwLatest` is the last completed (or in-progress) match, i.e. 直前の試合; a 直近の試合へ戻る button reappears once you navigate away.

The team schedule endpoint returns only a handful of events right after a season transition, so `mwList` merges Kashiwa's matches out of the league scoreboard with `getSeasonSchedule()` (team-schedule data wins on id collision). Round lookup is by event id, falling back to "a Kashiwa match within ±3 days" if the two sources ever disagree on ids.

For a match that has not been played yet, `fillPreview()` runs after the synchronous render and injects a preview into `#mw-preview`: stadium weather (±2 h around kick-off), head-to-head history for the current season, and a 5-match recent-form row for both clubs — the old 次の試合 tab's content. Everything is relative to the *displayed* fixture, not to today, so form and h2h only count matches before it. Results are cached per event id in `mwPreviewCache`, and `mwToken` discards responses that arrive after the user has already navigated on. Weather is skipped when the kick-off time is not fixed, when the venue is missing from `STADIUM_COORDS`, or when the date is past Open-Meteo's 16-day forecast window (`fetchWeather` returns `null` if fewer than 3 hourly slots match).

### REYSOL日程 tab (vertical season list)

`showSchedule` renders every entry of `mwList` as a compact row (`scheduleRow`) in one continuous timeline, grouped under sticky 年月 headings. Played matches show the score Kashiwa-first plus a W/D/L pill; upcoming ones show the kick-off time (or 時刻未定). A `NEXT` divider marks the boundary at the first unplayed fixture, and the tab scrolls that divider into view on first render — so opening the tab lands on 次の試合 rather than the top of the season. Tapping a row calls `openMatch(i)`, which switches to 試合結果 and opens that fixture; when the results tab has not been loaded yet the index is handed over through `mwPending`, which `showResults` consumes in place of `mwLatest`.

### Associates presale dates (🎟)

Upcoming **home** fixtures show the start of アソシエイツ会員先行（一次販売）. Away matches are sold by the opposing club, so nothing is shown for them.

`www.reysol.co.jp` sends no `Access-Control-Allow-Origin`, so the page cannot be fetched from the browser directly. `loadPresales()` goes through **`r.jina.ai`**, which returns the page as markdown *and* echoes the CORS header. `parsePresales()` splits on the `####` per-match headings, reads the fixture date from `［M/D(曜)HH:MM］`, and takes the **first data cell** of the table — the columns are `一次販売 | 二次販売 | 三次販売 | プレリク先行受付日 | 一般発売日`, and the first three sit under a `colspan=3` header reading ｱｿｼｴｲﾂ会員先行発売期間. The result is memoised in `presalePromise` and cached in `sessionStorage` for `TICKET_TTL`.

The official page lists only the matches whose sale is approaching and is updated as the season progresses, so most fixtures are absent from it. Those render as 🎟 販売日程 未発表 in the muted `.tbd` colour. **Never derive a sale date from the fixture date.** An earlier version extrapolated a rule from the three published matches; it was removed because a wrong ticket date causes a missed sale, and three samples cannot justify one — winter-break and midweek fixtures were already producing doubtful output.

Rendering never blocks on the fetch: rows draw immediately as 未発表, and `refreshPresales()` swaps in the real dates when the response lands. If `r.jina.ai` is unavailable everything simply stays 未発表. Once a published sale date has passed, the label becomes 🎟 アソシエイツ先行 発売中.

### Matchweek grouping

ESPN exposes no round/matchweek number for J.League, so `buildRounds()` reconstructs it: events are walked in date order and dropped into the first round that has fewer than (clubs ÷ 2) matches and does not already contain either club. This yields exactly 38 × 10 for a 20-club season and absorbs rescheduled matches into their original round.

### Kick-off times that are not yet fixed

For matches without a confirmed kick-off, ESPN stores a placeholder such as `18:00Z`, which shifts the date forward a day once converted to JST. `eventDateInfo(ev)` checks `competitions[0].timeValid` and, when false, builds the date from the `YYYY-MM-DD` part alone and reports `timeValid: false`, so the UI prints 時刻未定 and skips the weather forecast.

### Localisation tables

English → Japanese translations for team names, venue names, and competition names are handled by three static lookup objects at the top of the script: `TEAM_JP`, `VENUE_JP`, and the `leagueJP()` function. Stadium GPS coordinates for the weather API are in `STADIUM_COORDS`.

### Match detail modal

Clicking the match card calls `openResultDetail(eventId)`, which looks the parsed event up in `resultEventsMap` and opens a bottom-sheet modal and then asynchronously fetches the ESPN summary endpoint (`/summary?event=<id>`). Goal events are extracted from `scoringPlays`, falling back to `keyEvents` and then `plays` (ESPN returns different shapes depending on the match). Box-score stats (possession, shots, etc.) are rendered from `data.boxscore.teams`.
