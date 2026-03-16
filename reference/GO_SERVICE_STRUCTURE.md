# Go Service — Directory and File Reference
## space-wars-3002-text-generation

This document describes every directory and file in the Go dialogue generator service.

---

## Overview

The service is an offline Go process that reads vendor profiles from the database, generates flavour dialogue using a local `llama.cpp` instance, validates the output, and stores the results — either by writing directly to the database or by submitting lines to the PHP backend via its internal HTTP API.

It is designed to run as a batch job (cron, manual trigger, or CI step), not as a long-running HTTP server.

---

## Repository Root

```
space-wars-3002-text-generation/
├── cmd/
│   └── vendor-dialogue-generator/
│       └── main.go
├── docs/                          ← git submodule (shared docs repo)
├── internal/
│   ├── config/
│   ├── constants/
│   ├── db/
│   ├── dialogue/
│   ├── jobs/
│   ├── llm/
│   ├── logging/
│   ├── php/
│   ├── prompts/
│   ├── validation/
│   └── vendors/
├── .env                           ← gitignored local config
├── .env.example
├── .gitignore
├── .gitmodules
├── go.mod
├── go.sum
├── LICENSE.md
├── Makefile
├── README.md
└── vendor-dialogue-generator      ← compiled binary (gitignored)
```

---

## Root Files

### `go.mod`
Module declaration. Module name is `space-wars-3002-text-generation`. Requires Go 1.21. Two external dependencies:
- `github.com/go-sql-driver/mysql v1.7.1` — MariaDB/MySQL driver
- `github.com/joho/godotenv v1.5.1` — `.env` file loader

### `go.sum`
Dependency checksums. Managed automatically by `go mod tidy`. Do not edit manually.

### `Makefile`
Convenience targets:
- `make build` — compiles the binary to `./vendor-dialogue-generator`
- `make run` — runs the service directly via `go run`
- `make test` — runs all tests with `go test ./...`
- `make tidy` — runs `go mod tidy` to prune unused dependencies
- `make lint` — runs `go vet ./...` for static analysis

### `.env`
Local environment configuration. Gitignored. Copy from `.env.example` and fill in real values.

### `.env.example`
Template documenting all supported environment variables with safe defaults. See the Environment Variables section below for full details.

### `.gitignore`
Ignores: compiled binaries (`*.exe`, `*.so`, `*.dylib`), test artefacts (`*.out`, `coverage.*`), `.env`, `tags`, and the `docs/` submodule content.

### `.gitmodules`
Declares `docs/` as a git submodule pointing to the shared `space-wars-3002-docs` repository. All documentation for the project lives there.

### `README.md`
Brief one-paragraph description of the service purpose. The definitive reference is this document and the design docs in `docs/`.

### `LICENSE.md`
Project licence.

### `vendor-dialogue-generator`
The compiled binary produced by `make build`. Gitignored. Not committed to the repository.

---

## `cmd/`

Contains the service entry point. Go convention: one subdirectory per binary.

### `cmd/vendor-dialogue-generator/main.go`
The program entry point. Responsibilities:
1. Parses the `--rows` CLI flag (overrides `ROWS_TO_GENERATE` env var)
2. Loads config via `config.Load()`
3. Opens the database connection
4. Creates the PHP HTTP client if `USE_HTTP_API=true`, otherwise passes `nil`
5. Creates the LLM client
6. Creates and starts the worker pool
7. Calls `loadPendingVendors()` to fetch the work list
8. Submits each vendor to the pool and waits for completion

Contains two private functions:
- `loadPendingVendors()` — dispatches to `loadFromPHP` or `loadFromDB` based on `cfg.UseHTTPAPI`
- `loadFromPHP()` — calls `phpClient.FetchPending()`, maps `php.VendorRecord` → `vendors.VendorProfile`, applies batch limit
- `loadFromDB()` — calls `vendors.FetchPending()` directly against the database

---

## `internal/`

All application logic. Packages are deliberately small and single-purpose. Nothing in `internal/` is exported outside the module.

---

### `internal/constants/enums.go`

Package `constants`. Defines all canonical string constants that must match the PHP backend exactly. Grouped into:

