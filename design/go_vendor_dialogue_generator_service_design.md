# Vendor Dialogue Generator Service – Go Application Design
## Space Wars 3002

## Overview

This document defines the Go-based vendor dialogue generator service.

The service is responsible for:

- reading vendor records from the main MariaDB database
- identifying vendors that need dialogue generation or regeneration
- constructing prompts based on vendor profile data
- calling the local `llama.cpp` server
- validating and normalizing generated dialogue
- storing dialogue in the `vendor_dialogue` table
- updating generation status fields on `vendor_profiles`

This service is asynchronous and offline from the gameplay request path.

---

# 1. Responsibilities

The Go generator **must**:

- read from `vendor_profiles`
- write to `vendor_dialogue`
- update dialogue generation status fields on `vendor_profiles`
- batch generation work
- validate all LLM responses before storing them
- support regeneration via versioning

The Go generator **must not**:

- calculate gameplay prices
- decide negotiation outcomes
- serve player-facing API requests
- mutate vendor business logic
- rely on the PHP API for normal generation work

---

# 2. Source Data

The generator reads from `vendor_profiles`.

Current structure includes:

- `id`
- `uuid`
- `galaxy_id`
- `poi_id`
- `trading_post_id`
- `service_type`
- `criminality`
- `personality`
- `markup_base`
- `dialogue_generation_status`
- `dialogue_generation_version`
- `dialogue_generated_at`

Example `personality` payload:

```json
{
  "honesty": 0.45,
  "greed": 0.55,
  "charm": 0.44,
  "risk_tolerance": 0.76
}
```

The generator must parse this JSON and convert it into prompt instructions.

---

# 3. Target Data

The generator writes rows into `vendor_dialogue`.

Expected fields:

- `vendor_profile_id`
- `line_type`
- `interaction_bucket`
- `inventory_context`
- `line_text`
- `weight`
- `generation_version`

The generator should treat `vendor_dialogue` as append/replace data scoped by vendor and generation version.

---

# 4. Line Types

## Recommended v1 line types

Start small.

- `greeting`
- `inventory_pitch`
- `deal_accepted`
- `deal_rejected`
- `farewell`

## Recommended future line types

- `lockout_warning`
- `deal_counter`
- `deal_insulted`
- `returning_greeting`
- `rare_item_pitch`
- `criminal_offer`
- `reputation_reaction`

Do not implement all of these in v1.

---

# 5. Interaction Buckets

Use visit/relationship buckets instead of exact counts.

## Required v1 buckets

- `first_visit`
- `second_visit`
- `third_visit`
- `repeat_customer`

These buckets are stable enough for dialogue reuse and do not create absurd content volume.

---

# 6. Inventory Context Strategy

The generator may optionally create inventory-specific lines.

## v1 recommendation

Support inventory context only for `inventory_pitch`.

Examples:

- `shield_projector`
- `engine`
- `reactor`
- `hull_plating`
- `ship`

If no context-specific line exists, the API will fall back to generic lines.

Do not try to generate dialogue for every exact item SKU in v1.

---

# 7. Generation Flow

```text
load config
connect to MariaDB
fetch vendors with pending or stale dialogue
for each vendor:
    mark as generating
    for each required line_type / interaction_bucket pair:
        build prompt
        call llama-server
        parse response
        validate lines
        delete old rows for that scope and generation version
        insert new dialogue rows
    mark vendor complete
```

---

# 8. Prompt Strategy

The model should be asked for **small batches of short lines**.

## General prompt rules

Every prompt should specify:

- vendor role / service type
- interaction bucket
- personality values
- criminality
- markup base
- optional inventory example
- output schema
- hard stylistic rules

## Example line request

- 15 greetings for first visit
- 15 greetings for second visit
- 20 greetings for third/repeat visit
- 10 inventory pitches for generic ship
- 10 inventory pitches for a shield projector example

Keep batches small enough that validation and retries remain cheap.

---

# 9. Prompt Builder Rules

The generator should not pass raw JSON blindly and hope for the best.

It should derive prompt hints from vendor data.

## Example mappings

### `service_type`
Controls the vendor role:
- `salvage_yard` → rougher, opportunistic, used goods
- `shipyard` → polished, prideful, sales-oriented
- `trading_hub` → general merchant tone
- `market` → broader commercial tone

### `criminality`
Affects tone:
- low → more civil
- medium → sharper, more evasive
- high → rough, predatory, or illicit edge

### `markup_base`
Affects pricing posture:
- high markup → defensive, premium language
- low/negative markup → casual, bargain language

### `personality`
Suggested initial mappings:
- `honesty` → bluntness / fairness / sales truthfulness
- `greed` → defensive markup, harder sell
- `charm` → warmth, smoothness, social ease
- `risk_tolerance` → willingness to exaggerate or sell questionable goods

---

# 10. Example Prompt Template

## System message

```text
You generate short in-universe dialogue for NPC vendors in a space exploration and trading game.
Output JSON only.
Do not include explanations.
Keep every line as one sentence.
```

