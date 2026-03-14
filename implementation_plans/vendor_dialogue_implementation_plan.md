# Vendor Dialogue System — PHP Implementation Plan
## Source: docs/design/vendor_dialogue_phased_architecture.md

---

## Phase Completion Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Foundational storage and retrieval | ✅ Complete (2026-03-13) |
| 2 | Go dialogue generator + PHP support layer | Not started — see `vendor_dialogue_phase2_php_implementation_plan.md` |
| 3 | Interaction verbs | Not started — blocked on Phase 2 |
| 4 | Runtime item fact composition | Not started — blocked on Phase 3 |
| 5 | Relationship drift | Not started — blocked on Phase 4 |
| 6 | Generalised NPC framework | Partially done (vendor pool pattern complete) |

---

## Phase 1 — Remaining Work

### What's already done
- `vendor_dialogue` table with all required columns including `transaction_context`, `inventory_context NOT NULL DEFAULT 'none'`
- `galaxy_vendor_profiles` with generation status tracking fields
- `VendorDialogue` model with enum casts
- `VendorDialogueService` with deterministic selector and §10.5 fallback chain
- `DialogueLineType`, `InteractionBucket`, `TransactionContext`, `InventoryContext` enums
- Static fallbacks by service type

### 1.1 Remove `dialogue_pool`

#### `database/migrations/2026_03_13_000001_remove_dialogue_pool_from_vendor_profiles.php`
```
up():
    Schema::table('vendor_profiles', function (Blueprint $table) {
        $table->dropColumn('dialogue_pool');
    });

    // Also drop from trading_posts if still there
    Schema::table('trading_posts', function (Blueprint $table) {
        $table->dropColumn('dialogue_pool');
    });

down():
    re-add dialogue_pool as nullable JSON column on both tables
```

#### `app/Models/VendorProfile.php`
- Remove `'dialogue_pool'` from `$fillable`
- Remove `'dialogue_pool' => 'array'` from `$casts`
- Remove `getDialogue(string $context): string` method (reads from dialogue_pool)
- Remove `getDefaultDialogue(string $context): string` method

#### `app/Models/GalaxyVendorProfile.php`
- Remove `getDialogue(string $context): string` method (reads from vendorProfile->dialogue_pool)
- Remove `getDefaultDialogue(string $context): string` method

#### `app/Models/TradingPost.php`
- Remove `'dialogue_pool'` from `$fillable`
- Remove `'dialogue_pool' => 'array'` from `$casts`
- Remove any `getDialogue()` method

---

### 1.2 Auto-mark dialogue stale on GalaxyVendorProfile field changes

#### `app/Observers/GalaxyVendorProfileObserver.php` — NEW FILE

Watched fields: `service_type`, `criminality`, `personality`, `markup_base`

```
class GalaxyVendorProfileObserver
{
    private const WATCHED_FIELDS = ['service_type', 'criminality', 'personality', 'markup_base'];

    public function updating(GalaxyVendorProfile $vendor): void
        foreach WATCHED_FIELDS as $field
            if vendor->isDirty($field)
                vendor->dialogue_generation_status = 'pending'
                vendor->dialogue_generation_version = vendor->dialogue_generation_version + 1
                vendor->dialogue_generated_at = null
                break  // one dirty field is enough to trigger
}
```

#### `app/Providers/AppServiceProvider.php` — MODIFY
```
public function boot(): void
    // add:
    GalaxyVendorProfile::observe(GalaxyVendorProfileObserver::class);
```

---

### 1.3 Gameplay-facing vendor dialogue API endpoint

#### `app/Http/Controllers/Api/VendorDialogueController.php` — NEW FILE