| Constant group | Values |
|----------------|--------|
| Line types | `greeting`, `inventory_pitch`, `deal_accepted`, `deal_rejected`, `farewell` |
| Interaction buckets | `first_visit`, `second_visit`, `third_visit`, `repeat_customer` |
| Transaction contexts | `neutral`, `vendor_selling`, `vendor_buying` |
| Inventory contexts | `none`, `ship`, `shield_projector`, `engine`, `reactor`, `weapon`, `sensor_array`, `cargo_module`, `hull_plating`, `salvage_component` |
| Generation statuses | `pending`, `generating`, `complete`, `failed` |

**Rule:** Never use raw strings for these values anywhere in the service. Always reference these constants. If PHP changes a value, it changes here first.

---

### `internal/config/config.go`

Package `config`. Exports one type (`Config`) and one function (`Load()`).

**`Config` struct fields:**

| Field | Env var | Default | Purpose |
|-------|---------|---------|---------|
| `DBHost` | `DB_HOST` | required | Database host |
| `DBPort` | `DB_PORT` | `3306` | Database port |
| `DBUser` | `DB_USER` | required | Database user |
| `DBPassword` | `DB_PASSWORD` | `""` | Database password |
| `DBName` | `DB_NAME` | required | Database name |
| `LLMBaseURL` | `LLM_BASE_URL` | required | llama.cpp server base URL |
| `LLMModel` | `LLM_MODEL` | required | Model name to pass in requests |
| `LLMTemperature` | `LLM_TEMPERATURE` | `0.4` | Sampling temperature (keep low for small models) |
| `LLMTopP` | `LLM_TOP_P` | `0.7` | Nucleus sampling threshold |
| `LLMMaxTokens` | `LLM_MAX_TOKENS` | `500` | Max tokens per LLM response |
| `LLMTimeoutSeconds` | `LLM_TIMEOUT_SECONDS` | `60` | HTTP timeout for LLM calls |
| `WorkerCount` | `WORKER_COUNT` | `1` | Number of concurrent worker goroutines |
| `BatchSize` | `BATCH_SIZE` | `5` | Number of vendors fetched per run |
| `GenerationRetryMax` | `GENERATION_RETRY_MAX` | `2` | Max LLM retries per bucket |
| `RowsToGenerate` | `ROWS_TO_GENERATE` | `0` (unlimited) | Hard limit on vendors processed |
| `GenerationVersion` | `GENERATION_VERSION` | `1` | Written to `generation_version` on inserted rows |
| `UseHTTPAPI` | `USE_HTTP_API` | `false` | When true, use PHP HTTP API instead of direct DB |
| `PHPBaseURL` | `PHP_BASE_URL` | `""` | PHP app base URL (required if UseHTTPAPI) |
| `PHPInternalToken` | `PHP_INTERNAL_TOKEN` | `""` | Bearer token for PHP internal API (required if UseHTTPAPI) |
| `DryRun` | `DRY_RUN` | `false` | Skip LLM calls and DB writes when true |
| `LogLevelString` | `LOG_LEVEL` | `"info"` | Log verbosity: debug, info, warn, error |

**`Load()`** loads `.env` via godotenv (best-effort, no error if missing), then reads each env var, collecting all errors before returning so the operator sees every missing field at once.

**`LogLevel()`** converts `LogLevelString` to a `slog.Level`.

---

### `internal/db/db.go`

Package `db`. Wraps `*sql.DB` in a thin struct. Exports `New(cfg)` which builds the DSN, calls `sql.Open`, pings the database to verify connectivity, and returns the wrapper. All other packages use `*db.DB` directly and call standard `Query`/`Exec`/`Close` methods.

---

### `internal/logging/logger.go`

Package `logging`. Wraps `log/slog` with helper methods that accept a `map[string]interface{}` for structured fields, matching the style used throughout the service.

Methods: `Debugf`, `Infof`, `Warnf`, `Errorf` — each converts the fields map to `slog.Attr` values and calls `LogAttrs`.

`WithFields(fields)` returns a child logger with fields pre-attached (not currently used but available).

---

### `internal/llm/types.go`

Package `llm`. Plain Go structs for the llama.cpp OpenAI-compatible API:

- `ChatMessage` — `{role, content}`
- `ChatRequest` — outbound request body including `model`, `messages`, `temperature`, `top_p`, `max_tokens`, `stream` (always `false`)
- `ChatResponse` / `Choice` — inbound response; only the first choice's message content is used
- `DialogueOutput` — the expected JSON shape of the LLM response: `{"lines": [...]}`

---

