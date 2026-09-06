# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 出力言語

ユーザーとのやり取りはすべて日本語で行うこと。返答・説明・コメントは必ず日本語で出力する。

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
| `…/soccer/afc.champions` and `…/soccer/afc.cup` | ACL Elite / ACL Two fixtures (see below) |
| `https://api.open-meteo.com/v1/forecast` | Stadium weather forecast |
| `https://r.jina.ai/https://www.reysol.co.jp/ticket/tktscd.php` | Associates presale dates (see below) |
| `https://r.jina.ai/https://www.reysol.co.jp/game/results/` | 天皇杯 / ルヴァン / プレシーズン fixtures (see below) |
| `https://r.jina.ai/https://data.j-league.or.jp/SFMS01/search` | Levain Cup bracket (see below) |
| `https://r.jina.ai/https://www.jfa.jp/match/emperorscup_<year>/match/schedule.json` | Emperor's Cup bracket (see below) |

`TEAM_ID = '7476'` is Kashiwa Reysol's ESPN identifier. J1 requests are rooted at the `BASE` constant.

### Which competition comes from where

The REYSOL日程 and 試合結果 tabs list **every** Kashiwa fixture, not just J1, and each competition has a different source:

| Competition | Source | Why |
|---|---|---|
| J1 | ESPN `jpn.1` | Full data: live scores, goals, box score, standings |
| ACL Elite / ACL Two | ESPN `afc.champions` / `afc.cup` | Same, in separate ESPN leagues |
| 天皇杯, ルヴァンカップ, プレシーズン | reysol.co.jp via `r.jina.ai` | **ESPN has no such league** |

ESPN's soccer catalogue (`sports.core.api.espn.com/v2/sports/soccer/leagues`) registers only `jpn.1` and `jpn.world_challenge` for Japan — there is no Emperor's Cup or Levain Cup league at all, so those fixtures can only come from the club site. Do not go looking for an ESPN slug for them.

The club site is used **only** for what ESPN lacks. It is not a replacement for ESPN: it carries no team ids (so no head-to-head or opponent form), no live status, no goalscorers or box score, no other clubs' fixtures (needed for the standings, the rank chart, and 同じ節の他会場), and no year on its dates.

ESPN's per-team endpoint (`/teams/7476/schedule`) returns nothing under `afc.champions`, so cup fixtures are pulled from that league's `/scoreboard` over the season window and filtered to Kashiwa — `getCupEvents()` / `CUP_LEAGUES`. Events carry a `leagueJP` tag from the fetcher because scoreboard events have no `season.displayName` for `leagueJP()` to read.

Note that the fixture list and the standings tab use **different** sources for the same cup. The fixture list only needs Kashiwa's own matches, which the club site provides; the standings tab needs every club's, which it does not. See the next section.

### 順位表 tab — competition switcher

`COMPETITIONS` drives a chip row (`.comp-nav`) above the table; `switchComp` swaps `compTab` and re-renders `#comp-body`. Each competition is fetched on first selection and memoised in `compCache` (a failed fetch is deleted from the cache so it can be retried). `compToken` discards a response that lands after the user has already moved to another chip.

| Chip | Source | Rendered as |
|---|---|---|
| J1リーグ | `info.standings`, already fetched | `standingsTable` + 順位推移 chart |
| ACLエリート | ESPN `afc.champions/standings` | `standingsTable`, Kashiwa's group first |
| ルヴァンカップ | data.j-league.or.jp via `r.jina.ai` | `bracketHtml` |
| 天皇杯 | jfa.jp schedule JSON via `r.jina.ai` | `bracketHtml` |

Neither cup is in ESPN, and the club site (which the fixture list uses) only carries Kashiwa's own matches — a bracket needs the whole draw, so both come from the organiser instead:

- **Levain**: `data.j-league.or.jp/SFMS01/search?competition_years=<year>&competition_frame_ids=11`. `r.jina.ai`'s default markdown turns the result table into a markdown table, which `parseLevain` reads (cells are link syntax — take the label). Scores look like `0-0 (PK3-1)`, unplayed ones like `vs`. The last column carries `マッチＮｏ［２４］` in full-width digits (the match number) and an undecided slot reads `[39]w` — "winner of No.39".
- **Emperor's Cup**: `www.jfa.jp/match/emperorscup_<year>/match/schedule.json` — the file the official page's jQuery plugin loads. `r.jina.ai` returns it near-verbatim after a few header lines, so `parseEmperor` reads from the first `{`. Undecided ties come through as `No.71の勝者` and undecided dates as `未定`; winners are `homeWinFlag` / `awayWinFlag`, and PK results are `homePKScore` / `awayPKScore`.

