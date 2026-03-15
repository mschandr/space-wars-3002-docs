# Design Alignment Report (Updated)
## Vendor Dialogue Generator Service – Database Schema Aligned

**Date**: 2026-03-10 (Updated)
**Source**: Joint PHP/Go Design Document (vendor_dialogue_joint_design_go_php.md)
**Status**: Schema is defined; implementation gaps remain

---

## Database Schema CONFIRMED

### vendor_profiles (Source Table)

**Actual Schema per Joint Design:**
```sql
ALTER TABLE vendor_profiles
ADD COLUMN dialogue_generation_status ENUM('pending', 'generating', 'complete', 'failed') DEFAULT 'pending',
ADD COLUMN dialogue_generation_version INT UNSIGNED DEFAULT 1,
ADD COLUMN dialogue_generated_at TIMESTAMP NULL DEFAULT NULL;
```

**Current Go Implementation:**
```go
type VendorProfile struct {
    ID              int64
    ServiceType     string
    Criminality     float64
    MarkupBase      float64
    PersonalityJSON string
    Personality     Personality
    CreatedAt       time.Time
    UpdatedAt       time.Time
}
```

**Missing Fields:**
- ✗ `dialogue_generation_status` (ENUM: pending/generating/complete/failed)
- ✗ `dialogue_generation_version` (INT UNSIGNED, default 1)
- ✗ `dialogue_generated_at` (TIMESTAMP NULL)

**Action Required:**
Add three fields to VendorProfile struct:
```go
DialogueGenerationStatus string    // "pending", "generating", "complete", "failed"
DialogueGenerationVersion int      // version number for regeneration tracking
DialogueGeneratedAt *time.Time     // nullable timestamp
```

---

### vendor_dialogue (Target Table)

**Actual Schema per Joint Design:**
```sql
CREATE TABLE vendor_dialogue (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_profile_id BIGINT UNSIGNED NOT NULL,
    line_type ENUM(...) NOT NULL,
    interaction_bucket ENUM(...) NOT NULL,
    transaction_context ENUM('neutral', 'vendor_selling', 'vendor_buying') DEFAULT 'neutral',
    inventory_context VARCHAR(64) DEFAULT 'none',
    line_text VARCHAR(255) NOT NULL,
    weight DECIMAL(5,4) DEFAULT 1.0000,
    generation_version INT UNSIGNED DEFAULT 1,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

**Current Go INSERT:**
```sql
INSERT INTO vendor_dialogue (vendor_profile_id, line_type, interaction_bucket, line_text)
VALUES (?, ?, ?, ?)
```

**Missing in INSERT:**
- ✗ `transaction_context` (required field with default 'neutral')
- ✗ `weight` (optional, defaults to 1.0)
- ✗ `generation_version` (optional, track which version generated this)
- ✗ `created_at` (optional, set to NOW())

**Action Required:**
Update INSERT to:
```sql
INSERT INTO vendor_dialogue
(vendor_profile_id, line_type, interaction_bucket, transaction_context, inventory_context, line_text, weight, generation_version, created_at)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, NOW())
```

---

## Implementation Gaps Remaining

### 1. VendorProfile Struct (CRITICAL)

**Current:**
```go
type VendorProfile struct {
    ID              int64
    ServiceType     string
    Criminality     float64
    MarkupBase      float64
    PersonalityJSON string
    Personality     Personality
    CreatedAt       time.Time
    UpdatedAt       time.Time
}
```

**Required:**
```go
type VendorProfile struct {
    ID                        int64
    ServiceType               string
    Criminality               float64
    MarkupBase                float64
    PersonalityJSON           string
    Personality               Personality
    DialogueGenerationStatus  string    // NEW
    DialogueGenerationVersion int       // NEW
    DialogueGeneratedAt       *time.Time// NEW (nullable)
    CreatedAt                 time.Time
    UpdatedAt                 time.Time
}
```

**Action:**
- Add three fields to struct
- Update repository.go SELECT to fetch these columns
- Update rows.Scan() to map these fields

---

### 2. Database Query (CRITICAL)

**Current:**
```sql
SELECT id, service_type, criminality, markup_base, personality, created_at, updated_at
FROM vendor_profiles
ORDER BY id
LIMIT ?
```

**Required:**
```sql
SELECT id, service_type, criminality, markup_base, personality,
       dialogue_generation_status, dialogue_generation_version, dialogue_generated_at,
       created_at, updated_at
FROM vendor_profiles
WHERE dialogue_generation_status IN ('pending', 'failed')
ORDER BY id
LIMIT ?
```

**Why:**
- Missing status tracking columns in SELECT
- Missing WHERE clause to filter for vendors that need work
- Current implementation fetches ALL vendors

**Action:**
- Add three columns to SELECT
- Add WHERE filtering
- Update Scan() order to match

---

### 3. Status Update Functions (CRITICAL)

**Current State:** Functions removed because columns didn't exist

**Required Implementation:**
```go
func MarkGenerating(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE vendor_profiles
        SET dialogue_generation_status = 'generating'
        WHERE id = ?
    `, vendorID)
    return err
}

func MarkComplete(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE vendor_profiles
        SET dialogue_generation_status = 'complete', dialogue_generated_at = NOW()
        WHERE id = ?
    `, vendorID)
    return err
}

