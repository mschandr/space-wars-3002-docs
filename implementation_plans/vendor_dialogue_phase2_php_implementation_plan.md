# Vendor Dialogue System — Phase 2: PHP Support for Go Dialogue Generator
## Created: 2026-03-13
## Source: docs/design/vendor_dialogue_phased_architecture.md + vendor_dialogue_joint_design_go_php.md

---

## Overview

Phase 2 is primarily a Go service. However the PHP backend must provide supporting infrastructure for the Go generator to function cleanly. Specifically, PHP must:

1. Expose a **secure internal HTTP API** for the Go service to poll, submit, and update dialogue
2. **Validate** submitted dialogue lines server-side before storage (defense-in-depth)
3. Provide **artisan commands** for operators to queue vendors, check status, and run diagnostics
4. Add **configuration** for the internal token and generator endpoint

The alternative — letting Go write directly to the database — is architecturally valid but bypasses PHP-side validation and makes it harder to add side effects (cache invalidation, regeneration jobs) later. The HTTP API approach is recommended.

---

## What Already Exists (Do Not Re-Implement)

| Item | Location | Notes |
|------|----------|-------|
| `AdminVendorDialogueController` | `app/Http/Controllers/Admin/` | Pending list, regenerate, inspect endpoints |
| Admin routes | `routes/api.php` | `/api/admin/vendors/...` under Sanctum auth |
| `GalaxyVendorProfile::markDialogueStale()` | `app/Models/GalaxyVendorProfile.php` | Increments version, sets status to pending |
| `scopeNeedingDialogueGeneration()` | `app/Models/GalaxyVendorProfile.php` | Filters pending + failed |
| `GalaxyVendorProfileObserver` | `app/Observers/` | Auto-marks pending when service_type/criminality/personality/markup_base changes |
| Generation status tracking columns | `galaxy_vendor_profiles` table | dialogue_generation_status, dialogue_generation_version, dialogue_generated_at |
| `VendorDialogueService::getGenerationStatus()` | `app/Services/VendorDialogueService.php` | Used by admin inspect endpoint |

---

## Phase 2 Deliverables (PHP)

### 2.1 Configuration

#### `config/vendor_dialogue.php` — NEW FILE

```
return [

    /*
     * Internal API token used to authenticate the Go dialogue generator.
     * Go must send: Authorization: Bearer <token>
     * Set DIALOGUE_GENERATOR_TOKEN in .env
     */
    'internal_token' => env('DIALOGUE_GENERATOR_TOKEN', null),

    /*
     * Optional: URL of the Go generator service (used by TriggerVendorDialogueRegenerationJob in Phase 6)
     * Set DIALOGUE_GENERATOR_URL in .env
     */
    'generator_url' => env('DIALOGUE_GENERATOR_URL', null),

    /*
     * Maximum number of dialogue lines accepted per vendor per submission request.
     * Go submits lines per vendor per scope (line_type + bucket + context combination).
     */
    'max_lines_per_submission' => 20,

    /*
     * Validation constraints mirroring Go's validation rules.
     * Both sides must reject lines outside these bounds.
     */
    'validation' => [
        'min_words' => 6,
        'max_words' => 20,
        'max_characters' => 255,
    ],

    /*
     * Generation matrix: the exact (line_type, interaction_bucket, transaction_context, inventory_context)
     * combinations that the Go generator is expected to produce.
     * PHP uses this to count coverage on the status command.
     */
    'generation_matrix' => [
        // Greetings
        ['line_type' => 'greeting', 'bucket' => 'first_visit',      'transaction_context' => 'neutral',        'inventory_context' => 'none'],
        ['line_type' => 'greeting', 'bucket' => 'second_visit',     'transaction_context' => 'neutral',        'inventory_context' => 'none'],
        ['line_type' => 'greeting', 'bucket' => 'third_visit',      'transaction_context' => 'neutral',        'inventory_context' => 'none'],
        ['line_type' => 'greeting', 'bucket' => 'repeat_customer',  'transaction_context' => 'neutral',        'inventory_context' => 'none'],

        // Inventory pitches (vendor_selling)
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'ship'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'shield_projector'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'engine'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'reactor'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'weapon'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'sensor_array'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'cargo_module'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'hull_plating'],
        ['line_type' => 'inventory_pitch', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'salvage_component'],

        // Deal responses
        ['line_type' => 'deal_accepted', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'none'],
        ['line_type' => 'deal_accepted', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_buying',  'inventory_context' => 'none'],
        ['line_type' => 'deal_rejected', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_selling', 'inventory_context' => 'none'],
        ['line_type' => 'deal_rejected', 'bucket' => 'repeat_customer', 'transaction_context' => 'vendor_buying',  'inventory_context' => 'none'],

        // Farewell
        ['line_type' => 'farewell', 'bucket' => 'repeat_customer', 'transaction_context' => 'neutral', 'inventory_context' => 'none'],
    ],
];
```

