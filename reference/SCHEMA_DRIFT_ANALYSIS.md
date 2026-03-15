# Schema Drift Analysis
## Defined vs. Implemented

---

## TABLE: vendor_profiles

### DEFINED (Joint Design Document)

```sql
ALTER TABLE vendor_profiles ADD (
    dialogue_generation_status ENUM('pending', 'generating', 'complete', 'failed') DEFAULT 'pending',
    dialogue_generation_version INT UNSIGNED DEFAULT 1,
    dialogue_generated_at TIMESTAMP NULL DEFAULT NULL
);
```

**Expected columns in SELECT:**
- id
- service_type
- criminality
- personality
- markup_base
- dialogue_generation_status ← **TRACKING**
- dialogue_generation_version ← **TRACKING**
- dialogue_generated_at ← **TRACKING**
- created_at
- updated_at

**Plus (from earlier sections):**
- uuid
- galaxy_id
- poi_id
- trading_post_id

---

### CURRENTLY IN CODE

**repository.go FetchPending() SELECT:**
```sql
SELECT id, service_type, criminality, markup_base, personality,
       created_at, updated_at
FROM vendor_profiles
ORDER BY id
LIMIT ?
```

**VendorProfile struct:**
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

**Scan order in repository.go:**
```go
err := rows.Scan(
    &v.ID,
    &v.ServiceType,
    &v.Criminality,
    &v.MarkupBase,
    &v.PersonalityJSON,
    &v.CreatedAt,
    &v.UpdatedAt,
)
```

---

### DRIFT ANALYSIS

| Column | Defined? | Fetched? | In Struct? | Impact |
|--------|----------|----------|-----------|--------|
| id | ✓ | ✓ | ✓ | OK |
| service_type | ✓ | ✓ | ✓ | OK |
| criminality | ✓ | ✓ | ✓ | OK |
| personality | ✓ | ✓ | ✓ | OK |
| markup_base | ✓ | ✓ | ✓ | OK |
| **dialogue_generation_status** | ✓ | **✗** | **✗** | **BLOCKER** — Can't track status |
| **dialogue_generation_version** | ✓ | **✗** | **✗** | **BLOCKER** — Can't support versioning |
| **dialogue_generated_at** | ✓ | **✗** | **✗** | **BLOCKER** — Can't record when generated |
| created_at | ✓ | ✓ | ✓ | OK |
| updated_at | ✓ | ✓ | ✓ | OK |
| uuid | ✓ (from PHASE_5_9) | **✗** | **✗** | **NOT NEEDED** for v1 generation |
| galaxy_id | ✓ (from PHASE_5_9) | **✗** | **✗** | **NOT NEEDED** for v1 generation |
| poi_id | ✓ (from PHASE_5_9) | **✗** | **✗** | **NOT NEEDED** for v1 generation |
| trading_post_id | ✓ (from PHASE_5_9) | **✗** | **✗** | **NOT NEEDED** for v1 generation |

---

## TABLE: vendor_dialogue

### DEFINED (Joint Design Document)

```sql
CREATE TABLE vendor_dialogue (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_profile_id BIGINT UNSIGNED NOT NULL,
    line_type ENUM('greeting', 'inventory_pitch', 'deal_accepted', 'deal_rejected', 'farewell'),
    interaction_bucket ENUM('first_visit', 'second_visit', 'third_visit', 'repeat_customer'),
    transaction_context ENUM('neutral', 'vendor_selling', 'vendor_buying') DEFAULT 'neutral',
    inventory_context VARCHAR(64) DEFAULT 'none',
    line_text VARCHAR(255) NOT NULL,
    weight DECIMAL(5,4) DEFAULT 1.0000,
    generation_version INT UNSIGNED DEFAULT 1,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    ... indexes ...
);
```