### `internal/llm/client.go`

Package `llm`. Exports `Client` and `New(cfg)`.

**`Generate(system, user string, attempt int) ([]string, error)`**

- `attempt=0` uses the configured temperature; each subsequent attempt reduces it by 0.15 (floor 0.1) and appends a stronger JSON-only instruction to the user message
- Sends a POST to `{LLMBaseURL}/v1/chat/completions`
- Parses the response content as `{"lines": [...]}`
- Returns an error for non-200 status, malformed JSON, or an empty lines array

The `http.Client` is created with a timeout from `cfg.LLMTimeoutSeconds`.

---

### `internal/php/client.go`

Package `php`. Used only when `cfg.UseHTTPAPI = true`. Implements the three-endpoint contract with the PHP internal API.

**Types:**
- `VendorRecord` — shape of each vendor in the pending response (id, uuid, service_type, criminality, dialogue_generation_version, profile)
- `VendorProfileData` — nested profile block (archetype, personality map, markup_base)
- `PendingResponse` — top-level response from the pending endpoint
- `SubmitLinesRequest` — body for the lines submission endpoint

**Methods on `Client`:**

| Method | HTTP call | Purpose |
|--------|-----------|---------|
| `FetchPending()` | `GET /api/internal/vendor-dialogue/pending` | Returns all vendors with status `pending` or `failed` |
| `UpdateStatus(uuid, status, generatedAt)` | `PATCH /api/internal/vendor-dialogue/{uuid}/status` | Transitions a vendor's generation status |
| `SubmitLines(uuid, req)` | `POST /api/internal/vendor-dialogue/{uuid}/lines` | Sends a validated batch of lines for one dialogue scope |

All requests include `Authorization: Bearer {token}` and a 30-second HTTP timeout. `doRequest()` is a shared helper that sets headers and executes the request.

---

### `internal/vendors/vendor.go`

Package `vendors`. Defines the `VendorProfile` struct and the `Personality` type.

**`VendorProfile` fields:**

| Field | Source column | Notes |
|-------|--------------|-------|
| `ID` | `galaxy_vendor_profiles.id` | Primary key |
| `UUID` | `galaxy_vendor_profiles.uuid` | Used for PHP HTTP API calls |
| `ServiceType` | `galaxy_vendor_profiles.service_type` | `salvage_yard`, `shipyard`, `trading_hub`, `market` |
| `Criminality` | `galaxy_vendor_profiles.criminality` | 0.0–1.0 |
| `MarkupBase` | `vendor_profiles.markup_base` | From joined vendor_profiles row |
| `PersonalityJSON` | `vendor_profiles.personality` | Raw JSON string, parsed into `Personality` |
| `Personality` | — | Parsed map of trait name → float64 |
| `DialogueGenerationStatus` | `galaxy_vendor_profiles.dialogue_generation_status` | `pending`, `generating`, `complete`, `failed` |
| `DialogueGenerationVersion` | `galaxy_vendor_profiles.dialogue_generation_version` | Written to `generation_version` on inserted dialogue rows |
| `CreatedAt` / `UpdatedAt` | timestamps | Standard timestamps |

**`Personality`** is `map[string]float64`. `Get(key)` returns the trait value or `0.5` as default if the key is absent.

**`ParsePersonality()`** unmarshals `PersonalityJSON` into `Personality`. Called after scanning a DB row.

---

### `internal/vendors/repository.go`

Package `vendors`. All database interactions for vendor loading and status tracking.

**`FetchPending(database, limit, logger)`**
Queries `galaxy_vendor_profiles` joined to `vendor_profiles` with `WHERE dialogue_generation_status IN ('pending', 'failed')`. Scans results into `VendorProfile` structs, calling `ParsePersonality()` on each and skipping any row with malformed JSON (with a warning log). Returns up to `limit` vendors ordered by id.

**`MarkGenerating(database, vendorID)`**
`UPDATE galaxy_vendor_profiles SET dialogue_generation_status = 'generating' WHERE id = ?`

**`MarkComplete(database, vendorID)`**
`UPDATE ... SET dialogue_generation_status = 'complete', dialogue_generated_at = ? WHERE id = ?`

**`MarkFailed(database, vendorID)`**
`UPDATE ... SET dialogue_generation_status = 'failed' WHERE id = ?`

---

### `internal/prompts/builder.go`

