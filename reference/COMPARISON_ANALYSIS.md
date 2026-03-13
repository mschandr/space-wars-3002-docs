# Comparison: go_vendor_dialogue_generator_service_design.md vs Current Implementation

**Date**: 2026-03-10
**Status**: Analysis Complete

---

## Executive Summary

The design document specifies a **separate Go service** that generates vendor dialogue using **LLM (llama.cpp)** in batch mode. The current implementation is a **static PHP/Laravel table structure** with no generation capability or status tracking.

**Gap**: Design requires a Go generator service with LLM integration; current repo only has the data model.

---

## Part 1: Architectural Alignment

### ✅ ALIGNED

1. **Target Table Structure** (`vendor_dialogue`)
   - ✅ Created with correct columns
   - ✅ Correct field types
   - ✅ Correct indexes
   - ✅ Foreign key to vendor_profiles

2. **Line Types** (5 required)
   - ✅ `greeting` - implemented
   - ✅ `inventory_pitch` - implemented
   - ✅ `deal_accepted` - implemented
   - ✅ `deal_rejected` - implemented
   - ✅ `farewell` - implemented

3. **Interaction Buckets** (4 required)
   - ✅ `first_visit` - implemented
   - ✅ `second_visit` - implemented
   - ✅ `third_visit` - implemented
   - ✅ `repeat_customer` - implemented

4. **Inventory Context**
   - ✅ Optional field, nullable
   - ✅ VARCHAR(64) for item type references
   - ✅ Scoped to `inventory_pitch` line type

5. **Data Model**: `VendorDialogue` class
   - ✅ Proper model with relationships
   - ✅ Scopes and helper methods
   - ✅ Weight-based selection logic

---

## Part 2: MISSING COMPONENTS (Critical Gaps)

### 🔴 MISSING: Vendor Profile Status Fields

The design document (Section 2, 58-60) specifies these fields on `vendor_profiles`:

```
- `dialogue_generation_status`  → ENUM('pending', 'generating', 'complete', 'failed')
- `dialogue_generation_version` → INT UNSIGNED
- `dialogue_generated_at`       → TIMESTAMP
```

**Current State**:
- ❌ `dialogue_generation_status` - NOT added to vendor_profiles
- ❌ `dialogue_generation_version` - NOT added to vendor_profiles
- ❌ `dialogue_generated_at` - NOT added to vendor_profiles

**Impact**: Generator has no way to track which vendors need generation or their status.

**Solution Required**: Add migration to vendor_profiles with these 3 fields.

---

### 🔴 MISSING: Go Generator Service

The entire Go application is missing:

**Required but not implemented**:
- ❌ Go project structure (cmd/, internal/ packages)
- ❌ Database connection layer (Go)
- ❌ Vendor loading and status updates
- ❌ Prompt building and template system
- ❌ LLM HTTP client for llama.cpp
- ❌ JSON validation and normalization
- ❌ Batch processing and worker coordination
- ❌ Configuration management (environment variables)
- ❌ Logging system
- ❌ Retry logic with stricter prompts

**Sections from document covering this**:
- Section 13: Go Project Structure (packages: config, db, vendors, dialogue, prompts, llm, validation, jobs)
- Section 14-26: Configuration, SQL strategies, validation rules, retry logic

---

### 🔴 MISSING: LLM Integration

The design requires integration with **llama.cpp** OpenAI-compatible endpoint:

**Not implemented**:
- ❌ HTTP client for `/v1/chat/completions`
- ❌ Request/response handling
- ❌ Temperature/top_p/max_tokens configuration
- ❌ Prompt templates (Section 10)
- ❌ Model loading and selection

**Required configuration** (Section 14):
```
LLM_BASE_URL              → NOT in config
LLM_MODEL                 → NOT in config
LLM_TIMEOUT_SECONDS       → NOT in config
```

---

### 🔴 MISSING: Prompt Building System

The design specifies detailed prompt construction (Section 8-11):

**Not implemented**:
- ❌ Prompt template engine
- ❌ Vendor-to-prompt-hint mapping:
  - ❌ Service type → tone mapping (salvage_yard, shipyard, trading_hub, market)
  - ❌ Criminality → tone adjustment
  - ❌ Markup_base → pricing posture
  - ❌ Personality (honesty, greed, charm, risk_tolerance) → dialogue traits
- ❌ Context-specific inventory prompts
- ❌ System message generation

---

