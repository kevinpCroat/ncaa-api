# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a free API that scrapes and returns consumable JSON data from ncaa.com. It provides live scores, stats, rankings, standings, schedules, and game details (box scores, play-by-play, etc.) for NCAA sports.

Built with [ElysiaJS](https://elysiajs.com/) (a Bun web framework) and uses linkedom for HTML parsing.

## Commands

### Development
- `bun run dev` - Start development server with hot reload (watches src/index.ts)
- `bun run start` - Start production server
- `bun test` - Run test suite
- `bun run lint` - Run Biome linter with auto-fix

### Testing
Run tests with `bun test`. Tests are in `test/index.test.ts`.

## Architecture

### Data Source Strategy

The API uses a hybrid approach to fetch data from NCAA.com:

1. **New GraphQL API** (2025+ seasons): NCAA.com's GraphQL endpoint at `sdataprod.ncaa.com`
2. **Old JSON API** (pre-2025 seasons): Static JSON files at `data.ncaa.com/casablanca`
3. **HTML Scraping** (fallback): Parses HTML from ncaa.com pages using linkedom

The scoreboard route automatically determines which endpoint to use based on the date and sport using `doesSupportScoreboardNewApi()` in `src/codes.ts`.

### Key Files

- `src/index.ts` - Main application entry point with all route handlers
- `src/codes.ts` - Sport/division mappings, GraphQL hashes, and helper functions for determining which API to use
- `src/scoreboard/scoreboard.ts` - GraphQL scoreboard fetching and format conversion logic
- `src/scoreboard/types.ts` - TypeScript types for GraphQL responses
- `src/openapi.ts` - OpenAPI specification for documentation

### Route Architecture

Routes are validated in the `resolve` hook which checks against `validRoutes` Map and assigns appropriate caches. The `onBeforeHandle` hook handles cache retrieval before route execution.

**Main route types:**

1. **Scoreboard routes** (`/scoreboard/:sport/*`):
   - Complex logic to handle both new GraphQL and old JSON APIs
   - Uses semaphores to prevent duplicate concurrent requests for the same resource
   - Automatically converts new GraphQL format to old JSON format for backwards compatibility
   - Football playoff routes combine multiple weeks (16-20) into single response
   - 45s cache TTL

2. **Game detail routes** (`/game/:id/*`):
   - Fetches from GraphQL API with sport-specific hashes
   - Hash discovery: tries primary hash first, if response is empty, uses `__typename` to find correct hash
   - Sub-routes: base info, boxscore, play-by-play, scoring-summary, team-stats
   - 45s cache TTL

3. **Scraped routes** (stats, rankings, standings, history):
   - Handled by catch-all route at bottom of index.ts
   - Fetches HTML from ncaa.com and parses tables using linkedom
   - `parseTable()` extracts data from standard tables
   - `parseStandings()` handles complex multi-table standings pages
   - 30m cache TTL

4. **Bracket routes** (`/brackets/:sport/:division/:year`):
   - Hybrid approach: HTML parsing + GraphQL API
   - Fetches bracket HTML to extract structure (rounds, regions, bracket type)
   - Attempts to fetch championship games from GraphQL API
   - Merges structure with game data, organizing games by round
   - Falls back to structure-only if no game data available
   - 30m cache TTL

5. **Other routes**:
   - `/schools-index` - Fetches school list from `ncaa.com/json/schools`
   - `/schedule/:sport/:division/*` - Fetches from old JSON API
   - `/schedule-alt/:sport/:division/:year` - Fetches from new GraphQL API
   - `/news/:sport/:division` - Parses RSS feeds with custom CDATA/HTML handling
   - `/logo/:school` - Proxies SVG logos from ncaa.com

### Caching Strategy

Uses `expiry-map` for in-memory caching with two cache durations:
- `cache_45s` - 45 second TTL for live scores and game data
- `cache_30m` - 30 minute TTL for relatively static data (stats, rankings, standings)

Scoreboard routes use both cache key formats (original URL path and week-based) to enable shared caching across different URL formats that return the same data.

### GraphQL Hash Management

NCAA.com's GraphQL API uses persisted queries with SHA-256 hashes. Hashes are sport-specific and stored in `src/codes.ts`:
- `playByPlayHashes` - Play-by-play data hashes
- `boxscoreHashes` - Box score data hashes
- `teamStatsHashes` - Team stats hashes

When a hash returns empty data, the code uses the `__typename` field to identify the correct hash for that sport.

Championship bracket data uses hash `833b812cd33218fbffa93fa81646e843d0b9bbca75283b2a33a0bf6d65ef9d27` (query: `ScoringWidgetChampionship_ncaa`).

### Season Year Logic

NCAA seasons don't align with calendar years. The `getSeasonYear()` function in `src/codes.ts` calculates the season year:
- Months 0-6 (Jan-July) belong to the previous year's season
- Months 7-11 (Aug-Dec) belong to the current year's season
- Example: January 2026 games are part of the 2025 season

### Bracket Implementation

The brackets route uses a hybrid approach combining HTML parsing and GraphQL API:

1. **HTML Structure Parsing**: Uses linkedom to parse bracket page HTML and extract:
   - Bracket type/ID (e.g., `bracket_16` for 16-team tournaments)
   - Round names from `.rounds-header` elements
   - Region names from `.region` elements (for larger brackets)
   - Sport and title metadata

2. **Championship Games Fetching**: Attempts to fetch game data from GraphQL API:
   - Uses `ScoringWidgetChampionship_ncaa` hash
   - Returns array of championship games if available
   - Returns empty array if tournament hasn't occurred or data unavailable

3. **Data Merging**: Combines structure with games:
   - Matches games to rounds by `bracketRound` field
   - Falls back to round name matching if `bracketRound` not present
   - Returns structure-only if no games available (future/unavailable tournaments)

4. **Response Format**:
   ```typescript
   {
     sport: string,
     title: string,
     year: string,
     bracketId: string,
     regions: string[],
     rounds: Array<{
       name: string,
       games: Array<{
         gameId: string,
         startDate: string,
         gameState: string,
         teams: Array<{name, seed, score, winner}>
       }>
     }>
   }
   ```

### Security

Optional custom header authentication via `NCAA_HEADER_KEY` environment variable. When set, all requests must include matching `x-ncaa-key` header.

## Code Style

- Uses Biome for formatting/linting (not Prettier/ESLint)
- Double quotes, 2-space indentation, 100 char line width, always semicolons
- TypeScript strict mode enabled
- Target ES2022 with Node module resolution