#### `.env` additions (document in `.env.example`)

```
DIALOGUE_GENERATOR_TOKEN=change-me-to-a-secure-random-string
DIALOGUE_GENERATOR_URL=http://localhost:8090
```

---

### 2.2 Internal Authentication Middleware

#### `app/Http/Middleware/InternalApiTokenMiddleware.php` — NEW FILE

```
class InternalApiTokenMiddleware
{
    public function handle(Request $request, Closure $next): Response
        token = config('vendor_dialogue.internal_token')

        if !token
            // Token not configured — block all access
            return response()->json(['error' => 'Internal API not configured'], 503)

        bearer = $request->bearerToken()

        if !bearer || !hash_equals($token, $bearer)
            return response()->json(['error' => 'Unauthorized'], 401)

        return $next($request)
}
```

Register in `bootstrap/app.php` (Laravel 11 style) or `app/Http/Kernel.php`:

```
// In withMiddleware() callback:
$middleware->alias([
    'internal.token' => InternalApiTokenMiddleware::class,
]);
```

---

### 2.3 Internal Dialogue Generator Controller

#### `app/Http/Controllers/Internal/DialogueGeneratorController.php` — NEW FILE

Three endpoints for the Go service to use.

```
class DialogueGeneratorController extends Controller
{
    public function __construct(
        private DialogueValidationService $validationService,
    ) {}

    /**
     * GET /api/internal/vendor-dialogue/pending
     *
     * Returns all vendors needing dialogue generation with the full profile
     * data the Go generator requires to build prompts.
     *
     * Go polls this on startup and periodically.
     */
    public function pending(Request $request): JsonResponse
        vendors = GalaxyVendorProfile::needingDialogueGeneration()
            ->with('vendorProfile:id,name,archetype,service_type,criminality,personality,markup_base')
            ->get()

        return response()->json([
            'count' => vendors->count(),
            'vendors' => vendors->map(fn ($v) => [
                'id' => $v->id,
                'uuid' => $v->uuid,
                'service_type' => $v->service_type,
                'criminality' => (float) $v->criminality,
                'dialogue_generation_version' => $v->dialogue_generation_version,
                'profile' => [
                    'archetype' => $v->vendorProfile?->archetype?->value ?? 'honest_dealer',
                    'personality' => $v->vendorProfile?->personality ?? [],
                    'markup_base' => (float) ($v->vendorProfile?->markup_base ?? 0.10),
                ],
            ])
        ])

    /**
     * PATCH /api/internal/vendor-dialogue/{vendorUuid}/status
     *
     * Go calls this to update dialogue generation status as it works.
     * Body: { status: 'generating'|'complete'|'failed', generated_at?: ISO8601 }
     *
     * Status transitions:
     *   pending    → generating  (Go starts processing)
     *   generating → complete    (Go finished successfully)
     *   generating → failed      (Go gave up after retries)
     */
    public function updateStatus(Request $request, string $vendorUuid): JsonResponse
        validated = $request->validate([
            'status'       => 'required|in:generating,complete,failed',
            'generated_at' => 'nullable|date',
        ])

        vendor = GalaxyVendorProfile::findByUuid(vendorUuid)
        if !vendor return response()->json(['error' => 'Not found'], 404)

        updates = ['dialogue_generation_status' => validated['status']]

        if validated['status'] === 'complete'
            updates['dialogue_generated_at'] = validated['generated_at'] ?? now()

        vendor->update(updates)

        return response()->json(['ok' => true, 'status' => vendor->dialogue_generation_status])

    /**
     * POST /api/internal/vendor-dialogue/{vendorUuid}/lines
     *
     * Go submits generated lines for a specific (line_type, bucket, transaction_context,
     * inventory_context) scope. PHP validates and stores them.
     *
     * Replace strategy: existing rows matching the exact scope are deleted before insert.
     *
     * Body:
     * {
     *   "line_type": "greeting",
     *   "interaction_bucket": "first_visit",
     *   "transaction_context": "neutral",
     *   "inventory_context": "none",
     *   "generation_version": 3,
     *   "lines": ["Looking to trade?", "Back again, are we?"]
     * }
     */
    public function submitLines(Request $request, string $vendorUuid): JsonResponse
        validated = $request->validate([
            'line_type'           => 'required|in:greeting,inventory_pitch,deal_accepted,deal_rejected,farewell',
            'interaction_bucket'  => 'required|in:first_visit,second_visit,third_visit,repeat_customer',
            'transaction_context' => 'required|in:neutral,vendor_selling,vendor_buying',
            'inventory_context'   => 'required|string|max:64',
            'generation_version'  => 'required|integer|min:1',
            'lines'               => 'required|array|min:1|max:' . config('vendor_dialogue.max_lines_per_submission'),
            'lines.*'             => 'required|string|max:255',
        ])

        vendor = GalaxyVendorProfile::findByUuid(vendorUuid)
        if !vendor return response()->json(['error' => 'Not found'], 404)

        // Validate each line — reject submission if any line fails
        errors = validationService->validateLines(validated['lines'])
        if !empty(errors)
            return response()->json(['error' => 'Validation failed', 'failures' => errors], 422)

        // Deduplicate within submission
        uniqueLines = collect(validated['lines'])->unique()->values()

        DB::transaction(function () use ($vendor, $validated, $uniqueLines)
            // Delete old rows for this exact scope
            VendorDialogue::where('galaxy_vendor_profile_id', $vendor->id)
                ->where('line_type', $validated['line_type'])
                ->where('interaction_bucket', $validated['interaction_bucket'])
                ->where('transaction_context', $validated['transaction_context'])
                ->where('inventory_context', $validated['inventory_context'])
                ->delete()

            // Insert new rows
            now = now()
            rows = $uniqueLines->map(fn ($line) => [
                'galaxy_vendor_profile_id' => $vendor->id,
                'line_type'                => $validated['line_type'],
                'interaction_bucket'       => $validated['interaction_bucket'],
                'transaction_context'      => $validated['transaction_context'],
                'inventory_context'        => $validated['inventory_context'],
                'generation_version'       => $validated['generation_version'],
                'line_text'                => $line,
                'weight'                   => 1.0,
                'created_at'               => now,
                'updated_at'               => now,
            ])->toArray()

            VendorDialogue::insert($rows)
        )

        return response()->json([
            'ok'       => true,
            'stored'   => $uniqueLines->count(),
            'rejected' => count($validated['lines']) - $uniqueLines->count(),
        ])
}
```