### 🔴 MISSING: Validation & Normalization

The design specifies (Section 18-19):

**Not implemented**:
- ❌ JSON parsing and schema validation
- ❌ Line count thresholds (min 3-5, max based on context)
- ❌ Line length validation (6-20 words, max 255 chars)
- ❌ Empty string rejection
- ❌ Control character filtering
- ❌ Duplicate detection after normalization
- ❌ Punctuation validation (reject `!!!`)
- ❌ Meta-text rejection (`Here are 15 lines:`)

---

### 🔴 MISSING: Batch Processing

The design specifies (Section 22-24):

**Not implemented**:
- ❌ Batch generation matrix
- ❌ Worker coordination (WORKER_COUNT config)
- ❌ Batch size management (BATCH_SIZE config)
- ❌ Generation scope scheduling
- ❌ Retry logic with backoff

---

### 🔴 MISSING: Database Operations

Required SQL strategies (Section 15-17):

**Not implemented**:
- ❌ SELECT vendors with `dialogue_generation_status` filtering
- ❌ UPDATE vendor status to `generating`, `complete`, `failed`
- ❌ DELETE old dialogue rows before regeneration
- ❌ INSERT new rows in batch
- ❌ Transaction handling

---

### 🔴 MISSING: Regeneration Policy

Design Section 26 specifies regeneration triggers:

**Not implemented**:
- ❌ Personality change detection
- ❌ Criminality change detection
- ❌ Markup_base change detection
- ❌ Version tracking in dialogue rows
- ❌ Stale dialogue detection

---

## Part 3: What IS Correctly Implemented

### ✅ Table Schema (100% complete)

```php
vendor_dialogue table {
    ✅ id
    ✅ vendor_profile_id (FK)
    ✅ line_type (ENUM: greeting, inventory_pitch, deal_accepted, deal_rejected, farewell)
    ✅ interaction_bucket (ENUM: first_visit, second_visit, third_visit, repeat_customer)
    ✅ inventory_context (VARCHAR(64), nullable)
    ✅ line_text (VARCHAR(255))
    ✅ weight (DECIMAL(5,4), default 1.0)
    ✅ generation_version (INT UNSIGNED, default 1)
    ✅ timestamps
    ✅ indexes (vendor_line_type, vendor_lookup, inventory_context)
    ✅ cascade delete FK
}
```

### ✅ PHP Data Model

```php
✅ VendorDialogue model with:
    ✅ Proper relationships
    ✅ Scopes (forVendor, byLineType, byBucket, byVersion)
    ✅ Static getDialogue() with weighted random selection
    ✅ getVendorDialogue() with optional filters
    ✅ Factory with state methods
    ✅ Casts for enums and decimals

✅ Enums:
    ✅ DialogueLineType (greeting, inventory_pitch, deal_accepted, deal_rejected, farewell)
    ✅ InteractionBucket (first_visit, second_visit, third_visit, repeat_customer)
    ✅ Both with label() and description() methods

✅ VendorProfile integration:
    ✅ Added dialogueLines() hasMany relationship
```

---

## Part 4: Changes Required to Align with Design

### CRITICAL (Must implement)

#### 1. Add Status Fields to vendor_profiles Migration

**File**: `database/migrations/XXXX_add_dialogue_status_to_vendor_profiles.php`

```php
Schema::table('vendor_profiles', function (Blueprint $table) {
    $table->enum('dialogue_generation_status', [
        'pending',
        'generating',
        'complete',
        'failed'
    ])->default('pending')->after('markup_base');

    $table->unsignedInteger('dialogue_generation_version')
        ->default(1)
        ->after('dialogue_generation_status');

    $table->timestamp('dialogue_generated_at')
        ->nullable()
        ->after('dialogue_generation_version');

    $table->index('dialogue_generation_status');
});
```

---

#### 2. Create Go Generator Service Structure

**Directory**: `/vendor-dialogue-generator/` (separate repo or `/go/` subdir)

