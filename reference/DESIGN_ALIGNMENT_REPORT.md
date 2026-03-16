# Design Alignment Report
## Vendor Dialogue Generator Service

**Date**: 2026-03-10
**Current Status**: Partial implementation, significant gaps vs. design document

---

## Executive Summary

The current implementation covers the basic flow but diverges from the design in several critical areas:

1. **Database Schema Mismatch** — Missing required columns for tracking generation status and versioning
2. **Configuration Incomplete** — Missing version tracking, timeout settings, and filters
3. **Prompt Builder Simplified** — Generic mappings instead of design-specified service type logic
4. **LLM Settings Non-Conformant** — Temperatures/top_p too high; missing stream parameter
5. **Vendor Query Incomplete** — Only selects subset of expected columns
6. **Status Tracking Disabled** — Removed due to missing DB columns
7. **Inventory Context Not Scoped** — Applied too broadly, design limits to `inventory_pitch` only
8. **Generation Version Tracking Missing** — No versioning support

---

## Section-by-Section Comparison

### 1. Database Schema (`vendor_profiles` table)

**Design Expects:**
- `id`, `uuid`, `galaxy_id`, `poi_id`, `trading_post_id`
- `service_type`, `criminality`, `personality`, `markup_base`
- `dialogue_generation_status` (pending/generating/complete/failed)
- `dialogue_generation_version` (for regeneration tracking)
- `dialogue_generated_at` (timestamp)

**Current Implementation Has:**
- ✓ `id`, `service_type`, `criminality`, `personality`, `markup_base`, `created_at`, `updated_at`
- ✗ Missing: `uuid`, `galaxy_id`, `poi_id`, `trading_post_id`
- ✗ Missing: `dialogue_generation_status`, `dialogue_generation_version`, `dialogue_generated_at`

**Action Required:**
- Query actual database schema to determine which columns actually exist
- Either: Add missing columns to schema, OR update design to match actual schema
- Currently code skips these due to column non-existence; this is a blocker

---

### 2. Target Table (`vendor_dialogue`)

**Design Expects:**
- `vendor_profile_id`, `line_type`, `interaction_bucket`, `inventory_context`
- `line_text` (the dialogue line)
- `weight` (for weighting/priority)
- `generation_version` (tracks which generation produced this row)

**Current Implementation:**
- ✓ `vendor_profile_id`, `line_type`, `interaction_bucket`, `line_text` (INSERT implemented)
- ✗ Missing: `inventory_context`, `weight`, `generation_version`

**Action Required:**
- Add three missing columns to `vendor_dialogue` table OR confirm with user they're not needed
- Update INSERT to include these fields
- If generation_version missing, lose regeneration capability

---

### 3. Configuration Variables

**Design Requires:**
```
DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASS
LLM_BASE_URL, LLM_MODEL, LLM_TIMEOUT_SECONDS
GENERATION_VERSION, WORKER_COUNT, BATCH_SIZE, MAX_RETRIES
(Optional) SHOP_TYPE_FILTER, VENDOR_LIMIT, LOG_LEVEL, DRY_RUN
```

**Current Implementation Has:**
```
✓ DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME
✓ LLM_BASE_URL, LLM_MODEL
✗ Missing: LLM_TIMEOUT_SECONDS
✗ Missing: GENERATION_VERSION
✗ Missing: MAX_RETRIES (has GENERATION_RETRY_MAX, may be same)
✗ Missing: SHOP_TYPE_FILTER, VENDOR_LIMIT
✓ WORKER_COUNT, BATCH_SIZE
✓ LOG_LEVEL, DRY_RUN
✗ Has extras: LLM_TEMPERATURE, LLM_TOP_P (not in design; see #4)
✓ Has: ROWS_TO_GENERATE (added per user request)
```

**Action Required:**
- Add `GENERATION_VERSION` (required for versioning support)
- Add `LLM_TIMEOUT_SECONDS` with reasonable default (e.g., 60s)
- Rename or clarify: `GENERATION_RETRY_MAX` vs. `MAX_RETRIES` (likely same, clarify intent)
- LLM temperature/top_p: see section #4 below