---

### 2.4 PHP-Side Dialogue Validation Service

#### `app/Services/Vendor/DialogueValidationService.php` — NEW FILE

Mirrors Go's validation rules. Used by `DialogueGeneratorController::submitLines()`.

```
class DialogueValidationService
{
    private const META_COMMENTARY_PATTERNS = [
        '/\bsure\b.*\bhere\b/i',
        '/\bof course\b/i',
        '/\bcertainly\b/i',
        '/\bI\'ll generate\b/i',
        '/\bhere are\b/i',
        '/\bhere\'s\b/i',
        '/\bas requested\b/i',
        '/\bline\s+\d+/i',
    ]

    /**
     * Validate a list of dialogue lines.
     * Returns an array of failures: [['line' => string, 'reason' => string]]
     * Empty array means all lines passed.
     */
    public function validateLines(array $lines): array
        minWords   = config('vendor_dialogue.validation.min_words', 6)
        maxWords   = config('vendor_dialogue.validation.max_words', 20)
        maxChars   = config('vendor_dialogue.validation.max_characters', 255)
        failures   = []
        seen       = []

        foreach lines as $index => $line
            // Length check
            if strlen($line) > $maxChars
                failures[] = ['line' => $line, 'reason' => "Exceeds {$maxChars} characters"]
                continue

            // Word count check
            wordCount = str_word_count($line)
            if wordCount < $minWords
                failures[] = ['line' => $line, 'reason' => "Too short ({$wordCount} words, min {$minWords})"]
                continue
            if wordCount > $maxWords
                failures[] = ['line' => $line, 'reason' => "Too long ({$wordCount} words, max {$maxWords})"]
                continue

            // Control characters / malformed punctuation
            if $this->hasControlCharacters($line)
                failures[] = ['line' => $line, 'reason' => 'Contains control characters']
                continue

            // Meta-commentary check
            if $this->isMetaCommentary($line)
                failures[] = ['line' => $line, 'reason' => 'Contains meta-commentary']
                continue

            // Cross-submission duplicate check (within this batch)
            normalized = strtolower(trim($line))
            if isset($seen[$normalized])
                failures[] = ['line' => $line, 'reason' => 'Duplicate line in submission']
                continue
            $seen[$normalized] = true

        return failures

    private function hasControlCharacters(string $line): bool
        return preg_match('/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/', $line) === 1

    private function isMetaCommentary(string $line): bool
        foreach META_COMMENTARY_PATTERNS as $pattern
            if preg_match($pattern, $line)
                return true
        return false

    /**
     * Check if a line already exists in the DB for a given vendor+scope.
     * Used for cross-generation deduplication (optional, slower).
     */
    public function isDuplicate(int $galaxyVendorProfileId, string $lineType, string $lineText): bool
        return VendorDialogue::where('galaxy_vendor_profile_id', $galaxyVendorProfileId)
            ->where('line_type', $lineType)
            ->where('line_text', $lineText)
            ->exists()
}
```