```
class VendorDialogueController extends BaseApiController
{
    public function __construct(
        private VendorDialogueService $dialogueService
    ) {}

    /**
     * GET /api/vendors/{vendorUuid}/dialogue
     * Returns greeting and available dialogue for a player visiting a vendor.
     */
    public function index(Request $request, string $vendorUuid): JsonResponse
        player = resolvePlayer(request)
        vendor = GalaxyVendorProfile::findByUuid(vendorUuid) or 404

        relationship = PlayerVendorRelationship::firstOrCreate(
            [player_id, vendor_profile_id],
            [defaults]
        )

        bucket = dialogueService->mapInteractionBucket(relationship->visit_count)
        context = TransactionContext::NEUTRAL

        greeting = dialogueService->getDialogueWithFallback(
            vendor, 'greeting', player->id, relationship->visit_count, context
        )

        return success([
            'vendor' => [
                'uuid' => vendor->uuid,
                'service_type' => vendor->service_type,
                'criminality' => vendor->criminality,
            ],
            'dialogue' => [
                'greeting' => greeting,
            ],
            'interaction_bucket' => bucket,
            'dialogue_status' => vendor->dialogue_generation_status,
        ])
}
```

#### `routes/api.php` — MODIFY
```
// Add under authenticated routes:
Route::get('/vendors/{vendorUuid}/dialogue', [VendorDialogueController::class, 'index']);
```

---

## Phase 3 — Interaction Verbs

### 3.1 Interaction verb definitions

#### `app/Enums/Vendor/InteractionVerb.php` — NEW FILE

```
enum InteractionVerb: string
{
    // Always available
    case BROWSE_INVENTORY = 'browse_inventory'
    case ASK_PRICE = 'ask_price'
    case MAKE_OFFER = 'make_offer'
    case ACCEPT_OFFER = 'accept_offer'
    case REJECT_OFFER = 'reject_offer'
    case LEAVE = 'leave'

    // Role-gated
    case INSPECT_QUALITY = 'inspect_quality'        // requires chief_engineer
    case INSPECT_DEFECTS = 'inspect_defects'        // requires chief_engineer
    case INSPECT_COMPATIBILITY = 'inspect_compatibility' // requires chief_engineer
    case ASK_SPECS = 'ask_specs'                    // requires science_officer or chief_engineer
    case CHALLENGE_CLAIM = 'challenge_claim'        // requires science_officer or tactical_officer

    // Relationship-gated
    case REQUEST_DISCOUNT = 'request_discount'      // requires familiarity >= 0.6
    case ASK_BLACK_MARKET = 'ask_black_market'      // requires trust >= 0.7, vendor criminality >= 0.5

    public function label(): string
        return match($this) { ... }

    public function requiredCrewRole(): ?array
        // Returns list of CrewRole values that unlock this verb
        // null means no role requirement
        return match($this) {
            self::INSPECT_QUALITY => ['chief_engineer'],
            self::INSPECT_DEFECTS => ['chief_engineer'],
            self::INSPECT_COMPATIBILITY => ['chief_engineer'],
            self::ASK_SPECS => ['science_officer', 'chief_engineer'],
            self::CHALLENGE_CLAIM => ['science_officer', 'tactical_officer'],
            default => null,
        }

    public function isAlwaysAvailable(): bool
        return in_array($this, [
            self::BROWSE_INVENTORY,
            self::ASK_PRICE,
            self::MAKE_OFFER,
            self::ACCEPT_OFFER,
            self::REJECT_OFFER,
            self::LEAVE,
        ])

    public function requiredRelationshipField(): ?string
        // Returns the relationship field name and minimum value
        return match($this) {
            self::REQUEST_DISCOUNT => 'familiarity',
            self::ASK_BLACK_MARKET => 'trust',
            default => null,
        }

    public function requiredRelationshipThreshold(): float
        return match($this) {
            self::REQUEST_DISCOUNT => 0.6,
            self::ASK_BLACK_MARKET => 0.7,
            default => 0.0,
        }

    public function mapsToLineType(): string
        // What dialogue line type should the vendor respond with
        return match($this) {
            self::BROWSE_INVENTORY => 'greeting',
            self::ASK_PRICE, self::INSPECT_QUALITY, self::ASK_SPECS => 'inventory_pitch',
            self::ACCEPT_OFFER => 'deal_accepted',
            self::REJECT_OFFER, self::CHALLENGE_CLAIM => 'deal_rejected',
            self::LEAVE => 'farewell',
            default => 'inventory_pitch',
        }

    public function mapsToTransactionContext(): TransactionContext
        return match($this) {
            self::MAKE_OFFER, self::ACCEPT_OFFER, self::REJECT_OFFER => TransactionContext::VENDOR_SELLING,
            default => TransactionContext::NEUTRAL,
        }
}
```