## User message template

```text
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
- may acknowledge the player has visited before
- return JSON: {"lines":["..."]}
```

## Inventory pitch template

```text
Generate 10 inventory pitch lines for a salvage vendor selling a used shield projector.

Vendor context:
- service_type: salvage_yard
- interaction_bucket: repeat_customer
- criminality: 0.27
- markup_base: 0.2152

Personality:
- honesty: 0.45
- greed: 0.55
- charm: 0.44
- risk_tolerance: 0.76

Inventory example:
- item_type: shield_projector
- condition: 76

Rules:
- one sentence per line
- 8 to 18 words
- may reference used equipment
- should sound like a sales pitch
- return JSON: {"lines":["..."]}
```

---

# 11. LLM API Contract

Use the local `llama.cpp` OpenAI-compatible endpoint.

## Endpoint

`POST /v1/chat/completions`

## Required request fields

- `model`
- `messages`
- `temperature`
- `top_p`
- `max_tokens`
- `stream`

## Recommended defaults

- `temperature: 0.3 - 0.5`
- `top_p: 0.6 - 0.8`
- `max_tokens: 300 - 500`
- `stream: false`

Higher temperatures will make small models drift.

---

# 12. Example HTTP Request Payload

```json
{
  "model": "qwen2.5-1.5b-instruct",
  "messages": [
    {
      "role": "system",
      "content": "You generate short in-universe dialogue for NPC vendors in a space exploration and trading game. Output JSON only."
    },
    {
      "role": "user",
      "content": "Generate 15 greetings for a salvage vendor. Vendor context: service_type salvage_yard, interaction_bucket third_visit, criminality 0.27, markup_base 0.2152. Personality: honesty 0.45, greed 0.55, charm 0.44, risk_tolerance 0.76. Rules: one sentence per line, 8 to 16 words, slightly sarcastic but not abusive, may acknowledge prior visits, return JSON: {\"lines\":[\"...\"]}"
    }
  ],
  "temperature": 0.4,
  "top_p": 0.7,
  "max_tokens": 400,
  "stream": false
}
```

---

# 13. Go Project Structure

Suggested package layout:

```text
cmd/vendor-dialogue-generator/main.go
internal/config
internal/db
internal/vendors
internal/dialogue
internal/prompts
internal/llm
internal/validation
internal/logging
internal/jobs
```

## Package responsibilities

### `config`
- environment variables
- runtime settings
- feature flags

### `db`
- MariaDB connection
- SQL helpers / repository layer

### `vendors`
- vendor loading
- status updates
- vendor parsing

### `dialogue`
- orchestration for line generation and storage

### `prompts`
- prompt templates
- mapping vendor fields to prompt hints

### `llm`
- HTTP client for `llama.cpp`

### `validation`
- JSON parsing
- line validation
- dedupe

### `jobs`
- batching and worker coordination

---

# 14. Configuration

Use environment variables.

## Required

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASS

LLM_BASE_URL
LLM_MODEL
LLM_TIMEOUT_SECONDS

GENERATION_VERSION
WORKER_COUNT
BATCH_SIZE
MAX_RETRIES
```

## Optional

```text
SHOP_TYPE_FILTER
VENDOR_LIMIT
LOG_LEVEL
DRY_RUN
```

---

# 15. SQL Read Strategy

Fetch vendors that need work.

Example:

```sql
SELECT *
FROM vendor_profiles
WHERE dialogue_generation_status IN ('pending', 'failed')
ORDER BY id
LIMIT ?
```

For regeneration by version:

```sql
SELECT *
FROM vendor_profiles
WHERE dialogue_generation_version = ?
  AND dialogue_generation_status IN ('pending', 'failed')
ORDER BY id
LIMIT ?
```

---

# 16. Status Update Strategy

## Before generation

```sql
UPDATE vendor_profiles
SET dialogue_generation_status = 'generating'
WHERE id = ?;
```

## On success

```sql
UPDATE vendor_profiles
SET dialogue_generation_status = 'complete',
    dialogue_generated_at = NOW()
WHERE id = ?;
```

## On failure

```sql
UPDATE vendor_profiles
SET dialogue_generation_status = 'failed'
WHERE id = ?;
```

---

# 17. Replacement Strategy for Dialogue Rows

Before inserting new rows for a given scope, delete old rows for that vendor and generation version scope.

Example:

```sql
DELETE FROM vendor_dialogue
WHERE vendor_profile_id = ?
  AND line_type = ?
  AND interaction_bucket = ?;
