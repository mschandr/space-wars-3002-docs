# Space Wars 3002 – Vendor Dialogue System Design
## Joint Design Document for Go Dialogue Generator and PHP Backend API

## Document Purpose

This document defines the architecture, responsibilities, data model, and interaction flow for the vendor dialogue system in **Space Wars 3002**.

It is intended for two implementation agents:

- **Go AI** – responsible for the offline / asynchronous dialogue generator service
- **PHP BE AI** – responsible for the gameplay API integration and dialogue retrieval

This design reflects the following key decision:

> Vendor dialogue should be split into:
> 1. stable vendor-centric dialogue, which can be pre-generated
> 2. item-centric dialogue, which must be composed at runtime from live item facts

This avoids tying dialogue to static vendor inventory, which is invalid because inventory can move between vendors and players.

---

# 1. Core Design Principles

## 1.1 Dialogue must not block gameplay
No LLM calls may occur in the gameplay request path.

## 1.2 Vendor voice is stable, item facts are dynamic
The system should pre-generate the **vendor's voice**, not pre-generate dialogue for every exact item instance.

## 1.3 Inventory is movable
A vendor may buy and sell items that were not originally generated for that vendor. Therefore dialogue cannot be permanently bound to static shop inventory.

## 1.4 Runtime composition is required
For item-specific dialogue, the API should combine:
- a pre-generated vendor voice line
- structured item facts available at transaction time

## 1.5 Clear separation of responsibilities
- **Go AI** generates and stores dialogue pools
- **PHP backend** retrieves dialogue and performs runtime composition

---

# 2. System Overview

```text
vendor_profiles (source data)
        ↓
Go generator service
        ↓
vendor_dialogue (stable voice lines)
        ↓
PHP API request
        ↓
load vendor + item + interaction context
        ↓
retrieve vendor voice line
        ↓
compose item-specific response from runtime facts
        ↓
return final dialogue to player
```

---

# 3. Dialogue Categories

There are two categories of dialogue.

## 3.1 Vendor-Centric Dialogue (pre-generated)
These lines depend on the vendor, not on a specific moving item instance.

Examples:
- greetings
- returning greetings
- farewells
- deal accepted
- deal rejected
- generic sales pitch by item category
- generic buying reaction

These should be generated offline and stored in the database.

## 3.2 Item-Centric Dialogue (runtime-composed)
These lines depend on the current item involved in the transaction.

Examples:
- exact condition
- exact rarity
- exact defects
- exact provenance
- exact legality
- exact transaction direction

These should not be fully pre-generated per item instance. They should be composed at runtime using deterministic templates and structured item facts.

---

# 4. Responsibilities

# 4.1 Go AI Responsibilities

The Go dialogue generator must:

- read vendors from `vendor_profiles`
- identify vendors that need dialogue generation
- build prompts from vendor profile data
- call the local `llama.cpp` service
- validate generated dialogue
- store rows in `vendor_dialogue`
- update generation status fields in `vendor_profiles`

The Go generator must not:

- serve gameplay requests
- calculate transaction outcomes
- decide prices
- inspect live moving inventory for permanent storage
- generate one-off item instance dialogue for every item row

---

# 4.2 PHP BE AI Responsibilities

The PHP backend must:

- create and maintain vendor profiles
- mark dialogue as pending when vendor traits change
- retrieve dialogue rows from `vendor_dialogue`
- determine interaction buckets
- determine transaction context
- collect item facts at runtime
- compose final player-facing text
- return fallback dialogue if generated dialogue is missing

The PHP backend must not:

- call the LLM during gameplay
- generate vendor dialogue text from scratch
- mutate stored vendor dialogue dynamically
- store dialogue in `vendor_profiles`

---

# 5. Database Design

# 5.1 Source Table: `vendor_profiles`

Existing fields already include:

- `id`
- `uuid`
- `galaxy_id`
- `poi_id`
- `trading_post_id`
- `service_type`
- `criminality`
- `personality`
- `markup_base`
- timestamps

Required additional fields:

```sql
ALTER TABLE vendor_profiles
ADD COLUMN dialogue_generation_status ENUM(
    'pending',
    'generating',
    'complete',
    'failed'
) NOT NULL DEFAULT 'pending',
ADD COLUMN dialogue_generation_version INT UNSIGNED NOT NULL DEFAULT 1,
ADD COLUMN dialogue_generated_at TIMESTAMP NULL DEFAULT NULL;
```

`dialogue_pool` should be removed.

---

# 5.2 Dialogue Storage Table: `vendor_dialogue`

