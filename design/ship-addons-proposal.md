# Ship Add-Ons — Current System & Proposed Expansion

This document captures the current ship component architecture and a proposed expansion to 28 distinct add-on types.

---

## Current System (8 types, 8 slots)

The existing component system uses 8 `ComponentType` enum values mapped to 8 `SlotType` enum values. Six are core systems (one per ship), two are multi-slot (multiple allowed).

| ComponentType | SlotType | Core/Multi | Effect Keys |
|---|---|---|---|
| WEAPON | WEAPON | multi-slot | `damage`, `max_ammo` |
| SHIELD | SHIELD_GENERATOR | core (1) | `shield_boost` |
| ARMOR | HULL_PLATING | core (1) | `hull_boost` |
| ENGINE | ENGINE | core (1) | _(speed/warp — implicit)_ |
| SENSOR | SENSOR_ARRAY | core (1) | `sensor_boost` |
| FUEL_SYSTEM | REACTOR | core (1) | `fuel_capacity`, `fuel_regen` |
| CARGO | CARGO_MODULE | core (1) | `cargo_boost` |
| UTILITY | UTILITY | multi-slot | _(catch-all)_ |

### Key Files

| File | Purpose |
|---|---|
| `app/Enums/ComponentType.php` | 8-value enum with `label()`, maps to SlotType |
| `app/Enums/SlotType.php` | 8-value enum with `isCoreSystem()`, `isMultiSlot()`, `slotColumn()` |
| `app/Enums/RarityTier.php` | 6 tiers: common, uncommon, rare, epic, unique, exotic |
| `app/Models/ShipComponent.php` | Blueprint model — name, type, effects JSON, rarity, requirements |
| `app/Models/PlayerShipComponent.php` | Installed instance — condition, ammo, upgrade_level, mechanic_bonus |
| `app/Models/SalvageYardInventory.php` | For-sale listings — quantity, price, condition, source |
| `app/Services/ComponentUpgradeService.php` | Upgrade logic — cost scaling, level caps by rarity |
| `app/Services/SalvageYardService.php` | Purchase/sell/uninstall/lazy generation |
| `app/Http/Controllers/Api/ComponentUpgradeController.php` | Upgrade API endpoints |
| `app/Http/Controllers/Api/SalvageYardController.php` | Salvage yard API endpoints |

### Storage Schema

**ship_components** (blueprints):
- `id`, `uuid`, `name`, `type`, `slot_type`, `description`
- `slots_required` (default 1), `base_price`, `rarity`
- `effects` (JSON), `requirements` (JSON), `is_available`
- `max_upgrade_level`, `upgrade_cost_base`, `size_class`

**player_ship_components** (installed instances):
- `id`, `player_ship_id` (FK), `ship_component_id` (FK)
- `slot_type`, `slot_index` (which slot 1-N)
- `condition` (0-100), `ammo`, `max_ammo`
- `is_active`, `upgrade_level`, `mechanic_bonus`
- Unique constraint: `(player_ship_id, slot_type, slot_index)`

**salvage_yard_inventory** (for sale):
- `id`, `trading_hub_id` (FK), `ship_component_id` (FK)
- `quantity`, `current_price`, `condition`, `source` (salvage/manufactured/stolen)

### Effective Stat Calculation

```
PlayerShip.getEffective*() = base_stat + sum(component effects)
```

Per-component effective effect:
```
base_effect
  * (1 + upgrade_level * 0.15)   // upgrade multiplier
  * (condition / 100)            // condition degradation
  * (1 + mechanic_bonus)         // mechanic installation quality
```

### Upgrade Caps by Rarity

| Rarity | Max Upgrade Levels | Notes |
|---|---|---|
| Common | 5-7 | Cheap, high ceiling |
| Uncommon | 3-5 | |
| Rare | 2-3 | |
| Epic | 1-2 | |
| Unique | 1 | |
| Exotic | 0 | Cannot be upgraded |

### Limitations of Current System

- "Utility" is a catch-all with no distinct gameplay identity.
- Engines, warp drives, reactors, and fuel tanks are collapsed into two types (ENGINE, FUEL_SYSTEM).
- No distinction between energy weapons (lasers) and kinetic weapons (missiles/torpedoes).
- No stealth, electronic warfare, or deployable mechanics.
- No specialized cargo (smuggling, bio, refining).

---

## Proposed Expansion (28 types, 6 groups)

Each type has a clear gameplay identity and distinct effect keys. Types within a group share thematic purpose but differ mechanically.