---

### 3.2 Option gating service

#### `app/Services/Vendor/VendorInteractionOptionService.php` — NEW FILE

```
class VendorInteractionOptionService
{
    public function getAvailableVerbs(
        GalaxyVendorProfile $vendor,
        Player $player,
        ?string $focusedInventoryContext = null  // what item category is being looked at
    ): array
        available = []
        relationship = PlayerVendorRelationship::firstOrCreate(...)
        crewRoles = player->activeShip?->crew->pluck('role.value')->toArray() ?? []

        foreach InteractionVerb::cases() as $verb
            // Skip if locked out and verb is not LEAVE
            if relationship->is_locked_out && verb !== LEAVE
                continue

            // Always-available verbs
            if verb->isAlwaysAvailable()
                available[] = buildVerbOption(verb, reason: null)
                continue

            // Role-gated verbs
            if requiredRoles = verb->requiredCrewRole()
                if array_intersect(requiredRoles, crewRoles) is not empty
                    available[] = buildVerbOption(verb, reason: null)
                continue

            // Relationship-gated verbs
            if field = verb->requiredRelationshipField()
                threshold = verb->requiredRelationshipThreshold()
                relationshipValue = relationship->{$field} ?? 0.0
                if relationshipValue >= threshold
                    available[] = buildVerbOption(verb, reason: null)
                continue

        return available

    private function buildVerbOption(InteractionVerb $verb, ?string $reason): array
        return [
            'verb' => verb->value,
            'label' => verb->label(),
            'reason' => reason,  // null = available, string = why unavailable (for future locked-but-visible options)
        ]
}
```

---

### 3.3 Vendor interaction controller

#### `app/Http/Controllers/Api/VendorInteractionController.php` — NEW FILE

```
class VendorInteractionController extends BaseApiController
{
    public function __construct(
        private VendorDialogueService $dialogueService,
        private VendorInteractionOptionService $optionService,
    ) {}

    /**
     * POST /api/vendors/{vendorUuid}/enter
     * Player approaches vendor. Returns greeting + available verbs.
     */
    public function enter(Request $request, string $vendorUuid): JsonResponse
        player = resolvePlayer(request)
        vendor = GalaxyVendorProfile::findByUuid(vendorUuid) or 404
        relationship = getOrCreateRelationship(player, vendor)

        // Increment visit count
        relationship->increment('visit_count')
        relationship->update(['last_interaction_at' => now()])

        greeting = dialogueService->getDialogueWithFallback(
            vendor,
            'greeting',
            player->id,
            relationship->visit_count,
            TransactionContext::NEUTRAL,
            InventoryContext::NONE,
        )

        options = optionService->getAvailableVerbs(vendor, player)

        return success([
            'dialogue' => ['greeting' => greeting],
            'options' => options,
        ])

    /**
     * POST /api/vendors/{vendorUuid}/interact
     * Player executes an interaction verb.
     * Body: { verb: string, inventory_context?: string, item_uuid?: string }
     */
    public function interact(Request $request, string $vendorUuid): JsonResponse
        validated = request->validate([
            'verb' => 'required|string',
            'inventory_context' => 'nullable|string',
            'item_uuid' => 'nullable|uuid',
        ])

        player = resolvePlayer(request)
        vendor = GalaxyVendorProfile::findByUuid(vendorUuid) or 404
        relationship = getOrCreateRelationship(player, vendor)

        verb = InteractionVerb::from(validated['verb']) or 422

        // Validate verb is actually available
        availableVerbs = optionService->getAvailableVerbs(vendor, player)
        if verb not in availableVerbs
            return error('Verb not available', 403)

        invContext = InventoryContext::tryFrom(validated['inventory_context'] ?? 'none') ?? InventoryContext::NONE
        transContext = verb->mapsToTransactionContext()
        lineType = verb->mapsToLineType()

        response = dialogueService->getDialogueWithFallback(
            vendor,
            lineType,
            player->id,
            relationship->visit_count,
            transContext,
            invContext,
        )

        // Get updated options after this interaction
        options = optionService->getAvailableVerbs(vendor, player, invContext->value)

        return success([
            'verb' => verb->value,
            'dialogue' => ['response' => response],
            'options' => options,
        ])
}
```