```

Then insert fresh rows.

This keeps the dataset clean and avoids ambiguity.

---

# 18. Validation Rules

The generator must validate the model output before saving.

## Required checks

- valid JSON object
- contains `lines`
- `lines` is an array
- minimum line count threshold reached
- line length within limits
- no empty strings
- no duplicates after normalization
- no disallowed control characters

## Recommended line limits

- min words: 6
- max words: 20
- max characters: 255

## Optional checks

- reject lines containing repeated punctuation like `!!!`
- reject malformed quoting
- reject obvious meta output like `Here are 15 lines:`

---

# 19. Normalization and Deduplication

## Normalize by

- trimming whitespace
- lowercasing
- collapsing internal whitespace
- stripping duplicate terminal punctuation where sensible

## Exact dedupe

Hash normalized text and reject duplicates.

For v1, exact dedupe is sufficient.

---

# 20. Retry Logic

If generation fails:

## Retry cases

- malformed JSON
- not enough lines
- timeout
- all or most lines invalid

## Retry policy

- maximum 2 retries per bucket
- second retry uses a stricter prompt:
  - lower temperature
  - more explicit JSON-only instruction
  - shorter requested line count if needed

If retries fail, mark vendor as `failed`.

---

# 21. Worker Model

Because the machine is modest, keep the worker count low.

## Recommendation

- `WORKER_COUNT=1` or `2`
- `BATCH_SIZE=10` to `25` vendors per pass

Do not try to saturate the machine.

---

# 22. Initial Generation Scope

Recommended v1 generation matrix:

## Greetings
- `greeting` + `first_visit` → 15 lines
- `greeting` + `second_visit` → 15 lines
- `greeting` + `third_visit` → 15 lines
- `greeting` + `repeat_customer` → 20 lines

## Inventory pitches
- `inventory_pitch` + `repeat_customer` + `ship` → 10 lines
- `inventory_pitch` + `repeat_customer` + `shield_projector` → 10 lines

## Deal accepted
- `deal_accepted` + `repeat_customer` → 10 lines

## Deal rejected
- `deal_rejected` + `repeat_customer` → 10 lines

## Farewell
- `farewell` + `repeat_customer` → 10 lines

That is enough to prove the architecture.

---

# 23. Suggested Example Data Flow

```text
vendor_profiles row loaded
        ↓
personality parsed
        ↓
prompt hints derived
        ↓
prompt built for greeting / third_visit
        ↓
llama.cpp called
        ↓
JSON parsed
        ↓
15 valid lines accepted
        ↓
rows inserted into vendor_dialogue
        ↓
vendor marked complete
```

---

# 24. Logging

Log every generation pass with:

- vendor ID
- line type
- interaction bucket
- prompt hash
- response size
- inserted row count
- validation failures
- retry count
- final status

Do not log entire prompts and raw responses at debug level forever in production, or the logs will become garbage.

Recommended:
- raw prompt/response logging only in debug mode

---

# 25. Prompt Hashing

Store a deterministic hash of the built prompt template if needed for debugging and reproducibility.

This helps answer:
- which prompt template produced these rows
- whether a change in prompt logic caused bad output

Use SHA-256 over normalized prompt text.

---

# 26. Regeneration Policy

Regenerate dialogue when:

- `dialogue_generation_version` changes
- `personality` changes
- `criminality` changes
- `markup_base` changes
- prompt template version changes
- model changes materially

The PHP backend should only mark vendors stale.
The Go generator performs the actual regeneration.

---

# 27. Failure Modes

## LLM unavailable
- mark vendor failed
- retry later

## Bad output
- retry with stricter settings
- if still bad, fail cleanly

## DB error
- do not mark complete
- return vendor to `failed`

## Partial success
- if one bucket fails, mark vendor failed overall in v1
- later you can move to bucket-level status if needed

---

# 28. Future Improvements

- personality-to-tone mapping table
- archetype-specific prompt templates
- item-specific pitch generation by component type
- near-duplicate detection
- per-service-type specialized workers
- optional generation queue table for explicit scheduling

---

# 29. Minimal Queue Table (Optional)

If you later want more explicit control:

```sql
CREATE TABLE vendor_dialogue_generation_jobs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_profile_id BIGINT UNSIGNED NOT NULL,
    status ENUM('pending','running','complete','failed') NOT NULL DEFAULT 'pending',
    attempts INT NOT NULL DEFAULT 0,
    scheduled_at TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
    started_at TIMESTAMP NULL DEFAULT NULL,
    finished_at TIMESTAMP NULL DEFAULT NULL,
    error_message TEXT NULL,
    created_at TIMESTAMP NULL DEFAULT NULL,
    updated_at TIMESTAMP NULL DEFAULT NULL,

    INDEX idx_status_scheduled (status, scheduled_at),
    CONSTRAINT fk_vendor_dialogue_job_vendor
        FOREIGN KEY (vendor_profile_id)
        REFERENCES vendor_profiles(id)
        ON DELETE CASCADE
);
```

This is optional. You can skip it in v1.

---

# 30. Summary

The Go generator service should:

- read vendor profiles that need dialogue
- convert vendor fields into prompt instructions
- call the local `llama.cpp` API
- validate and normalize returned lines
- store rows in `vendor_dialogue`
- update generation status on `vendor_profiles`

This service should remain fully separate from the gameplay API.

That separation ensures:
- no gameplay latency
- cleaner architecture
- easier regeneration
- simpler debugging
