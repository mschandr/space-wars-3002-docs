# Vendor Dialogue System — Phase 2: Go Dialogue Generator Implementation Plan
## Created: 2026-03-13
## Source: docs/design/vendor_dialogue_phased_architecture.md + vendor_dialogue_go_technical_design.md + vendor_dialogue_joint_design_go_php.md

---

## Overview

The Go service lives at `/home/mdhas/workspace/space-wars-3002-text-generation/`. A partial implementation exists. This plan documents every file that needs to change, what specifically needs to change, and provides pseudocode stubs for each method.

---

## Architecture Decision: Direct DB vs. HTTP API

Two modes are possible:

| Mode | Pros | Cons |
|------|------|------|
| Direct DB write | Simpler, no HTTP overhead | Bypasses PHP validation; no side-effect hooks |
| HTTP API (recommended) | PHP validates, cache/job hooks possible | Requires PHP internal API (see Phase 2 PHP plan) |

**This plan covers the HTTP API approach** (Go polls PHP for pending vendors, submits lines via HTTP). The direct-DB fallback is noted where relevant.

Config env var `USE_HTTP_API=true/false` should gate the mode.

---

## Current State Summary

### What exists and works
- `internal/config/config.go` — Config loader (DB, LLM, worker settings)
- `internal/db/db.go` — MariaDB connection wrapper
- `internal/logging/logger.go` — Structured slog logger
- `internal/llm/client.go` + `llm/types.go` — llama.cpp HTTP client
- `internal/validation/validator.go` — Line validation (word count, chars, control chars, dedup)
- `internal/vendors/vendor.go` — `VendorProfile` struct + `Personality.Get()`
- `internal/vendors/repository.go` — `FetchPending()` (but has critical gaps — see below)
- `internal/prompts/builder.go` — Prompt builder (but has critical gaps — see below)
- `internal/dialogue/orchestrator.go` — Generation loop (but has critical gaps — see below)
- `internal/jobs/worker.go` — Worker pool
- `cmd/vendor-dialogue-generator/main.go` — Entry point

### Critical gaps
1. `FetchPending()` — no WHERE clause on status; missing UUID/version fields
2. No status tracking (`MarkGenerating`, `MarkComplete`, `MarkFailed`)
3. `DialogueBucket` missing `TransactionContext`; generation matrix incomplete
4. DELETE/INSERT in orchestrator missing `transaction_context`, `inventory_context`, `weight`, `generation_version`
5. Prompt builder uses wrong service types; missing transaction context, criminality/markup tone
6. LLM defaults too high (temp 0.7 → should be 0.4; top_p 0.9 → should be 0.7)
7. `ChatRequest` missing `Stream: false`
8. Retry logic doesn't lower temperature or strengthen JSON instruction on retry
9. Validator missing full meta-commentary pattern list from design
10. No PHP HTTP API client

---

## Files to Create

### NEW: `internal/php/client.go`

HTTP client for the PHP internal API. Used when `USE_HTTP_API=true`.

```go
package php

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

type Client struct {
    baseURL string
    token   string
    http    *http.Client
}

// VendorRecord is the shape returned by PHP's /api/internal/vendor-dialogue/pending
type VendorRecord struct {
    ID                        int64       `json:"id"`
    UUID                      string      `json:"uuid"`
    ServiceType               string      `json:"service_type"`
    Criminality               float64     `json:"criminality"`
    DialogueGenerationVersion int         `json:"dialogue_generation_version"`
    Profile                   VendorProfileData `json:"profile"`
}

type VendorProfileData struct {
    Archetype   string             `json:"archetype"`
    Personality map[string]float64 `json:"personality"`
    MarkupBase  float64            `json:"markup_base"`
}

type PendingResponse struct {
    Count   int            `json:"count"`
    Vendors []VendorRecord `json:"vendors"`
}

type SubmitLinesRequest struct {
    LineType            string   `json:"line_type"`
    InteractionBucket   string   `json:"interaction_bucket"`
    TransactionContext  string   `json:"transaction_context"`
    InventoryContext    string   `json:"inventory_context"`
    GenerationVersion   int      `json:"generation_version"`
    Lines               []string `json:"lines"`
}

func New(baseURL, token string) *Client {
    return &Client{
        baseURL: baseURL,
        token:   token,
        http:    &http.Client{Timeout: 30 * time.Second},
    }
}

// FetchPending polls PHP for vendors needing dialogue generation.
// GET /api/internal/vendor-dialogue/pending
func (c *Client) FetchPending() ([]VendorRecord, error) {
    // build GET request to c.baseURL + "/api/internal/vendor-dialogue/pending"
    // set Authorization: Bearer c.token header
    // execute request
    // check status 200 or return error
    // decode JSON into PendingResponse
    // return PendingResponse.Vendors, nil
}

// UpdateStatus tells PHP the generation status for a vendor.
// PATCH /api/internal/vendor-dialogue/{uuid}/status
// status must be one of: "generating", "complete", "failed"
// generatedAt is only sent when status == "complete" (ISO8601 string or empty)
func (c *Client) UpdateStatus(vendorUUID, status, generatedAt string) error {
    // build PATCH request to c.baseURL + "/api/internal/vendor-dialogue/" + vendorUUID + "/status"
    // body: {"status": status, "generated_at": generatedAt}  (omit generated_at if empty)
    // set Authorization: Bearer c.token header
    // set Content-Type: application/json header
    // execute request
    // check status 200 or return error
    // return nil
}

// SubmitLines sends generated lines for one (lineType, bucket, transactionContext, inventoryContext) scope.
// POST /api/internal/vendor-dialogue/{uuid}/lines
func (c *Client) SubmitLines(vendorUUID string, req SubmitLinesRequest) error {
    // marshal req to JSON
    // build POST request to c.baseURL + "/api/internal/vendor-dialogue/" + vendorUUID + "/lines"
    // set Authorization: Bearer c.token header
    // set Content-Type: application/json header
    // execute request
    // check status 200/201 or return descriptive error (include response body on failure)
    // return nil
}

// doRequest is a shared helper for all HTTP calls.
func (c *Client) doRequest(method, url string, body io.Reader) (*http.Response, error) {
    // create http.NewRequest(method, url, body)
    // set Authorization header from c.token
    // set Content-Type if body != nil
    // execute c.http.Do(req)
    // return response, err
}
```