**Expected INSERT columns:**
- vendor_profile_id ← PROVIDED
- line_type ← PROVIDED
- interaction_bucket ← PROVIDED
- transaction_context ← **REQUIRED**
- inventory_context ← **REQUIRED**
- line_text ← PROVIDED
- weight ← **REQUIRED**
- generation_version ← **REQUIRED**
- created_at ← **REQUIRED**
- updated_at ← OPTIONAL (usually set to created_at in v1)

---

### CURRENTLY IN CODE

**orchestrator.go INSERT:**
```sql
INSERT INTO vendor_dialogue (vendor_profile_id, line_type, interaction_bucket, line_text)
VALUES (?, ?, ?, ?)
```

**Parameters passed:**
```go
vendor.ID, bucket.LineType, bucket.Bucket, line
```

---

### DRIFT ANALYSIS

| Column | Defined? | Provided? | Passed to DB? | Impact |
|--------|----------|-----------|---------------|--------|
| vendor_profile_id | ✓ | ✓ | ✓ | OK |
| line_type | ✓ | ✓ | ✓ | OK |
| interaction_bucket | ✓ | ✓ | ✓ | OK |
| **transaction_context** | ✓ | **✗** | **✗** | **ERROR** — Required, no default on INSERT |
| **inventory_context** | ✓ | **PARTIAL** | **✗** | **ERROR** — Partially available (only for pitch), not passed |
| line_text | ✓ | ✓ | ✓ | OK |
| **weight** | ✓ | **✗** | **✗** | **WARNING** — Uses DB default (1.0) |
| **generation_version** | ✓ | **✗** | **✗** | **ERROR** — Critical for versioning, not passed |
| **created_at** | ✓ | **✗** | **✗** | **WARNING** — Uses implicit NOW() or DB default |
| updated_at | ✓ | **✗** | **✗** | OK — Usually left NULL in v1 |

---

## DialogueBucket Struct Drift

### DEFINED (Design)
Each generation should specify:
- line_type
- interaction_bucket
- transaction_context (or let it default to 'neutral')
- inventory_context (or let it default to 'none')
- requested_count (number of lines)

### CURRENTLY IN CODE
```go
type DialogueBucket struct {
    LineType        string
    Bucket          string
    InventoryCtx    string
    RequestedCount  int
}
```

| Field | Defined? | In Struct? | Notes |
|-------|----------|-----------|-------|
| LineType | ✓ | ✓ | Maps to line_type |
| Bucket | ✓ | ✓ | Maps to interaction_bucket |
| **TransactionContext** | ✓ | **✗** | **MISSING** — Always 'neutral' for v1 |
| InventoryCtx | ✓ | ✓ | Maps to inventory_context |
| RequestedCount | ✓ | ✓ | Request parameter |

---

## Configuration Drift

### DEFINED (Design Section #14)

```
DB_HOST ✓
DB_PORT ✓
DB_USER ✓
DB_PASS ✓
DB_NAME ✓

LLM_BASE_URL ✓
LLM_MODEL ✓
LLM_TIMEOUT_SECONDS ✗ (missing)

GENERATION_VERSION ✗ (missing)
WORKER_COUNT ✓
BATCH_SIZE ✓
MAX_RETRIES (or GENERATION_RETRY_MAX) ✓

(Optional)
SHOP_TYPE_FILTER ✗
VENDOR_LIMIT ✗
LOG_LEVEL ✓
DRY_RUN ✓
```

### CURRENTLY IN CODE

```go
type Config struct {
    // Database ✓
    DBHost, DBPort, DBUser, DBPassword, DBName

    // LLM
    LLMBaseURL ✓
    LLMModel ✓
    LLMTemperature 0.7 (✓ but wrong value: design says 0.3-0.5)
    LLMTopP 0.9 (✓ but wrong value: design says 0.6-0.8)
    LLMMaxTokens 500 ✓

    // Job processing
    WorkerCount ✓
    BatchSize ✓
    GenerationRetryMax ✓
    RowsToGenerate ✓ (user-requested, not in design)

    // Debug
    DryRun ✓
    LogLevelString ✓

    // MISSING
    // GenerationVersion ✗
    // LLMTimeoutSeconds ✗
}
```

