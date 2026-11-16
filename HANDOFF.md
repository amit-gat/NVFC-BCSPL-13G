# NVFC BCSPL 13G — weekly data handoff

This repository is a static GitHub Pages site. The app is one file, `index.html`, at the repository root. It loads Tailwind from the CDN and renders with vanilla JavaScript. All data is embedded in that file as `BASELINE_STANDINGS`, `ROSTER`, and `BASELINE_MATCHES`.

Every table and chart is computed from those arrays when the page loads. There is no build step.

The old Google Sheet sync was removed on purpose. Do not re-add it, and do not fetch standings or matches from any external source at runtime.

GitHub Pages serves the `main` branch from `/`. `.nojekyll` is an empty file so Pages publishes the HTML as-is instead of running Jekyll.

Live site: https://amit-gat.github.io/NVFC-BCSPL-13G/

The header badge must read `N Matches Loaded`, where N is `BASELINE_MATCHES.length` (currently 10).

## Source

Each week, transcribe the new BC Soccer Comet screenshots dropped in the Google Drive folder **NVFC BCSPL U13 Game Reports**:

- **Lineup** — starting XI, substitutes, goalkeeper, captain
- **Timeline** — score, goal minutes and halves, scorers, opponent goals, cards
- **League Table** — the full 16-team table, including last-5 form

Use only what is on those screenshots. Open the rendered page afterward and check it against the same screenshots.

## Where to edit

Edit only the data in the `<script>` of `index.html`:

- Append one object to `BASELINE_MATCHES`.
- Replace `BASELINE_STANDINGS` in full (all 16 teams) from the latest league table.
- Change `ROSTER` only when a new player appears. Keep existing ids stable.

Leave the rendering functions alone. `renderAll()` calls `computeStats()` and redraws the KPI row, league table, goal timeline, starts chart, top scorers, 20-minute interval chart, home/away split, goalkeeper cards, bench-impact bar, player leaderboard, and match log. Putting the new numbers in the arrays is what refreshes them.

The venue filter (All Matches / Home / Away) recomputes the KPIs, charts, leaderboard, and match log from the filtered matches. The standings table always shows the full league table.

## Roster ids

Use these ids in `gk`, `cap`, `starters`, `subs`, and `GL` scorer ids.

| Id | Player | Notes |
| --- | --- | --- |
| 1 | Aine Matthews | Goalkeeper |
| 3 | Kate Hargreaves | |
| 4 | Zoe Baker | |
| 7 | Ellie Hewgill | |
| 8 | Molly Webber | |
| 9 | Sara Zanatta | |
| 10 | Nahla McArdle | |
| 11 | Mackenzie Hermkens | |
| 12 | Soleil MacDougall | Captain |
| 13 | Mika Gatenyo | Wore #21 as GK in BCSPL-01; her id stays 13 |
| 14 | Maya Soliman | |
| 15 | Hollie Searles | |
| 16 | Alaina Langston | |
| 17 | Ella Thompson | |
| 19 | Annabelle Gottloeber | |
| 25 | Aurie Granander | |
| 210 | London Cucca | Goalkeeper, displayed as #21 (`dispNum: '21'`) |

Do not create a roster id `21`. London Cucca is `210` so her display number does not collide with Mika Gatenyo's BCSPL-01 jersey. On the BCSPL-01 lineup detail, Mika is shown as #21; everywhere else she is #13. That is a jersey mapping for that one match, not a highlight.

New players get a new numeric id. Do not renumber anyone already in the table.

## Match schema

```js
{
  id: 'BCSPL-NN',          // sequential, zero-padded: BCSPL-01 … next is BCSPL-11
  date: 'YYYY-MM-DD',
  time: '9:30 AM',         // kickoff as printed on the Comet sheet
  opp: 'TSS Rovers 13G',
  venue: 'North Vancouver',
  ha: 'Home',              // 'Home' or 'Away'
  gf: 2,
  ga: 1,
  res: 'W',                // 'W', 'D', or 'L' from NVFC's perspective
  gk: 1,                   // roster id of the starting goalkeeper
  cap: 12,                 // roster id of the captain
  starters: [1, 4, 7],     // Starting XI roster ids, including the GK
  subs: [3, 9],            // bench roster ids
  goals: [GL(min, half, scorerId, note)],
  oppGoals: [OG(min, half, player)],
  cards: [CD(min, half, num, card, team)]
}
```