---

## Files to Modify

### 1. `internal/config/config.go`

**Changes required:**

Add fields to `Config` struct:
```go
// Existing fields stay unchanged.
// Add:
GenerationVersion int     // GENERATION_VERSION env var (default: 1)
LLMTimeoutSeconds int     // LLM_TIMEOUT_SECONDS env var (default: 60)
UseHTTPAPI        bool    // USE_HTTP_API env var (default: false)
PHPBaseURL        string  // PHP_BASE_URL env var (required if UseHTTPAPI)
PHPInternalToken  string  // PHP_INTERNAL_TOKEN env var (required if UseHTTPAPI)
```

Update `Load()`:
```go
func Load() (*Config, error) {
    // ... existing parsing ...

    // Add GenerationVersion
    cfg.GenerationVersion = 1
    if v := os.Getenv("GENERATION_VERSION"); v != "" {
        if n, err := strconv.Atoi(v); err == nil {
            cfg.GenerationVersion = n
        } else {
            errs = append(errs, "GENERATION_VERSION must be numeric")
        }
    }

    // Add LLMTimeoutSeconds
    cfg.LLMTimeoutSeconds = 60
    if v := os.Getenv("LLM_TIMEOUT_SECONDS"); v != "" {
        if n, err := strconv.Atoi(v); err == nil {
            cfg.LLMTimeoutSeconds = n
        } else {
            errs = append(errs, "LLM_TIMEOUT_SECONDS must be numeric")
        }
    }

    // Fix: lower default temperature from 0.7 to 0.4
    cfg.LLMTemperature = 0.4

    // Fix: lower default top_p from 0.9 to 0.7
    cfg.LLMTopP = 0.7

    // Add UseHTTPAPI
    cfg.UseHTTPAPI = strings.ToLower(os.Getenv("USE_HTTP_API")) == "true"

    // Add PHPBaseURL (required if UseHTTPAPI)
    cfg.PHPBaseURL = os.Getenv("PHP_BASE_URL")
    if cfg.UseHTTPAPI && cfg.PHPBaseURL == "" {
        errs = append(errs, "PHP_BASE_URL is required when USE_HTTP_API=true")
    }

    // Add PHPInternalToken (required if UseHTTPAPI)
    cfg.PHPInternalToken = os.Getenv("PHP_INTERNAL_TOKEN")
    if cfg.UseHTTPAPI && cfg.PHPInternalToken == "" {
        errs = append(errs, "PHP_INTERNAL_TOKEN is required when USE_HTTP_API=true")
    }
}
```

---

### 2. `internal/llm/types.go`

**Changes required:**

Add `Stream` field to `ChatRequest`:
```go
type ChatRequest struct {
    Model       string        `json:"model"`
    Messages    []ChatMessage `json:"messages"`
    Temperature float64       `json:"temperature"`
    TopP        float64       `json:"top_p"`
    MaxTokens   int           `json:"max_tokens"`
    Stream      bool          `json:"stream"` // ADD: always false
}
```

---

### 3. `internal/llm/client.go`

**Changes required:**

- Add timeout to `http.Client` using `cfg.LLMTimeoutSeconds`
- Add `attempt` parameter to `Generate()` to support temperature lowering on retry
- Set `Stream: false` in `ChatRequest`

```go
func New(cfg *config.Config) *Client {
    return &Client{
        baseURL: cfg.LLMBaseURL,
        model:   cfg.LLMModel,
        client:  &http.Client{
            Timeout: time.Duration(cfg.LLMTimeoutSeconds) * time.Second, // ADD timeout
        },
        cfg: cfg,
    }
}

// Generate calls the LLM. attempt=0 is first try; attempt>0 uses stricter settings.
func (c *Client) Generate(system, user string, attempt int) ([]string, error) {
    // Lower temperature on retry
    temperature := c.cfg.LLMTemperature
    if attempt > 0 {
        temperature = max(0.1, temperature-0.15) // reduce by 0.15 each retry, floor at 0.1
    }

    // Strengthen JSON instruction on retry
    if attempt > 0 {
        user = user + "\n\nIMPORTANT: Output ONLY valid JSON. No explanations. No preamble. No markdown."
    }

    req := ChatRequest{
        Model:       c.model,
        Messages:    []ChatMessage{
            {Role: "system", Content: system},
            {Role: "user", Content: user},
        },
        Temperature: temperature,
        TopP:        c.cfg.LLMTopP,
        MaxTokens:   c.cfg.LLMMaxTokens,
        Stream:      false, // ADD: explicit false
    }

    // ... rest of existing logic unchanged ...
}
```

---

### 4. `internal/vendors/vendor.go`

**Changes required:**

Add `UUID`, `DialogueGenerationStatus`, `DialogueGenerationVersion` fields:

```go
package vendors

import (
    "encoding/json"
    "time"
)

type Personality map[string]float64

type VendorProfile struct {
    ID                        int64
    UUID                      string     // ADD
    ServiceType               string
    Criminality               float64
    MarkupBase                float64
    PersonalityJSON           string
    Personality               Personality
    DialogueGenerationStatus  string     // ADD: pending|generating|complete|failed
    DialogueGenerationVersion int        // ADD
    CreatedAt                 time.Time
    UpdatedAt                 time.Time
}

// ParsePersonality unchanged.

// Get unchanged.
```

---

### 5. `internal/vendors/repository.go`

**Changes required:**

