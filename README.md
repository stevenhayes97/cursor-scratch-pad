# Sleeper Lineup Advisor (POC)

A start/sit advisor for a Sleeper fantasy football team. A Python script
pulls **objective, structured** signals about your current roster on
demand (no persistence needed — call it fresh every time); a Cursor
**subagent** layers on **qualitative** reasoning (trends, expert consensus,
injury nuance, game-script judgment) via live web search to produce the
actual recommendation.

## Why this split

Struct data (schedule, weather, official injury designation, Vegas
lines) is cheap and reliable to fetch from APIs. "Is this WR trending up,"
"is this RB playing through a nagging injury that's capping his workload,"
and "what do three different analysts think" are not things a scraper does
well — that's exactly what an LLM agent with web search is good at. So:

- **`sleeper_advisor/` (Python)** — gathers the hard data into one JSON/Markdown bundle.
- **`.cursor/agents/lineup-advisor.md` (subagent)** — runs the script, fills the gaps with web research, and writes the final recommendation.

## What it pulls in, and from where

| Signal | Source | Auth needed |
|---|---|---|
| Your roster, starters, official injury status | [Sleeper API](https://docs.sleeper.com/) | No |
| NFL opponent, home/away, kickoff time, venue (dome/outdoor/retractable) | ESPN scoreboard (public, undocumented) | No |
| Weather forecast at kickoff (temp, wind, precip) | [Open-Meteo](https://open-meteo.com/) | No |
| Vegas spread / total / implied team total / game-script flag (blowout risk, low-total grind-it-out game) | [The Odds API](https://the-odds-api.com/) | Yes (free tier: 500 req/mo) |

Everything except The Odds API works with zero configuration. Without an
odds key, you still get schedule/venue/weather/injury data — you just lose
the "weak opponent / takes their foot off the gas" signal, which is
specifically what spread + total data proxies for.

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env
# then fill in .env:
#   SLEEPER_LEAGUE_ID   -- from your league URL: sleeper.com/leagues/<LEAGUE_ID>/team
#   SLEEPER_ROSTER_ID   -- or set SLEEPER_USERNAME instead and it'll be resolved for you
#   ODDS_API_KEY        -- optional, free tier at the-odds-api.com
```

## Usage

Standalone (what the POC's hardcoded values give you today):

```bash
python -m sleeper_advisor.cli --format markdown
python -m sleeper_advisor.cli --format json > context.json
```

Every value can also be passed as a flag, overriding `.env` — this is what
makes the same code reusable behind a future multi-tenant API without
rewriting it:

```bash
python -m sleeper_advisor.cli --league-id <id> --roster-id <id> --week <n> --format json
```

As a subagent (from Cursor chat / Cloud Agent):

```
/lineup-advisor who should I start this week?
```

The subagent runs the script, reads the structured output, does its own
web research to fill in trend/injury/expert-opinion nuance, and returns a
recommendation with reasoning and confidence levels.

## Running tests

```bash
python -m pytest tests/ -v
```

Tests cover the pure/deterministic pieces (stadium reference data, the
game-script classification logic) — no network calls, so they run
anywhere.

## Known limitations (POC scope)

- **ESPN schedule endpoint can lag right at a new season's start.** It's
  been observed to fall back to the most recently *completed* season's
  dates for a given week/year until the new season's schedule is fully
  published on their side. This resolves once the season is underway;
  worth a sanity check against the actual current week if testing in the
  preseason.
- **Injury nuance is shallow by design.** Sleeper's `injury_status` is a
  blunt designation (Questionable/Doubtful/Out) refreshed on the league's
  schedule, not a snap-share or practice-participation trend. The subagent
  is expected to supplement this via web search — the script does not try
  to scrape injury reports itself.
- **Game-script thresholds are heuristics** (`BLOWOUT_SPREAD_THRESHOLD`,
  `LOW_COMPETITIVENESS_TOTAL_CEILING` in `context_builder.py`), not
  statistically fit values. Tune them or have the subagent treat them as
  a prompt, not gospel.
- **The Odds API free tier is limited** (500 requests/month) — fine for a
  personal weekly check, not for a multi-user product without a paid plan
  or caching layer.
- **No persistence, on purpose** — every run hits live APIs fresh, matching
  the requirement that lineups can change week to week and shouldn't need
  a stale cache (the one exception is Sleeper's ~5MB player dictionary,
  cached to `/tmp` for 12h since Sleeper explicitly asks callers not to
  refetch it every request).

## Roadmap: subagent → reusable API + UI

The current design was chosen specifically to make this transition
low-friction:

1. **API layer**: wrap `sleeper_advisor.context_builder.build_context()` in
   a thin FastAPI (or similar) service. League ID / roster ID / week /
   odds key become request parameters instead of env vars — the function
   signature is already shaped for this (see `AdvisorConfig`).
2. **Agent invocation**: replace the manual "run script, then reason"
   subagent flow with a server-side call to the
   [Cursor Cloud Agents API](https://cursor.com/docs/cloud-agent/api/endpoints.md)
   (or `@cursor/sdk`), passing the structured context (or having the agent
   fetch it itself via a tool call) plus the same lineup-advisor prompt.
   This lets other users' requests each spin up an isolated cloud agent run.
3. **Multi-tenancy**: each user supplies their own Sleeper league/roster
   (public, no auth needed on the Sleeper side) and optionally their own
   Odds API key; the backend needs basic per-user config storage and rate
   limiting, but no change to the core advisor logic.
4. **UI**: a simple form (league URL or ID + team name) that calls the API
   and streams/displays the agent's recommendation — this is a thin
   client, all the logic already lives in the API layer above.
