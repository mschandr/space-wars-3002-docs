# Database Tables Reference

**Total Tables**: 88
**Last Updated**: 2026-03-08

## Design Philosophy

The database uses a **Profile Pool Pattern** for galaxy instantiation:

1. **Global Profile Pools** are maintained across galaxy instances:
   - `pirate_captains` (100s of unique captain profiles)
   - `crew_members` (333 unique crew profiles)
   - `vendor_profiles` (73 unique vendor instances with personalities)
   - `pirate_factions`, `trading_posts`, `minerals`, `ships`, `plans`

2. When a new galaxy is created:
   - The system randomly **samples** from these global pools
   - Creates galaxy-specific instances (POIs, trading hubs, etc.)
   - Assigns sampled profiles to provide personality & variety

3. When a galaxy is flushed with `galaxy:flush`:
   - **Deletes** all galaxy-specific instances
   - **Preserves** global profile pools for reuse
   - Use `--destroy-global` only to reset the entire game world

---

## Tables Flushed by `galaxy:flush` Command

### Level 1: System & Auth (NOT FLUSHED - Preserved)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `users` | User accounts & authentication | ❌ No |
| `personal_access_tokens` | Laravel Sanctum API tokens | ❌ No |
| `password_reset_tokens` | Password reset tokens | ❌ No |
| `sessions` | Session management | ❌ No |

### Level 1: Ephemeral Data (ALWAYS FLUSHED)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `sessions` | Session state & auth | ✅ Always |
| `cache` | Cache store | ✅ Always |
| `cache_locks` | Cache locks | ✅ Always |
| `jobs` | Queue jobs | ✅ Always |
| `failed_jobs` | Failed queue jobs | ✅ Always |

### Level 2: Framework (NOT FLUSHED - Preserved)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `migrations` | Database migration history | ❌ No |
| `job_batches` | Job batch tracking | ❌ No |
| `telescope_entries` | Laravel Telescope debugging | ❌ No |
| `telescope_entries_tags` | Telescope tags | ❌ No |
| `telescope_monitoring` | Telescope monitoring | ❌ No |

### Level 3: Global Profile Pools & Seed Data (PRESERVED by default, use `--destroy-global` to delete)

**Design**: These are global profile pools sampled randomly when instantiating new galaxies. Preserved across galaxy flushes to maintain variety and personality across instances.

| Table | Purpose | Count | Flushed |
|-------|---------|-------|---------|
| `minerals` | 27 tradeable commodity definitions (Water Ice, Carbon, Iron Ore, etc.) with base prices | 27 | 🟡 Only with `--destroy-global` |
| `ships` | 10 ship blueprints/templates (Sparrow-class, Wraith-class, Leviathan-class, etc.) | 10 | 🟡 Only with `--destroy-global` |
| `plans` | 18 upgrade module plans (fuel tanks, weapons, shields, etc.) | 18 | 🟡 Only with `--destroy-global` |
| `pirate_factions` | Pirate faction definitions (Crimson Raiders, Black Star Syndicate, etc.) | 12 | 🟡 Only with `--destroy-global` |
| `pirate_captains` | Pirate captain profiles - pool for random selection when populating galaxies | 100s | 🟡 Only with `--destroy-global` |
| `crew_members` | Crew member profiles - pool for hireable crew across galaxies | 333 | 🟡 Only with `--destroy-global` |
| `vendor_profiles` | Vendor instances with unique personalities & dialogue - pool for galaxy population | 73 | 🟡 Only with `--destroy-global` |
| `trading_posts` | Vendor templates (35 base templates) used as blueprints | 36 | 🟡 Only with `--destroy-global` |
| `ship_components` | Ship component type definitions | N/A | 🟡 Only with `--destroy-global` |
| `poi_types` | POI type definitions | N/A | ❌ Never |

---

## Galaxy-Specific Tables (ALWAYS FLUSHED with `--galaxy` or full flush)

