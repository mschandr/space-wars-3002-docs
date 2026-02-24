# Space Wars 3002 - Documentation Index

Master index of all project documentation, organized by category.

## Directory Structure

```
docs/
├── README.md                         ← you are here
├── api/                              ← API endpoint reference (~150 endpoints)
│   ├── README.md                     ← API index, response format, auth guide
│   ├── authentication.md             ← Register, login, logout, tokens
│   ├── galaxies.md                   ← Galaxy CRUD, map, stats, creation, NPCs
│   ├── players.md                    ← Player CRUD, status, settings
│   ├── navigation-travel.md          ← Location, warp gates, coordinate jumps, fuel
│   ├── ships.md                      ← Active ship, fuel regen, upgrades, damage
│   ├── ship-services.md              ← Repairs, shipyard, salvage yard, plans shop
│   ├── trading.md                    ← Trading hubs, buy/sell minerals, cargo
│   ├── combat.md                     ← PvE pirates, PvP challenges, team combat
│   ├── colonies.md                   ← Establish, manage, buildings, mining
│   ├── scanning-exploration.md       ← System scans, star charts, exploration log
│   ├── world-data.md                 ← POI types, habitable/mineable lookups
│   ├── leaderboards-victory.md       ← Rankings, victory conditions, progress
│   ├── notifications.md              ← List, read, clear, unread count
│   ├── pirate-factions.md            ← Factions, captains, player reputation
│   └── special-content.md            ← Mirror universe, precursor content, market events
├── design/                           ← Game design specs
│   ├── character-sheet.md            ← Player character sheet layout and fields
│   ├── flotilla.md                   ← Flotilla (fleet) system design
│   ├── fog-of-war.md                 ← Fog-of-war implementation design
│   └── merchant-commentary.md        ← Dynamic merchant dialogue system
├── guides/                           ← Workflow and integration guides
│   ├── player-workflow.md            ← End-to-end API workflow for a player session
│   ├── fe-api-mismatch-report.md     ← Frontend/API response format mismatch audit
│   ├── implementation-plan.md        ← 12-phase frontend implementation roadmap
│   └── learning-guide.md             ← File-by-file Svelte 5 learning walkthrough
├── reference/                        ← Lookup tables and glossaries
│   ├── ship-stats.md                 ← Ship class stats, components, upgrade paths
│   └── lexical-jargon.md             ← In-game terminology and lore glossary
└── types/                            ← TypeScript type definitions
    ├── galaxy-creation.ts            ← Galaxy creation request/response types
    └── scanning.ts                   ← Scanning system types
```

## Quick Links

| Looking for... | Go to |
|----------------|-------|
| API endpoint reference | [api/README.md](api/README.md) |
| How a full player session works | [guides/player-workflow.md](guides/player-workflow.md) |
| Ship classes and stats | [reference/ship-stats.md](reference/ship-stats.md) |
| Game terminology | [reference/lexical-jargon.md](reference/lexical-jargon.md) |
| Fog-of-war design | [design/fog-of-war.md](design/fog-of-war.md) |
| FE/API format mismatches | [guides/fe-api-mismatch-report.md](guides/fe-api-mismatch-report.md) |
| Implementation roadmap | [guides/implementation-plan.md](guides/implementation-plan.md) |
| Learning Svelte 5 walkthrough | [guides/learning-guide.md](guides/learning-guide.md) |
| TypeScript types | [types/](types/) |
