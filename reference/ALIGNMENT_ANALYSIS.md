# Alignment Analysis: vendor_profile_dialogue_changes.md

**Date**: 2026-03-10
**Current Repo State**: Partial implementation (table + model only)
**Target Document**: vendor_profile_dialogue_changes.md (PHP backend spec)

---

## Executive Summary

The design document specifies what the **PHP API backend must do** to support the separate Go dialogue generator service. We've implemented 40% of the PHP backend requirements (table + model). We need to add:

- Vendor profile status tracking (3 new fields)
- Dialogue retrieval logic
- Deterministic line selection (no random)
- Fallback dialogue system
- Admin endpoints
- Personality change handling

**Effort**: ~2-3 days for a Laravel developer

---

## Part 1: Database Changes

### ✅ COMPLETED: vendor_dialogue Table

```php
✅ Table created with:
    ✅ vendor_profile_id (FK with cascade delete)
    ✅ line_type ENUM (greeting, inventory_pitch, deal_accepted, deal_rejected, farewell)
    ✅ interaction_bucket ENUM (first_visit, second_visit, third_visit, repeat_customer)
    ✅ inventory_context VARCHAR(50/64) nullable
    ✅ line_text VARCHAR(255)
    ✅ weight DECIMAL(5,4)
    ✅ generation_version INT
    ✅ timestamps
    ✅ 3 indexes (vendor_line_type, vendor_lookup, inventory_context)
```

---

### 🔴 MISSING: vendor_profiles Status Fields

**Required migration**: Add 3 new columns to `vendor_profiles`

```php
// NEW MIGRATION NEEDED
Schema::table('vendor_profiles', function (Blueprint $table) {
    $table->enum('dialogue_generation_status', [
        'pending',
        'generating',
        'complete',
        'failed',
    ])->default('pending')->after('personality');

    $table->unsignedInteger('dialogue_generation_version')
        ->default(1)
        ->after('dialogue_generation_status');

    $table->timestamp('dialogue_generated_at')
        ->nullable()
        ->after('dialogue_generation_version');
});
```

**Impact**:
- Signals Go service which vendors need dialogue
- Tracks generation progress
- Enables regeneration on version bump

---

### 🟡 FUTURE: Drop dialogue_pool Column

**Deferred** (Section 1.1):
```php
// LATER, after Go service is live and tested
Schema::table('vendor_profiles', function (Blueprint $table) {
    $table->dropColumn('dialogue_pool');
});
```

**When**: After 2+ weeks of Go service running successfully in production

---

## Part 2: Laravel Models

### ✅ COMPLETED: VendorDialogue Model

```php
✅ Created with:
    ✅ Correct table name ('vendor_dialogue')
    ✅ Correct casts (weight→float, generation_version→int)
    ✅ belongsTo(VendorProfile) relationship
    ✅ Scopes (forVendor, byLineType, byBucket, byVersion)
    ✅ Helper methods (getDialogue, getVendorDialogue)
    ✅ Factory with state builders
```

---

### 🔴 MISSING: VendorProfile Updates

**Required changes to VendorProfile model**:

```php
// ADD THESE FIELDS TO CASTS
protected $casts = [
    // ... existing ...
    'personality' => 'array',
    'criminality' => 'float',
    'markup_base' => 'float',
    'dialogue_generated_at' => 'datetime',  // NEW
];

// ADD THIS RELATIONSHIP
public function dialogue(): HasMany
{
    return $this->hasMany(VendorDialogue::class, 'vendor_profile_id');
}

// ADD THESE FILLABLE FIELDS
protected $fillable = [
    // ... existing ...
    'dialogue_generation_status',       // NEW
    'dialogue_generation_version',      // NEW
    'dialogue_generated_at',            // NEW
];
```

---

## Part 3: Vendor Creation Behavior

### 🔴 MISSING: Initialization of Dialogue Fields

**Where**: VendorProfile creation (factory, seeder, API controller)

**Required**:
```php
$vendor = VendorProfile::create([
    // ... existing fields ...
    'dialogue_generation_status' => 'pending',      // NEW
    'dialogue_generation_version' => 1,             // NEW
    'dialogue_generated_at' => null,                // NEW
]);
```

**Locations to update**:
1. VendorProfileSeeder (database/seeders/)
2. VendorProfile factory (database/factories/)
3. API endpoint that creates vendors
4. Tests that create vendor fixtures

---

## Part 4: Interaction Bucket Logic

### 🔴 MISSING: Bucket Mapping Function

**Required logic** (Section 4):