### Player & Ship Data
| Table | Purpose | Flushed |
|-------|---------|---------|
| `players` | Player accounts within galaxy | ✅ Yes |
| `player_ships` | Ships owned by players | ✅ Yes |
| `player_cargos` | Cargo held on player ships | ✅ Yes |
| `player_plans` | Upgrade plans owned by players | ✅ Yes |
| `player_star_charts` | Star charts discovered by players | ✅ Yes |
| `player_notifications` | Game notifications sent to players | ✅ Yes |
| `player_precursor_rumors` | Rumor data discovered by players | ✅ Yes |
| `player_ship_components` | Installed components on player ships | ✅ Yes |
| `player_ship_fighters` | Fighter squadrons on player ships | ✅ Yes |
| `pilot_lane_knowledge` | Player's known warp lanes | ✅ Yes |

### Player State & Progression
| Table | Purpose | Flushed |
|-------|---------|---------|
| `player_system_knowledge` | Player's system discovery knowledge | ✅ Yes |
| `player_trade_transactions` | Player's trading history | ✅ Yes |
| `player_price_sightings` | Commodity prices seen by player | ✅ Yes |
| `player_vendor_relationships` | Player reputation with vendors | ✅ Yes |
| `player_items` | Player inventory items | ✅ Yes |
| `player_jump_bookmarks` | Player's saved jump coordinates | ✅ Yes |
| `player_contract_reputation` | Player's contract completion reputation | ✅ Yes |

### Galaxy Geography & Points of Interest
| Table | Purpose | Flushed |
|-------|---------|---------|
| `galaxies` | Galaxy instances (procedurally generated) | ✅ Yes |
| `points_of_interest` | All POIs (stars, planets, stations, etc.) | ✅ Yes |
| `sectors` | Grid-based regions within galaxy | ✅ Yes |
| `warp_gates` | Connections between POIs | ✅ Yes |
| `warp_lane_pirates` | Pirate placements on warp lanes | ✅ Yes |

### Trading & Economy
| Table | Purpose | Flushed |
|-------|---------|---------|
| `trading_hubs` | Market stations at POIs | ✅ Yes |
| `trading_hub_inventories` | Hub's commodity stock levels | ✅ Yes |
| `trading_hub_plans` | Plans available for purchase at hubs | ✅ Yes |
| `trading_hub_ships` | Ships available at shipyards | ✅ Yes |
| `salvage_yard_inventory` | Salvage items at salvage yards | ✅ Yes |
| `market_events` | Dynamic market price events | ✅ Yes |
| `vendor_profiles` | Per-hub vendor instances (with personality) | ✅ Yes |
| `trading_posts` | Vendor templates (global, but linked to galaxy data) | ❌ No |

### Crew & Personnel
| Table | Purpose | Flushed |
|-------|---------|---------|
| `crew_members` | Hirable crew pool (per galaxy) | ✅ Yes |
| `crew_assignments` | Crew assignments to trading hubs | ✅ Yes |

### Combat & Encounters
| Table | Purpose | Flushed |
|-------|---------|---------|
| `combat_sessions` | Recorded combat encounters | ✅ Yes |
| `combat_participants` | Participants in combat (players/NPCs/pirates) | ✅ Yes |
| `pirate_fleets` | Pirate fleet instances | ✅ Yes |
| `pirate_cargo` | Cargo on pirate ships | ✅ Yes |
| `pirate_bands` | *(Unknown - check schema)* | ❌ No |

### Colonies & Expansion
| Table | Purpose | Flushed |
|-------|---------|---------|
| `colonies` | Player-controlled colonies | ✅ Yes |
| `colony_buildings` | Buildings within colonies | ✅ Yes |
| `colony_missions` | Colony mission queue | ✅ Yes |
| `colony_ship_production` | Ship production queue at colonies | ✅ Yes |
| `system_defenses` | Orbital & ground defenses at POIs | ✅ Yes |
| `orbital_structures` | Orbital platforms & stations | ✅ Yes |