func MarkFailed(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE vendor_profiles
        SET dialogue_generation_status = 'failed'
        WHERE id = ?
    `, vendorID)
    return err
}
```

**Action:**
- Add these functions back to vendors/repository.go
- Restore calls in jobs/worker.go processVendor()

---

### 4. Dialogue INSERT (HIGH PRIORITY)

**Current:**
```sql
INSERT INTO vendor_dialogue (vendor_profile_id, line_type, interaction_bucket, line_text)
VALUES (?, ?, ?, ?)
```

**Required:**
```sql
INSERT INTO vendor_dialogue
(vendor_profile_id, line_type, interaction_bucket, transaction_context,
 inventory_context, line_text, weight, generation_version, created_at)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, NOW())
```

**Changes Needed:**
1. Add `transaction_context` parameter (default: 'neutral')
2. Add `inventory_context` parameter (from bucket.InventoryCtx or 'none')
3. Add `weight` parameter (default: 1.0)
4. Add `generation_version` parameter (from config)

**Where to Update:**
- dialogue/orchestrator.go line ~113 (INSERT statement)
- Add transaction_context to DialogueBucket struct or hardcode 'neutral' for v1
- Pass generation_version from orchestrator

---

### 5. Configuration (HIGH PRIORITY)

**Design Requires:**
```
GENERATION_VERSION (required)
LLM_TIMEOUT_SECONDS (optional, good practice)
```

**Current:** Missing both

**Action:**
- Add to internal/config/config.go:
  ```go
  GenerationVersion int    // default: 1
  LLMTimeoutSeconds int    // default: 60
  ```
- Add to .env.example
- Add to flag parsing in main.go

---

### 6. LLM Client Settings (HIGH PRIORITY)

**Design Recommends:**
- temperature: 0.3-0.5 (current: 0.7 ❌)
- top_p: 0.6-0.8 (current: 0.9 ❌)
- max_tokens: 300-500 (current: 500 ✓)
- stream: false (missing ❌)

**Action:**
- Lower LLM_TEMPERATURE default to 0.4
- Lower LLM_TOP_P default to 0.7
- Add stream: false to ChatRequest
- Update .env.example

---

### 7. Retry Logic (MEDIUM PRIORITY)

**Current:** Reduces count by 0.8x only

**Design Requires:**
- Max 2 retries
- Second retry: lower temperature + stricter JSON instruction

**Action:**
- Check if second retry adjusts temperature
- Add explicit JSON instruction on second attempt
- Consider reducing temperature from 0.4 → 0.2 on retry

---

### 8. Prompt Builder (MEDIUM PRIORITY)

**Current:** Generic tone building

**Design Specifies:** Service type mappings + personality mappings

**Action:**
- Reference design section #8-#10
- Expand service_type handling
- Improve personality trait utilization
- Match example prompt structure

---

### 9. Transaction Context (LOW PRIORITY for v1)

**Schema Includes:** `transaction_context` ENUM('neutral', 'vendor_selling', 'vendor_buying')

**Current Implementation:** Only handles 'neutral'

**Action for v1:** Hardcode all generation to 'neutral'
- Add to DialogueBucket as constant
- Pass to INSERT

**Future:** PHP backend could specify context when calling API

---

## Priority Implementation Order

### Phase 1: Critical (Blocks Core Functionality)
1. ✅ Update VendorProfile struct (add 3 fields)
2. ✅ Update repository SELECT (add WHERE clause, 3 columns)
3. ✅ Update Scan() mapping
4. ✅ Reinstate MarkGenerating/Complete/Failed functions
5. ✅ Restore worker.processVendor() calls

### Phase 2: High (Schema Compliance)
6. ✅ Update INSERT to include transaction_context, weight, generation_version
7. ✅ Add GENERATION_VERSION config
8. ✅ Lower LLM defaults (temp 0.4, top_p 0.7)
9. ✅ Add stream: false

### Phase 3: Medium (Design Compliance)
10. Update prompt builder with design mappings
11. Improve retry logic
12. Add LLM_TIMEOUT_SECONDS config

---

## Estimated Effort

- Phase 1: 30 minutes (struct updates, query fixes, status functions)
- Phase 2: 45 minutes (INSERT updates, config, LLM settings)
- Phase 3: 60 minutes (prompt builder, retry logic)

**Total: ~2 hours for full compliance**

---

## Code Changes Summary

### Files to Modify

1. **internal/vendors/vendor.go**
   - Add 3 fields to VendorProfile struct

2. **internal/vendors/repository.go**
   - Update SELECT query (add WHERE, 3 columns)
   - Update Scan() (add 3 fields)
   - Add back 3 Mark* functions

3. **internal/dialogue/orchestrator.go**
   - Update INSERT (add 5 parameters)
   - Add transaction_context to DialogueBucket
   - Pass generation_version

4. **internal/config/config.go**
   - Add GenerationVersion field
   - Add LLMTimeoutSeconds field
   - Load from .env

5. **internal/llm/types.go**
   - Add Stream field to ChatRequest

6. **internal/llm/client.go**
   - Set stream: false in requests
   - Add timeout from config

7. **internal/jobs/worker.go**
   - Restore MarkGenerating/Complete/Failed calls

8. **.env.example**
   - Add GENERATION_VERSION=1
   - Add LLM_TIMEOUT_SECONDS=60
   - Update LLM_TEMPERATURE=0.4
   - Update LLM_TOP_P=0.7

9. **cmd/vendor-dialogue-generator/main.go**
   - Add flag for generation version if desired

---

## Verification Checklist

After implementation:
- [ ] Code compiles without errors
- [ ] `go vet ./...` passes
- [ ] Query fetches correct columns from vendor_profiles
- [ ] WHERE clause filters to pending/failed only
- [ ] Status updates work (mark generating → complete/failed)
- [ ] INSERT includes all 9 columns
- [ ] LLM temperature/top_p are within design ranges
- [ ] Stream parameter is false
- [ ] generation_version is tracked
- [ ] Tested against real database