---

### 2.5 Artisan Commands

#### `app/Console/Commands/VendorDialogueQueueCommand.php` — NEW FILE

```
class VendorDialogueQueueCommand extends Command
{
    protected $signature = 'vendor:dialogue-queue
                            {--galaxy= : Only queue vendors in this galaxy UUID}
                            {--all    : Queue all vendors, including those with status=complete}
                            {--force  : Also re-queue vendors currently status=generating}'

    protected $description = 'Mark galaxy vendor profiles as pending for dialogue generation'

    public function handle(): int
        query = GalaxyVendorProfile::query()

        // Scope to galaxy if provided
        if galaxyUuid = this->option('galaxy')
            galaxy = Galaxy::findByUuid(galaxyUuid)
            if !galaxy
                this->error("Galaxy not found: {$galaxyUuid}")
                return Command::FAILURE
            query->where('galaxy_id', $galaxy->id)

        // Determine which statuses to re-queue
        if this->option('all')
            // Re-queue everything, including complete
            statuses = ['pending', 'failed', 'complete']
        elseif this->option('force')
            statuses = ['pending', 'failed', 'generating']
        else
            // Default: only pending and failed
            statuses = ['pending', 'failed']

        count = query->whereIn('dialogue_generation_status', $statuses)
            ->update([
                'dialogue_generation_status' => 'pending',
                'dialogue_generated_at'      => null,
            ])

        this->info("Queued {$count} vendors for dialogue generation.")
        return Command::SUCCESS
}
```

#### `app/Console/Commands/VendorDialogueStatusCommand.php` — NEW FILE

> **Note**: This was previously planned in Phase 6. It belongs in Phase 2 because it's an operational tool for Go generator monitoring.

```
class VendorDialogueStatusCommand extends Command
{
    protected $signature = 'vendor:dialogue-status {galaxyUuid?}'
    protected $description = 'Show dialogue generation status for vendors in a galaxy or across all galaxies'

    public function handle(): int
        query = GalaxyVendorProfile::with('galaxy:id,uuid,name', 'vendorProfile:id,name')

        if galaxyUuid = this->argument('galaxyUuid')
            galaxy = Galaxy::findByUuid(galaxyUuid)
            if !galaxy
                this->error("Galaxy not found: {$galaxyUuid}")
                return Command::FAILURE
            query->where('galaxy_id', $galaxy->id)

        vendors = query->get()

        if vendors->isEmpty()
            this->info('No vendors found.')
            return Command::SUCCESS

        // Summary table grouped by status
        grouped = vendors->groupBy('dialogue_generation_status')
        this->table(['Status', 'Count'], grouped->map(fn ($g, $s) => [$s, $g->count()])->values())

        // Coverage: how many matrix slots are filled per vendor?
        matrixSize = count(config('vendor_dialogue.generation_matrix', []))

        // Detail table
        this->table(
            ['Galaxy', 'Vendor', 'Service', 'Status', 'Version', 'Line Count', 'Generated At'],
            vendors->map(fn ($v) => [
                $v->galaxy->name ?? $v->galaxy->uuid,
                $v->vendorProfile->name ?? $v->uuid,
                $v->service_type,
                $v->dialogue_generation_status,
                $v->dialogue_generation_version,
                $v->dialogueLines()->count() . '/' . $matrixSize,
                $v->dialogue_generated_at?->toDateTimeString() ?? '—',
            ])
        )

        return Command::SUCCESS
}
```