#### `routes/api.php` — MODIFY
```
Route::post('/vendors/{vendorUuid}/enter', [VendorInteractionController::class, 'enter']);
Route::post('/vendors/{vendorUuid}/interact', [VendorInteractionController::class, 'interact']);
```

---

## Phase 4 — Runtime Item Fact Composition

### 4.1 Item fact clause builder

#### `app/Services/Vendor/ItemFactClauseBuilder.php` — NEW FILE

```
class ItemFactClauseBuilder
{
    // Condition percent ranges to descriptive phrases
    private const CONDITION_CLAUSES = [
        [90, 100, ['Pristine condition.', 'Excellent shape.', 'Like new.']],
        [70,  89, ['Good working order.', 'Solid condition.', 'Minor wear, fully functional.']],
        [50,  69, ['Moderate wear.', 'Some signs of use.', 'Works, but not mint.']],
        [30,  49, ['Heavy wear.', 'Visibly used.', 'Functional but beaten up.']],
        [ 0,  29, ['Poor condition.', 'Barely holding together.', 'Needs work.']],
    ]

    private const RARITY_CLAUSES = [
        'common'    => ['Standard issue.', 'Common spec.', 'Nothing special, but reliable.'],
        'uncommon'  => ['Decent find.', 'Not easy to come by.', 'Above average spec.'],
        'rare'      => ['Hard to source.', 'Rare configuration.', 'Not many of these around.'],
        'legendary' => ['One of a kind.', 'You won\'t find another.', 'Exceptional provenance.'],
    ]

    private const LEGALITY_CLAUSES = [
        'contraband' => ['Hot goods. You didn\'t hear it from me.', 'Officially, this doesn\'t exist.'],
        'restricted' => ['Technically licensed goods.', 'Requires the right paperwork.'],
    ]

    private const PROVENANCE_CLAUSES = [
        'military_surplus' => ['Military surplus.', 'Ex-navy spec.', 'Decommissioned service unit.'],
        'salvage'          => ['Salvage origin.', 'Pulled from a wreck.', 'Field-stripped from a derelict.'],
        'corporate'        => ['Corporate manufacture.', 'Factory direct.', 'OEM build.'],
    ]

    /**
     * Build a condition clause from a percent value.
     * Returns deterministic phrase using $seed.
     */
    public function buildConditionClause(int $conditionPercent, int $seed): ?string
        foreach CONDITION_CLAUSES as [$min, $max, $phrases]
            if conditionPercent >= min && conditionPercent <= max
                index = abs($seed) % count(phrases)
                return phrases[index]
        return null

    /**
     * Build a rarity clause.
     */
    public function buildRarityClause(string $rarity, int $seed): ?string
        phrases = RARITY_CLAUSES[rarity] ?? null
        if !phrases return null
        index = abs($seed + 1) % count(phrases)
        return phrases[index]

    /**
     * Build a defect clause from an array of defect tags.
     * Returns the most significant defect phrase or null.
     */
    public function buildDefectClause(array $defects, int $seed): ?string
        if empty(defects) return null
        // Pick one defect deterministically
        index = abs($seed + 2) % count(defects)
        defect = defects[index]
        // Map defect tag to human phrase
        return defectTagToPhrase(defect)  // internal lookup map

    /**
     * Build a legality clause.
     */
    public function buildLegalityClause(?string $legalityStatus, int $seed): ?string
        if !legalityStatus || legalityStatus === 'legal' return null
        phrases = LEGALITY_CLAUSES[legalityStatus] ?? null
        if !phrases return null
        index = abs($seed + 3) % count(phrases)
        return phrases[index]

    /**
     * Build a provenance clause.
     */
    public function buildProvenanceClause(?string $provenance, int $seed): ?string
        if !provenance return null
        phrases = PROVENANCE_CLAUSES[provenance] ?? null
        if !phrases return null
        index = abs($seed + 4) % count(phrases)
        return phrases[index]

    /**
     * Build all applicable clauses for a set of item facts.
     * Returns an ordered array of clause strings, most specific first.
     */
    public function buildClausesForItem(array $itemFacts, int $seed): array
        clauses = []

        if isset(itemFacts['condition_percent'])
            clause = buildConditionClause(itemFacts['condition_percent'], seed)
            if clause clauses[] = clause

        if isset(itemFacts['defects']) && !empty(itemFacts['defects'])
            clause = buildDefectClause(itemFacts['defects'], seed)
            if clause clauses[] = clause

        if isset(itemFacts['legality'])
            clause = buildLegalityClause(itemFacts['legality'], seed)
            if clause clauses[] = clause

        if isset(itemFacts['provenance'])
            clause = buildProvenanceClause(itemFacts['provenance'], seed)
            if clause clauses[] = clause

        if isset(itemFacts['rarity']) && empty(clauses)
            // Only add rarity if nothing more specific was added
            clause = buildRarityClause(itemFacts['rarity'], seed)
            if clause clauses[] = clause

        return clauses
}
```