---

### 4. LLM Client Settings

**Design Recommends:**
```
temperature: 0.3 - 0.5
top_p: 0.6 - 0.8
max_tokens: 300 - 500
stream: false (explicit requirement)
```

**Current Implementation:**
```
temperature: 0.7 (TOO HIGH — design max is 0.5)
top_p: 0.9 (TOO HIGH — design max is 0.8)
max_tokens: 500 (within range)
stream: not set (missing from ChatRequest struct)
```

**Action Required:**
- Lower default temperature from 0.7 → 0.4
- Lower default top_p from 0.9 → 0.7
- Add `stream: false` to ChatRequest struct and set it in requests
- Update .env.example with corrected defaults
- Document that lower temps prevent small model drift

---

### 5. Vendor Query

**Design Example Query:**
```sql
SELECT * FROM vendor_profiles
WHERE dialogue_generation_status IN ('pending', 'failed')
ORDER BY id LIMIT ?
```

**Current Implementation Query:**
```sql
SELECT id, service_type, criminality, markup_base, personality, created_at, updated_at
FROM vendor_profiles
ORDER BY id LIMIT ?
```

**Issues:**
- No WHERE clause filtering (fetches ALL vendors, not just pending/failed)
- Missing: dialogue_generation_status column (can't filter anyway)
- Missing: uuid, galaxy_id, poi_id, trading_post_id in SELECT
- Missing: dialogue_generation_version (can't support regeneration by version)

**Action Required:**
- Clarify actual schema with user (which columns actually exist?)
- Update query to match: either add WHERE clause OR select all needed columns
- Once status columns exist, add WHERE filtering back

---

### 6. Status Update Logic

**Design Requires:**
```sql
-- Before generation
UPDATE vendor_profiles SET dialogue_generation_status = 'generating' WHERE id = ?;

-- On success
UPDATE vendor_profiles
SET dialogue_generation_status = 'complete', dialogue_generated_at = NOW()
WHERE id = ?;

-- On failure
UPDATE vendor_profiles SET dialogue_generation_status = 'failed' WHERE id = ?;
```

**Current Implementation:**
- ✗ Removed all status update functions (`MarkGenerating`, `MarkComplete`, `MarkFailed`)
- Reason: `dialogue_generation_status` column doesn't exist in actual schema
- Effect: No tracking of which vendors completed vs. failed

**Action Required:**
- **Option A:** Add `dialogue_generation_status` and `dialogue_generated_at` columns to vendor_profiles, reinstate update logic
- **Option B:** Clarify with user if status tracking is truly not needed (differs from design)
- Current workaround is insufficient for production use

---

### 7. Prompt Builder

**Design Specifies Service Type Mappings:**

```
salvage_yard    → rougher, opportunistic, used goods
shipyard        → polished, prideful, sales-oriented
trading_hub     → general merchant tone
market          → broader commercial tone
```

**Current Implementation:**
- ✓ Has basic service_type switch (armory, shield_specialist, supply_merchant)
- ✗ Mappings are generic, not design-specified
- ✗ Missing: salvage_yard, shipyard, trading_hub, market

**Design Specifies Personality Mappings:**

```
honesty         → bluntness / fairness / sales truthfulness
greed           → defensive markup, harder sell
charm           → warmth, smoothness, social ease
risk_tolerance  → willingness to exaggerate or sell questionable goods
```

**Current Implementation:**
- ✓ Uses personality.Get() to safely access traits
- ✗ Mappings are simplistic (honesty > 0.7 = "straightforward")
- ✗ Missing: nuanced tone building per design examples

**Action Required:**
- Update prompt builder to match design service type list
- Expand personality trait usage with design-specified mappings
- Reference example templates in design section #10 for exact wording
- Ensure prompt hints drive better LLM output alignment

---

### 8. Prompt Templates

**Design Provides Example System Message:**
```
You generate short in-universe dialogue for NPC vendors in a space exploration and trading game.
Output JSON only.
Do not include explanations.
Keep every line as one sentence.
```

**Current Implementation System Prompt:**
```
Generic merchant tone derived from service_type/personality
"Respond with exactly: {\"lines\": [...]}"
```

**Design Provides Detailed User Message Template:**
- Lists vendor context (service_type, interaction_bucket, criminality, markup_base)
- Lists personality traits with values
- Specifies rules (8-16 words, one sentence, tone hints)
- Shows exact JSON schema expected

**Current Implementation User Prompt:**
- Minimal, generic
- Doesn't match design verbosity/structure

**Action Required:**
- Rewrite `buildSystemPrompt()` and `buildUserPrompt()` to match design templates
- Reference design section #10 for exact template language
- Include service_type-specific tone hints
- Include inventory context example when applicable

---

### 9. Interaction Buckets & Line Types

**Design v1 Scope:**

**Line Types:**
- greeting, inventory_pitch, deal_accepted, deal_rejected, farewell ✓

**Interaction Buckets:**
- first_visit, second_visit, third_visit, repeat_customer ✓

**Current Implementation:**
- Implements all 5 line types ✓
- Implements all 4 buckets ✓

**Generation Matrix (Design Section #22):**

```
greeting + first_visit        → 15 lines
greeting + second_visit       → 15 lines
greeting + third_visit        → 15 lines
greeting + repeat_customer    → 20 lines

inventory_pitch + repeat_customer + ship              → 10 lines
inventory_pitch + repeat_customer + shield_projector → 10 lines

deal_accepted + repeat_customer   → 10 lines
deal_rejected + repeat_customer   → 10 lines
farewell + repeat_customer        → 10 lines
```

**Current Implementation:**
- Has this matrix hardcoded in `getDialogueBuckets()` ✓

**Status:** ✓ This section aligns well

---

### 10. Inventory Context Scoping

**Design States (Section #6):**
> "Support inventory context only for `inventory_pitch`"

**Design Examples:**
- `shield_projector`, `engine`, `reactor`, `hull_plating`, `ship`

**Current Implementation:**
- Includes `InventoryCtx` field in DialogueBucket
- Applied to inventory_pitch buckets only ✓
- Correctly limited to ship + shield_projector ✓

**Status:** ✓ This section aligns

---

### 11. Validation Rules

**Design (Section #18):**
- min words: 6, max words: 20 ✓
- max characters: 255 ✓
- no empty strings ✓
- no duplicates after normalization (SHA-256) ✓
- no control characters ✓
- reject repeated punctuation (!!! or ???) ✓
- reject meta output ✓
- minimum threshold: not specified in detail, current = 60% ✓

**Current Implementation:**
- `validateLine()` covers all required checks ✓
- `Validate()` checks 60% threshold ✓
- `normalizeLine()` normalizes properly ✓

**Status:** ✓ This section aligns well

---

### 12. Retry Logic

**Design (Section #20):**
```
Max 2 retries per bucket
Second retry uses:
- lower temperature
- more explicit JSON-only instruction
- shorter requested line count if needed
```

**Current Implementation:**
```go
for attempt := 0; attempt <= o.cfg.GenerationRetryMax; attempt++ {
    lines, err := o.callLLM(...)
    if err != nil {
        // Reduce count by 0.8x, continue to next attempt
        requestedCount = int(float64(requestedCount) * 0.8)
        continue
    }
    ...
}
```

**Issues:**
- ✓ Has max retry count
- ✗ Does not lower temperature on retry
- ✗ Does not use stricter prompt on retry
- ✗ Only reduces count, doesn't strengthen JSON instruction
- ✗ No configurable LLM parameter adjustment per retry

**Action Required:**
- Add retry-aware prompt building (stricter on 2nd attempt)
- Lower temperature when retrying (e.g., 0.4 → 0.2 or use config override)
- Add explicit JSON-only instruction on retry
- Track retry attempt in logging

---

### 13. Malformed JSON Handling

**Design (Section #20):** Retry on malformed JSON

**Current Implementation:**
- Gracefully skips vendors with malformed personality JSON ✓
- Logs vendor_id and error ✓

**Status:** ✓ This section aligns

---

### 14. Error Handling & Logging

**Design (Section #24) Logs:**
- vendor ID ✓
- line type ✓
- interaction bucket ✓
- prompt hash ✓
- response size ✗
- inserted row count ✓
- validation failures ✓
- retry count ✗
- final status ✗

**Current Implementation:**
- Logs vendor_id, line_type, bucket ✓
- Logs accepted/rejected counts ✓
- Missing: response_size, retry_count, final_status details

**Action Required:**
- Add response size logging
- Add retry attempt number to logs
- Add final status summary per vendor

---

### 15. Generation Version Tracking

**Design (Section #26):**
> Regenerate when dialogue_generation_version changes, personality/criminality changes, etc.

**Current Implementation:**
- No version tracking
- No regeneration by version support
- Fetches all vendors (or filtered by user --rows flag)

**Action Required:**
- Add GENERATION_VERSION env var (required config)
- Pass version to orchestrator
- Update vendor_dialogue INSERT to include generation_version
- Add version parameter to SELECT WHERE clause (design section #15)
- Without this, regeneration capability is non-functional

---

## Summary of Required Changes

### Critical (Blocks Full Functionality)

1. **Resolve Database Schema** — Query actual vendor_profiles schema; clarify which columns exist
   - If `dialogue_generation_status` doesn't exist: add it OR confirm with user it's not needed
   - If `dialogue_generation_version` doesn't exist: add it OR remove regeneration feature
   - If extra columns (uuid, galaxy_id, poi_id, trading_post_id) don't exist: confirm they're not needed

2. **Add vendor_dialogue Columns** — Ensure table has: inventory_context, weight, generation_version
   - Or confirm these aren't needed and close regeneration capability

3. **Reinstate Status Updates** — Add MarkGenerating/MarkComplete/MarkFailed back once DB schema exists
   - Without this, can't track vendor generation progress

4. **Add GENERATION_VERSION Config** — Required for versioning support
   - Add to config struct, .env.example, and flag parsing

### High Priority (Design Compliance)

5. **Adjust LLM Defaults** — Lower temperature (0.7 → 0.4), top_p (0.9 → 0.7), add stream: false
   - Prevents small model drift per design intent

6. **Rewrite Prompt Templates** — Match design section #10 examples exactly
   - Service type tone mappings
   - Personality trait utilization
   - Word count guidance (8-16 for greeting, 8-18 for pitch)

7. **Implement Stricter Retry Logic** — Lower temperature, harden JSON instruction on 2nd attempt
   - Current: only reduces count
   - Need: temperature adjustment + prompt hardening

8. **Add Missing Config Variables** — LLM_TIMEOUT_SECONDS, possibly MAX_RETRIES alias
   - Low effort, improves production readiness

### Medium Priority (Polish)

9. **Enhance Logging** — Add response_size, retry_count, final_status_summary
   - Improves observability

10. **Update Query WHERE Clause** — Add dialogue_generation_status filtering (once column exists)
    - Currently fetches all vendors; design expects filtering

---

## Implementation Roadmap

### Phase 1: Clarify Database Schema (BLOCKING)
- User confirms which columns actually exist in vendor_profiles and vendor_dialogue
- Based on answer, either:
  - Add missing columns to schema, OR
  - Update design expectations to match actual schema

### Phase 2: Reinstate Core Features (1-2 hours)
- Add dialogue_generation_status/version columns and update logic
- Add missing vendor_dialogue columns
- Add GENERATION_VERSION config

### Phase 3: Compliance (2-3 hours)
- Update LLM client defaults
- Rewrite prompt builder with design examples
- Implement stricter retry logic

### Phase 4: Polish (1 hour)
- Enhance logging
- Test against real database
- Verify dialogue quality matches design intent