- Fix `FetchPending()` to filter on `dialogue_generation_status` and select UUID/version
- Add `MarkGenerating()`, `MarkComplete()`, `MarkFailed()` functions

Note: These functions target the `galaxy_vendor_profiles` table (where the PHP side stores generation status). The table maps to `galaxy_vendor_profiles` in the actual schema.

```go
package vendors

import (
    "fmt"
    "time"

    "space-wars-3002-text-generation/internal/db"
    "space-wars-3002-text-generation/internal/logging"
)

// FetchPending loads vendors with dialogue_generation_status IN ('pending', 'failed').
// Queries galaxy_vendor_profiles joined to vendor_profiles for personality/markup data.
func FetchPending(database *db.DB, limit int, logger *logging.Logger) ([]VendorProfile, error) {
    rows, err := database.Query(`
        SELECT
            gvp.id,
            gvp.uuid,
            gvp.service_type,
            gvp.criminality,
            vp.markup_base,
            vp.personality,
            gvp.dialogue_generation_status,
            gvp.dialogue_generation_version,
            gvp.created_at,
            gvp.updated_at
        FROM galaxy_vendor_profiles gvp
        JOIN vendor_profiles vp ON vp.id = gvp.vendor_profile_id
        WHERE gvp.dialogue_generation_status IN ('pending', 'failed')
        ORDER BY gvp.id
        LIMIT ?
    `, limit)

    // ... scan rows into VendorProfile structs as before ...
    // Scan: gvp.id, gvp.uuid, gvp.service_type, gvp.criminality,
    //        vp.markup_base, vp.personality, gvp.dialogue_generation_status,
    //        gvp.dialogue_generation_version, gvp.created_at, gvp.updated_at
    // Call v.ParsePersonality(), skip on error with warning log
    // Return []VendorProfile, nil
}

// MarkGenerating sets dialogue_generation_status = 'generating' for a vendor.
func MarkGenerating(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE galaxy_vendor_profiles
        SET dialogue_generation_status = 'generating'
        WHERE id = ?
    `, vendorID)
    if err != nil {
        return fmt.Errorf("failed to mark vendor %d as generating: %w", vendorID, err)
    }
    return nil
}