---

### 4.2 Dialogue composition service

#### `app/Services/Vendor/DialogueCompositionService.php` — NEW FILE

```
class DialogueCompositionService
{
    public function __construct(
        private ItemFactClauseBuilder $clauseBuilder,
    ) {}

    /**
     * Compose a final dialogue string from a stored voice line and live item facts.
     *
     * Strategy:
     * - vendor_selling: voice line + condition/defect clause
     * - vendor_buying:  skeptical voice line + defect clause (focus on negatives)
     * - neutral:        voice line only, or with rarity clause if available
     */
    public function compose(
        string $voiceLine,
        array $itemFacts,
        TransactionContext $transactionContext,
        int $seed,
    ): string
        if empty(itemFacts)
            return voiceLine

        clauses = clauseBuilder->buildClausesForItem(itemFacts, seed)

        if empty(clauses)
            return voiceLine

        // For vendor_buying, prefer defect/condition clauses (skeptical framing)
        // For vendor_selling, prefer condition/provenance (positive framing)
        // Both use the same clause pool but the stored voice line already sets tone

        // Pick first clause (most significant)
        primaryClause = clauses[0]

        // Compose: voice + clause, ensuring clean punctuation
        voiceNoPeriod = rtrim(voiceLine, '.')
        return "{$voiceNoPeriod}. {$primaryClause}"

    /**
     * Full pipeline: retrieve voice line, compose with item facts, return final string.
     */
    public function getComposedDialogue(
        GalaxyVendorProfile $vendor,
        string $lineType,
        Player $player,
        int $interactionCount,
        TransactionContext $transactionContext,
        InventoryContext $inventoryContext,
        array $itemFacts = [],
    ): string
        // Get base voice line via existing fallback chain
        voiceLine = dialogueService->getDialogueWithFallback(
            vendor,
            lineType,
            player->id,
            interactionCount,
            transactionContext,
            inventoryContext,
        )

        if empty(itemFacts)
            return voiceLine

        seed = crc32("{$player->id}:{$vendor->id}:{$interactionCount}:{$lineType}")

        return compose(voiceLine, itemFacts, transactionContext, seed)
}
```

#### `app/Services/VendorDialogueService.php` — MODIFY
- Inject `DialogueCompositionService`
- Add `getComposedDialogue()` public method that delegates to `DialogueCompositionService`

---

## Phase 5 — Relationship Drift

### 5.1 Drift fields on PlayerVendorRelationship

#### `database/migrations/2026_03_13_000002_add_drift_fields_to_player_vendor_relationships.php` — NEW FILE
```
up():
    Schema::table('player_vendor_relationships', function (Blueprint $table) {
        $table->decimal('trust', 4, 3)->default(0.500)->after('markup_modifier')
        $table->decimal('respect', 4, 3)->default(0.500)->after('trust')
        $table->decimal('resentment', 4, 3)->default(0.000)->after('respect')
        $table->decimal('caution', 4, 3)->default(0.500)->after('resentment')
        $table->decimal('familiarity', 4, 3)->default(0.000)->after('caution')
        $table->decimal('perceived_competence', 4, 3)->default(0.500)->after('familiarity')
        $table->decimal('willingness_to_discount', 4, 3)->default(0.500)->after('perceived_competence')
    })

down():
    Schema::table('player_vendor_relationships', function (Blueprint $table) {
        $table->dropColumn([
            'trust', 'respect', 'resentment', 'caution',
            'familiarity', 'perceived_competence', 'willingness_to_discount',
        ])
    })
```