```php
// Add to VendorProfile or VendorDialogueService

private function mapInteractionBucket(int $interactionCount): string
{
    return match (true) {
        $interactionCount <= 1 => 'first_visit',
        $interactionCount === 2 => 'second_visit',
        $interactionCount === 3 => 'third_visit',
        default => 'repeat_customer',
    };
}
```

**Usage**: Convert player interaction counts into dialogue buckets for line selection

---

## Part 5: Dialogue Retrieval

### 🔴 MISSING: Proper Retrieval Query Logic

**Current issue**: We have `getDialogue()` using weighted random. Design requires **deterministic selection**.

**Required query** (Section 5):

```php
// Add to VendorDialogueService or VendorProfile

public function getDialogueLines(
    string $lineType,
    string $interactionBucket,
    ?string $inventoryContext = null
): Collection {
    $query = $this->dialogue()
        ->where('line_type', $lineType)
        ->where('interaction_bucket', $interactionBucket);

    if ($inventoryContext !== null) {
        $query->where(function ($q) use ($inventoryContext) {
            $q->whereNull('inventory_context')
              ->orWhere('inventory_context', $inventoryContext);
        });
    } else {
        $query->whereNull('inventory_context');
    }

    return $query->orderBy('id')->get();
}
```

**Key differences from current code**:
- Does NOT use random selection within query
- Returns all matching lines (sorted by id)
- Deterministic ordering (by id, not rand)
- Supports inventory context fallback (null matches any)

---

## Part 6: Deterministic Dialogue Selection

### 🔴 MISSING: Deterministic Line Picker

**Replace** current weighted random with deterministic selection (Section 6):

```php
// Add to VendorDialogueService

private function selectDeterministicLine(
    Collection $lines,
    int $playerId,
    int $vendorId,
    int $interactionCount,
    string $lineType
): ?VendorDialogue {
    if ($lines->isEmpty()) {
        return null;
    }

    // Create deterministic seed from player + vendor + interaction + line type
    $seed = crc32($playerId . ':' . $vendorId . ':' . $interactionCount . ':' . $lineType);
    $index = $seed % $lines->count();

    return $lines->values()->get($index);
}
```

**Why**: Ensures same player gets same dialogue from same vendor at same bucket level (reproducible, fair, no unfair RNG)

---

## Part 7: Fallback Dialogue System

### 🔴 MISSING: Multi-Tier Fallback Logic

**Required** (Section 7):

```php
// Add to VendorDialogueService

private const STATIC_FALLBACKS = [
    'salvage_yard' => [
        'greeting' => 'Welcome. See anything worth salvaging?',
        'inventory_pitch' => 'Got some quality merchandise in stock.',
        'deal_accepted' => 'Not a bad deal.',
        'deal_rejected' => 'Your loss.',
        'farewell' => 'Try not to get yourself spaced.',
    ],
    'shipyard' => [
        'greeting' => 'Looking for a new hull?',
        'inventory_pitch' => 'This vessel is built to last.',
        'deal_accepted' => 'Fine choice. You won\'t regret it.',
        'deal_rejected' => 'Come back when you\'re ready.',
        'farewell' => 'Come back when you want something faster.',
    ],
    'trading_hub' => [
        'greeting' => 'Looking to trade?',
        'inventory_pitch' => 'I have exactly what you need.',
        'deal_accepted' => 'Pleasure doing business.',
        'deal_rejected' => 'Your decision.',
        'farewell' => 'Safe travels.',
    ],
    'market' => [
        'greeting' => 'Take a look around.',
        'inventory_pitch' => 'Everything here is quality goods.',
        'deal_accepted' => 'Good choice.',
        'deal_rejected' => 'Suit yourself.',
        'farewell' => 'Come again.',
    ],
];

public function getDialogueWithFallback(
    VendorProfile $vendor,
    string $lineType,
    int $playerId,
    int $interactionCount
): string {
    $bucket = $this->mapInteractionBucket($interactionCount);

    // Tier 1: Same vendor, line type, repeat_customer bucket
    if ($bucket !== 'repeat_customer') {
        $fallbackLines = $vendor->getDialogueLines($lineType, 'repeat_customer');
        if (!$fallbackLines->isEmpty()) {
            return $this->selectDeterministicLine(
                $fallbackLines,
                $playerId,
                $vendor->id,
                4,  // Use 4 for repeat_customer tier
                $lineType
            )->line_text;
        }
    }

    // Tier 2: Same vendor, line type, any bucket
    $allLines = $vendor->getDialogueLines($lineType, $bucket);
    if (!$allLines->isEmpty()) {
        return $this->selectDeterministicLine(
            $allLines,
            $playerId,
            $vendor->id,
            $interactionCount,
            $lineType
        )->line_text;
    }

    // Tier 3: Static fallback by service type
    return $this->STATIC_FALLBACKS[$vendor->service_type][$lineType]
        ?? 'Hmph.';  // Ultimate fallback
}
```