// MarkComplete sets status = 'complete' and dialogue_generated_at = NOW().
func MarkComplete(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE galaxy_vendor_profiles
        SET dialogue_generation_status = 'complete',
            dialogue_generated_at = ?
        WHERE id = ?
    `, time.Now().UTC().Format("2006-01-02 15:04:05"), vendorID)
    if err != nil {
        return fmt.Errorf("failed to mark vendor %d as complete: %w", vendorID, err)
    }
    return nil
}

// MarkFailed sets dialogue_generation_status = 'failed'.
func MarkFailed(database *db.DB, vendorID int64) error {
    _, err := database.Exec(`
        UPDATE galaxy_vendor_profiles
        SET dialogue_generation_status = 'failed'
        WHERE id = ?
    `, vendorID)
    if err != nil {
        return fmt.Errorf("failed to mark vendor %d as failed: %w", vendorID, err)
    }
    return nil
}
```

---

### 6. `internal/prompts/builder.go`

**Complete rewrite.** Current implementation has wrong service types and missing transaction context, criminality tone, and markup tone.

```go
package prompts

import (
    "crypto/sha256"
    "fmt"
    "strings"

    "space-wars-3002-text-generation/internal/vendors"
)

type Prompt struct {
    System string
    User   string
    Hash   string
}

// BuildPrompt constructs the LLM prompt for a specific dialogue scope.
// lineType: greeting|inventory_pitch|deal_accepted|deal_rejected|farewell
// bucket: first_visit|second_visit|third_visit|repeat_customer
// transactionContext: neutral|vendor_selling|vendor_buying
// inventoryContext: none|ship|engine|etc.
// count: number of lines to request
func BuildPrompt(vendor vendors.VendorProfile, lineType, bucket, transactionContext, inventoryContext string, count int) *Prompt {
    system := buildSystemPrompt()
    user := buildUserPrompt(vendor, lineType, bucket, transactionContext, inventoryContext, count)

    normalized := strings.TrimSpace(system + "\n" + user)
    hash := fmt.Sprintf("%x", sha256.Sum256([]byte(normalized)))

    return &Prompt{
        System: system,
        User:   user,
        Hash:   hash,
    }
}

// buildSystemPrompt returns the invariant system message per design §9.1.
func buildSystemPrompt() string {
    return strings.TrimSpace(`
You generate short in-universe dialogue for NPC vendors in a space exploration and trading game.
Output JSON only.
Each line must be one sentence.
Do not include explanations.
Do not include markdown.
`)
}

// buildUserPrompt assembles the vendor-specific user message per design §9.2.
func buildUserPrompt(vendor vendors.VendorProfile, lineType, bucket, transactionContext, inventoryContext string, count int) string {
    var b strings.Builder

    serviceTypeTone := deriveServiceTypeTone(vendor.ServiceType)
    criminalityTone := deriveCriminalityTone(vendor.Criminality)
    markupTone := deriveMarkupTone(vendor.MarkupBase)
    honestyHint := deriveHonestyHint(vendor.Personality.Get("honesty"))
    greedHint := deriveGreedHint(vendor.Personality.Get("greed"))
    charmHint := deriveCharmHint(vendor.Personality.Get("charm"))
    riskHint := deriveRiskHint(vendor.Personality.Get("risk_tolerance"))

    minWords, maxWords := lineTypeWordLimits(lineType)

    fmt.Fprintf(&b, "Generate %d %s lines for a %s vendor.\n\n", count, lineType, vendor.ServiceType)
    fmt.Fprintf(&b, "Context:\n")
    fmt.Fprintf(&b, "- service_type: %s (%s)\n", vendor.ServiceType, serviceTypeTone)
    fmt.Fprintf(&b, "- interaction_bucket: %s\n", bucket)
    fmt.Fprintf(&b, "- transaction_context: %s\n", transactionContext)
    fmt.Fprintf(&b, "- inventory_context: %s\n", inventoryContext)
    fmt.Fprintf(&b, "- criminality_tone: %s\n", criminalityTone)
    fmt.Fprintf(&b, "- markup_tone: %s\n\n", markupTone)
    fmt.Fprintf(&b, "Vendor voice:\n")
    fmt.Fprintf(&b, "- honesty_hint: %s\n", honestyHint)
    fmt.Fprintf(&b, "- greed_hint: %s\n", greedHint)
    fmt.Fprintf(&b, "- charm_hint: %s\n", charmHint)
    fmt.Fprintf(&b, "- risk_hint: %s\n\n", riskHint)
    fmt.Fprintf(&b, "Rules:\n")
    fmt.Fprintf(&b, "- one sentence per line\n")
    fmt.Fprintf(&b, "- %d to %d words per line\n", minWords, maxWords)
    fmt.Fprintf(&b, "- keep lines in character\n")
    fmt.Fprintf(&b, "- do not mention exact live item condition percentages\n")
    fmt.Fprintf(&b, "- do not invent exact live defects\n")
    fmt.Fprintf(&b, "- return JSON: {\"lines\":[\"...\"]}\n")

    return b.String()
}

// deriveServiceTypeTone maps service_type to a human-readable tone hint per design §8.1.
func deriveServiceTypeTone(serviceType string) string {
    switch serviceType {
    case "salvage_yard":
        return "used goods dealer, practical tone, rough edges allowed, may reference salvage and wear"
    case "shipyard":
        return "polished and professional, prideful about product quality, stronger sales posture"
    case "trading_hub":
        return "general merchant, broad product framing, commercial tone"
    case "market":
        return "open commerce marketplace, general trade language, less technical specificity"
    default:
        return "general merchant in a space trading hub"
    }
}

// deriveCriminalityTone maps 0.0–1.0 criminality to a tone hint per design §8.2.
func deriveCriminalityTone(criminality float64) string {
    switch {
    case criminality >= 0.7:
        return "predatory edge, rougher language, dubious but within content rules"
    case criminality >= 0.4:
        return "sharper tone, more opportunistic, some roughness permitted"
    default:
        return "cleaner language, low menace, straightforward"
    }
}

// deriveMarkupTone maps markup_base to a price posture hint per design §8.3.
func deriveMarkupTone(markupBase float64) string {
    switch {
    case markupBase > 0.3:
        return "defensive about pricing, premium framing, worth-the-price language"
    case markupBase < 0.05:
        return "casual bargain framing, lower price defensiveness, less pompous"
    default:
        return "moderate pricing stance, matter-of-fact"
    }
}

// deriveHonestyHint maps honesty trait to a voice hint per design §8.4.
func deriveHonestyHint(honesty float64) string {
    switch {
    case honesty >= 0.7:
        return "blunt, transparent, straightforward, minimal exaggeration"
    case honesty <= 0.3:
        return "evasive, exaggerates benefits, downplays flaws"
    default:
        return "occasionally candid but still a salesperson"
    }
}

// deriveGreedHint maps greed trait to a price defensiveness hint per design §8.4.
func deriveGreedHint(greed float64) string {
    switch {
    case greed >= 0.7:
        return "hard sell, price defensive, scarcity framing, pushy"
    case greed <= 0.3:
        return "relaxed about price, not pushing hard for margin"
    default:
        return "wants to close the deal, mild price defensiveness"
    }
}

// deriveCharmHint maps charm trait to a social tone hint per design §8.4.
func deriveCharmHint(charm float64) string {
    switch {
    case charm >= 0.7:
        return "smooth, friendly, socially at ease, persuasive without pressure"
    case charm <= 0.3:
        return "blunt, socially curt, no pleasantries"
    default:
        return "professional but not especially warm"
    }
}

// deriveRiskHint maps risk_tolerance trait to a willingness to hype questionable goods per design §8.4.
func deriveRiskHint(riskTolerance float64) string {
    switch {
    case riskTolerance >= 0.7:
        return "happy to sell questionable goods, rough confidence, used-but-usable framing"
    case riskTolerance <= 0.3:
        return "only sells clearly legitimate goods, conservative framing"
    default:
        return "will sell grey-area goods but frames them diplomatically"
    }
}

// lineTypeWordLimits returns min and max word counts per line type.
func lineTypeWordLimits(lineType string) (min, max int) {
    switch lineType {
    case "greeting":
        return 6, 16
    case "inventory_pitch":
        return 8, 18
    case "deal_accepted", "deal_rejected":
        return 6, 14
    case "farewell":
        return 6, 14
    default:
        return 6, 20
    }
}
```

---

### 7. `internal/dialogue/orchestrator.go`

**Changes required:**

- Add `TransactionContext` to `DialogueBucket`
- Expand `getDialogueBuckets()` with full design matrix including `transaction_context`
- Fix DELETE query to include `transaction_context` and `inventory_context`
- Fix INSERT to include `transaction_context`, `inventory_context`, `weight`, `generation_version`
- Add status tracking calls before/after generation
- Pass `attempt` to `callLLM()` for retry temperature adjustment
- Add `phpClient` field for HTTP mode

```go
package dialogue

import (
    "fmt"

    "space-wars-3002-text-generation/internal/config"
    "space-wars-3002-text-generation/internal/db"
    "space-wars-3002-text-generation/internal/llm"
    "space-wars-3002-text-generation/internal/logging"
    "space-wars-3002-text-generation/internal/php"
    "space-wars-3002-text-generation/internal/prompts"
    "space-wars-3002-text-generation/internal/validation"
    "space-wars-3002-text-generation/internal/vendors"
)

// DialogueBucket represents one scope in the generation matrix.
type DialogueBucket struct {
    LineType           string
    Bucket             string
    TransactionContext string // ADD: neutral|vendor_selling|vendor_buying
    InventoryContext   string // ADD: none|ship|engine|etc.
    RequestedCount     int
}

type Orchestrator struct {
    db        *db.DB
    llm       *llm.Client
    logger    *logging.Logger
    cfg       *config.Config
    phpClient *php.Client // nil if UseHTTPAPI is false
}

func New(database *db.DB, llmClient *llm.Client, phpClient *php.Client, logger *logging.Logger, cfg *config.Config) *Orchestrator {
    return &Orchestrator{
        db:        database,
        llm:       llmClient,
        logger:    logger,
        cfg:       cfg,
        phpClient: phpClient,
    }
}

// GenerateForVendor runs the full generation matrix for one vendor.
// It marks the vendor as generating before starting and complete/failed after.
func (o *Orchestrator) GenerateForVendor(vendor vendors.VendorProfile) error {
    // Step 1: Mark generating
    if err := o.markGenerating(vendor); err != nil {
        return fmt.Errorf("failed to mark vendor %d as generating: %w", vendor.ID, err)
    }

    buckets := getDialogueBuckets()
    var lastErr error

    for _, bucket := range buckets {
        if err := o.generateBucket(vendor, bucket); err != nil {
            o.logger.Errorf("failed to generate bucket", map[string]interface{}{
                "vendor_id":           vendor.ID,
                "line_type":           bucket.LineType,
                "bucket":              bucket.Bucket,
                "transaction_context": bucket.TransactionContext,
                "inventory_context":   bucket.InventoryContext,
                "error":               err.Error(),
            })
            lastErr = err
            // Continue to next bucket; partial generation is better than none.
        }
    }

    // Step 2: Mark complete or failed
    if lastErr != nil {
        _ = o.markFailed(vendor)
        return lastErr
    }

    return o.markComplete(vendor)
}

// generateBucket generates lines for one (lineType, bucket, transactionContext, inventoryContext) scope.
func (o *Orchestrator) generateBucket(vendor vendors.VendorProfile, bucket DialogueBucket) error {
    o.logger.Infof("generating bucket", map[string]interface{}{
        "vendor_id":           vendor.ID,
        "vendor_uuid":         vendor.UUID,
        "line_type":           bucket.LineType,
        "bucket":              bucket.Bucket,
        "transaction_context": bucket.TransactionContext,
        "inventory_context":   bucket.InventoryContext,
        "requested_count":     bucket.RequestedCount,
    })

    var lastErr error
    requestedCount := bucket.RequestedCount

    for attempt := 0; attempt <= o.cfg.GenerationRetryMax; attempt++ {
        lines, promptHash, err := o.callLLM(vendor, bucket, requestedCount, attempt)
        if err != nil {
            o.logger.Infof("LLM call failed", map[string]interface{}{
                "vendor_id":   vendor.ID,
                "attempt":     attempt,
                "error":       err.Error(),
            })
            lastErr = err
            requestedCount = max(3, int(float64(requestedCount)*0.8))
            continue
        }

        accepted, rejected, err := validation.Validate(lines)
        if err != nil {
            o.logger.Infof("validation failed", map[string]interface{}{
                "vendor_id":   vendor.ID,
                "attempt":     attempt,
                "rejected":    len(rejected),
                "error":       err.Error(),
            })
            lastErr = err
            requestedCount = max(3, int(float64(requestedCount)*0.8))
            continue
        }

        o.logger.Infof("validation succeeded", map[string]interface{}{
            "vendor_id":   vendor.ID,
            "accepted":    len(accepted),
            "rejected":    len(rejected),
            "prompt_hash": promptHash,
            "retry_count": attempt,
        })

        // Store the lines
        if !o.cfg.DryRun {
            if err := o.storeLines(vendor, bucket, accepted); err != nil {
                return fmt.Errorf("failed to store lines: %w", err)
            }
        }

        o.logger.Infof("bucket complete", map[string]interface{}{
            "vendor_id": vendor.ID,
            "inserted":  len(accepted),
        })
        return nil
    }

    return lastErr
}

// callLLM builds a prompt and calls the LLM. Returns lines, prompt hash, error.
func (o *Orchestrator) callLLM(vendor vendors.VendorProfile, bucket DialogueBucket, count, attempt int) ([]string, string, error) {
    p := prompts.BuildPrompt(
        vendor,
        bucket.LineType,
        bucket.Bucket,
        bucket.TransactionContext,
        bucket.InventoryContext,
        count,
    )

    if o.cfg.LogLevelString == "debug" {
        o.logger.Debugf("calling LLM", map[string]interface{}{
            "vendor_id":   vendor.ID,
            "prompt_hash": p.Hash,
            "system":      p.System,
            "user":        p.User,
        })
    }

    if o.cfg.DryRun {
        o.logger.Infof("dry run: skipping LLM call", nil)
        return nil, p.Hash, nil
    }

    lines, err := o.llm.Generate(p.System, p.User, attempt)
    return lines, p.Hash, err
}

// storeLines deletes old rows for the scope and inserts fresh lines.
// Uses HTTP API if configured, otherwise writes directly to DB.
func (o *Orchestrator) storeLines(vendor vendors.VendorProfile, bucket DialogueBucket, lines []string) error {
    if o.cfg.UseHTTPAPI {
        return o.storeLinesViaHTTP(vendor, bucket, lines)
    }
    return o.storeLinesDirectDB(vendor, bucket, lines)
}

// storeLinesViaHTTP submits lines to PHP's internal API endpoint.
func (o *Orchestrator) storeLinesViaHTTP(vendor vendors.VendorProfile, bucket DialogueBucket, lines []string) error {
    req := php.SubmitLinesRequest{
        LineType:           bucket.LineType,
        InteractionBucket:  bucket.Bucket,
        TransactionContext: bucket.TransactionContext,
        InventoryContext:   bucket.InventoryContext,
        GenerationVersion:  vendor.DialogueGenerationVersion,
        Lines:              lines,
    }
    return o.phpClient.SubmitLines(vendor.UUID, req)
}

// storeLinesDirectDB deletes old rows for the exact scope and inserts new ones.
func (o *Orchestrator) storeLinesDirectDB(vendor vendors.VendorProfile, bucket DialogueBucket, lines []string) error {
    // Delete old rows for this exact (vendor, lineType, bucket, transactionContext, inventoryContext) scope
    _, err := o.db.Exec(`
        DELETE FROM vendor_dialogue
        WHERE galaxy_vendor_profile_id = ?
          AND line_type = ?
          AND interaction_bucket = ?
          AND transaction_context = ?
          AND inventory_context = ?
    `, vendor.ID, bucket.LineType, bucket.Bucket, bucket.TransactionContext, bucket.InventoryContext)
    if err != nil {
        return fmt.Errorf("failed to delete old dialogue rows: %w", err)
    }

    // Insert new rows
    for _, line := range lines {
        _, err := o.db.Exec(`
            INSERT INTO vendor_dialogue
                (galaxy_vendor_profile_id, line_type, interaction_bucket, transaction_context,
                 inventory_context, line_text, weight, generation_version, created_at, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, 1.0000, ?, NOW(), NOW())
        `, vendor.ID, bucket.LineType, bucket.Bucket, bucket.TransactionContext,
            bucket.InventoryContext, line, vendor.DialogueGenerationVersion)
        if err != nil {
            return fmt.Errorf("failed to insert dialogue line: %w", err)
        }
    }

    return nil
}

// markGenerating updates vendor status via HTTP API or direct DB.
func (o *Orchestrator) markGenerating(vendor vendors.VendorProfile) error {
    if o.cfg.UseHTTPAPI {
        return o.phpClient.UpdateStatus(vendor.UUID, "generating", "")
    }
    return vendors.MarkGenerating(o.db, vendor.ID)
}

// markComplete updates vendor status on success.
func (o *Orchestrator) markComplete(vendor vendors.VendorProfile) error {
    if o.cfg.UseHTTPAPI {
        generatedAt := /* time.Now().UTC().Format(time.RFC3339) */ ""
        return o.phpClient.UpdateStatus(vendor.UUID, "complete", generatedAt)
    }
    return vendors.MarkComplete(o.db, vendor.ID)
}

// markFailed updates vendor status on failure.
func (o *Orchestrator) markFailed(vendor vendors.VendorProfile) error {
    if o.cfg.UseHTTPAPI {
        return o.phpClient.UpdateStatus(vendor.UUID, "failed", "")
    }
    return vendors.MarkFailed(o.db, vendor.ID)
}

// getDialogueBuckets returns the full v1 generation matrix per design §6.
// Every entry is a (lineType, bucket, transactionContext, inventoryContext) scope.
func getDialogueBuckets() []DialogueBucket {
    return []DialogueBucket{
        // --- Greetings (neutral, no inventory context) ---
        {LineType: "greeting", Bucket: "first_visit",     TransactionContext: "neutral", InventoryContext: "none", RequestedCount: 15},
        {LineType: "greeting", Bucket: "second_visit",    TransactionContext: "neutral", InventoryContext: "none", RequestedCount: 15},
        {LineType: "greeting", Bucket: "third_visit",     TransactionContext: "neutral", InventoryContext: "none", RequestedCount: 15},
        {LineType: "greeting", Bucket: "repeat_customer", TransactionContext: "neutral", InventoryContext: "none", RequestedCount: 15},

        // --- Inventory pitches — vendor selling by category ---
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "ship",              RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "shield_projector",  RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "engine",            RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "reactor",           RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "weapon",            RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "sensor_array",      RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "cargo_module",      RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "hull_plating",      RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "salvage_component", RequestedCount: 10},

        // --- Inventory pitches — vendor buying (skeptical/appraising) ---
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_buying", InventoryContext: "ship",              RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_buying", InventoryContext: "engine",            RequestedCount: 10},
        {LineType: "inventory_pitch", Bucket: "repeat_customer", TransactionContext: "vendor_buying", InventoryContext: "salvage_component", RequestedCount: 10},

        // --- Deal outcomes ---
        {LineType: "deal_accepted", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "none", RequestedCount: 10},
        {LineType: "deal_accepted", Bucket: "repeat_customer", TransactionContext: "vendor_buying",  InventoryContext: "none", RequestedCount: 10},
        {LineType: "deal_rejected", Bucket: "repeat_customer", TransactionContext: "vendor_selling", InventoryContext: "none", RequestedCount: 10},
        {LineType: "deal_rejected", Bucket: "repeat_customer", TransactionContext: "vendor_buying",  InventoryContext: "none", RequestedCount: 10},

        // --- Farewells ---
        {LineType: "farewell", Bucket: "repeat_customer", TransactionContext: "neutral", InventoryContext: "none", RequestedCount: 10},
    }
}
```

---

### 8. `internal/validation/validator.go`

**Changes required:**

Expand `validateLine()` to include the full meta-commentary pattern list from design §13.

```go
// validateLine checks a single normalized line against all rejection criteria.
func validateLine(line string) error {
    // ... existing checks (empty, word count, char count, control chars, repeated punctuation) ...

    // Expanded meta-commentary check
    lower := strings.ToLower(line)
    metaPatterns := []string{
        "here are",
        "here's",
        "certainly",
        "of course",
        "as requested",
        "i'll generate",
        "sure, here",
        "[system]",
        "<thinking>",
        "{thinking}",
    }
    for _, pattern := range metaPatterns {
        if strings.Contains(lower, pattern) {
            return fmt.Errorf("meta commentary detected: matched %q", pattern)
        }
    }

    // Check for "line N:" prefix pattern (e.g. "Line 1: Here is your dialogue")
    lineNumberPrefix := regexp.MustCompile(`(?i)^line\s+\d+\s*:`)
    if lineNumberPrefix.MatchString(line) {
        return fmt.Errorf("meta commentary: line number prefix detected")
    }

    return nil
}
```

---

### 9. `internal/jobs/worker.go`

No structural changes needed — the pool already calls `orchestrator.GenerateForVendor()`. Once the orchestrator handles status updates internally, the worker remains correct.

**Minor change:** Update `NewPool` signature to accept `phpClient`:

```go
func NewPool(database *db.DB, llmClient *llm.Client, phpClient *php.Client, logger *logging.Logger, cfg *config.Config) *Pool {
    return &Pool{
        workers:      cfg.WorkerCount,
        vendorChan:   make(chan vendors.VendorProfile, cfg.WorkerCount),
        orchestrator: dialogue.New(database, llmClient, phpClient, logger, cfg),
        logger:       logger,
        db:           database,
    }
}
```

---

### 10. `cmd/vendor-dialogue-generator/main.go`

**Changes required:**

- Construct `php.Client` if `cfg.UseHTTPAPI`
- Pass `phpClient` to `jobs.NewPool`
- When using HTTP API, use `phpClient.FetchPending()` instead of `vendors.FetchPending()`

```go
func main() {
    // ... existing flag parse + config load ...

    // Create PHP client (nil if not in HTTP API mode)
    var phpClient *php.Client
    if cfg.UseHTTPAPI {
        phpClient = php.New(cfg.PHPBaseURL, cfg.PHPInternalToken)
    }

    // ... existing logger, db, llmClient setup ...

    pool := jobs.NewPool(database, llmClient, phpClient, logger, cfg)
    pool.Start()

    // Load pending vendors
    var pendingVendors []vendors.VendorProfile
    var err error

    if cfg.UseHTTPAPI {
        // Fetch via PHP HTTP API
        phpVendors, fetchErr := phpClient.FetchPending()
        if fetchErr != nil {
            logger.Errorf("failed to fetch pending vendors via HTTP", map[string]interface{}{"error": fetchErr.Error()})
            os.Exit(1)
        }
        // Map php.VendorRecord → vendors.VendorProfile
        for _, pv := range phpVendors {
            personality := vendors.Personality(pv.Profile.Personality)
            pendingVendors = append(pendingVendors, vendors.VendorProfile{
                ID:                        pv.ID,
                UUID:                      pv.UUID,
                ServiceType:               pv.ServiceType,
                Criminality:               pv.Criminality,
                MarkupBase:                pv.Profile.MarkupBase,
                Personality:               personality,
                DialogueGenerationVersion: pv.DialogueGenerationVersion,
                DialogueGenerationStatus:  "pending",
            })
        }
        // Apply batch limit if configured
        if cfg.BatchSize > 0 && len(pendingVendors) > cfg.BatchSize {
            pendingVendors = pendingVendors[:cfg.BatchSize]
        }
    } else {
        // Fetch directly from DB
        fetchLimit := cfg.BatchSize
        if cfg.RowsToGenerate > 0 && cfg.RowsToGenerate < cfg.BatchSize {
            fetchLimit = cfg.RowsToGenerate
        }
        pendingVendors, err = vendors.FetchPending(database, fetchLimit, logger)
        if err != nil {
            logger.Errorf("failed to fetch pending vendors", map[string]interface{}{"error": err.Error()})
            os.Exit(1)
        }
    }

    logger.Infof("fetched pending vendors", map[string]interface{}{"count": len(pendingVendors)})

    for _, vendor := range pendingVendors {
        pool.Submit(vendor)
    }

    pool.Close()

    logger.Infof("vendor dialogue generator completed", map[string]interface{}{"processed": len(pendingVendors)})
    os.Exit(0)
}
```

---

## `.env.example` Updates

Add the following new variables:

```env
# Dialogue generator mode
USE_HTTP_API=false

# Required if USE_HTTP_API=true: PHP internal API connection
PHP_BASE_URL=http://localhost:8000
PHP_INTERNAL_TOKEN=change-me-to-a-secure-random-string

# Generation versioning
GENERATION_VERSION=1

# LLM settings — use lower values for small local models
LLM_TEMPERATURE=0.4
LLM_TOP_P=0.7
LLM_MAX_TOKENS=500
LLM_TIMEOUT_SECONDS=60
```

---

## Canonical Enums (must match PHP exactly)

Both Go and PHP must treat these as strict enumerations. No ad-hoc strings.

### Line types
`greeting`, `inventory_pitch`, `deal_accepted`, `deal_rejected`, `farewell`

### Interaction buckets
`first_visit`, `second_visit`, `third_visit`, `repeat_customer`

### Transaction contexts
`neutral`, `vendor_selling`, `vendor_buying`

### Inventory contexts
`none`, `ship`, `shield_projector`, `engine`, `reactor`, `weapon`, `sensor_array`, `cargo_module`, `hull_plating`, `salvage_component`

Consider defining these as package-level constants in a new `internal/constants/enums.go` file:

```go
package constants

const (
    // LineTypes
    LineTypeGreeting       = "greeting"
    LineTypeInventoryPitch = "inventory_pitch"
    LineTypeDealAccepted   = "deal_accepted"
    LineTypeDealRejected   = "deal_rejected"
    LineTypeFarewell       = "farewell"

    // InteractionBuckets
    BucketFirstVisit     = "first_visit"
    BucketSecondVisit    = "second_visit"
    BucketThirdVisit     = "third_visit"
    BucketRepeatCustomer = "repeat_customer"

    // TransactionContexts
    TxNeutral       = "neutral"
    TxVendorSelling = "vendor_selling"
    TxVendorBuying  = "vendor_buying"

    // InventoryContexts
    InvNone              = "none"
    InvShip              = "ship"
    InvShieldProjector   = "shield_projector"
    InvEngine            = "engine"
    InvReactor           = "reactor"
    InvWeapon            = "weapon"
    InvSensorArray       = "sensor_array"
    InvCargoModule       = "cargo_module"
    InvHullPlating       = "hull_plating"
    InvSalvageComponent  = "salvage_component"
)
```

---

## Generation Matrix (full v1 — 22 scopes)

| line_type       | bucket           | transaction_context | inventory_context   | count |
|-----------------|------------------|---------------------|---------------------|-------|
| greeting        | first_visit      | neutral             | none                | 15    |
| greeting        | second_visit     | neutral             | none                | 15    |
| greeting        | third_visit      | neutral             | none                | 15    |
| greeting        | repeat_customer  | neutral             | none                | 15    |
| inventory_pitch | repeat_customer  | vendor_selling      | ship                | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | shield_projector    | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | engine              | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | reactor             | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | weapon              | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | sensor_array        | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | cargo_module        | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | hull_plating        | 10    |
| inventory_pitch | repeat_customer  | vendor_selling      | salvage_component   | 10    |
| inventory_pitch | repeat_customer  | vendor_buying       | ship                | 10    |
| inventory_pitch | repeat_customer  | vendor_buying       | engine              | 10    |
| inventory_pitch | repeat_customer  | vendor_buying       | salvage_component   | 10    |
| deal_accepted   | repeat_customer  | vendor_selling      | none                | 10    |
| deal_accepted   | repeat_customer  | vendor_buying       | none                | 10    |
| deal_rejected   | repeat_customer  | vendor_selling      | none                | 10    |
| deal_rejected   | repeat_customer  | vendor_buying       | none                | 10    |
| farewell        | repeat_customer  | neutral             | none                | 10    |

Total: **22 scopes × ~10 lines = ~220 lines per vendor**

---

## Implementation Order

```
Step 1 — internal/constants/enums.go (new file, no dependencies)
Step 2 — internal/config/config.go (add GenerationVersion, LLMTimeout, HTTP API fields, fix LLM defaults)
Step 3 — internal/llm/types.go (add Stream field)
Step 4 — internal/llm/client.go (add timeout, add attempt param, set Stream false)
Step 5 — internal/vendors/vendor.go (add UUID, generation status/version fields)
Step 6 — internal/vendors/repository.go (fix FetchPending query, add Mark* functions)
Step 7 — internal/php/client.go (new file — HTTP API client)
Step 8 — internal/prompts/builder.go (full rewrite: correct service types, add transaction context, tone derivation)
Step 9 — internal/validation/validator.go (expand meta-commentary patterns)
Step 10 — internal/dialogue/orchestrator.go (add TransactionContext to DialogueBucket, full matrix, fix DELETE/INSERT, add status calls, add HTTP/DB store switch)
Step 11 — internal/jobs/worker.go (update NewPool signature to accept phpClient)
Step 12 — cmd/vendor-dialogue-generator/main.go (wire phpClient, switch FetchPending source)
Step 13 — .env.example (add new env vars)
```

---

## Files Summary

| File | Action | Key Change |
|------|--------|------------|
| `internal/constants/enums.go` | **Create** | Canonical string constants for all enums |
| `internal/php/client.go` | **Create** | HTTP client for PHP internal API (FetchPending, UpdateStatus, SubmitLines) |
| `internal/config/config.go` | **Modify** | Add GenerationVersion, LLMTimeoutSeconds, UseHTTPAPI, PHPBaseURL, PHPInternalToken; fix temp/top_p defaults |
| `internal/llm/types.go` | **Modify** | Add `Stream bool` to ChatRequest |
| `internal/llm/client.go` | **Modify** | Add HTTP timeout; accept attempt param; set Stream=false; lower temp on retry |
| `internal/vendors/vendor.go` | **Modify** | Add UUID, DialogueGenerationStatus, DialogueGenerationVersion fields |
| `internal/vendors/repository.go` | **Modify** | Fix FetchPending (WHERE status, UUID in SELECT, correct table); add MarkGenerating/Complete/Failed |
| `internal/prompts/builder.go` | **Rewrite** | Correct service type mappings; add transaction context; criminality/markup/personality tone hints |
| `internal/validation/validator.go` | **Modify** | Expand meta-commentary pattern list |
| `internal/dialogue/orchestrator.go` | **Modify** | Add TransactionContext to DialogueBucket; full 22-scope matrix; fix DELETE/INSERT; status tracking; HTTP/DB store switch |
| `internal/jobs/worker.go` | **Modify** | Update NewPool to accept phpClient |
| `cmd/vendor-dialogue-generator/main.go` | **Modify** | Wire phpClient; switch FetchPending source based on UseHTTPAPI |
| `.env.example` | **Modify** | Add USE_HTTP_API, PHP_BASE_URL, PHP_INTERNAL_TOKEN, GENERATION_VERSION; fix LLM defaults |

---

## Notes on Direct DB Mode vs. HTTP API Mode

If `USE_HTTP_API=false` (direct DB mode):
- `FetchPending` queries `galaxy_vendor_profiles` directly
- `MarkGenerating`/`MarkComplete`/`MarkFailed` UPDATE `galaxy_vendor_profiles` directly
- `storeLinesDirectDB` DELETEs and INSERTs into `vendor_dialogue` using `galaxy_vendor_profile_id` FK
- PHP-side validation (`DialogueValidationService`) does NOT run on these rows

If `USE_HTTP_API=true`:
- `phpClient.FetchPending()` polls PHP for work
- `phpClient.UpdateStatus()` handles status transitions
- `phpClient.SubmitLines()` sends lines to PHP, which validates and stores them
- PHP-side validation runs, cache hooks can be added later without Go changes

**Recommended default for development: `USE_HTTP_API=false` (simpler). Switch to `true` once PHP internal API is deployed.**