```sql
CREATE TABLE vendor_dialogue (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,

    vendor_profile_id BIGINT UNSIGNED NOT NULL,

    line_type ENUM(
        'greeting',
        'inventory_pitch',
        'deal_accepted',
        'deal_rejected',
        'farewell'
    ) NOT NULL,

    interaction_bucket ENUM(
        'first_visit',
        'second_visit',
        'third_visit',
        'repeat_customer'
    ) NOT NULL,

    transaction_context ENUM(
        'neutral',
        'vendor_selling',
        'vendor_buying'
    ) NOT NULL DEFAULT 'neutral',

    inventory_context VARCHAR(64) NOT NULL DEFAULT 'none',

    line_text VARCHAR(255) NOT NULL,

    weight DECIMAL(5,4) NOT NULL DEFAULT 1.0000,

    generation_version INT UNSIGNED NOT NULL DEFAULT 1,

    created_at TIMESTAMP NULL DEFAULT NULL,
    updated_at TIMESTAMP NULL DEFAULT NULL,

    INDEX idx_vendor_line_type (
        vendor_profile_id,
        line_type
    ),

    INDEX idx_vendor_lookup (
        vendor_profile_id,
        line_type,
        interaction_bucket,
        transaction_context,
        inventory_context
    ),

    INDEX idx_vendor_context (
        vendor_profile_id,
        transaction_context,
        inventory_context
    ),

    CONSTRAINT fk_vendor_dialogue_vendor
        FOREIGN KEY (vendor_profile_id)
        REFERENCES vendor_profiles(id)
        ON DELETE CASCADE
);
```

---

# 5.3 Why `transaction_context` is required

This is necessary because the same vendor sounds different depending on the direction of the transaction.

Examples:

## Vendor selling to player
- boastful
- persuasive
- minimizing defects

## Vendor buying from player
- skeptical
- dismissive
- trying to lower value

So the API needs dialogue that distinguishes:

- `vendor_selling`
- `vendor_buying`

`neutral` is used for generic greetings and farewells.

---

# 5.4 Why `inventory_context` must be controlled

Item-specific vendor voice lines need a clear subject category.

Examples of canonical contexts:

- `none`
- `ship`
- `shield_projector`
- `engine`
- `reactor`
- `weapon`
- `sensor_array`
- `cargo_module`
- `hull_plating`
- `salvage_component`

For v1, this should be stored as `VARCHAR(64)` but treated as a strict canonical enum in code in both Go and PHP.

Do not allow arbitrary strings.

---

# 6. Interaction Buckets

Exact visit counts should be mapped into buckets.

Mapping:

- `1` → `first_visit`
- `2` → `second_visit`
- `3` → `third_visit`
- `>=4` → `repeat_customer`

This keeps dialogue volume reasonable and retrieval simple.

---

# 7. Runtime Composition Model

## 7.1 Stored line
A stored line should provide the **voice**.

Example:
- "This engine still has some fight left in it."
- "Back again? Either I impress you or concern you."
- "You can do worse than this hull, and many have."

## 7.2 Runtime facts
The API should then pull actual item facts such as:

- item type
- condition percent
- rarity tier
- legality / contraband
- known defects
- source / provenance
- transaction direction

## 7.3 Final composed result
The API may produce:

- stored line + runtime clause
- runtime clause only if no stored pitch exists
- stored pitch alone if item facts are not available

Example:
- Stored voice line: "This engine still has some fight left in it."
- Runtime clause: "Seventy-six percent condition, minor coolant instability."
- Final response: "This engine still has some fight left in it. Seventy-six percent condition, minor coolant instability."

This gives specificity without requiring live LLM generation.

---

# 8. Canonical Inventory Contexts

Both Go and PHP must share the same canonical item categories.

## Recommended v1 set

- `none`
- `ship`
- `shield_projector`
- `engine`
- `reactor`
- `weapon`
- `sensor_array`
- `cargo_module`
- `hull_plating`
- `salvage_component`

Both codebases must define these as constants or enums.

---

# 9. Go Generator Design

# 9.1 Input Data
The Go service reads:

- `vendor_profiles.id`
- `service_type`
- `criminality`
- `personality`
- `markup_base`
- generation status/version fields

## Example personality payload

```json
{
  "honesty": 0.45,
  "greed": 0.55,
  "charm": 0.44,
  "risk_tolerance": 0.76
}
```

---

# 9.2 Generation Targets

The generator should create stable voice lines, not exact item-instance descriptions.

## Recommended v1 generation matrix