**Packages needed**:
```
cmd/vendor-dialogue-generator/
  main.go

internal/
  config/
    config.go         → Parse env vars (DB_*, LLM_*, GENERATION_*, WORKER_*, etc.)

  db/
    client.go         → MariaDB connection
    queries.go        → Vendor fetching, status updates, dialogue storage

  vendors/
    loader.go         → Load vendor profiles
    parser.go         → Parse personality JSON to prompt hints

  prompts/
    builder.go        → Build prompts from vendor data
    templates.go      → Prompt template definitions
    mapping.go        → Service type / criminality / personality mappings

  llm/
    client.go         → HTTP client for llama.cpp
    request.go        → Build OpenAI-compatible requests
    response.go       → Parse JSON responses

  dialogue/
    generator.go      → Main orchestration
    validator.go      → Validate, normalize, dedupe lines

  jobs/
    worker.go         → Worker pool
    batch.go          → Batch processing

  logging/
    logger.go         → Structured logging
```

---

#### 3. Environment Configuration

**Required** (no defaults):
```bash
DB_HOST=localhost
DB_PORT=3306
DB_NAME=space_wars_3002
DB_USER=user
DB_PASS=pass

LLM_BASE_URL=http://localhost:8000
LLM_MODEL=qwen2.5-1.5b-instruct
LLM_TIMEOUT_SECONDS=30

GENERATION_VERSION=1
WORKER_COUNT=1
BATCH_SIZE=10
MAX_RETRIES=2
```

**Optional**:
```bash
SHOP_TYPE_FILTER=salvage_yard  # If filtering to specific type
VENDOR_LIMIT=0                  # 0 = unlimited
LOG_LEVEL=info
DRY_RUN=false
```

---

#### 4. Prompt Templates

**Required**: Define prompt templates that map vendor data to LLM prompts.

**Example system message**:
```
You generate short in-universe dialogue for NPC vendors in a space exploration trading game.
Output JSON only. Do not include explanations. Keep every line as one sentence.
```

**Example user message template**:
```
Generate 15 greetings for a salvage vendor.

Vendor context:
- service_type: salvage_yard
- interaction_bucket: third_visit
- criminality: 0.27
- markup_base: 0.2152

Personality:
- honesty: 0.45
- greed: 0.55
- charm: 0.44
- risk_tolerance: 0.76

Rules:
- one sentence per line
- 8 to 16 words
- tone slightly sarcastic but not abusive
- may acknowledge prior visits
- return JSON: {"lines":["..."]}
```

---

#### 5. Validation Rules

**Required validation before storing**:

```go
func ValidateLines(lines []string) (valid []string, err error) {
    // Must implement:

    ✅ JSON structure check (parse successful)
    ✅ "lines" array exists and is array
    ✅ Minimum line count (3-5 depending on context)
    ✅ Maximum line count (based on request)
    ✅ Line length: 6-20 words, max 255 chars
    ✅ No empty strings
    ✅ No control characters (\x00, \x01, etc.)
    ✅ No duplicate punctuation (!!!, ???, etc.)
    ✅ No meta-text (`Here are 15 lines:`)
    ✅ Exact deduplication after normalization
    ✅ Trim whitespace
    ✅ Collapse internal whitespace
}
```

---

#### 6. Generation Flow Orchestration

**Required workflow**:

```
Main loop:
  ├─ Load config
  ├─ Connect to MariaDB
  ├─ Fetch vendors: WHERE dialogue_generation_status IN ('pending', 'failed')
  ├─ For each vendor:
  │  ├─ UPDATE to 'generating'
  │  ├─ Parse personality JSON
  │  ├─ For each required line_type/bucket pair:
  │  │  ├─ Build prompt
  │  │  ├─ Call llama.cpp
  │  │  ├─ Parse response
  │  │  ├─ Validate lines (retry 2x if needed)
  │  │  ├─ DELETE old rows (vendor_id, line_type, bucket)
  │  │  ├─ INSERT new rows
  │  │  └─ Log result
  │  ├─ UPDATE vendor to 'complete' + dialogue_generated_at
  │  └─ Log final status
  └─ End
```

---

#### 7. Retry Logic

**Required**:
- First attempt: temperature 0.4, top_p 0.7, full request
- Second attempt: temperature 0.2, top_p 0.5, more explicit JSON instruction, maybe lower line count
- Third attempt: fail vendor, mark as `failed`

---

### HIGH PRIORITY (Implement for MVP)

#### 8. Vendor Profile Model Updates

```php
// In VendorProfile model

protected $fillable = [
    // ... existing ...
    'dialogue_generation_status',
    'dialogue_generation_version',
    'dialogue_generated_at',
];

protected $casts = [
    // ... existing ...
    'dialogue_generation_status' => 'string',  // or custom enum
    'dialogue_generation_version' => 'integer',
    'dialogue_generated_at' => 'datetime',
];

// Add scope for finding vendors needing work
public function scopeNeedingDialogueGeneration($query) {
    return $query->whereIn('dialogue_generation_status', ['pending', 'failed']);
}
```

