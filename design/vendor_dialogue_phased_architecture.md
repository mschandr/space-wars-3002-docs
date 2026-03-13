# Space Wars 3002 – Vendor Dialogue System
## Phased Architecture and Branch Plan

## Purpose

This document defines a phased implementation plan for the vendor dialogue and NPC interaction system in Space Wars 3002.

The goal is to break delivery into discrete, cumulative packages so work can proceed branch-by-branch with clear benefits at each stage.

Suggested branch naming:

- feat/vendor_dialogue_phase_1
- feat/vendor_dialogue_phase_2
- feat/vendor_dialogue_phase_3
- feat/vendor_dialogue_phase_4
- feat/vendor_dialogue_phase_5
- feat/vendor_dialogue_phase_6

Each phase should leave the game in a working state.

---

# Core Design Objective

Build a dialogue system that is:

- deterministic in mechanics
- flavourful in presentation
- scalable to many NPCs
- independent of real-time LLM calls
- able to support local drift in behaviour based on player interaction

The system is based on:

1. interaction verbs
2. mechanical resolution
3. NPC relationship drift
4. pre-generated voice lines
5. runtime composition using live item facts

---

# Phase 1
## Branch: feat/vendor_dialogue_phase_1
## Goal: Foundational dialogue storage and retrieval

## Summary
Introduce the database structures and backend retrieval logic required to move away from static dialogue_pool blobs and into structured dialogue rows.

## Scope

### Database
- remove dialogue_pool from vendor_profiles
- add:
  - dialogue_generation_status
  - dialogue_generation_version
  - dialogue_generated_at

### Create vendor_dialogue
The table should support:
- line type
- interaction bucket
- transaction context
- inventory context
- generation version

### PHP backend
Implement:
- VendorDialogue model
- interaction bucket mapper
- dialogue retrieval service
- deterministic dialogue selector
- static fallback lines by service type

### Dialogue categories for this phase
- greeting
- inventory_pitch
- deal_accepted
- deal_rejected
- farewell

### Interaction buckets
- first_visit
- second_visit
- third_visit
- repeat_customer

### Transaction contexts
- neutral
- vendor_selling
- vendor_buying

### Inventory contexts
Use canonical code constants:
- none
- ship
- shield_projector
- engine
- reactor
- weapon
- sensor_array
- cargo_module
- hull_plating
- salvage_component

## Benefit
At the end of Phase 1:
- the API can retrieve structured vendor dialogue
- dialogue is no longer trapped in a JSON blob
- the schema is ready for generator integration
- the game can serve deterministic, context-aware flavour text

## Deliverables
- migrations
- models
- retrieval service
- fallback service
- constants/enums
- updated vendor/shop API response fields

---

# Phase 2
## Branch: feat/vendor_dialogue_phase_2
## Goal: Offline Go dialogue generator integrated with vendor profiles

## Summary
Build the Go service that reads vendor profiles, generates dialogue using local llama.cpp, validates the output, and stores it in vendor_dialogue.

## Scope

### Go service
Implement:
- MariaDB connection
- vendor loading
- prompt builder
- llama.cpp API client
- response parser
- validation and dedupe
- insert/update dialogue rows
- status updates on vendor_profiles

### Generation targets
Generate stable vendor-centric lines only:
- greetings by interaction bucket
- inventory pitches by item category
- deal accepted
- deal rejected
- farewells

### Validation
Reject:
- malformed JSON
- duplicates
- lines outside length limits
- meta commentary
- malformed punctuation

### Config
Add environment-driven config for:
- DB
- LLM endpoint
- model name
- retries
- worker count
- batch size
- generation version

## Benefit
At the end of Phase 2:
- vendor dialogue can be generated asynchronously
- gameplay is unaffected by LLM latency
- new vendors can automatically enter dialogue generation queues via status flags
- flavour content becomes scalable

## Deliverables
- Go service
- prompt builder
- validation layer
- insert/update strategy
- operational config

---

# Phase 3
## Branch: feat/vendor_dialogue_phase_3
## Goal: Controlled interaction verbs for vendors

## Summary
Add a proper interaction verb framework so vendor conversations are driven by explicit player intents rather than freeform chat.

## Scope

### Introduce vendor interaction verbs
Examples:
- browse_inventory
- ask_price
- inspect_quality
- inspect_compatibility
- ask_specs
- challenge_claim
- make_offer
- accept_offer
- reject_offer
- leave

### Add role-gated options
For example:
- chief_engineer gets:
  - inspect_quality
  - inspect_defects
  - inspect_compatibility
  - ask_specs

### Add option gating rules
Available options =:
- base options
- role options
- context options
- relationship options
- blocked options

### API changes
Return available interaction options along with dialogue.

## Benefit
At the end of Phase 3:
- interactions become structured and scalable
- different roles get meaningful options
- vendor interaction logic becomes extensible without hardcoded scene scripting

## Deliverables
- interaction verb definitions
- option gating logic
- API payload structure for option lists
- first vendor-only interaction controller/service layer

---

# Phase 4
## Branch: feat/vendor_dialogue_phase_4
## Goal: Runtime item fact composition

## Summary
Separate vendor voice from live item facts so moving inventory can still produce coherent dialogue.

## Scope