#### `app/Models/PlayerVendorRelationship.php` — MODIFY
```
// Add to $fillable:
'trust', 'respect', 'resentment', 'caution',
'familiarity', 'perceived_competence', 'willingness_to_discount',

// Add to $casts:
'trust' => 'decimal:3',
'respect' => 'decimal:3',
'resentment' => 'decimal:3',
'caution' => 'decimal:3',
'familiarity' => 'decimal:3',
'perceived_competence' => 'decimal:3',
'willingness_to_discount' => 'decimal:3',

// Add method:
public function getEffectiveTone(): string
    // trust+familiarity pulls toward warm
    // resentment pulls toward cold
    warmScore = (float) this->trust + (float) this->familiarity - (float) this->resentment
    if warmScore >= 1.2 return 'warm'
    if warmScore <= 0.3 return 'cold'
    return 'neutral'
```

---

### 5.2 Vendor drift service

#### `app/Services/Vendor/VendorDriftService.php` — NEW FILE

```
class VendorDriftService
{
    private const DECAY_FACTOR = 0.99;
    private const REGEN_THRESHOLD = 0.15;  // drift magnitude that triggers dialogue regen
    private const DRIFT_FIELDS = [
        'trust', 'respect', 'resentment', 'caution',
        'familiarity', 'perceived_competence', 'willingness_to_discount',
    ];

    /**
     * Delta table: InteractionVerb => outcome => trait => delta
     * Positive delta = increase that trait. All values small (0.01-0.05).
     */
    private const DELTA_TABLE = [
        'browse_inventory' => [
            'any' => ['familiarity' => +0.01],
        ],
        'inspect_quality' => [
            'defect_found' => ['respect' => +0.03, 'trust' => -0.02],
            'no_defect'    => ['respect' => +0.01],
        ],
        'challenge_claim' => [
            'correct'   => ['respect' => +0.04, 'trust' => -0.03, 'resentment' => +0.02],
            'incorrect' => ['respect' => -0.02, 'resentment' => +0.01],
        ],
        'make_offer' => [
            'lowball'   => ['resentment' => +0.03, 'trust' => -0.01, 'willingness_to_discount' => -0.02],
            'fair'      => ['trust' => +0.02, 'resentment' => -0.01],
            'generous'  => ['trust' => +0.03, 'willingness_to_discount' => +0.02],
        ],
        'accept_offer' => [
            'any' => ['trust' => +0.02, 'familiarity' => +0.02, 'willingness_to_discount' => +0.01],
        ],
        'reject_offer' => [
            'any' => ['resentment' => +0.01, 'caution' => +0.01],
        ],
        'request_discount' => [
            'granted'  => ['familiarity' => +0.02, 'willingness_to_discount' => -0.03],
            'declined' => ['caution' => +0.01],
        ],
    ]

    /**
     * Apply drift to a relationship record based on what just happened.
     *
     * $outcome is a string key into the delta table for that verb,
     * e.g. 'lowball', 'defect_found', 'any'
     */
    public function applyDrift(
        PlayerVendorRelationship $relationship,
        GalaxyVendorProfile $vendor,
        InteractionVerb $verb,
        string $outcome = 'any',
    ): void
        // Get delta vector for this verb+outcome
        delta = getDeltaVector(verb, outcome)

        if empty(delta) return

        updates = []
        foreach DRIFT_FIELDS as field
            current = (float) relationship->{$field}
            d = delta[field] ?? 0.0

            // Apply decay first, then delta
            decayed = current * DECAY_FACTOR
            newValue = clamp(decayed + d, 0.0, 1.0)
            updates[field] = round(newValue, 3)

        relationship->update(updates)

        // Check if dialogue should be regenerated based on accumulated drift
        checkRegenerationThreshold(vendor, relationship)

    /**
     * Get the delta vector for a verb+outcome combination.
     */
    private function getDeltaVector(InteractionVerb $verb, string $outcome): array
        verbTable = DELTA_TABLE[verb->value] ?? []
        if empty(verbTable) return []

        // Prefer specific outcome, fall back to 'any'
        return verbTable[outcome] ?? verbTable['any'] ?? []

    /**
     * Check if personality drift on the galaxy vendor profile has crossed
     * the regeneration threshold. If so, mark dialogue stale.
     *
     * This is based on cumulative relationship shifts across all players
     * (tracked separately on galaxy_vendor_profiles.personality_drift).
     *
     * NOTE: galaxy-level drift (personality_drift column) is a Phase 5 addition.
     * This method stub is a placeholder for that logic.
     */
    private function checkRegenerationThreshold(
        GalaxyVendorProfile $vendor,
        PlayerVendorRelationship $relationship,
    ): void
        // TODO Phase 5: accumulate per-player drift into galaxy-level drift
        // For now: if resentment or trust has drifted far from neutral,
        // mark this vendor instance stale so Go regenerates its lines
        if abs((float) relationship->resentment - 0.0) > REGEN_THRESHOLD
            || abs((float) relationship->trust - 0.5) > REGEN_THRESHOLD
            vendor->markDialogueStale()

    /**
     * Apply passive decay to all drift fields (call on a schedule or on next visit).
     * Slowly returns all traits toward their defaults without interaction.
     */
    public function applyPassiveDecay(PlayerVendorRelationship $relationship): void
        defaults = [
            'trust' => 0.500, 'respect' => 0.500, 'resentment' => 0.000,
            'caution' => 0.500, 'familiarity' => 0.000,
            'perceived_competence' => 0.500, 'willingness_to_discount' => 0.500,
        ]
        updates = []
        foreach DRIFT_FIELDS as field
            current = (float) relationship->{$field}
            default = defaults[field]
            // Nudge toward default
            newValue = current + ((default - current) * 0.005)
            updates[field] = round(clamp(newValue, 0.0, 1.0), 3)

        relationship->update(updates)
}
```

