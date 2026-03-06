# ADR 0001: Construction Output Deferral

**Status:** Accepted
**Date:** 2026-03-04
**Deciders:** Backend Architecture Team

## Context

Phase 3 of the economy overhaul implements a construction system with the following requirements:
1. Players can initiate construction jobs at trading hubs
2. Inputs (commodities) are consumed from hub inventory via the ledger economy
3. Construction jobs are tracked with a maturation timer
4. Jobs complete asynchronously during economy ticks

The challenge: The current `Blueprint` model references `output_item_code` as a simple string (e.g., `"ship_fighter_mk1"`), but there is **no `player_items` table** to store the actual constructed items. Building a full item storage layer would bloat Phase 3 scope significantly.

## Decision

**Phase 3 implements consumption + job tracking only. Output item delivery is deferred to Phase N+.**

- `ConstructionService::build()` creates ledger entries to consume inputs ✅
- `ConstructionJob` tracks what was built, when it started, and when it matures ✅
- `ConstructionTickService` matures pending jobs during economy ticks ✅
- `ConstructionService::completeJob()` is a **stub that logs "item ready"** — no actual delivery ⏸️

## Rationale

### Why Defer?

1. **Schema dependency:** Item delivery requires a `player_items` table with fields like:
   - `id`, `uuid`, `player_id`, `item_code`, `item_type`, `quantity`
   - Timeable delivery logic (pickup deadline, storage limit, etc.)
   - Relationships to variants, upgrades, and ship modules
   - This is a design discussion by itself

2. **Scope explosion:** Implementing both economy conservation AND item storage in one phase would:
   - Double the testing burden
   - Increase risk of regression
   - Prevent delivery of the core economy features on schedule

3. **Economic correctness:** Ledger-based consumption is the critical piece. Item delivery is a separate concern that can be verified in isolation later.

### Why It's Safe

- **Stub is explicit:** `completeJob()` is clearly marked "TODO: Future phases"
- **No silent failures:** A player receives an error "Item ready but not delivered" (phase N+ message)
- **Ledger is immutable:** Inputs consumed now are permanently recorded; delivery cannot corrupt conservation
- **API contract:** Response includes `output_item_code` so clients know what *will* be delivered

## Alternatives Considered

### A. Instant Delivery (Rejected)
Store items in player cargo directly upon completion. Problems:
- No delivery deadline or pickup logic
- Undermines future "item storage" design
- Doesn't work for ships, blueprints, or other non-cargo items

### B. Full Async Queue (Rejected for Phase 3)
Implement complete item delivery system now:
- `player_items` table + schema
- Delivery deadlines and storage limits
- Ship/module/upgrade delivery logic
- Pickup notification flow

**Verdict:** This is the *right* answer architecturally, but wrong scope for Phase 3. Defer to Phase N+ when it can be designed properly.

### C. Consumption + Job Tracking Only (Chosen)
- ✅ Highest confidence in correctness
- ✅ Minimal scope
- ✅ Allows concurrent work on item storage layer
- ✅ Ledger foundation is complete and tested

## Consequences

### Positive
- Economy conservation is rock-solid; ledger cannot be bypassed
- Test coverage is simple: verify consumption, verify maturation
- Non-blocking: item storage can be designed independently
- Migration path: Replace `completeJob()` stub with real logic later without touching ledger

### Negative
- Players see construction jobs complete but can't pick up items yet
- Requires client-side messaging: "Your construction is ready, but item delivery is not yet implemented"
- Minimal immediate value from construction jobs (they consume resources but don't produce anything)

## Evolution Path

**Phase N+:** Implement item delivery
1. Create `PlayerItem` model and `player_items` table
2. Create `ItemDeliveryService`
3. Replace `ConstructionService::completeJob()` stub:
   ```php
   public function completeJob(ConstructionJob $job): void
   {
       // Mark as complete
       $job->status = 'COMPLETE';
       $job->completed_at = now();
       $job->save();

       // Deliver the item
       ItemDeliveryService::deliver($job);

       // Notify player
       $job->player->notify(new ConstructionComplete($job));
   }
   ```

This change requires **no modifications** to:
- Migration schema (already has `output_item_code`)
- Ledger entries (consumption already recorded)
- Controller APIs (response format unchanged)
- Tests (new tests only for delivery logic)

## Related Issues
- None yet (Phase 3 is initial implementation)

## References
- Phase 3 Plan: `/docs/guides/PHASE_3_PLAN.md`
- Blueprint Model: `app/Models/Blueprint.php`
- Construction Service: `app/Services/Economy/ConstructionService.php`