Helpers already in the file:

- `GL(min, half, scorerId, note)` — NVFC goal. `note` is optional and defaults to `''`.
- `OG(min, half, player)` — opponent goal. `player` is the scorer text from the timeline (for example `'#10 Claudia Jerke'`).
- `CD(min, half, num, card, team)` — card. `team` is `'NVFC'` or `'Opp'`. Omitted team defaults to `'NVFC'`.

Minute strings keep the Comet mark, including the apostrophe: `"8'"`, `"40+2'"`, `"80+1'"`. Use `"—"` when the sheet does not list a minute or scorer. `half` is `1` or `2`. `scorerId` is a roster id, or `0` when the scorer is unlisted (`'Scorer Unlisted in Comet'`).

For an NVFC card, `num` is the roster id. For an opponent card, `num` is the shirt number printed on the sheet and `team` is `'Opp'`.

The 20-minute chart bins goals by half and base minute (stoppage uses the number before `+`):

- Half 1, minute ≤ 20 → 1'–20'
- Half 1, minute > 20 → 21'–40+'
- Half 2, minute ≤ 60 → 41'–60'
- Half 2, minute > 60 → 61'–80+'

A goal with minute `"—"` is omitted from the interval chart. A `scorerId` of `0` still counts toward GF but not toward any player's goals or the XI / bench split.

## Standings schema

Replace the whole `BASELINE_STANDINGS` array every week. All 16 teams, in table order. Do not patch a single row and leave the rest stale.

```js
{ pos, team, isNVFC, mp, w, d, l, gf, ga, gd, pts, last5 }
```

- `pos` — place on the latest league table (1–16).
- `team` — name as shown on the Comet table.
- `isNVFC` — `true` only for NVFC BCSPL 13G. The page highlights that row from this flag.
- `mp`, `w`, `d`, `l`, `gf`, `ga`, `pts` — copied from the table.
- `gd` — goal difference as a signed number (`gf - ga`).
- `last5` — the five most recent results on the table, oldest to newest, each `'W'`, `'D'`, or `'L'`. The standings caption reads "Last 5 form shown oldest to newest".

`pos` is what the KPI "League Standing" card uses. Match-derived record, GF, GA, and GD on the KPI row come from `BASELINE_MATCHES`, so the NVFC standings row and the match list have to describe the same set of games.

## Rules

1. **No duplicates.** Before adding a match, confirm the `id` is unused and that no existing match has the same `date` and `opp`.
2. **Sub goals.** A goal by a player listed in that match's `subs` gets note `'Sub Goal'`. If that goal was the deciding goal, the note is `'Sub Goal - Match-Winner'`. The charts also treat a scorer as a sub goal when their id is not in `starters`, so the note and the arrays have to agree.
3. **Deciding goal.** The deciding goal is the match-winner: the goal that decided the result. Starter match-winners already use the note `'Match-Winner'`. Other notes already in the file are `'Hat-Trick Goal'` and `'Scorer Unlisted in Comet'`.
4. **No special player highlighting.** Every player renders the same way: same row style, same bar colors, same badge treatment. Do not add a color, outline, sort bias, or callout for any player. Shirt numbers come from `dispNum` for everyone.
5. **Refresh and verify.** Every update replaces standings in full and adds the match (or corrects the arrays). Then load the page and check the badge, KPI row, all 16 standings rows, the goal timeline, starts and scorers charts, interval chart, home/away split, goalkeeper cards, leaderboard, and match log against the Lineup, Timeline, and League Table screenshots.

## Weekly steps

1. Take the new Lineup, Timeline, and League Table screenshots from **NVFC BCSPL U13 Game Reports**.
2. Reject the update if that `id` already exists, or if the same date and opponent are already in `BASELINE_MATCHES`.
3. Append the next `BCSPL-NN` object. The current last id is `BCSPL-10` (2026-11-08, TSS Rovers 13G).
4. Replace all 16 objects in `BASELINE_STANDINGS` from the league table, including `last5`.
5. Add a roster row only for a player who is not already listed.
6. Open `index.html` locally, or the Pages URL after `main` updates, and verify it against the screenshots. The badge should show the new match count (11 after the next game).
7. Commit to `main`. Pages publishes the root of `main`.