### Greetings
- `greeting` + `first_visit` + `neutral` + `none`
- `greeting` + `second_visit` + `neutral` + `none`
- `greeting` + `third_visit` + `neutral` + `none`
- `greeting` + `repeat_customer` + `neutral` + `none`

### Inventory pitches
- `inventory_pitch` + `repeat_customer` + `vendor_selling` + `ship`
- `inventory_pitch` + `repeat_customer` + `vendor_selling` + `shield_projector`
- `inventory_pitch` + `repeat_customer` + `vendor_selling` + `engine`
- `inventory_pitch` + `repeat_customer` + `vendor_selling` + `salvage_component`

### Deal accepted
- `deal_accepted` + `repeat_customer` + `vendor_selling` + `none`
- `deal_accepted` + `repeat_customer` + `vendor_buying` + `none`

### Deal rejected
- `deal_rejected` + `repeat_customer` + `vendor_selling` + `none`
- `deal_rejected` + `repeat_customer` + `vendor_buying` + `none`

### Farewell
- `farewell` + `repeat_customer` + `neutral` + `none`

---

# 9.3 Prompt Builder Rules

The generator must derive human-readable prompt hints from source data.

## `service_type`
Maps vendor role and tone:
- `salvage_yard` → used goods, rough edges, opportunistic tone
- `shipyard` → cleaner, prouder, more professional
- `trading_hub` → merchant-generalist
- `market` → commercial marketplace

## `criminality`
Influences legal/moral tone:
- low → cleaner language
- high → sharper or more dubious tone

## `markup_base`
Influences price posture:
- high → more defensive, premium language
- low or negative → more casual or bargain language

## `personality`
Suggested mapping:
- honesty → bluntness / trustworthiness
- greed → harder sell / price defensiveness
- charm → smoothness / friendliness
- risk_tolerance → willingness to hype questionable goods

---

# 9.4 Example Prompt

## System message

```text
You generate short in-universe dialogue for NPC vendors in a space exploration and trading game.
Output JSON only.
Each line must be one sentence.
```

## User message

```text
Generate 15 inventory pitch lines for a salvage vendor selling an engine.

Vendor context:
- service_type: salvage_yard
- interaction_bucket: repeat_customer
- transaction_context: vendor_selling
- criminality: 0.27
- markup_base: 0.2152

Personality:
- honesty: 0.45
- greed: 0.55
- charm: 0.44
- risk_tolerance: 0.76

Rules:
- one sentence per line
- 8 to 18 words
- should sound like a sales pitch
- may describe used or salvaged equipment
- do not include exact percentages or defects
- return JSON: {"lines":["..."]}
```

Important rule:
- the generator should **not** ask the LLM to invent exact live item facts in stored lines

Those facts belong to runtime composition.

---

# 9.5 LLM Endpoint

Use the local `llama.cpp` server:

`POST /v1/chat/completions`

Recommended parameters:
- `temperature: 0.3 - 0.5`
- `top_p: 0.6 - 0.8`
- `max_tokens: 300 - 500`
- `stream: false`

---

# 9.6 Validation Rules

The Go generator must validate all output.

Reject lines if:
- invalid JSON
- missing `lines` array
- line count too low
- duplicate text
- line < 6 words
- line > 20 words
- line > 255 characters
- meta commentary appears
- malformed punctuation or control characters appear

Retry at most twice.

---

# 9.7 Storage Strategy

Before inserting new rows for a given scope, delete old rows matching:

- `vendor_profile_id`
- `line_type`
- `interaction_bucket`
- `transaction_context`
- `inventory_context`

Then insert fresh lines with the current generation version.

---

# 9.8 Status Updates

Before generation:
- set `dialogue_generation_status = 'generating'`

On success:
- set `dialogue_generation_status = 'complete'`
- set `dialogue_generated_at = NOW()`

On failure:
- set `dialogue_generation_status = 'failed'`

---

# 10. PHP Backend Design

# 10.1 Vendor creation/update rules

When a vendor is created:
- set `dialogue_generation_status = 'pending'`
- set `dialogue_generation_version = 1`
- set `dialogue_generated_at = null`

When relevant vendor fields change:
- `service_type`
- `criminality`
- `personality`
- `markup_base`

then:
- increment `dialogue_generation_version`
- set `dialogue_generation_status = 'pending'`
- set `dialogue_generated_at = null`

The PHP backend does not regenerate directly.

---

# 10.2 Dialogue retrieval rules

For each gameplay interaction, the PHP backend must determine:

- `vendor_profile_id`
- `line_type`
- `interaction_bucket`
- `transaction_context`
- `inventory_context`

