# Vendor Profile Dialogue System -- PHP API Backend Changes

## Overview

The PHP API backend must transition from storing dialogue in
`vendor_profiles.dialogue_pool` to using a dedicated `vendor_dialogue`
table. Dialogue generation will be performed by a separate Go service
using a local LLM. The PHP API will only read dialogue and serve it to
players during gameplay.

The responsibilities of the PHP API backend are:

1.  Own vendor profile data
2.  Retrieve dialogue for gameplay interactions
3.  Track dialogue generation status
4.  Provide deterministic dialogue selection
5.  Provide fallback dialogue if generated dialogue is missing

The PHP API must never generate dialogue itself.

------------------------------------------------------------------------

# 1. Database Changes

## 1.1 Drop `dialogue_pool`

Once the new dialogue system is in place, remove the old column.

Laravel migration:

``` php
Schema::table('vendor_profiles', function (Blueprint $table) {
    $table->dropColumn('dialogue_pool');
});
```

------------------------------------------------------------------------

## 1.2 Add Dialogue Generation Tracking Fields

These fields allow the generator service to know which vendors require
dialogue generation.

``` php
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

------------------------------------------------------------------------

## 1.3 Create `vendor_dialogue` Table

``` php
Schema::create('vendor_dialogue', function (Blueprint $table) {
    $table->id();

    $table->foreignId('vendor_profile_id')
        ->constrained('vendor_profiles')
        ->cascadeOnDelete();

    $table->enum('line_type', [
        'greeting',
        'inventory_pitch',
        'deal_accepted',
        'deal_rejected',
        'farewell',
    ]);

    $table->enum('interaction_bucket', [
        'first_visit',
        'second_visit',
        'third_visit',
        'repeat_customer',
    ]);

    $table->string('inventory_context', 50)->nullable();
    $table->string('line_text', 255);
    $table->decimal('weight', 5, 4)->default(1.0000);
    $table->unsignedInteger('generation_version')->default(1);

    $table->timestamps();

    $table->index(['vendor_profile_id', 'line_type']);
    $table->index(['vendor_profile_id', 'line_type', 'interaction_bucket'], 'vendor_dialogue_lookup_idx');
    $table->index(['vendor_profile_id', 'inventory_context']);
});
```

------------------------------------------------------------------------

# 2. Laravel Models

## VendorProfile Model

``` php
class VendorProfile extends Model
{
    protected $table = 'vendor_profiles';

    protected $casts = [
        'personality' => 'array',
        'criminality' => 'float',
        'markup_base' => 'float',
        'dialogue_generated_at' => 'datetime',
    ];

    public function dialogue()
    {
        return $this->hasMany(VendorDialogue::class, 'vendor_profile_id');
    }
}
```

------------------------------------------------------------------------

## VendorDialogue Model

``` php
class VendorDialogue extends Model
{
    protected $table = 'vendor_dialogue';

    protected $casts = [
        'weight' => 'float',
        'generation_version' => 'int',
    ];

    public function vendorProfile()
    {
        return $this->belongsTo(VendorProfile::class, 'vendor_profile_id');
    }
}
```

------------------------------------------------------------------------

# 3. Vendor Creation Behaviour

When the API creates a new vendor profile it must set dialogue
generation fields.

``` php
$vendor = VendorProfile::create([
    'uuid' => (string) Str::uuid(),
    'galaxy_id' => $galaxyId,
    'poi_id' => $poiId,
    'trading_post_id' => $tradingPostId,
    'service_type' => $serviceType,
    'criminality' => $criminality,
    'personality' => $personality,
    'markup_base' => $markupBase,
    'dialogue_generation_status' => 'pending',
    'dialogue_generation_version' => 1,
    'dialogue_generated_at' => null,
]);
```

This signals the Go generator to create dialogue for the vendor.

------------------------------------------------------------------------

# 4. Interaction Bucket Logic

Interaction counts must be mapped into buckets.

``` php
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

------------------------------------------------------------------------

# 5. Dialogue Retrieval

Example retrieval query:

``` php
$query = VendorDialogue::query()
    ->where('vendor_profile_id', $vendor->id)
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

$lines = $query->orderBy('id')->get();
```

------------------------------------------------------------------------

# 6. Deterministic Dialogue Selection

The API must not use `ORDER BY RAND()`.

``` php
private function selectDeterministicLine(Collection $lines, int $playerId, int $vendorId, int $interactionCount, string $lineType): ?VendorDialogue
{
    if ($lines->isEmpty()) {
        return null;
    }

    $seed = crc32($playerId . ':' . $vendorId . ':' . $interactionCount . ':' . $lineType);
    $index = $seed % $lines->count();

    return $lines->values()->get($index);
}
```

------------------------------------------------------------------------

# 7. Fallback Dialogue

If generated dialogue does not exist, fallback dialogue must be used.

Example configuration:

``` php
private const STATIC_FALLBACKS = [
    'salvage_yard' => [
        'greeting' => 'Welcome. See anything worth salvaging?',
        'farewell' => 'Try not to get yourself spaced.',
    ],
    'shipyard' => [
        'greeting' => 'Looking for a new hull?',
        'farewell' => 'Come back when you want something faster.',
    ],
    'trading_hub' => [
        'greeting' => 'Looking to trade?',
        'farewell' => 'Safe travels.',
    ],
    'market' => [
        'greeting' => 'Take a look around.',
        'farewell' => 'Come again.',
    ],
];
```

Fallback order:

1.  Same vendor and line type using `repeat_customer`
2.  Same vendor and line type any bucket
3.  Static fallback by service type

------------------------------------------------------------------------

# 8. API Response Structure

Example vendor interaction response:

``` json
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

------------------------------------------------------------------------

# 9. Personality Change Handling

If vendor personality fields change, dialogue must be marked stale.

``` php
$vendor->update([
    'personality' => $newPersonality,
    'dialogue_generation_status' => 'pending',
    'dialogue_generation_version' => $vendor->dialogue_generation_version + 1,
    'dialogue_generated_at' => null,
]);
```

The Go generator will regenerate dialogue later.

------------------------------------------------------------------------

# 10. Admin Endpoints

Recommended internal endpoints:

### List vendors needing dialogue

    GET /api/admin/vendors/dialogue/pending

### Force regeneration

    POST /api/admin/vendors/{uuid}/dialogue/regenerate

### Inspect dialogue

    GET /api/admin/vendors/{uuid}/dialogue

------------------------------------------------------------------------

# 11. Final Responsibility Separation

The PHP API backend must:

-   manage vendor profiles
-   track dialogue generation status
-   retrieve dialogue rows
-   select deterministic dialogue lines
-   provide fallback dialogue
-   return dialogue in gameplay responses

The PHP backend must **not**:

-   call the LLM
-   generate dialogue
-   construct prompts
-   mutate dialogue text dynamically

Dialogue generation is the responsibility of the Go generator service.