Both are keyed by `info.year` (the season's starting year) — **never hard-code the competition year**.

`groupRounds` orders rounds by `ROUND_SEQ`, the knockout progression, **not by date**: Levain's ACL-bound clubs enter at 4回戦, so that round (10/3) is played before 3回戦 (10/14) and a date sort would list them backwards.

`bracketHtml` renders two things — Kashiwa's run through the draw (`.brk-step` cards, one per round it appears in) and then the draw itself as an ordinary left-to-right tournament bracket. `isKashiwa` matches `'柏'` exactly (the J.League site's abbreviation) or a name containing `'柏レイソル'` (JFA's full name).

#### How the bracket is built

Neither source ships a tree, so `linkBracket` reconstructs one. Both give every match a number (`no`), and an undecided slot names the match that feeds it — `No.57の勝者` (JFA) or `[39]w` (J.League); `parseEmperor` / `parseLevain` turn those into `srcH` / `srcA` and display them as `No.57 勝者`. Once a tie is played the reference is **replaced by the winning club's name**, so for the decided part of the draw the edge is instead found by looking up which match in the previous round that club won. Every match therefore ends up with `kidH` / `kidA`.

`bracketLayout` is a plain tree layout over that: a post-order walk from the roots (matches nothing feeds into, deepest column first) gives each leaf the next vertical slot, and every parent sits at the midpoint of its children. A second left-to-right pass re-centres each parent on its children's final positions and pushes apart anything that would overlap inside a column. Columns are round order (`ROUND_SEQ`), **not tree depth** — Levain's ACL-bound clubs enter at 4回戦 with no preceding match, and they belong in the 4回戦 column all the same.

The result is absolutely positioned inside `.brk-canvas` (`BRK_W` / `BRK_H` / `BRK_GAP` in the script are the single source of truth for the geometry, including the SVG connector paths), under a sticky `.brk-heads` row, in a `.brk-wrap` that scrolls in both directions. The Emperor's Cup draw is ~1400 × 2200 px, so `focusBracket()` scrolls Kashiwa's furthest match into the middle right after render.

### Tab system and data loading

Three tabs — 順位表 (standings), 試合結果 (results), and REYSOL日程 (schedule) — are lazy-loaded: a tab's data is fetched only on first activation, tracked by the `loaded` object. The active tab's content is rendered into `#tab-<name>`. `switchTab` maps tab elements to names positionally, so the order of `.tab` elements and the `TABS` array must stay in sync.

The 試合結果 tab absorbed what used to be separate 全試合 and 次の試合 tabs, so everything fixture-related now lives there.

Switching tabs resets scroll to the top, except that REYSOL日程 restores its previous offset from `scrollPos` — the list is long and it is normal to leave it and come back.

### Season detection

J.League moved to an autumn–spring calendar in 2026-27, and ESPN models it as `season=2026, seasontype=4` ("Regular Season", 2026-07-01 → 2027-07-01) alongside the transitional `seasontype=1` (2026 J1 100 Year Vision League). **Never hard-code the season or season type.** `getSeasonInfo()` calls the standings endpoint with no season parameters — ESPN then returns whatever season/type is currently live — and caches `{ year, type, start, label, teamCount, standings }`. `label` ("2026-27") drives the header and section headings.

### Schedule caching

`fetchTeamSchedule(teamId)` merges two calls to `/teams/<id>/schedule`: one pinned to the current season/type, one with ESPN's default (which may still return the previous season during a transition). If neither yields a current-season match, it falls back to `getLeagueScoreboard()` — the league-wide `/scoreboard?dates=…` covering the whole season, cached once and shared by all callers. ESPN rejects date ranges of 366 days or more, so the window is 364 days from the season start.

`getSchedule()` caches Kashiwa's merged events; `getSeasonSchedule()` filters those to the current season via `isCurrentSeason()`. Both cover J1 only — cup fixtures come from `getCupEvents()` and `getOfficialEvents()` and are merged in `buildMwList()`.

### Club-site fixtures (天皇杯 / ルヴァン / プレシーズン)

`loadOfficialEvents()` fetches `https://www.reysol.co.jp/game/results/` through `r.jina.ai` with the header **`x-respond-with: html`** and parses the result with `DOMParser`. The header matters: r.jina.ai's default markdown output silently drops the `.game-meta` block, so every fixture loses its date and kick-off time. The header is allowed through preflight, so the browser can send it.

`OFFICIAL_SECTIONS` names the page anchors to read — `#levain`, `#emperors-cup`, `#preseason`. **`#j1league` and `#acl` are deliberately excluded**: those matches already come from ESPN with better data, and taking both would list them twice.

`officialEvent()` shapes each `.game-item` into an ESPN-like event so `parseEvent()` can read it unchanged, and flags it `synthetic: true`. Synthetic events have no ESPN summary, so `heroCard` refuses to open the detail modal for them, and they have no `oppId`, which is why the head-to-head filter in `fillPreview` requires a non-empty `e.oppId` (otherwise every synthetic match would match every other one).

The page prints "8月26日" with no year, so `officialEvent` resolves it against the season start: a month earlier than the season's first month belongs to the following year. Times may read "現地19:00／日本21:15" — the 日本 value wins. Venues are abbreviations ("三協F柏"), expanded through `VENUE_SHORT_JP` so they line up with `STADIUM_COORDS` for the weather forecast.

Results are cached in `sessionStorage` for `TICKET_TTL`. A failed fetch yields `[]`, so J1 and ACL still render.

### Data flow

1. `getSchedule()` / `getSeasonSchedule()` → raw ESPN events array (ascending by date)
2. `parseEvent(ev)` → normalises each event into a flat object with `ourScore`, `oppScore`, `isHome`, `ourLogo`, `oppLogo`, `oppId`, `completed`, `timeValid`, etc.
3. Tab renderers (`showStandings`, `showResults`, `showSchedule`) consume parsed events and build HTML strings injected via `innerHTML`.

### `mwList` — the shared fixture list

`buildMwList()` produces `mwList`, Kashiwa's fixtures for the season in ascending date order, as `[{ ev, e, round, others }]` — all competitions in one timeline. Both the 試合結果 and REYSOL日程 tabs render from it, so it is built exactly once: `ensureMwList()` memoises the *promise* in `mwListPromise` and returns the season info. It also sets `mwLatest`, the index of the last completed (or in-progress) match.

Only J1 fixtures get a `round` and `others`; anything tagged `leagueJP` (i.e. every cup fixture) is forced to `round: null`, which makes the UI print the competition name in place of 第N節. This is not cosmetic — `roundOf`'s "a Kashiwa match within ±3 days" fallback would otherwise attach a cup match to whatever J1 round happens to sit next to it.

### 試合結果 tab (one match at a time)

`showResults` does not list every result. It consumes `mwList` and shows exactly one entry at a time: a large card for Kashiwa's match, followed by the other matches of the same matchweek (`others`, resolved through `buildRounds`). ◀▶ (`moveMatchweek`) and horizontal swipe (`mwTouchStart` / `mwTouchEnd`) move between fixtures, with a slide-in animation (`.mw-slide.in-left` / `.in-right`). The initial position `mwLatest` is the last completed (or in-progress) match, i.e. 直前の試合; a 直近の試合へ戻る button reappears once you navigate away.

The team schedule endpoint returns only a handful of events right after a season transition, so `mwList` merges Kashiwa's matches out of the league scoreboard with `getSeasonSchedule()` (team-schedule data wins on id collision). Round lookup is by event id, falling back to "a Kashiwa match within ±3 days" if the two sources ever disagree on ids.

For a match that has not been played yet, `fillPreview()` runs after the synchronous render and injects a preview into `#mw-preview`: stadium weather (±2 h around kick-off), head-to-head history for the current season, and a 5-match recent-form row for both clubs — the old 次の試合 tab's content. Everything is relative to the *displayed* fixture, not to today, so form and h2h only count matches before it. Results are cached per event id in `mwPreviewCache`, and `mwToken` discards responses that arrive after the user has already navigated on. Weather is skipped when the kick-off time is not fixed, when the venue is missing from `STADIUM_COORDS`, or when the date is past Open-Meteo's 16-day forecast window (`fetchWeather` returns `null` if fewer than 3 hourly slots match).

### REYSOL日程 tab (vertical season list)

`showSchedule` renders every entry of `mwList` as a compact row (`scheduleRow`) in one continuous timeline, grouped under sticky 年月 headings.

The left column shows 第N節 for J1 and a coloured badge for everything else, keyed by `LEAGUE_BADGE` (`.sch-round.cup` plus a per-competition class). J1 is left as plain text on purpose: it is ~38 of ~50 rows, so badging it too would flatten the contrast that makes cup fixtures easy to spot. The badge names are abbreviated to fit `.sch-meta` (66px, 60px on narrow screens); the plain `.sch-round` carries the same vertical padding as the badge so rows keep a uniform height. Played matches show the score Kashiwa-first plus a W/D/L pill; upcoming ones show the kick-off time (or 時刻未定). A `NEXT` divider marks the boundary at the first unplayed fixture, and the tab scrolls that divider into view on first render — so opening the tab lands on 次の試合 rather than the top of the season. Tapping a row calls `openMatch(i)`, which switches to 試合結果 and opens that fixture; when the results tab has not been loaded yet the index is handed over through `mwPending`, which `showResults` consumes in place of `mwLatest`.

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

English → Japanese translations for team names, venue names, and competition names are handled by three static lookup objects at the top of the script: `TEAM_JP`, `VENUE_JP`, and the `leagueJP()` function. `TEAM_JP` and `VENUE_JP` also cover the ACL opponents and their away grounds. `VENUE_SHORT_JP` is separate: it maps the club site's Japanese abbreviations to full names, not English to Japanese. Stadium GPS coordinates for the weather API are in `STADIUM_COORDS`.

### Match detail modal

Clicking the match card calls `openResultDetail(eventId)`, which looks the parsed event up in `resultEventsMap` and opens a bottom-sheet modal and then asynchronously fetches the ESPN summary endpoint (`/summary?event=<id>`). Goal events are extracted from `scoringPlays`, falling back to `keyEvents` and then `plays` (ESPN returns different shapes depending on the match). Box-score stats (possession, shots, etc.) are rendered from `data.boxscore.teams`.