Then query `vendor_dialogue`.

Example query shape:

```sql
SELECT line_text
FROM vendor_dialogue
WHERE vendor_profile_id = ?
  AND line_type = ?
  AND interaction_bucket = ?
  AND transaction_context = ?
  AND inventory_context IN (?, 'none')
ORDER BY id;
```

The API must prefer the exact `inventory_context` first, then fall back to `none`.

---

# 10.3 Deterministic selection

Do not use `ORDER BY RAND()`.

Use application-side deterministic selection.

Example seed components:
- player ID
- vendor ID
- interaction count
- line type
- transaction context
- item category

Selection:
- `index = hash(seed) % count`

This provides stable variation.

---

# 10.4 Runtime item fact composition

The PHP backend should own runtime phrase composition for live items.

## Example item facts available at runtime
- condition percent
- rarity label
- defect tag
- provenance tag
- legality / contraband flag

## Example composition patterns

### Vendor selling
- "{voice_line} {condition_clause}."
- "{voice_line} {defect_clause}, but {positive_clause}."

### Vendor buying
- "{skeptical_line} {condition_clause}."
- "{skeptical_line} {defect_clause}."

Examples:
- "This shield projector still has some life left in it. Seventy-six percent integrity."
- "Ugly casing, solid output. Minor coolant leak, but the core still runs."

This composition should be deterministic and template-driven, not LLM-driven.

---

# 10.5 Fallback behaviour

If no dialogue rows are found, fallback order should be:

1. exact line type + exact bucket + exact transaction context + `inventory_context = none`
2. exact line type + exact bucket + `transaction_context = neutral`
3. exact line type + `repeat_customer` + `inventory_context = none`
4. static hardcoded fallback by service type

There must always be a fallback string.

---

# 10.6 Response shape

Example API response:

```json
{
  "success": true,
  "data": {
    "vendor": {
      "uuid": "53568459-6bb0-4540-b032-918138ced0be",
      "service_type": "salvage_yard",
      "criminality": 0.27
    },
    "dialogue": {
      "greeting": "Back again? Either I impress you or concern you.",
      "inventory_pitch": "This engine still has some fight left in it. Seventy-six percent condition."
    }
  }
}
```

---

# 11. Shared Code Contracts

Both Go and PHP must agree on:

## Interaction buckets
- `first_visit`
- `second_visit`
- `third_visit`
- `repeat_customer`

## Transaction contexts
- `neutral`
- `vendor_selling`
- `vendor_buying`

## Inventory contexts
- `none`
- `ship`
- `shield_projector`
- `engine`
- `reactor`
- `weapon`
- `sensor_array`
- `cargo_module`
- `hull_plating`
- `salvage_component`

These must be implemented as canonical constants/enums in both codebases.

---

# 12. Suggested Implementation Order

## Phase 1
- PHP: add generation-tracking columns to `vendor_profiles`
- PHP: create `vendor_dialogue`
- PHP: stop using `dialogue_pool`

## Phase 2
- Go: implement vendor loading
- Go: implement prompt building
- Go: implement llama.cpp client
- Go: implement validation
- Go: insert dialogue rows

## Phase 3
- PHP: implement dialogue retrieval service
- PHP: implement deterministic selection
- PHP: implement runtime item clause composition
- PHP: implement static fallbacks

## Phase 4
- PHP: mark vendors pending when traits change
- Go: support regeneration by version
- both: finalize shared enum constants

---

# 13. Risks and Mitigations

## Risk: dialogue becomes too generic
Mitigation:
- increase category-specific lines
- improve personality-to-prompt mapping
- add more transaction-specific voice lines

## Risk: item-specific dialogue feels repetitive
Mitigation:
- use multiple runtime clause templates
- use deterministic seeded rotation for condition/defect phrasing

## Risk: Go generator stores lines that incorrectly reference exact condition values
Mitigation:
- explicit prompt rule forbidding exact live item facts in stored lines

## Risk: retrieval mismatches due to inconsistent inventory_context values
Mitigation:
- strict shared enums/constants in both services

---

# 14. Final Summary

## Go AI must build and store:
- stable vendor voice
- category-level inventory pitches
- transaction-aware but item-instance-agnostic dialogue

## PHP BE AI must:
- retrieve stored voice lines
- determine interaction bucket and transaction context
- inject live item facts at runtime
- return coherent final dialogue

This design avoids:
- gameplay-time LLM calls
- inventory-location coupling
- impossible pre-generation of all moving item dialogue

It also provides:
- good flavour
- scalable storage
- deterministic behaviour
- clean separation of responsibilities