---

### 5.3 Wire drift into interaction controller

#### `app/Http/Controllers/Api/VendorInteractionController.php` — MODIFY

```
// Add VendorDriftService injection
// In interact() method, after resolving verb outcome, add:

driftService->applyDrift(relationship, vendor, verb, $outcome)
```

---

### 5.4 Relationship-aware dialogue selection

#### `app/Services/VendorDialogueService.php` — MODIFY

```
// Modify getDialogueWithFallback() to accept optional relationship tone:
public function getDialogueWithFallback(
    GalaxyVendorProfile $vendor,
    string $lineType,
    int $playerId,
    int $interactionCount,
    TransactionContext $transactionContext = TransactionContext::NEUTRAL,
    InventoryContext $inventoryContext = InventoryContext::NONE,
    string $relationshipTone = 'neutral',   // NEW: 'warm', 'neutral', 'cold'
): string

// In selectDeterministicLine(), incorporate tone into seed so warm/cold
// players get different lines from the same pool:
private function selectDeterministicLine(..., string $tone = 'neutral'): ?VendorDialogue
    seed = crc32("{$playerId}:{$vendorId}:{$interactionCount}:{$lineType}:{$tone}")
    index = abs(seed) % lines->count()
    return lines->values()->get(index)
```

---

## Phase 6 — Remaining PHP Work

### 6.1 Dialogue regeneration job

#### `app/Jobs/TriggerVendorDialogueRegenerationJob.php` — NEW FILE

```
class TriggerVendorDialogueRegenerationJob implements ShouldQueue
{
    public function __construct(
        private int $galaxyVendorProfileId,
    ) {}

    public function handle(): void
        vendor = GalaxyVendorProfile::find(this->galaxyVendorProfileId)
        if !vendor return

        // Ensure status is pending (the Go service polls for pending vendors)
        if vendor->dialogue_generation_status !== 'pending'
            vendor->update(['dialogue_generation_status' => 'pending'])

        // Optionally: HTTP call to Go service to wake it up
        // Http::post(config('services.go_dialogue_generator.url') . '/trigger', [
        //     'galaxy_vendor_profile_id' => vendor->id
        // ])
```

#### `app/Models/GalaxyVendorProfile.php` — MODIFY

```
// Update markDialogueStale() to also dispatch the regen job:
public function markDialogueStale(): void
    this->update([
        'dialogue_generation_status' => 'pending',
        'dialogue_generation_version' => this->dialogue_generation_version + 1,
        'dialogue_generated_at' => null,
    ])
    TriggerVendorDialogueRegenerationJob::dispatch(this->id)->delay(now()->addSeconds(5))
```