**Fallback order**:
1. Same vendor + line type + repeat_customer bucket
2. Same vendor + line type + any bucket
3. Static fallback by service_type
4. Generic fallback

---

## Part 8: API Response Structure

### 🔴 MISSING: Dialogue Integration in API Responses

**Required format** (Section 8):

```php
// Example vendor interaction endpoint response

{
  "success": true,
  "data": {
    "vendor": {
      "uuid": "53568459-6bb0-4540-b032-918138ced0be",
      "service_type": "salvage_yard",
      "criminality": 0.06,
      "markup_base": 0.2152
    },
    "dialogue": {
      "greeting": "Back again? My inventory must be treating you well.",
      "inventory_pitch": "This shield projector still holds seventy-six percent integrity."
    }
  }
}
```

**Implementation**:
```php
// In VendorController or TradingController

$vendor = VendorProfile::find($vendorId);
$playerId = auth()->user()->id;
$playerInteractionCount = $vendor->playerInteractionCount($playerId);

return $this->success([
    'vendor' => [
        'uuid' => $vendor->uuid,
        'service_type' => $vendor->service_type,
        'criminality' => $vendor->criminality,
        'markup_base' => $vendor->markup_base,
    ],
    'dialogue' => [
        'greeting' => $this->dialogueService->getDialogueWithFallback(
            $vendor, 'greeting', $playerId, $playerInteractionCount
        ),
        'inventory_pitch' => $this->dialogueService->getDialogueWithFallback(
            $vendor, 'inventory_pitch', $playerId, $playerInteractionCount
        ),
    ],
]);
```

---

## Part 9: Personality Change Handling

### 🔴 MISSING: Stale Dialogue Detection

**Required** (Section 9):

When vendor personality changes, mark dialogue stale:

```php
// Add to VendorProfile model or controller

public function updatePersonality(array $newPersonality): void
{
    $this->update([
        'personality' => $newPersonality,
        'dialogue_generation_status' => 'pending',
        'dialogue_generation_version' => $this->dialogue_generation_version + 1,
        'dialogue_generated_at' => null,
    ]);

    // Go service will pick this up and regenerate
}
```

**Triggers for regeneration**:
- `personality` changed
- `criminality` changed
- `markup_base` changed
- Manual regeneration requested

---

## Part 10: Admin Endpoints

### 🔴 MISSING: Three Admin Routes

**Required** (Section 10):

```php
// routes/api.php or admin routes

// 1. List vendors needing dialogue
Route::get('/admin/vendors/dialogue/pending', [AdminVendorController::class, 'pendingDialogue']);

// 2. Force regeneration
Route::post('/admin/vendors/{uuid}/dialogue/regenerate', [AdminVendorController::class, 'regenerateDialogue']);

// 3. Inspect dialogue
Route::get('/admin/vendors/{uuid}/dialogue', [AdminVendorController::class, 'inspectDialogue']);
```

**Implementation examples**:

```php
class AdminVendorController extends Controller
{
    // 1. List pending vendors
    public function pendingDialogue()
    {
        $vendors = VendorProfile::whereIn('dialogue_generation_status', ['pending', 'failed'])
            ->get();

        return $this->success([
            'count' => $vendors->count(),
            'vendors' => $vendors->map(fn($v) => [
                'uuid' => $v->uuid,
                'service_type' => $v->service_type,
                'status' => $v->dialogue_generation_status,
                'version' => $v->dialogue_generation_version,
            ]),
        ]);
    }

    // 2. Force regeneration
    public function regenerateDialogue(string $uuid)
    {
        $vendor = VendorProfile::findByUuid($uuid);

        $vendor->update([
            'dialogue_generation_status' => 'pending',
            'dialogue_generation_version' => $vendor->dialogue_generation_version + 1,
            'dialogue_generated_at' => null,
        ]);

        return $this->success(['message' => 'Regeneration scheduled']);
    }

    // 3. Inspect dialogue
    public function inspectDialogue(string $uuid)
    {
        $vendor = VendorProfile::findByUuid($uuid);

        $dialogue = $vendor->dialogue()
            ->orderBy('line_type')
            ->orderBy('interaction_bucket')
            ->get()
            ->groupBy('line_type');

        return $this->success([
            'vendor' => [
                'uuid' => $vendor->uuid,
                'service_type' => $vendor->service_type,
                'status' => $vendor->dialogue_generation_status,
            ],
            'dialogue' => $dialogue,
        ]);
    }
}
```

---

## Part 11: Service Architecture

