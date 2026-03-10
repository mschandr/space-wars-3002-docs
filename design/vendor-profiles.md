# Vendor Profiles & Negotiation System

Design document for the vendor personality, relationship, negotiation, and dialogue systems in Space Wars 3002.

---

## Table of Contents

1. [Motivation](#motivation)
2. [Architecture Overview](#architecture-overview)
3. [Vendor Profiles](#vendor-profiles)
   - [Personality Traits](#personality-traits)
   - [Archetypes](#archetypes)
   - [Name Generation](#name-generation)
   - [Vendor-per-Hub Model](#vendor-per-hub-model)
4. [Player-Vendor Relationships](#player-vendor-relationships)
   - [Markup Escalation](#markup-escalation)
   - [Markup Decay](#markup-decay)
   - [Goodwill Milestones](#goodwill-milestones)
5. [Price Formula](#price-formula)
6. [Active Negotiation](#active-negotiation)
   - [Negotiation Roll](#negotiation-roll)
   - [Special Cases](#special-cases)
7. [Crew Pay Negotiation](#crew-pay-negotiation)
8. [Repair Shop Integration](#repair-shop-integration)
9. [Vendor Dialogue (LLM)](#vendor-dialogue-llm)
   - [System Prompt Construction](#system-prompt-construction)
   - [Context Window](#context-window)
   - [Fallback Strategy](#fallback-strategy)
10. [Commentary Integration](#commentary-integration)
11. [Configuration Reference](#configuration-reference)
12. [Database Schema](#database-schema)
13. [File Reference](#file-reference)

---

## Motivation

Before this system, all vendors were faceless — prices were static, there was no personality, and no persistent relationship between players and the merchants they traded with. This system adds economic depth:

- **Each trading hub** gets a unique named vendor with personality-driven pricing.
- **Player behavior matters** — buying without negotiating makes future prices rise.
- **Crew become valuable** — crew stats directly power the negotiation roll.
- **Precursor items are sacred** — haggling over precursor artifacts carries permanent consequences.
- **Vendor dialogue** gives each vendor a distinct voice through an optional self-hosted LLM.

---

## Architecture Overview

```
                          VendorProfileService
                         (generation, name pool)
                                  |
                                  v
    Galaxy Generation -----> VendorProfile (1:1 with TradingHub)
                                  |
          +-----------------------+------------------------+
          |                       |                        |
          v                       v                        v
  VendorNegotiation       VendorDialogue         MerchantCommentary
     Service                 Service                 Service
  (pricing, rolls,        (LLM dialogue,          (vendor-aware
   decay, lockout)         static fallback)         tag scoring)
          |
          v
  PlayerVendorRelationship
  (per-player state: markup, goodwill, lockout)
```

---

## Vendor Profiles

### Personality Traits

Each vendor has four personality traits stored as a JSON column, each ranging from 0.0 to 1.0:

| Trait            | Effect on Gameplay                                                          |
|------------------|-----------------------------------------------------------------------------|
| `greed`          | Higher = larger base markup, faster escalation per purchase                 |
| `business_acumen`| Higher = harder to negotiate against (increases vendor resistance)          |
| `luck`           | Affects negotiation roll swing; high luck benefits vendor, low luck helps player |
| `apathy`         | Higher = faster time-based markup decay (vendor "forgets" escalation)       |

**Why JSON?** New trait dimensions (e.g., `wit`, `patience`) can be added with zero schema migration — just update the generation logic and any service that reads the new trait. The `trait()` helper returns a configurable default (0.5) for traits that don't exist yet.

### Archetypes

Archetypes are derived deterministically from trait values, not stored independently. The derivation logic:

1. **Check balanced first**: If all four traits are between 0.3 and 0.7 → `HONEST_DEALER`
2. **Check compound archetypes**: Two-trait combinations with both traits above threshold:
   - `greed >= 0.6 && business_acumen >= 0.6` → `SHARK`
   - `business_acumen < 0.4 && apathy >= 0.6` → `PUSHOVER`
   - `business_acumen >= 0.6 && apathy < 0.3` → `PARANOID`
   - `luck >= 0.6 && business_acumen < 0.4` → `GAMBLER`
3. **Single dominant trait**: Whichever trait is highest wins. Ties broken by priority order: greed > business_acumen > luck > apathy.

| Archetype       | Dominant Trait(s)                  | Gameplay Feel                        |
|-----------------|-----------------------------------|--------------------------------------|
| OPPORTUNIST     | High greed                        | Aggressive markup, escalates fast    |
| PROFESSIONAL    | High business_acumen              | Fair baseline, tough negotiator      |
| LUCKY_FENCE     | High luck                         | Erratic pricing, occasional deals    |
| INDIFFERENT     | High apathy                       | Barely marks up, barely reacts       |
| SHARK           | High greed + high business_acumen | Max markup, hardest to negotiate     |
| PUSHOVER        | Low business_acumen + high apathy | Easy to negotiate, doesn't escalate  |
| PARANOID        | High business_acumen + low apathy | Remembers everything, slow to decay  |
| GAMBLER         | High luck + low business_acumen   | Wild outcome swings                  |
| HONEST_DEALER   | Balanced (all within 0.3-0.7)     | Moderate markup, fair negotiation    |

### Name Generation

Vendor names are generated from two PHP arrays in `VendorProfileService`:

- ~50 first names (space-themed: Vex, Kira, Torq, Nyx, etc.)
- ~30 last names (Korrath, Voss, Dren, etc.)

This yields 1,500+ unique name combinations — more than enough for even the largest galaxies (840 vendors max).

### Vendor-per-Hub Model

Every `TradingHub` has exactly one `VendorProfile` (1:1 relationship via `vendorProfile(): HasOne`). One vendor personality covers all services at a location: mineral trading, salvage yard, ship shop, plans shop, and repairs.

**Galaxy scale:**

| Galaxy Size | TradingHubs | Unique Vendors |
|-------------|-------------|----------------|
| Small       | ~84         | ~84            |
| Medium      | ~252        | ~252           |
| Large       | ~420        | ~420           |
| Massive     | ~840        | ~840           |

Uniqueness comes from float trait values, not just archetype labels. Two "Opportunist" vendors have different exact greed/apathy values, producing different escalation rates and markup ceilings.

---

## Player-Vendor Relationships

Each player has an independent relationship with each vendor they interact with, tracked in `player_vendor_relationships`.

### Markup Escalation

Every time a player buys at the asking price (without negotiating), the vendor's markup for that player increases:

```
delta = vendor.greed * ESCALATION_PER_BUY (0.05)
markup_escalation = min(MAX_ESCALATION=0.50, escalation + delta)
```

A vendor with `greed = 0.80` adds `0.80 * 0.05 = 4%` per purchase. A vendor with `greed = 0.20` adds only `0.20 * 0.05 = 1%` per purchase.

The maximum escalation is capped at 50%.

### Markup Decay

Markup decreases through two independent mechanisms:

**Time decay** (applied lazily on next visit):
```
days = days_since(last_interaction)
decay_factor = max(0, 1 - vendor.apathy * 0.01 * days)
markup_escalation *= decay_factor
```

High-apathy vendors decay faster (they "forget" the escalation). A vendor with `apathy = 0.80` decays at 0.8% per day; one with `apathy = 0.20` decays at 0.2% per day.

**Spend decay** (goodwill milestones):
Described below.

### Goodwill Milestones

For every 5,000 credits spent at a vendor (cumulative), the player earns a goodwill milestone:

```
per milestone:
  markup_escalation *= 0.90  (10% reduction)
  goodwill += 0.05
```

Goodwill is capped at 1.0. This rewards loyal customers while still maintaining the escalation pressure.

---

## Price Formula

The complete price calculation stack for any purchase:

```
vendor_base_markup  = vendor.greed * MAX_GREED_MARKUP (0.15)
relationship_markup = relationship.markup_escalation

quoted_price = base_price * (1 + vendor_base_markup) * (1 + relationship_markup)

crew_discount = best applicable crew getNegotiationDiscount()
final_price   = quoted_price * (1 - crew_discount)
```

**Key design decision:** Crew discount and vendor negotiation stack **multiplicatively** — crew discount is applied first, then negotiation on top. This means crew discounts are always valuable regardless of vendor markup.

**Example:**
- Base price: 1,000 credits
- Vendor greed: 0.60 → markup = 0.60 * 0.15 = 9%
- Relationship escalation: 5%
- Quoted price: 1,000 * 1.09 * 1.05 = 1,144.50
- Crew discount: 2%
- Final price: 1,144.50 * 0.98 = 1,121.61

---

## Active Negotiation

Players can propose a price below the quoted price. The outcome depends on a roll comparing player crew power vs. vendor resistance.

### Negotiation Roll

```
player_power     = max over active ship crew: (cool * luck * experience)
vendor_resistance = vendor.business_acumen * (1 - vendor.apathy)
luck_swing       = (vendor.luck - 0.5) * 0.2
random           = uniform(-0.10, 0.10)

roll = player_power - vendor_resistance + luck_swing + random
```

**Outcomes:**

| Roll      | Outcome   | Price Result                                           |
|-----------|-----------|--------------------------------------------------------|
| > 0.30    | SUCCESS   | Player gets proposed price (floored at 50% of base)   |
| > 0.00    | PARTIAL   | Price = average of quoted and proposed (floored at 50%)|
| <= 0.00   | FAIL      | Price unchanged (no penalty)                           |

The 50% floor prevents items from being negotiated below half their base value.

### Special Cases

**Precursor items** (`is_precursor = true`):
If the player proposes any price below the quoted price, the negotiation immediately fails with `INSULTED` outcome. The vendor permanently increases markup escalation by 1-5% (random within range). This penalty is never forgiven.

**Predatory proposals** (proposed price < 20% of base price, or negative):
The vendor is outraged. Goodwill drops by 0.5 and the player is locked out of this vendor for 24-48 hours (random). During lockout, all negotiation attempts immediately return `LOCKOUT`.

---

## Crew Pay Negotiation

When hiring crew, players can optionally propose a lower pay rate. The crew candidate acts as their own "vendor" — their stats drive the negotiation.

```
candidate_resistance = experience * (1 - (morale - 0.5) * 0.3)
player_power         = player_level / 20
luck_swing           = (candidate.luck - 0.5) * 0.2
random               = uniform(-0.10, 0.10)

roll = player_power - candidate_resistance + luck_swing + random
```

**Outcomes:**
- **roll > 0.30** (`accepted`): Candidate accepts proposed rate
- **roll > 0.00** (`counter`): Candidate counters at average of proposed and rolled
- **roll <= 0.00** (`rejected`): Candidate insists on their rolled rate

**Walk-away:** If `proposed_pay_rate < 0.01`, the candidate refuses entirely and the hire fails.

---

## Repair Shop Integration

Vendor greed affects repair costs. The hull repair rate per point is:

```
vendor_rate = BASE_RATE * (1 + greed * MAX_REPAIR_MARKUP)
```

With defaults: `BASE_RATE = 10`, `MAX_REPAIR_MARKUP = 0.50`:
- Vendor with `greed = 0.00`: 10 credits/point (no markup)
- Vendor with `greed = 0.50`: 12.5 credits/point
- Vendor with `greed = 1.00`: 15 credits/point (maximum)

Component repair costs are similarly affected by the vendor greed multiplier.

---

## Vendor Dialogue (LLM)

An optional self-hosted LLM (Ollama) provides unique voiced dialogue for each vendor. This is purely presentational — all pricing and game outcomes remain deterministic PHP code.

### System Prompt Construction

The system prompt is built deterministically from vendor traits:

```
You are {name}, a vendor at a trading hub in a distant galaxy.
Personality: {archetype_label}
You are {greed_descriptor} (greed=0.72), {acumen_descriptor} (business_acumen=0.45),
{luck_descriptor} (luck=0.31), and {apathy_descriptor} (apathy=0.58).
Keep responses to 1-2 sentences. Stay in character. Respond to this trading interaction.
```

Descriptors are mapped from trait values via threshold buckets:
- `>= 0.7`: "ruthlessly profit-driven" (greed), "shrewd and calculating" (acumen), etc.
- `>= 0.4`: "moderately profit-motivated", "reasonably business-savvy", etc.
- `< 0.4`: "not particularly greedy", "naive about business", etc.

### Context Window

Each LLM call includes the last N turns (default: 10) from the vendor-player dialogue history, providing conversational continuity. Messages are stored in `vendor_dialogue_history` with role (system/user/assistant) and context type.

### Fallback Strategy

The system is designed to work perfectly without the LLM:

1. **Ollama available**: LLM generates dialogue, persisted to history
2. **Ollama unavailable** (timeout, connection refused): Falls back to archetype-specific commentary pool (static strings per archetype)
3. **Ollama disabled** (`OLLAMA_URL=null`): Uses static commentary pool directly
4. **No archetype pool match**: Generic fallback strings

No exception ever bubbles to the caller. The dialogue layer is fire-and-forget.

**Pre-generation:** When a `VendorProfile` is created during galaxy generation, a greeting is pre-generated and cached to avoid cold-start latency on the first player visit.

---

## Commentary Integration

The `MerchantCommentaryService` tag-based scoring system is extended with vendor-aware tags:

### New Tags

| Tag                  | Values                                                        | Source                                            |
|----------------------|---------------------------------------------------------------|---------------------------------------------------|
| `vendor_archetype`   | `opportunist`, `professional`, `shark`, `pushover`, etc.      | From `VendorProfile.archetype`                    |
| `relationship_tier`  | `stranger`, `regular`, `favorite_mark`, `persona_non_grata`   | Derived from relationship state                   |

### Relationship Tier Derivation

| Tier                | Condition                           |
|---------------------|-------------------------------------|
| `stranger`          | 0 interactions                      |
| `regular`           | 5+ interactions                     |
| `favorite_mark`     | markup_escalation > 0.20            |
| `persona_non_grata` | goodwill < -0.5                     |

### New Commentary Pools

Vendor-archetype and relationship-tier combinations produce unique merchant commentary:

- `vendor_archetype:opportunist` — "Another customer, another opportunity..."
- `vendor_archetype:shark` — "I didn't get rich by giving discounts."
- `vendor_archetype:pushover` — "Oh, um, welcome! Everything's negotiable, I guess..."
- `vendor_archetype:indifferent` — "The prices are on the screen."
- `relationship_tier:favorite_mark` — "Ah, my favorite customer returns..."
- `relationship_tier:persona_non_grata` — "Oh. It's you again."

---

## Configuration Reference

All vendor configuration lives in `config/game_config.php` under the `vendor` key:

```php
'vendor' => [
    // Trait keys (extensible — add new traits here)
    'traits' => ['greed', 'business_acumen', 'luck', 'apathy'],

    // Pricing
    'max_greed_markup'       => 0.15,   // Max base markup from greed (15%)
    'max_repair_markup'      => 0.50,   // Greed * this = repair rate premium
    'base_repair_rate'       => 10,     // Credits per hull point (base)

    // Escalation
    'escalation_per_buy'     => 0.05,   // * greed per buy-at-asking
    'max_markup_escalation'  => 0.50,   // Hard cap (50%)

    // Decay
    'markup_decay_rate'      => 0.01,   // Per day * apathy
    'goodwill_milestone'     => 5000,   // Credits spent to earn goodwill tick
    'spend_decay_reduction'  => 0.10,   // Markup * 0.90 per milestone

    // Protection
    'precursor_insult_range' => [0.01, 0.05],  // Permanent penalty range
    'predatory_threshold'    => 0.20,          // Below 20% = predatory
    'lockout_hours'          => [24, 48],      // Random lockout duration

    // Negotiation
    'negotiation_floor'      => 0.50,   // Can never negotiate below 50% of base

    // Crew
    'crew_walkaway_days'     => 7,      // Days before crew candidate reappears

    // Dialogue (LLM)
    'dialogue' => [
        'ollama_url'         => env('OLLAMA_URL', null),
        'model'              => env('OLLAMA_MODEL', 'mistral'),
        'max_context_turns'  => 10,
        'timeout_seconds'    => 10,
    ],
],
```

---

## Database Schema

### `vendor_profiles`

| Column           | Type           | Notes                                      |
|------------------|----------------|--------------------------------------------|
| `id`             | bigint PK      | Auto-increment                             |
| `uuid`           | uuid unique    | External identifier                        |
| `trading_hub_id` | FK unique      | One vendor per hub, cascadeOnDelete         |
| `name`           | string(100)    | Generated full name                        |
| `traits`         | JSON           | `{"greed": 0.72, "business_acumen": 0.45, ...}` |
| `archetype`      | string         | VendorArchetype enum value                 |
| `created_at`     | timestamp      |                                            |
| `updated_at`     | timestamp      |                                            |

### `player_vendor_relationships`

| Column               | Type              | Notes                                  |
|----------------------|-------------------|----------------------------------------|
| `id`                 | bigint PK         |                                        |
| `player_id`          | FK → players      | cascadeOnDelete                        |
| `vendor_profile_id`  | FK → vendor_profiles | cascadeOnDelete                    |
| `goodwill`           | float             | -1.0 to 1.0, default 0.0              |
| `markup_escalation`  | float             | 0.0 to 0.50, default 0.0              |
| `interaction_count`  | int               | Total purchases, default 0            |
| `total_spent`        | int               | Cumulative credits spent, default 0    |
| `locked_until`       | timestamp nullable| Lockout expiry after predatory proposal|
| `last_interaction_at`| timestamp nullable| Drives time-based decay                |
| `created_at`         | timestamp         |                                        |
| `updated_at`         | timestamp         |                                        |

Unique constraint: `[player_id, vendor_profile_id]`

### `vendor_dialogue_history`

| Column               | Type                           | Notes                          |
|----------------------|--------------------------------|--------------------------------|
| `id`                 | bigint PK                      |                                |
| `player_id`          | FK → players                   | cascadeOnDelete                |
| `vendor_profile_id`  | FK → vendor_profiles           | cascadeOnDelete                |
| `role`               | enum(system, user, assistant)  | LLM message role               |
| `content`            | text                           | Message content                |
| `context_type`       | string                         | See context types below        |
| `created_at`         | timestamp                      |                                |
| `updated_at`         | timestamp                      |                                |

Index: `[player_id, vendor_profile_id, created_at]`

**Context types:** `greeting`, `negotiation_propose`, `negotiation_success`, `negotiation_partial`, `negotiation_fail`, `buy_at_asking`, `precursor_insult`, `lockout_warning`

---

## File Reference

### New Files

| File | Purpose |
|------|---------|
| `app/Enums/VendorArchetype.php` | 9 archetypes with `label()`, `dominantTrait()`, `commentaryPool()` |
| `app/Enums/NegotiationOutcome.php` | SUCCESS, PARTIAL, FAIL, INSULTED, LOCKOUT with `label()` |
| `app/Models/VendorProfile.php` | HasUuid, HasFactory, `trait()` helper for JSON access |
| `app/Models/PlayerVendorRelationship.php` | Relationship state, `isLockedOut()` |
| `app/Models/VendorDialogueHistory.php` | LLM message log |
| `app/DataObjects/NegotiationResult.php` | Readonly value object with `toArray()` |
| `app/Services/VendorProfileService.php` | Generation, archetype derivation, name pool |
| `app/Services/VendorNegotiationService.php` | Pricing, negotiation rolls, decay, lockout |
| `app/Services/VendorDialogueService.php` | Ollama HTTP client, system prompt, fallback |
| `app/Http/Controllers/Api/VendorController.php` | GET profile + POST negotiate |
| `database/factories/VendorProfileFactory.php` | States: `opportunist()`, `shark()`, `pushover()`, etc. |
| `database/migrations/2026_02_27_100001_create_vendor_profiles_table.php` | Schema |
| `database/migrations/2026_02_27_100002_create_player_vendor_relationships_table.php` | Schema |
| `database/migrations/2026_02_27_100003_create_vendor_dialogue_history_table.php` | Schema |

### Modified Files

| File | Change |
|------|--------|
| `app/Models/TradingHub.php` | Added `vendorProfile(): HasOne` relationship |
| `app/Services/CoreSystemGenerator.php` | Generates VendorProfile after TradingHub creation |
| `app/Services/GalaxyGeneration/Generators/TradingInfrastructureGenerator.php` | Bulk generates vendor profiles after hub bulk insert |
| `app/Services/SalvageYardService.php` | Vendor markup applied before crew discount |
| `app/Services/ShipyardInventoryService.php` | Vendor markup on ship purchases |
| `app/Http/Controllers/Api/PlansShopController.php` | Vendor markup on plan purchases |
| `app/Services/TradingService.php` | Vendor markup on mineral buy prices |
| `app/Services/ShipRepairService.php` | Vendor-adjusted repair rates |
| `app/Http/Controllers/Api/ShipServiceController.php` | Passes vendor to repair service |
| `app/Http/Controllers/Api/CrewController.php` | Optional pay negotiation at hire |
| `app/Services/MerchantCommentaryService.php` | Vendor archetype + relationship tier tags |
| `app/Console/Commands/GalaxyFlushCommand.php` | Flushes vendor tables |
| `config/game_config.php` | Vendor config block added |
| `routes/api.php` | Vendor + negotiation routes |