### Economy & Commodities (Phase 0-3)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `commodities` | Commodity definitions | ❌ No |
| `commodity_ledger_entries` | Immutable trading ledger | ✅ Yes |
| `hub_commodity_stats` | Supply/demand/price tracking | ✅ Yes |
| `resource_deposits` | Mining deposits at POIs | ✅ Yes |
| `economic_shocks` | Price shock events | ✅ Yes |
| `construction_jobs` | Ship construction in progress | ✅ Yes |

### Contracts & Quests (Phase 1-2)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `contracts` | Job board contracts | ✅ Yes |
| `contract_events` | Contract state change audit trail | ✅ Yes |

### Scanning & Exploration
| Table | Purpose | Flushed |
|-------|---------|---------|
| `system_scans` | Scan data for systems | ✅ Yes |

### Exploration & Discovery
| Table | Purpose | Flushed |
|-------|---------|---------|
| `stellar_cartographers` | Star chart vendors at trading hubs | ✅ Yes |
| `jump_plans` | Navigation plotting data | ✅ Yes |

### NPC & Dynamic Entities
| Table | Purpose | Flushed |
|-------|---------|---------|
| `npcs` | Non-player characters | ✅ Yes |
| `npc_ships` | Ships owned by NPCs | ✅ Yes |
| `npc_cargos` | Cargo on NPC ships | ✅ Yes |
| `precursor_ships` | Mysterious precursor ship encounters | ✅ Yes |

### PvP & Competitive
| Table | Purpose | Flushed |
|-------|---------|---------|
| `pvp_challenges` | PvP match challenges between players | ✅ Yes |
| `pvp_team_invitations` | PvP team invitations | ✅ Yes |

### Customs & Security (Phase 6)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `customs_officials` | Customs agents at trading hubs | ✅ Yes |
| `galaxy_customs_records` | Customs interaction records | ✅ Yes |

### Vendor & NPC State (Phase 6+)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `galaxy_vendor_states` | Vendor state changes per galaxy | ✅ Yes |
| `ship_personas` | Crew composition & ship personality | ✅ Yes |

### Blueprints & Construction (Phase 3+)
| Table | Purpose | Flushed |
|-------|---------|---------|
| `blueprints` | Ship construction blueprints | ✅ Yes |
| `blueprint_inputs` | Commodity inputs required for blueprints | ✅ Yes |

### Policy & Governance
| Table | Purpose | Flushed |
|-------|---------|---------|
| `reserve_policies` | Economic reserve policies | ✅ Yes |

### Shipyard Data
| Table | Purpose | Flushed |
|-------|---------|---------|
| `shipyard_inventory` | Ships available at shipyards | ✅ Yes |

---

## Summary

**Total Tables**: 88

| Category | Count | Flushed |
|----------|-------|---------|
| System & Auth | 4 | ❌ Never |
| Framework | 5 | ❌ Never |
| Ephemeral Data | 5 | ✅ Always |
| Global Profile Pools & Seed Data | 10 | 🟡 Only with `--destroy-global` |
| Galaxy-Specific Instances | **60** | ✅ Always |

**Key Principles**:
- **Ephemeral data** (sessions, cache, jobs) is **ALWAYS flushed**
- **Global profile pools** (pirate captains, crew members, vendors, etc.) are **PRESERVED by default**
  - These are sampled randomly when populating new galaxies
  - Maintains variety and personality across galaxy instances
- **User accounts & framework** tables are **NEVER flushed**
- Use `--destroy-global` flag **only** to completely reset the game world (delete all profile pools)

---

## Command Options

```bash
# Flush one galaxy (preserve all global seed data: minerals, ships, plans, pirate factions)
php artisan galaxy:flush --galaxy=51

# Flush one galaxy AND destroy global seed data (careful!)
php artisan galaxy:flush --galaxy=51 --destroy-global

# Flush all galaxies (preserve global data)
php artisan galaxy:flush --force

# Flush all galaxies AND destroy global data
php artisan galaxy:flush --force --destroy-global

# Preview what would be deleted
php artisan galaxy:flush --galaxy=51 --dry-run

# Preview with global destruction
php artisan galaxy:flush --galaxy=51 --destroy-global --dry-run
```