---

### 6.2 Dialogue generation status command

#### `app/Console/Commands/VendorDialogueStatusCommand.php` — NEW FILE

```
class VendorDialogueStatusCommand extends Command
{
    protected $signature = 'vendor:dialogue-status {galaxyUuid?}'
    protected $description = 'Show dialogue generation status for vendors in a galaxy'

    public function handle(): void
        query = GalaxyVendorProfile::with('vendorProfile')

        if galaxyUuid = this->argument('galaxyUuid')
            galaxy = Galaxy::findByUuid(galaxyUuid) or fail
            query->where('galaxy_id', galaxy->id)

        vendors = query->get()

        // Group by status and render table
        grouped = vendors->groupBy('dialogue_generation_status')

        this->table(
            ['Status', 'Count'],
            grouped->map(fn($group, $status) => [$status, count($group)])->values()
        )

        this->table(
            ['UUID', 'Service Type', 'Status', 'Version', 'Generated At'],
            vendors->map(fn($v) => [
                v->uuid, v->service_type, v->dialogue_generation_status,
                v->dialogue_generation_version, v->dialogue_generated_at,
            ])
        )
```

---

## Implementation Order

Follow the phase dependency chain from the design doc:

```
Phase 1 remaining  →  Phase 3  →  Phase 4  →  Phase 5  →  Phase 6
                   ↗
Phase 2 (Go)
```

### Suggested sprint order

**Sprint A** — Phase 1 cleanup (small, quick)
1. Migration to drop `dialogue_pool`
2. Clean `VendorProfile`, `GalaxyVendorProfile`, `TradingPost` models
3. `GalaxyVendorProfileObserver`
4. `VendorDialogueController` + route

**Sprint B** — Phase 3 (interaction verbs)
1. `InteractionVerb` enum
2. `VendorInteractionOptionService`
3. `VendorInteractionController` + routes

**Sprint C** — Phase 4 (item composition)
1. `ItemFactClauseBuilder`
2. `DialogueCompositionService`
3. Wire into `VendorDialogueService` and controller

**Sprint D** — Phase 5 (drift)
1. Migration for drift fields
2. Update `PlayerVendorRelationship`
3. `VendorDriftService`
4. Wire drift calls into `VendorInteractionController`
5. Update `VendorDialogueService` with tone-aware selection

**Sprint E** — Phase 6 cleanup
1. `TriggerVendorDialogueRegenerationJob`
2. Update `markDialogueStale()` to dispatch job
3. `VendorDialogueStatusCommand`

---

## Files Summary

| File | Action | Phase |
|------|--------|-------|
| `database/migrations/..._remove_dialogue_pool.php` | Create | 1 |
| `app/Models/VendorProfile.php` | Modify | 1 |
| `app/Models/GalaxyVendorProfile.php` | Modify | 1, 6 |
| `app/Models/TradingPost.php` | Modify | 1 |
| `app/Observers/GalaxyVendorProfileObserver.php` | Create | 1 |
| `app/Providers/AppServiceProvider.php` | Modify | 1 |
| `app/Http/Controllers/Api/VendorDialogueController.php` | Create | 1 |
| `routes/api.php` | Modify | 1, 3 |
| `app/Enums/Vendor/InteractionVerb.php` | Create | 3 |
| `app/Services/Vendor/VendorInteractionOptionService.php` | Create | 3 |
| `app/Http/Controllers/Api/VendorInteractionController.php` | Create | 3, 5 |
| `app/Services/Vendor/ItemFactClauseBuilder.php` | Create | 4 |
| `app/Services/Vendor/DialogueCompositionService.php` | Create | 4 |
| `app/Services/VendorDialogueService.php` | Modify | 4, 5 |
| `database/migrations/..._add_drift_fields.php` | Create | 5 |
| `app/Models/PlayerVendorRelationship.php` | Modify | 5 |
| `app/Services/Vendor/VendorDriftService.php` | Create | 5 |
| `app/Jobs/TriggerVendorDialogueRegenerationJob.php` | Create | 6 |
| `app/Console/Commands/VendorDialogueStatusCommand.php` | Create | 6 |