### Add runtime composition layer
Stored vendor lines provide:
- voice
- tone
- stance

Runtime item facts provide:
- condition
- rarity
- defects
- legality
- provenance
- compatibility

### Implement item fact clause builder
Examples:
- Seventy-six percent condition.
- Minor coolant instability.
- Military surplus.
- Compatible with medium exploration hulls.

### Compose final dialogue
Example:
- voice line + condition clause
- skeptical buying line + defect clause

### PHP backend
Implement deterministic clause selection.

## Benefit
At the end of Phase 4:
- dialogue remains coherent even though inventory moves between vendors and players
- exact item instances do not need permanent dialogue rows
- vendor voice and item specificity are cleanly separated

## Deliverables
- runtime composition service
- item fact clause templates
- deterministic clause selection
- API response integration

---

# Phase 5
## Branch: feat/vendor_dialogue_phase_5
## Goal: Local drift and player/NPC relationship overlays

## Summary
Introduce local behavioural drift so vendors can learn from player interactions without requiring live AI reasoning.

## Scope

### Add vendor-player relationship table
Suggested fields:
- trust
- respect
- resentment
- caution
- familiarity
- perceived_competence
- willingness_to_discount

### Update mechanical resolution to modify relationship state
Examples:
- spotting genuine defects increases respect
- repeated lowball offers increase resentment
- fair dealing improves trust
- threats increase fear/caution

### Dialogue selection responds to drift
Either:
- via tone buckets
or
- via selection rules using relationship thresholds

### Keep this vendor-specific in v1
Do not generalize to pirates/crew yet unless time permits.

## Benefit
At the end of Phase 5:
- vendors feel like they remember the player
- negotiation consequences accumulate over time
- dialogue and pricing posture become more personalized

## Deliverables
- relationship schema
- drift update rules
- relationship-aware selection inputs
- first wave of vendor memory mechanics

---

# Phase 6
## Branch: feat/vendor_dialogue_phase_6
## Goal: Generalized NPC framework for vendors, pirates, and crew

## Summary
Expand the system into a generalized NPC profile/instance/relationship model.

## Scope

### Introduce global profile pools
- vendor_profiles
- crew_profiles
- pirate_captain_profiles

### Introduce galaxy-local instances
- galaxy_vendor_instances
- galaxy_crew_instances
- galaxy_pirate_instances

### Add relationship overlays by player
- vendor_player_relationships
- crew_player_relationships
- pirate_player_relationships

### Adapt dialogue selection to instance-level or relationship-level state
Dialogue should eventually be keyed to:
- base profile
- local instance
- player relationship state
- current interaction context

### Add optional dialogue refresh jobs
Only regenerate dialogue when drift crosses meaningful thresholds.

## Benefit
At the end of Phase 6:
- NPCs become reusable, assignable assets across galaxies
- local drift creates galaxy-specific variants
- the same base NPC can diverge meaningfully between galaxies
- the design supports pirates and crew without rewriting the architecture

## Deliverables
- generalized NPC profile/instance schema
- drift-capable relationship overlay tables
- instance-aware dialogue selection strategy
- regeneration threshold logic

---

# Phase Dependencies

## Mandatory sequence
- Phase 1 before everything else
- Phase 2 depends on Phase 1
- Phase 3 depends on Phase 1
- Phase 4 depends on Phases 1 and 3
- Phase 5 depends on Phases 3 and 4
- Phase 6 depends on all earlier phases

---

# Suggested Engineering Order

## First deliver
Phase 1 + Phase 2

This gives:
- working storage
- working generation
- immediate flavour improvements

## Second deliver
Phase 3 + Phase 4

This gives:
- controlled interactions
- coherent item-specific dialogue

## Third deliver
Phase 5

This gives:
- memory
- behavioural consequences

## Final deliver
Phase 6

This gives:
- generalized NPC architecture

---

# Shared Contracts Across Phases

The following must remain canonical:

## Interaction buckets
- first_visit
- second_visit
- third_visit
- repeat_customer

## Transaction contexts
- neutral
- vendor_selling
- vendor_buying

## Inventory contexts
- none
- ship
- shield_projector
- engine
- reactor
- weapon
- sensor_array
- cargo_module
- hull_plating
- salvage_component

These must be shared consistently across:
- Go generator
- PHP backend
- future generalized NPC code

---

# Risks

## Risk: phases become too coupled
Mitigation:
- each phase must produce a usable partial system
- APIs should degrade gracefully to fallbacks

## Risk: generator output is too generic
Mitigation:
- improve prompt builder in Phase 2
- add tone buckets later

## Risk: local drift becomes unbalanced
Mitigation:
- cap all relationship values
- add decay
- make personality influence learning speed and memory

## Risk: dialogue selection logic becomes duplicated
Mitigation:
- centralize constants and selection rules early

---

# Final Summary

This phased plan ensures the vendor dialogue system evolves in clean steps:

- Phase 1 gives structure
- Phase 2 gives generated flavour
- Phase 3 gives controlled interaction verbs
- Phase 4 gives item-aware runtime composition
- Phase 5 gives relationship memory and drift
- Phase 6 generalizes the architecture for all NPC types

This keeps delivery manageable while providing cumulative gameplay value at every branch.