Package `prompts`. Constructs the system and user messages sent to the LLM for a specific dialogue scope.

**`BuildPrompt(vendor, lineType, bucket, transactionContext, inventoryContext, count)`**
Returns a `Prompt` struct with `System`, `User`, and `Hash` (SHA-256 of the combined prompt, used for logging).

**`buildSystemPrompt()`**
Returns the invariant system message instructing the model to output JSON only, one sentence per line, no markdown.

**`buildUserPrompt(...)`**
Assembles the vendor-specific context block by calling all tone derivation helpers, then formats the full user message including context block, vendor voice block, and rules.

**Tone derivation functions** — each maps a numeric value to a human-readable instruction string:

| Function | Input | Maps |
|----------|-------|------|
| `deriveServiceTypeTone` | `service_type` string | `salvage_yard`→rough/practical, `shipyard`→polished/prideful, `trading_hub`→general merchant, `market`→open commerce |
| `deriveCriminalityTone` | 0.0–1.0 | low→clean/straightforward, mid→opportunistic, high→predatory edge |
| `deriveMarkupTone` | 0.0–1.0+ | high→premium/defensive, low→bargain/casual |
| `deriveHonestyHint` | 0.0–1.0 | high→blunt/transparent, low→evasive/exaggerates |
| `deriveGreedHint` | 0.0–1.0 | high→hard sell/scarcity, low→relaxed about price |
| `deriveCharmHint` | 0.0–1.0 | high→smooth/friendly, low→curt/blunt |
| `deriveRiskHint` | 0.0–1.0 | high→happy selling questionable goods, low→conservative |

**`lineTypeWordLimits(lineType)`** returns (min, max) word counts per line type: greetings 6–16, inventory pitches 8–18, deal outcomes and farewells 6–14.

---

### `internal/validation/validator.go`

Package `validation`. Filters raw LLM output into accepted and rejected lines.

**`Validate(lines)`**
Runs each line through `normalizeLine` then `validateLine`. Tracks seen hashes to deduplicate. Returns `(accepted, rejected, error)`. Returns an error if fewer than 60% of input lines pass — this signals the caller to retry.

**`normalizeLine(line)`**
1. Trims surrounding whitespace
2. Collapses internal whitespace runs
3. Strips leading line-number prefixes (`1.`, `2)`, `Line 3:`)
4. Removes duplicate terminal punctuation (`..`, `!!`, `??`)