---

### 2.6 Route Registration

#### `routes/api.php` — MODIFY

Add an internal route group protected by the `internal.token` middleware, separate from Sanctum:

```php
// Internal API — authenticated by shared token, used by Go dialogue generator
Route::prefix('internal')->middleware('internal.token')->group(function () {
    Route::get('vendor-dialogue/pending', [DialogueGeneratorController::class, 'pending']);
    Route::patch('vendor-dialogue/{vendorUuid}/status', [DialogueGeneratorController::class, 'updateStatus']);
    Route::post('vendor-dialogue/{vendorUuid}/lines', [DialogueGeneratorController::class, 'submitLines']);
});
```

---

### 2.7 Middleware Registration

#### `bootstrap/app.php` — MODIFY (Laravel 11)

```php
->withMiddleware(function (Middleware $middleware) {
    // ... existing entries ...
    $middleware->alias([
        'internal.token' => \App\Http\Middleware\InternalApiTokenMiddleware::class,
    ]);
})
```

---

## Go Dialogue Generator — DB / HTTP Contract Reference

This section documents what the Go service reads and writes, so both sides stay in sync.

### Go reads from PHP

**Poll endpoint:**
```
GET /api/internal/vendor-dialogue/pending
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
```

Response shape:
```json
{
  "count": 12,
  "vendors": [
    {
      "id": 42,
      "uuid": "53568459-6bb0-4540-b032-918138ced0be",
      "service_type": "salvage_yard",
      "criminality": 0.27,
      "dialogue_generation_version": 3,
      "profile": {
        "archetype": "opportunist",
        "personality": {
          "honesty": 0.45,
          "greed": 0.55,
          "charm": 0.44,
          "risk_tolerance": 0.76,
          "ego_drive": 0.62,
          "empathy": 0.31,
          "curiosity": 0.58
        },
        "markup_base": 0.2152
      }
    }
  ]
}
```

### Go writes to PHP

**Mark as generating (begin processing a vendor):**
```
PATCH /api/internal/vendor-dialogue/{uuid}/status
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
Content-Type: application/json

{ "status": "generating" }
```

**Submit lines for one scope:**
```
POST /api/internal/vendor-dialogue/{uuid}/lines
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
Content-Type: application/json

{
  "line_type": "greeting",
  "interaction_bucket": "first_visit",
  "transaction_context": "neutral",
  "inventory_context": "none",
  "generation_version": 3,
  "lines": [
    "What do you need, stranger?",
    "Don't get lost in here. I've seen it happen.",
    "You've got the look of someone who wants something."
  ]
}
```

**Mark as complete:**
```
PATCH /api/internal/vendor-dialogue/{uuid}/status
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
Content-Type: application/json

{ "status": "complete", "generated_at": "2026-03-13T14:30:00Z" }
```

**Mark as failed:**
```
PATCH /api/internal/vendor-dialogue/{uuid}/status
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
Content-Type: application/json

{ "status": "failed" }
```

### Canonical Shared Enums

Both Go and PHP must treat these as strict enumerations. No ad-hoc strings allowed.

**line_type**: `greeting`, `inventory_pitch`, `deal_accepted`, `deal_rejected`, `farewell`

**interaction_bucket**: `first_visit`, `second_visit`, `third_visit`, `repeat_customer`

**transaction_context**: `neutral`, `vendor_selling`, `vendor_buying`

**inventory_context**: `none`, `ship`, `shield_projector`, `engine`, `reactor`, `weapon`, `sensor_array`, `cargo_module`, `hull_plating`, `salvage_component`

---

## Generation Matrix

The Go service should generate the following scopes per vendor. PHP's `generation_matrix` config lists these for coverage reporting.

| line_type       | bucket           | transaction_context | inventory_context   |
|-----------------|------------------|---------------------|---------------------|
| greeting        | first_visit      | neutral             | none                |
| greeting        | second_visit     | neutral             | none                |
| greeting        | third_visit      | neutral             | none                |
| greeting        | repeat_customer  | neutral             | none                |
| inventory_pitch | repeat_customer  | vendor_selling      | ship                |
| inventory_pitch | repeat_customer  | vendor_selling      | shield_projector    |
| inventory_pitch | repeat_customer  | vendor_selling      | engine              |
| inventory_pitch | repeat_customer  | vendor_selling      | reactor             |
| inventory_pitch | repeat_customer  | vendor_selling      | weapon              |
| inventory_pitch | repeat_customer  | vendor_selling      | sensor_array        |
| inventory_pitch | repeat_customer  | vendor_selling      | cargo_module        |
| inventory_pitch | repeat_customer  | vendor_selling      | hull_plating        |
| inventory_pitch | repeat_customer  | vendor_selling      | salvage_component   |
| deal_accepted   | repeat_customer  | vendor_selling      | none                |
| deal_accepted   | repeat_customer  | vendor_buying       | none                |
| deal_rejected   | repeat_customer  | vendor_selling      | none                |
| deal_rejected   | repeat_customer  | vendor_buying       | none                |
| farewell        | repeat_customer  | neutral             | none                |