---

#### 9. Optional: Dialogue Generation Queue Table

**Design Section 29** - Optional for v1, useful for explicit scheduling:

```php
// Migration
Schema::create('vendor_dialogue_generation_jobs', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('vendor_profile_id');
    $table->enum('status', ['pending', 'running', 'complete', 'failed'])
        ->default('pending');
    $table->unsignedInteger('attempts')->default(0);
    $table->timestamp('scheduled_at')->useCurrent();
    $table->timestamp('started_at')->nullable();
    $table->timestamp('finished_at')->nullable();
    $table->text('error_message')->nullable();
    $table->timestamps();

    $table->index(['status', 'scheduled_at']);
    $table->foreign('vendor_profile_id')
        ->references('id')->on('vendor_profiles')
        ->onDelete('cascade');
});
```

**Status**: OPTIONAL for v1, skip if not needed

---

### MEDIUM PRIORITY (Implement for completeness)

#### 10. PHP API to Trigger Generator

Optional: Add endpoint to manually trigger generation:

```php
// API Controller
Route::post('/api/vendors/{uuid}/regenerate-dialogue', [VendorController::class, 'regenerateDialogue']);

// Sets vendor.dialogue_generation_status = 'pending'
// Go service picks it up on next run
```

---

#### 11. Logging and Monitoring

**Implement** (in Go service):
- Log every generation pass with: vendor_id, line_type, bucket, prompt_hash, response_size, inserted_count, validation_failures, retry_count, status
- Do NOT log raw prompts/responses in production (only in DEBUG mode)

---

## Part 5: Summary of Changes Needed

### To Align Repository with Design Document:

| Component | Status | Priority | Effort |
|-----------|--------|----------|--------|
| vendor_dialogue table | ✅ Complete | — | Done |
| VendorDialogue model | ✅ Complete | — | Done |
| DialogueLineType enum | ✅ Complete | — | Done |
| InteractionBucket enum | ✅ Complete | — | Done |
| vendor_profiles status fields | ❌ Missing | CRITICAL | 30min |
| Go generator service | ❌ Missing | CRITICAL | 2-3 days |
| LLM integration (llama.cpp) | ❌ Missing | CRITICAL | 1-2 days |
| Prompt building system | ❌ Missing | CRITICAL | 1 day |
| Validation & normalization | ❌ Missing | CRITICAL | 4-6 hours |
| Batch processing + workers | ❌ Missing | HIGH | 4-6 hours |
| Database operations layer (Go) | ❌ Missing | HIGH | 4-6 hours |
| Retry logic | ❌ Missing | HIGH | 2-3 hours |
| Configuration management | ❌ Missing | HIGH | 2 hours |
| Logging system | ❌ Missing | MEDIUM | 1-2 hours |
| Generation queue table | ❌ Missing | MEDIUM | 30min |
| PHP API trigger endpoint | ❌ Missing | MEDIUM | 1 hour |

---

## Part 6: Recommended Implementation Order

### Phase 1: Foundation (1 day)
1. Add 3 status fields to vendor_profiles
2. Start Go project structure
3. Set up database connection layer (Go)
4. Implement vendor loader

### Phase 2: LLM Integration (2 days)
1. Build prompt templates
2. Implement llama.cpp HTTP client
3. Build prompt builder (vendor → prompt hints)
4. Test with manual prompt calls

### Phase 3: Validation & Storage (1 day)
1. Implement JSON validation and normalization
2. Implement batch insertion logic
3. Write dialogue deletion/replace logic

### Phase 4: Orchestration (1 day)
1. Build main generation loop
2. Implement retry logic
3. Build worker pool
4. Status update logic

### Phase 5: Polish (1 day)
1. Logging system
2. Configuration management
3. Error handling
4. Manual testing

---

## Conclusion

**Current State**: PHP data model is complete and correct. Table structure matches design perfectly.

**Missing**: The **entire Go generator service**. This is the core component that brings dialogue to life.

**Effort to Align**: ~2-3 weeks for a competent Go developer, or ~4 weeks for iterative development.

**Recommendation**:
1. Commit current PHP/Laravel work (table + model)
2. Create separate Go project
3. Implement in phases above
4. Integrate via status fields on vendor_profiles