**`validateLine(line)`** rejects a line if:
- Empty after normalisation
- Over 255 characters
- Fewer than 6 or more than 20 words
- Contains any Unicode control character
- Contains `!!!` or `???`
- Matches any meta-commentary pattern: `here are`, `here's`, `certainly`, `of course`, `as requested`, `i'll generate`, `sure, here`, `[system]`, `<thinking>`, `{thinking}`, ` ``` `, `json`
- Still begins with a line-number prefix after normalisation

---

### `internal/dialogue/orchestrator.go`

Package `dialogue`. The core generation loop. Drives a single vendor through the full 22-scope generation matrix.

**`DialogueBucket`** struct — one entry in the generation matrix:
- `LineType` — e.g. `greeting`
- `Bucket` — e.g. `first_visit`
- `TransactionContext` — e.g. `vendor_selling`
- `InventoryContext` — e.g. `engine`
- `RequestedCount` — how many lines to ask the LLM for

**`Orchestrator`** struct — holds references to `*db.DB`, `*llm.Client`, `*php.Client` (nil in direct-DB mode), `*logging.Logger`, and `*config.Config`.

**`New(...)`** constructor.

**`GenerateForVendor(vendor)`**
1. Calls `markGenerating` (best-effort, logs warning on failure but continues)
2. Iterates all 22 buckets via `generateBucket`
3. Continues past per-bucket failures — partial generation is better than none
4. Calls `markFailed` if any buckets failed, `markComplete` otherwise

**`generateBucket(vendor, bucket)`**
Retry loop (up to `GenerationRetryMax`):
1. `callLLM` — builds prompt, calls LLM, returns lines + prompt hash
2. `validation.Validate` — filters lines
3. On success: calls `storeLines`, logs result, returns
4. On failure: reduces `requestedCount` by 20%, continues to next attempt

**`callLLM(vendor, bucket, count, attempt)`**
Builds the prompt via `prompts.BuildPrompt` and calls `llm.Generate`. Skips the LLM call in dry-run mode.

**`storeLines(vendor, bucket, lines)`**
Routes to `storeLinesViaHTTP` or `storeLinesDirectDB` based on `cfg.UseHTTPAPI`.

**`storeLinesViaHTTP`**
Calls `phpClient.SubmitLines` with a `php.SubmitLinesRequest`. PHP handles the delete-and-replace storage strategy.

**`storeLinesDirectDB`**
Executes a `DELETE` for the exact (galaxy_vendor_profile_id, line_type, interaction_bucket, transaction_context, inventory_context) scope, then inserts each accepted line with `weight=1.0000` and the current `GenerationVersion`.

**`markGenerating` / `markComplete` / `markFailed`**
Route to `phpClient.UpdateStatus` or the direct `vendors.Mark*` functions depending on mode.

**`getDialogueBuckets()`** returns the full v1 generation matrix — 22 scopes:

| Count | Scope group |
|-------|-------------|
| 4 | Greetings (first/second/third/repeat_customer × neutral × none) |
| 9 | Inventory pitches, vendor_selling (repeat_customer × all 9 item categories) |
| 3 | Inventory pitches, vendor_buying (repeat_customer × ship, engine, salvage_component) |
| 2 | deal_accepted (vendor_selling + vendor_buying) |
| 2 | deal_rejected (vendor_selling + vendor_buying) |
| 1 | farewell (repeat_customer × neutral × none) |
| **22** | **Total** |

---

### `internal/jobs/worker.go`

Package `jobs`. A fixed-size goroutine pool.

**`Pool`** struct — holds worker count, a buffered `chan vendors.VendorProfile`, a `sync.WaitGroup`, and the `*dialogue.Orchestrator`.

**`NewPool(database, llmClient, phpClient, logger, cfg)`**
Creates the pool and the orchestrator. `phpClient` may be `nil`.

**`Start()`** — launches `WorkerCount` goroutines, each reading from `vendorChan`.

**`Submit(vendor)`** — sends a vendor into `vendorChan` (blocks if the channel is full).

**`Close()`** — closes `vendorChan` and waits for all workers to drain and exit.

**`worker(id)`** — reads from the channel and calls `processVendor` for each vendor.

**`processVendor(workerID, vendor)`** — calls `orchestrator.GenerateForVendor`, logs success or failure.

---

## `docs/` (git submodule)

The `docs/` directory is a git submodule pointing to the shared `space-wars-3002-docs` repository. It contains all documentation for the broader project (PHP app, Go service, frontend). It has its own git history and is updated independently.

All new documentation for this service must be placed under `docs/` in an appropriate subdirectory. Never create documentation files in the root of this repository.

### Subdirectory guide

| Directory | Purpose |
|-----------|---------|
| `docs/design/` | Architecture and design documents — vendor dialogue system, phased architecture, Go/PHP joint design, service-specific technical designs |
| `docs/implementation_plans/` | Step-by-step implementation plans with file-level detail and pseudocode |
| `docs/api/` | API reference documentation for all game systems |
| `docs/guides/` | How-to guides, phase implementation summaries, testing manuals |
| `docs/reference/` | Quick reference material — database tables, ship stats, alignment analyses, codebase structure (this file) |
| `docs/benchmarks/` | Performance and complexity baseline measurements |
| `docs/ADR/` | Architecture Decision Records |
| `docs/TODO/` | Tracked refactoring work, audits, unimplemented feature lists |
| `docs/types/` | TypeScript type definitions shared with the frontend |

---

## Data Flow Summary

```
Startup
  └── config.Load()
  └── db.New()
  └── php.New()          (if USE_HTTP_API=true)
  └── llm.New()
  └── jobs.NewPool()
  └── loadPendingVendors()
        ├── php.FetchPending()       (USE_HTTP_API=true)
        └── vendors.FetchPending()   (USE_HTTP_API=false)

Per vendor (parallel workers)
  └── orchestrator.GenerateForVendor(vendor)
        └── markGenerating()
        └── for each of 22 DialogueBuckets:
              └── prompts.BuildPrompt()
              └── llm.Generate()           → llama.cpp
              └── validation.Validate()
              └── storeLines()
                    ├── php.SubmitLines()       (USE_HTTP_API=true)
                    └── db DELETE + INSERT      (USE_HTTP_API=false)
        └── markComplete() or markFailed()
```