---

## LLM Client Settings Drift

### DEFINED (Design Section #11-12)

```json
{
  "model": "<configured>",
  "messages": [...],
  "temperature": 0.3-0.5,
  "top_p": 0.6-0.8,
  "max_tokens": 300-500,
  "stream": false
}
```

### CURRENTLY IN CODE

```go
type ChatRequest struct {
    Model      string
    Messages   []ChatMessage
    Temperature float64    // Currently 0.7 ❌
    TopP       float64     // Currently 0.9 ❌
    MaxTokens  int         // 500 ✓
    // Stream field MISSING ❌
}

// Set in New():
cfg.LLMTemperature = 0.7  // Should be 0.4
cfg.LLMTopP = 0.9         // Should be 0.7
```

---

## WHERE Clause Drift

### DEFINED (Design Section #15)

```sql
SELECT * FROM vendor_profiles
WHERE dialogue_generation_status IN ('pending', 'failed')
ORDER BY id
LIMIT ?
```

### CURRENTLY IN CODE

```sql
SELECT ... FROM vendor_profiles
ORDER BY id
LIMIT ?
-- NO WHERE CLAUSE
```

**Effect:** Fetches ALL vendors, including those already complete. No filtering for work needed.

---

## Status Update Drift

### DEFINED (Design Section #16)

Before generation:
```sql
UPDATE vendor_profiles SET dialogue_generation_status = 'generating' WHERE id = ?;
```

On success:
```sql
UPDATE vendor_profiles
SET dialogue_generation_status = 'complete', dialogue_generated_at = NOW()
WHERE id = ?;
```

On failure:
```sql
UPDATE vendor_profiles SET dialogue_generation_status = 'failed' WHERE id = ?;
```

### CURRENTLY IN CODE

- All three functions removed (MarkGenerating, MarkComplete, MarkFailed)
- No status updates occur
- No calls to update functions in worker.go

**Effect:** No tracking of generation progress; can't distinguish complete from pending.

---

## Summary Table

| Category | Drift | Severity | Impact |
|----------|-------|----------|--------|
| **vendor_profiles SELECT** | Missing 3 columns | CRITICAL | Can't track status/version |
| **vendor_profiles WHERE** | No filter clause | HIGH | Fetches unnecessary vendors |
| **vendor_dialogue INSERT** | Missing 5 parameters | CRITICAL | INSERT will fail or ignore fields |
| **Status updates** | Functions removed | CRITICAL | No progress tracking |
| **DialogueBucket struct** | Missing transaction_context | MEDIUM | Hardcoded to 'neutral' for v1 |
| **LLM temperatures** | Too high (0.7/0.9) | MEDIUM | Model drift, poor output |
| **LLM stream field** | Missing | LOW | Async vs streaming not explicit |
| **Config variables** | Missing GENERATION_VERSION | HIGH | Can't support regeneration |

---

## Quick Fix Checklist

- [ ] Add 3 fields to VendorProfile struct
- [ ] Update SELECT query to fetch those 3 columns
- [ ] Add WHERE clause to filter pending/failed
- [ ] Update Scan() to map new fields
- [ ] Restore MarkGenerating/Complete/Failed functions
- [ ] Add transaction_context to DialogueBucket or hardcode 'neutral'
- [ ] Update INSERT to 9 columns (add transaction_context, inventory_context, weight, generation_version, created_at)
- [ ] Lower LLMTemperature to 0.4
- [ ] Lower LLMTopP to 0.7
- [ ] Add stream: false to ChatRequest
- [ ] Add GenerationVersion config var
- [ ] Add LLMTimeoutSeconds config var
- [ ] Restore worker.processVendor() calls to Mark* functions