Total: **18 scopes × N lines per scope** (recommended: 5–10 lines per scope)

---

## Validation Rules Reference

PHP and Go must agree on rejection criteria. PHP enforces these on submission; Go enforces them before submission.

| Rule | Detail |
|------|--------|
| Min words | 6 words |
| Max words | 20 words |
| Max characters | 255 characters |
| No control characters | `\x00–\x08`, `\x0B`, `\x0C`, `\x0E–\x1F`, `\x7F` |
| No meta-commentary | Must not contain: "here are", "here's", "certainly", "of course", "as requested", "line N:", "I'll generate", "sure, here" |
| No exact item facts | Must not include specific percentages, defect names, or live item data (those belong in runtime composition) |
| No duplicates | Within-scope deduplication: same text string rejected |

---

## File Summary

| File | Action | Notes |
|------|--------|-------|
| `config/vendor_dialogue.php` | Create | Internal token, generator URL, validation bounds, generation matrix |
| `.env.example` | Modify | Add `DIALOGUE_GENERATOR_TOKEN`, `DIALOGUE_GENERATOR_URL` |
| `app/Http/Middleware/InternalApiTokenMiddleware.php` | Create | Bearer token check for Go service |
| `app/Http/Controllers/Internal/DialogueGeneratorController.php` | Create | pending, updateStatus, submitLines endpoints |
| `app/Services/Vendor/DialogueValidationService.php` | Create | Line validation, deduplication, meta-commentary detection |
| `app/Console/Commands/VendorDialogueQueueCommand.php` | Create | `vendor:dialogue-queue` — marks vendors pending |
| `app/Console/Commands/VendorDialogueStatusCommand.php` | Create | `vendor:dialogue-status` — status report (moved from Phase 6) |
| `routes/api.php` | Modify | Add `/api/internal/` route group with `internal.token` middleware |
| `bootstrap/app.php` | Modify | Register `internal.token` middleware alias |

---

## Implementation Order

```
1. config/vendor_dialogue.php + .env.example
2. InternalApiTokenMiddleware + register alias
3. DialogueValidationService (pure logic, easy to unit test)
4. DialogueGeneratorController (depends on service)
5. Routes
6. VendorDialogueQueueCommand
7. VendorDialogueStatusCommand
```

---

## Testing Notes

| Test | Type | What to assert |
|------|------|----------------|
| `InternalApiTokenMiddleware` | Unit | Correct token → passes; wrong token → 401; missing config → 503 |
| `DialogueValidationService::validateLines()` | Unit | Min/max words, max chars, control chars, meta-commentary, duplicates |
| `DialogueGeneratorController::pending()` | Feature | Returns pending+failed vendors; profile data present |
| `DialogueGeneratorController::updateStatus()` | Feature | Status transitions write correctly; bad UUID → 404; bad status → 422 |
| `DialogueGeneratorController::submitLines()` | Feature | Lines stored correctly; replace strategy deletes old scope; validation failures return 422 |
| `VendorDialogueQueueCommand` | Feature | `--all` re-queues complete; `--galaxy` scopes correctly; `--force` includes generating |
| `VendorDialogueStatusCommand` | Feature | Renders table without errors; galaxy UUID scope |

---

## Notes on Direct DB Access (Fallback)

If the Go service is configured to write directly to the database (bypassing the HTTP API), the PHP side still needs to:

- Ensure `VendorDialogue` rows have the correct `galaxy_vendor_profile_id` FK (not `vendor_profile_id`)
- Ensure `dialogue_generation_status` on `galaxy_vendor_profiles` is kept in sync by Go
- Accept that PHP-side validation (`DialogueValidationService`) will not run on those rows

The HTTP API approach is recommended because it keeps all write validation in PHP and allows side effects to be added later without Go changes.