### Propulsion & Power

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Engine** | core | Sublight thrust — determines travel speed and escape velocity | `speed`, `evasion_bonus` |
| **Warp Drive** | core | FTL system — determines jump range and fuel efficiency | `warp_range`, `fuel_efficiency` |
| **Reactor** | core | Power plant — total energy budget for all systems | `power_output`, `overcharge_capacity` |
| **Fuel Tank** | stackable | Raw fuel storage — multiple tanks allowed | `fuel_capacity` |
| **Afterburner** | single | Burst speed at high fuel cost — combat escape/pursuit | `burst_speed`, `fuel_burn_rate` |

### Offense

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Laser Cannon** | multi-slot | Energy weapon — no ammo, draws reactor power, steady DPS | `energy_damage`, `power_draw` |
| **Missile Pod** | multi-slot | Kinetic launcher — ammo-based, high burst, limited supply | `kinetic_damage`, `max_ammo`, `splash_radius` |
| **Torpedo Tube** | multi-slot | Heavy ordnance — very high single-target damage, slow reload | `torpedo_damage`, `max_ammo`, `reload_time` |
| **Mining Laser** | multi-slot | Dual-purpose extraction tool — low combat damage, mineral yield bonus | `mining_yield`, `energy_damage` |
| **Point Defense** | multi-slot | Anti-missile/fighter turret — defensive fire | `intercept_chance`, `fighter_damage` |

### Defense

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Shield Generator** | core | Energy barrier — regenerates over time | `shield_strength`, `shield_regen` |
| **Hull Plating** | core | Armor — flat damage reduction, no regen | `hull_boost`, `damage_reduction` |
| **ECM Suite** | single | Electronic countermeasures — missile evasion, scan jamming | `missile_evasion`, `scan_jamming` |
| **Cloaking Device** | single | Stealth — avoid pirate encounters, surprise attacks | `detection_reduction`, `ambush_bonus` |
| **Flare Launcher** | single | Disposable decoys — ammo-based missile defense | `decoy_chance`, `max_ammo` |

### Sensors & Intel

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Sensor Array** | core | Primary scanners — scan depth, detection range | `sensor_boost`, `scan_range` |
| **Probe Launcher** | single | Expendable remote scanners — scan without traveling | `probe_range`, `max_ammo` |
| **Comm Relay** | single | Extended communications — trade price data at range | `intel_range`, `price_freshness` |

### Cargo & Logistics

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Cargo Module** | stackable | Standard hold expansion — multiple modules allowed | `cargo_boost` |
| **Hidden Compartment** | single | Smuggling hold — invisible to scans, small capacity | `hidden_cargo`, `scan_resistance` |
| **Refrigerated Bay** | single | Perishable/bio cargo — enables special trade goods | `bio_cargo_capacity` |
| **Ore Processor** | single | Refines raw minerals in-flight — higher sell value | `refining_bonus`, `processing_rate` |
| **Tractor Beam** | single | Salvage collection — auto-collect debris, larger salvage radius | `salvage_range`, `salvage_bonus` |

### Special / Utility

| Add-On | Slot Behaviour | Description | Key Effects |
|---|---|---|---|
| **Fighter Bay** | multi-slot | Carrier module — launches NPC fighters in combat | `fighter_capacity`, `fighter_launch_rate` |
| **Colonist Pod** | single | Passenger transport — colonists for colony founding | `colonist_capacity` |
| **Repair Drone** | single | Passive hull repair over time (slow, no dock needed) | `hull_regen`, `component_repair_rate` |
| **Warp Beacon** | single | Deployable — lets you or allies warp to its location | `beacon_range`, `beacon_duration` |
| **Salvage Scoop** | single | Automatic post-combat loot collection bonus | `loot_bonus`, `salvage_speed` |

### Slot Behaviour Key

| Behaviour | Meaning |
|---|---|
| **core** | One per ship. Swapping replaces the existing one. |
| **multi-slot** | Multiple allowed up to the ship's slot count for that type. |
| **stackable** | Like multi-slot but uses generic expansion slots. Each unit adds capacity. |
| **single** | Only one of this specific type allowed, occupies a utility/expansion slot. |

---

## Summary

| Group | Count | Core | Multi | Stackable | Single |
|---|---|---|---|---|---|
| Propulsion & Power | 5 | 3 | 0 | 1 | 1 |
| Offense | 5 | 0 | 5 | 0 | 0 |
| Defense | 5 | 2 | 0 | 0 | 3 |
| Sensors & Intel | 3 | 1 | 0 | 0 | 2 |
| Cargo & Logistics | 5 | 0 | 0 | 1 | 4 |
| Special / Utility | 5 | 0 | 1 | 0 | 4 |
| **Total** | **28** | **6** | **6** | **2** | **14** |
