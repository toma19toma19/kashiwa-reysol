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

`TEAM_ID = '7476'` is Kashiwa Reysol's ESPN identifier. All ESPN requests are rooted at the `BASE` constant.

### Tab system and data loading

Three tabs — 順位表 (standings), 試合結果 (results), 次の試合 (next match) — are lazy-loaded: a tab's data is fetched only on first activation, tracked by the `loaded` object. The active tab's content is rendered into `#tab-<name>`.

### Schedule caching

`getSchedule()` fetches `/teams/7476/schedule` once and caches the result in `scheduleCache`. All functions that need historical match data call `getSchedule()` rather than fetching directly.

### Data flow

1. `getSchedule()` → raw ESPN events array
2. `parseEvent(ev)` → normalises each event into a flat object with `ourScore`, `oppScore`, `isHome`, `ourLogo`, `oppLogo`, `oppId`, `completed`, etc.
3. Tab renderers (`showStandings`, `showResults`, `showNext`) consume parsed events and build HTML strings injected via `innerHTML`.

### Localisation tables

English → Japanese translations for team names, venue names, and competition names are handled by three static lookup objects at the top of the script: `TEAM_JP`, `VENUE_JP`, and the `leagueJP()` function. Stadium GPS coordinates for the weather API are in `STADIUM_COORDS`.

### Match detail modal

Clicking a result card calls `openResultDetail(idx)`, which opens a bottom-sheet modal and then asynchronously fetches the ESPN summary endpoint (`/summary?event=<id>`). Goal events are extracted from `scoringPlays`, falling back to `keyEvents` and then `plays` (ESPN returns different shapes depending on the match). Box-score stats (possession, shots, etc.) are rendered from `data.boxscore.teams`.

### Next-match tab

`showNext` fetches both `teamData` (for `nextEvent`) and the opponent's schedule in parallel. It then builds: weather slots (±2 h around kick-off), head-to-head history from the current season, and a 5-match recent-form row for both clubs.