### 🔴 MISSING: Dialogue Service Class

**Recommended**: Create `VendorDialogueService`

```php
// app/Services/VendorDialogueService.php

class VendorDialogueService
{
    public function getDialogueWithFallback(
        VendorProfile $vendor,
        string $lineType,
        int $playerId,
        int $interactionCount
    ): string {
        // Implementation from Part 7
    }

    private function mapInteractionBucket(int $count): string {
        // Implementation from Part 4
    }

    private function selectDeterministicLine(
        Collection $lines,
        int $playerId,
        int $vendorId,
        int $interactionCount,
        string $lineType
    ): ?VendorDialogue {
        // Implementation from Part 6
    }

    // ... other methods
}
```

---

## Summary Table: What Needs to be Done

| Component | Status | Effort | Priority |
|-----------|--------|--------|----------|
| vendor_dialogue table | ✅ Done | — | — |
| VendorDialogue model | ✅ Done | — | — |
| Add 3 fields to vendor_profiles | ❌ Missing | 30min | CRITICAL |
| Update VendorProfile model | ❌ Missing | 1-2 hours | CRITICAL |
| Vendor creation initialization | ❌ Missing | 1-2 hours | CRITICAL |
| Interaction bucket mapping | ❌ Missing | 30min | HIGH |
| Dialogue retrieval logic | ❌ Missing | 1-2 hours | HIGH |
| Deterministic line selection | ❌ Missing | 1 hour | HIGH |
| Fallback dialogue system | ❌ Missing | 2-3 hours | HIGH |
| API response integration | ❌ Missing | 2-3 hours | HIGH |
| Personality change handling | ❌ Missing | 1 hour | HIGH |
| Admin endpoints | ❌ Missing | 2-3 hours | MEDIUM |
| Dialogue service class | ❌ Missing | 3-4 hours | MEDIUM |
| Migration to drop dialogue_pool | ❌ Missing | 30min | LOW (deferred) |

**Total Effort**: ~17-23 hours (2-3 developer days)

---

## Implementation Roadmap

### Day 1: Foundations
1. Create migration for 3 vendor_profiles fields (30min)
2. Update VendorProfile model with fields + relationship (1hr)
3. Update vendor creation in seeder/factory (1-2hrs)
4. Create VendorDialogueService skeleton (30min)

### Day 2: Core Logic
5. Implement interaction bucket mapping (30min)
6. Implement dialogue retrieval logic (1-2hrs)
7. Implement deterministic line selection (1hr)
8. Implement fallback dialogue system (2-3hrs)

### Day 3: Integration + Admin
9. Integrate dialogue into API responses (2-3hrs)
10. Implement personality change handling (1hr)
11. Create admin endpoints (2-3hrs)
12. Testing and refinement (2-3hrs)

---

## What This Design Does NOT Require

- ❌ Go service (separate project)
- ❌ LLM integration (in PHP)
- ❌ Prompt generation
- ❌ Random dialogue generation
- ❌ Dynamic text manipulation

**These are all responsibility of the Go service**, which reads vendor_profiles and writes to vendor_dialogue.

---

## Key Architectural Points

### Separation of Concerns
- **PHP backend**: Manages vendors, selects dialogue, provides fallbacks
- **Go service**: Generates dialogue asynchronously using LLM
- **No coupling**: PHP doesn't know about Go; Go doesn't need PHP at runtime

### Dialogue Generation Status
- `pending` → needs generation (Go service picks up)
- `generating` → Go service is working
- `complete` → dialogue ready, PHP serves it
- `failed` → Go service had trouble, try again

### Deterministic, Not Random
- Same player + same vendor + same interaction bucket = same dialogue
- No ORDER BY RAND()
- CRC32 seed for reproducibility
- Prevents unfair RNG scenarios

### Fallbacks
- Generated dialogue preferred
- Multi-tier fallback ensures something always returned
- Static fallbacks by service_type as last resort

---

## Critical Success Factors

1. ✅ vendor_dialogue table structure is correct
2. ✅ VendorDialogue model is correct
3. ❌ **Must add status fields to vendor_profiles**
4. ❌ **Must implement deterministic selection (not random)**
5. ❌ **Must implement fallback chain**
6. ❌ **Must initialize fields when creating vendors**
7. ✅ Can defer dropping dialogue_pool until Go service is live

---

## Next Steps

1. **Run migration** to add 3 status fields to vendor_profiles
2. **Update VendorProfile model** with new fields and relationship
3. **Create VendorDialogueService** with all required methods
4. **Update vendor creation** (seeder/factory/API) to initialize status fields
5. **Create admin endpoints** for monitoring
6. **Test** with mock data before Go service integration
