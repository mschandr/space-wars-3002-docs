# Job Board Contract System - Implementation Guide

**Version:** 1.0
**Status:** Ready for Implementation
**Scope:** Phase 1 (Transport & Supply Contracts)
**Estimated Hours:** 20-25
**Last Updated:** 2026-03-06

---

## Table of Contents

1. [Implementation Checklist](#implementation-checklist)
2. [Database Schema](#database-schema)
3. [Models](#models)
4. [Service Layer](#service-layer)
5. [API Endpoints](#api-endpoints)
6. [Business Logic Rules](#business-logic-rules)
7. [Integration Points](#integration-points)
8. [Testing Strategy](#testing-strategy)
9. [Rollout Plan](#rollout-plan)

---

## Implementation Checklist

### Phase 1a: Database & Models (2-3 hours)
- [ ] Create migration: `create_contracts_table`
- [ ] Create migration: `create_contract_events_table`
- [ ] Create migration: `create_player_contract_reputation_table`
- [ ] Create `Contract` model with relationships
- [ ] Create `ContractEvent` model with relationships
- [ ] Create `PlayerContractReputation` model with relationships
- [ ] Add `contracts()` and `activeContracts()` relationship to `Player`
- [ ] Add `postedContracts()` relationship to `PointOfInterest` (bar location)

### Phase 1b: Services (8-10 hours)
- [ ] Create `ContractService`
  - [ ] `listContractsAtLocation($poi, $filters)`
  - [ ] `acceptContract($contract, $player)`
  - [ ] `completeContract($contract, $player, $cargo_manifest)`
  - [ ] `canAcceptContract($contract, $player)` — validation
  - [ ] `getPlayerReputation($player)`
- [ ] Create `ContractGenerationService`
  - [ ] `generateContractsForHub($hub)` — run during economy tick
  - [ ] `generateTransportContract($origin, $destination, $commodity, $quantity)`
  - [ ] `generateSupplyContract($destination, $commodity, $quantity)`
- [ ] Create `ReputationService`
  - [ ] `recordSuccess($player, $contract)`
  - [ ] `recordFailure($player, $contract, $reason)`
  - [ ] `calculateReliabilityScore($player)` — 0-100
- [ ] Create `ContractExpiryService`
  - [ ] `processExpirations()` — scheduled job, marks POSTED contracts as EXPIRED
  - [ ] `processFailures()` — marks ACCEPTED past deadline as FAILED

### Phase 1c: Controllers & Routes (4-6 hours)
- [ ] Create `ContractController`
  - [ ] `listContracts($poi_uuid)` — GET /api/trading-hubs/{uuid}/contracts
  - [ ] `acceptContract($contract_uuid)` — POST /api/contracts/{uuid}/accept
  - [ ] `deliverCargo($contract_uuid)` — POST /api/contracts/{uuid}/deliver
  - [ ] `getMyContracts()` — GET /api/players/{uuid}/contracts
  - [ ] `getReputation()` — GET /api/players/{uuid}/reputation
- [ ] Register routes in `routes/api.php`

### Phase 1d: Scheduled Jobs (1-2 hours)
- [ ] Create artisan command: `ContractExpiryCommand`
  - [ ] Runs every hour
  - [ ] Expires POSTED contracts past `expires_at`
  - [ ] Fails ACCEPTED contracts past `deadline_at`
  - [ ] Applies reputation penalties

### Phase 1e: Testing (3-4 hours)
- [ ] Unit tests for `ContractService`
- [ ] Unit tests for `ContractGenerationService`
- [ ] Unit tests for `ReputationService`
- [ ] Feature tests for API endpoints
- [ ] Contract lifecycle tests (POSTED → ACCEPTED → COMPLETED)
- [ ] Validation tests (acceptance, delivery, expiry)

---

## Database Schema

### `contracts` Table

```php
Schema::create('contracts', function (Blueprint $table) {
    $table->id();
    $table->uuid()->unique();

    // Contract identity
    $table->enum('type', ['TRANSPORT', 'SUPPLY'])->default('TRANSPORT');
    $table->enum('status', ['POSTED', 'ACCEPTED', 'COMPLETED', 'FAILED', 'EXPIRED', 'CANCELLED'])->default('POSTED');
    $table->enum('scope', ['LOCAL', 'REGIONAL', 'GALACTIC'])->default('LOCAL');

    // Issuance
    $table->foreignId('bar_location_id')->constrained('points_of_interest')->cascadeOnDelete();
    $table->enum('issuer_type', ['SYSTEM', 'COLONY', 'FACTION', 'PLAYER'])->default('SYSTEM');
    $table->unsignedBigInteger('issuer_id')->nullable(); // polymorphic: colony_id, faction_id, player_id

    // Contract details
    $table->string('title'); // "Transport Titanium to Tau Ceti"
    $table->text('description')->nullable();

    // Routing
    $table->foreignId('origin_location_id')->constrained('points_of_interest')->restrictOnDelete();
    $table->foreignId('destination_location_id')->constrained('points_of_interest')->restrictOnDelete();

    // Cargo
    $table->json('cargo_manifest'); // [{commodity_id, quantity}, ...]

    // Compensation & risk
    $table->integer('reward_credits');
    $table->enum('risk_rating', ['LOW', 'MEDIUM', 'HIGH'])->default('LOW');

    // Requirements
    $table->integer('reputation_min')->default(0); // 0-100, player must have >= this
    $table->integer('active_contract_limit')->default(5);

    // Lifecycle
    $table->dateTime('posted_at');
    $table->dateTime('expires_at'); // POSTED contracts expire after this
    $table->dateTime('deadline_at'); // ACCEPTED contracts must complete by this
    $table->dateTime('completed_at')->nullable();
    $table->dateTime('failed_at')->nullable();
    $table->text('failure_reason')->nullable();

    // Player acceptance
    $table->foreignId('accepted_by_player_id')->nullable()->constrained('players')->nullOnDelete();
    $table->dateTime('accepted_at')->nullable();

    // Determinism (for testing/replay)
    $table->unsignedInteger('seed')->nullable();

    $table->timestamps();

    // Indexes
    $table->index(['bar_location_id', 'status']);
    $table->index(['accepted_by_player_id', 'status']);
    $table->index(['origin_location_id', 'destination_location_id']);
    $table->index('expires_at');
    $table->index('deadline_at');
});
```

### `contract_events` Table

```php
Schema::create('contract_events', function (Blueprint $table) {
    $table->id();
    $table->uuid()->unique();

    $table->foreignId('contract_id')->constrained('contracts')->cascadeOnDelete();

    // Event tracking
    $table->enum('event_type', ['POSTED', 'ACCEPTED', 'COMPLETED', 'FAILED', 'EXPIRED', 'CANCELLED'])->indexed();
    $table->enum('actor_type', ['SYSTEM', 'PLAYER', 'ADMIN'])->default('SYSTEM');
    $table->unsignedBigInteger('actor_id')->nullable();

    // Payload for rich logging
    $table->json('payload')->nullable(); // {reason, cargo_delivered, reputation_delta, ...}

    $table->timestamps();

    $table->index('contract_id');
    $table->index('created_at');
});
```

### `player_contract_reputation` Table

```php
Schema::create('player_contract_reputation', function (Blueprint $table) {
    $table->id();

    $table->foreignId('player_id')->unique()->constrained('players')->cascadeOnDelete();

    // Scoring
    $table->integer('reliability_score')->default(50); // 0-100, starts at 50
    $table->integer('completed_count')->default(0);
    $table->integer('failed_count')->default(0);
    $table->integer('abandoned_count')->default(0);
    $table->integer('expired_count')->default(0);

    // Penalties (cumulative reductions from 100)
    $table->integer('failure_penalty')->default(0);
    $table->integer('abandonment_penalty')->default(0);

    $table->timestamps();
});
```

---

## Models

### Contract Model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Contract extends Model
{
    use HasUuid;

    protected $fillable = [
        'uuid', 'type', 'status', 'scope', 'bar_location_id', 'issuer_type', 'issuer_id',
        'title', 'description', 'origin_location_id', 'destination_location_id',
        'cargo_manifest', 'reward_credits', 'risk_rating', 'reputation_min',
        'posted_at', 'expires_at', 'deadline_at', 'accepted_by_player_id', 'accepted_at',
        'completed_at', 'failed_at', 'failure_reason', 'seed',
    ];

    protected $casts = [
        'cargo_manifest' => 'array',
        'posted_at' => 'datetime',
        'expires_at' => 'datetime',
        'deadline_at' => 'datetime',
        'accepted_at' => 'datetime',
        'completed_at' => 'datetime',
        'failed_at' => 'datetime',
    ];

    // Relationships
    public function barLocation(): BelongsTo
    {
        return $this->belongsTo(PointOfInterest::class, 'bar_location_id');
    }

    public function originLocation(): BelongsTo
    {
        return $this->belongsTo(PointOfInterest::class, 'origin_location_id');
    }

    public function destinationLocation(): BelongsTo
    {
        return $this->belongsTo(PointOfInterest::class, 'destination_location_id');
    }

    public function acceptedBy(): BelongsTo
    {
        return $this->belongsTo(Player::class, 'accepted_by_player_id');
    }

    public function events(): HasMany
    {
        return $this->hasMany(ContractEvent::class);
    }

    // Scopes
    public function scopePosted($query)
    {
        return $query->where('status', 'POSTED');
    }

    public function scopeAccepted($query)
    {
        return $query->where('status', 'ACCEPTED');
    }

    public function scopeCompleted($query)
    {
        return $query->where('status', 'COMPLETED');
    }

    public function scopeByStatus($query, $status)
    {
        return $query->where('status', $status);
    }

    public function scopeAtLocation($query, $location_id)
    {
        return $query->where('bar_location_id', $location_id)->posted();
    }

    public function scopeExpired($query)
    {
        return $query->where('expires_at', '<=', now())->posted();
    }

    public function scopeOverdue($query)
    {
        return $query->where('deadline_at', '<=', now())->accepted();
    }

    // Helpers
    public function isPosted(): bool { return $this->status === 'POSTED'; }
    public function isAccepted(): bool { return $this->status === 'ACCEPTED'; }
    public function isCompleted(): bool { return $this->status === 'COMPLETED'; }
    public function isFailed(): bool { return $this->status === 'FAILED'; }
    public function isExpired(): bool { return $this->status === 'EXPIRED'; }
    public function isCancelled(): bool { return $this->status === 'CANCELLED'; }
}
```

### Player Extension

Add to `Player` model:

```php
public function contracts(): HasMany
{
    return $this->hasMany(Contract::class, 'accepted_by_player_id');
}

public function activeContracts(): HasMany
{
    return $this->contracts()->whereIn('status', ['POSTED', 'ACCEPTED']);
}

public function contractReputation(): HasOne
{
    return $this->hasOne(PlayerContractReputation::class);
}
```

---

## Service Layer

### ContractService

```php
namespace App\Services;

use App\Models\Contract;
use App\Models\Player;
use App\Models\PointOfInterest;
use Illuminate\Database\Eloquent\Collection;

class ContractService
{
    public function __construct(
        private ReputationService $reputationService,
        private InventoryService $inventoryService,
    ) {}

    /**
     * List contracts available at a location's bar
     *
     * @param PointOfInterest $location
     * @param array $filters {type?, status?, min_reward?, max_risk?}
     * @return Collection
     */
    public function listContractsAtLocation(PointOfInterest $location, array $filters = []): Collection
    {
        $query = Contract::where('bar_location_id', $location->id)->posted();

        if (isset($filters['type'])) {
            $query->where('type', $filters['type']);
        }
        if (isset($filters['min_reward'])) {
            $query->where('reward_credits', '>=', $filters['min_reward']);
        }
        if (isset($filters['max_risk'])) {
            $risk_levels = ['LOW' => 1, 'MEDIUM' => 2, 'HIGH' => 3];
            $query->whereRaw("FIELD(risk_rating, 'LOW', 'MEDIUM', 'HIGH') <= ?", [$risk_levels[$filters['max_risk']]]);
        }

        return $query->get();
    }

    /**
     * Check if player can accept a contract
     *
     * Validation:
     * - Contract must be POSTED
     * - Player reputation >= contract minimum
     * - Player doesn't exceed active contract limit
     * - Player not already accepted this contract
     *
     * @return array {success: bool, reason?: string}
     */
    public function canAcceptContract(Contract $contract, Player $player): array
    {
        if (!$contract->isPosted()) {
            return ['success' => false, 'reason' => 'Contract is no longer available'];
        }

        $reputation = $this->reputationService->getPlayerReputation($player);
        if ($reputation < $contract->reputation_min) {
            return ['success' => false, 'reason' => "Reputation too low (need {$contract->reputation_min}, have {$reputation})"];
        }

        $active = $player->activeContracts()->count();
        if ($active >= $contract->active_contract_limit) {
            return ['success' => false, 'reason' => "Too many active contracts ({$active}/{$contract->active_contract_limit})"];
        }

        return ['success' => true];
    }

    /**
     * Accept a contract
     *
     * Atomically:
     * - Validate acceptance
     * - Mark contract ACCEPTED
     * - Record event
     *
     * @throws \Exception
     */
    public function acceptContract(Contract $contract, Player $player): Contract
    {
        $canAccept = $this->canAcceptContract($contract, $player);
        if (!$canAccept['success']) {
            throw new \Exception($canAccept['reason']);
        }

        $contract->update([
            'status' => 'ACCEPTED',
            'accepted_by_player_id' => $player->id,
            'accepted_at' => now(),
        ]);

        ContractEvent::create([
            'contract_id' => $contract->id,
            'event_type' => 'ACCEPTED',
            'actor_type' => 'PLAYER',
            'actor_id' => $player->id,
        ]);

        return $contract->refresh();
    }

    /**
     * Complete a contract via cargo delivery
     *
     * Validates:
     * - Contract is ACCEPTED by this player
     * - Player is at destination
     * - Player has correct cargo
     * - Cargo quantity matches manifest
     *
     * Atomically:
     * - Remove cargo from ship
     * - Add cargo to destination hub inventory
     * - Mark contract COMPLETED
     * - Pay reward to player
     * - Record reputation success
     * - Record event
     *
     * @throws \Exception
     */
    public function completeContract(Contract $contract, Player $player, array $cargo_delivered): Contract
    {
        if (!$contract->isAccepted() || $contract->accepted_by_player_id !== $player->id) {
            throw new \Exception('You have not accepted this contract');
        }

        if ($player->current_poi_id !== $contract->destination_location_id) {
            throw new \Exception('You are not at the contract destination');
        }

        // Validate cargo
        $this->validateCargoDelivery($contract, $cargo_delivered);

        DB::transaction(function () use ($contract, $player, $cargo_delivered) {
            // Remove cargo from ship
            foreach ($cargo_delivered as $commodity_id => $quantity) {
                $this->inventoryService->removeCargoFromShip($player->activeShip, $commodity_id, $quantity);
            }

            // Add to destination hub inventory
            $hub = $contract->destinationLocation->tradingHub;
            foreach ($cargo_delivered as $commodity_id => $quantity) {
                $this->inventoryService->addToHubInventory($hub, $commodity_id, $quantity);
            }

            // Mark complete
            $contract->update([
                'status' => 'COMPLETED',
                'completed_at' => now(),
            ]);

            // Pay reward
            $player->addCredits($contract->reward_credits);

            // Record success
            $this->reputationService->recordSuccess($player, $contract);

            // Record event
            ContractEvent::create([
                'contract_id' => $contract->id,
                'event_type' => 'COMPLETED',
                'actor_type' => 'PLAYER',
                'actor_id' => $player->id,
                'payload' => [
                    'reward_credits' => $contract->reward_credits,
                    'cargo_delivered' => $cargo_delivered,
                ],
            ]);
        });

        return $contract->refresh();
    }

    /**
     * Validate cargo matches contract manifest
     *
     * @throws \Exception
     */
    private function validateCargoDelivery(Contract $contract, array $cargo_delivered): void
    {
        $manifest = collect($contract->cargo_manifest)->keyBy('commodity_id');
        $delivered = collect($cargo_delivered);

        foreach ($manifest as $commodity_id => $manifest_item) {
            if (!isset($cargo_delivered[$commodity_id])) {
                throw new \Exception("Missing commodity {$commodity_id}");
            }
            if ($cargo_delivered[$commodity_id] != $manifest_item['quantity']) {
                throw new \Exception("Quantity mismatch for commodity {$commodity_id}");
            }
        }

        if (count($cargo_delivered) > count($manifest)) {
            throw new \Exception('Extra cargo in delivery');
        }
    }

    /**
     * Get player's reputation score (0-100)
     */
    public function getPlayerReputation(Player $player): int
    {
        return $this->reputationService->getPlayerReputation($player);
    }
}
```

### ContractGenerationService

```php
namespace App\Services;

use App\Models\Contract;
use App\Models\TradingHub;
use App\Models\Commodity;
use Illuminate\Support\Str;

class ContractGenerationService
{
    /**
     * Generate contracts for a trading hub during economy tick
     *
     * Called from EconomyTickCommand::handle()
     *
     * - 0-3 contracts per hub per day
     * - Based on supply/demand levels
     * - Low demand commodity = transport contract needed
     * - High demand commodity = supply contract needed
     */
    public function generateContractsForHub(TradingHub $hub, int $count = 2): array
    {
        $contracts = [];

        for ($i = 0; $i < $count; $i++) {
            // Pick a commodity
            $commodity = Commodity::inRandomOrder()->first();

            // Pick origin/destination
            $origin = $hub->pointOfInterest;
            $destination = PointOfInterest::where('galaxy_id', $hub->pointOfInterest->galaxy_id)
                ->inRandomOrder()
                ->first();

            if (!$destination || $destination->id === $origin->id) {
                continue;
            }

            // Pick type based on demand
            $inventory = $hub->inventory()->where('commodity_id', $commodity->id)->first();
            if (!$inventory) {
                continue;
            }

            $type = $inventory->demand_level > 60 ? 'SUPPLY' : 'TRANSPORT';

            // Generate contract
            if ($type === 'TRANSPORT') {
                $contracts[] = $this->generateTransportContract($origin, $destination, $commodity, rand(100, 500));
            } else {
                $contracts[] = $this->generateSupplyContract($destination, $commodity, rand(200, 1000));
            }
        }

        return $contracts;
    }

    /**
     * Generate transport contract (move cargo A → B)
     */
    public function generateTransportContract(PointOfInterest $origin, PointOfInterest $destination, Commodity $commodity, int $quantity): Contract
    {
        $distance = $origin->distanceTo($destination);
        $base_reward = $commodity->base_value * $quantity * 0.1; // 10% of cargo value
        $distance_factor = max(1, $distance / 1000); // bonus for long hauls
        $reward = (int) ceil($base_reward * $distance_factor);

        $risk_rating = $distance > 2000 ? 'HIGH' : ($distance > 1000 ? 'MEDIUM' : 'LOW');

        return Contract::create([
            'uuid' => Str::uuid(),
            'type' => 'TRANSPORT',
            'status' => 'POSTED',
            'scope' => 'LOCAL',
            'bar_location_id' => $origin->tradingHub->id,
            'issuer_type' => 'SYSTEM',
            'title' => "Transport {$quantity} {$commodity->name} to {$destination->name}",
            'description' => "Haul {$quantity} units of {$commodity->name} from {$origin->name} to {$destination->name}. Reward: {$reward} CR",
            'origin_location_id' => $origin->id,
            'destination_location_id' => $destination->id,
            'cargo_manifest' => [
                ['commodity_id' => $commodity->id, 'quantity' => $quantity],
            ],
            'reward_credits' => $reward,
            'risk_rating' => $risk_rating,
            'reputation_min' => 0,
            'posted_at' => now(),
            'expires_at' => now()->addDays(3), // Posted for 3 days
            'deadline_at' => now()->addDays(7), // 7 days to complete once accepted
            'seed' => rand(1, 999999),
        ]);
    }

    /**
     * Generate supply contract (deliver materials to destination)
     */
    public function generateSupplyContract(PointOfInterest $destination, Commodity $commodity, int $quantity): Contract
    {
        $reward = (int) ceil($commodity->base_value * $quantity * 0.15); // 15% of cargo value

        // Random origin from anywhere in galaxy
        $origin = PointOfInterest::where('galaxy_id', $destination->galaxy_id)
            ->inRandomOrder()
            ->first();

        if (!$origin || $origin->id === $destination->id) {
            return null;
        }

        return Contract::create([
            'uuid' => Str::uuid(),
            'type' => 'SUPPLY',
            'status' => 'POSTED',
            'scope' => 'LOCAL',
            'bar_location_id' => $origin->tradingHub->id,
            'issuer_type' => 'SYSTEM',
            'title' => "Supply {$quantity} {$commodity->name} to {$destination->name}",
            'description' => "Deliver {$quantity} units of {$commodity->name} to {$destination->name}. Reward: {$reward} CR",
            'origin_location_id' => $origin->id,
            'destination_location_id' => $destination->id,
            'cargo_manifest' => [
                ['commodity_id' => $commodity->id, 'quantity' => $quantity],
            ],
            'reward_credits' => $reward,
            'risk_rating' => 'MEDIUM',
            'reputation_min' => 10,
            'posted_at' => now(),
            'expires_at' => now()->addDays(2),
            'deadline_at' => now()->addDays(5),
            'seed' => rand(1, 999999),
        ]);
    }
}
```

### ReputationService

```php
namespace App\Services;

use App\Models\Player;
use App\Models\PlayerContractReputation;
use App\Models\Contract;

class ReputationService
{
    /**
     * Get player's contract reputation (0-100)
     * Creates record if doesn't exist (defaults to 50)
     */
    public function getPlayerReputation(Player $player): int
    {
        $rep = PlayerContractReputation::firstOrCreate(
            ['player_id' => $player->id],
            ['reliability_score' => 50]
        );

        return $rep->reliability_score;
    }

    /**
     * Record successful contract completion
     * Increases score up to 100
     */
    public function recordSuccess(Player $player, Contract $contract): void
    {
        $rep = PlayerContractReputation::firstOrCreate(
            ['player_id' => $player->id],
            ['reliability_score' => 50]
        );

        $rep->increment('completed_count');
        $rep->reliability_score = min(100, $rep->reliability_score + 2);
        $rep->save();
    }

    /**
     * Record contract failure (deadline missed)
     * Reduces score, applies cumulative penalty
     */
    public function recordFailure(Player $player, Contract $contract, string $reason = null): void
    {
        $rep = PlayerContractReputation::firstOrCreate(
            ['player_id' => $player->id],
            ['reliability_score' => 50]
        );

        $rep->increment('failed_count');
        $rep->failure_penalty = min(50, $rep->failure_penalty + 5);
        $rep->reliability_score = max(0, 50 - $rep->failure_penalty - $rep->abandonment_penalty);
        $rep->save();
    }

    /**
     * Record abandoned contract (accepted but never completed or failed)
     */
    public function recordAbandonment(Player $player, Contract $contract): void
    {
        $rep = PlayerContractReputation::firstOrCreate(
            ['player_id' => $player->id],
            ['reliability_score' => 50]
        );

        $rep->increment('abandoned_count');
        $rep->abandonment_penalty = min(50, $rep->abandonment_penalty + 8);
        $rep->reliability_score = max(0, 50 - $rep->failure_penalty - $rep->abandonment_penalty);
        $rep->save();
    }
}
```

### ContractExpiryService

```php
namespace App\Services;

use App\Models\Contract;
use App\Models\ContractEvent;
use Illuminate\Support\Facades\DB;

class ContractExpiryService
{
    public function __construct(private ReputationService $reputationService) {}

    /**
     * Process contract expirations
     * Called hourly from ContractExpiryCommand
     *
     * - Mark POSTED contracts past expires_at as EXPIRED
     * - Mark ACCEPTED contracts past deadline_at as FAILED
     * - Apply reputation penalties
     *
     * @return array {expired: int, failed: int}
     */
    public function processExpirations(): array
    {
        $now = now();
        $expired_count = 0;
        $failed_count = 0;

        DB::transaction(function () use (&$expired_count, &$failed_count, $now) {
            // Expire POSTED contracts
            $expired = Contract::where('status', 'POSTED')
                ->where('expires_at', '<=', $now)
                ->get();

            foreach ($expired as $contract) {
                $contract->update(['status' => 'EXPIRED']);

                ContractEvent::create([
                    'contract_id' => $contract->id,
                    'event_type' => 'EXPIRED',
                    'actor_type' => 'SYSTEM',
                    'payload' => ['reason' => 'Contract expired without acceptance'],
                ]);

                $expired_count++;
            }

            // Fail ACCEPTED contracts past deadline
            $overdue = Contract::where('status', 'ACCEPTED')
                ->where('deadline_at', '<=', $now)
                ->get();

            foreach ($overdue as $contract) {
                $contract->update([
                    'status' => 'FAILED',
                    'failed_at' => now(),
                    'failure_reason' => 'Deadline exceeded',
                ]);

                // Apply reputation penalty
                $player = $contract->acceptedBy;
                $this->reputationService->recordFailure($player, $contract, 'Deadline exceeded');

                ContractEvent::create([
                    'contract_id' => $contract->id,
                    'event_type' => 'FAILED',
                    'actor_type' => 'SYSTEM',
                    'actor_id' => $player->id,
                    'payload' => ['reason' => 'Deadline exceeded'],
                ]);

                $failed_count++;
            }
        });

        return [
            'expired' => $expired_count,
            'failed' => $failed_count,
        ];
    }
}
```

---

## API Endpoints

### List Contracts at Location

```
GET /api/trading-hubs/{uuid}/contracts
```

**Query Parameters:**
```
?type=TRANSPORT|SUPPLY
?min_reward=5000
?max_risk=MEDIUM
```

**Response:**
```json
{
  "data": [
    {
      "uuid": "...",
      "type": "TRANSPORT",
      "title": "Transport 200 Titanium to Tau Ceti",
      "origin": { "uuid": "...", "name": "..." },
      "destination": { "uuid": "...", "name": "..." },
      "cargo_manifest": [
        { "commodity_id": 1, "commodity_name": "Titanium", "quantity": 200 }
      ],
      "reward_credits": 12000,
      "risk_rating": "MEDIUM",
      "deadline_days_remaining": 5,
      "requires_reputation": 0,
      "posted_at": "2026-03-06T10:00:00Z",
      "expires_at": "2026-03-09T10:00:00Z"
    }
  ],
  "meta": {
    "total": 15,
    "per_page": 10
  }
}
```

---

### Accept Contract

```
POST /api/contracts/{uuid}/accept
```

**Request:**
```json
{}
```

**Response (Success 200):**
```json
{
  "data": {
    "uuid": "...",
    "status": "ACCEPTED",
    "accepted_at": "2026-03-06T14:30:00Z",
    "deadline_at": "2026-03-13T10:00:00Z",
    "reward_credits": 12000,
    "destination": { "uuid": "...", "name": "Tau Ceti" }
  },
  "message": "Contract accepted. You have until 2026-03-13 to deliver."
}
```

**Response (Failure 422):**
```json
{
  "message": "Reputation too low (need 25, have 15)"
}
```

---

### Deliver Cargo

```
POST /api/contracts/{uuid}/deliver
```

**Request:**
```json
{
  "cargo": {
    "1": 200  // commodity_id: quantity
  }
}
```

**Response (Success 200):**
```json
{
  "data": {
    "uuid": "...",
    "status": "COMPLETED",
    "completed_at": "2026-03-08T09:15:00Z",
    "reward_credits": 12000,
    "cargo_delivered": { "1": 200 },
    "reputation_change": "+2 (now 47/100)"
  },
  "message": "Contract completed! 12,000 credits awarded."
}
```

**Response (Failure 422):**
```json
{
  "message": "Missing commodity 1 (Titanium)"
}
```

---

### Get Player's Contracts

```
GET /api/players/{uuid}/contracts?status=ACCEPTED|COMPLETED
```

**Response:**
```json
{
  "data": [
    {
      "uuid": "...",
      "status": "ACCEPTED",
      "type": "TRANSPORT",
      "title": "...",
      "destination": { "uuid": "...", "name": "..." },
      "deadline_at": "2026-03-13T10:00:00Z",
      "time_remaining_hours": 78,
      "reward_credits": 12000
    }
  ],
  "meta": {
    "completed": 5,
    "active": 2,
    "failed": 0
  }
}
```

---

### Get Player Reputation

```
GET /api/players/{uuid}/reputation
```

**Response:**
```json
{
  "data": {
    "reliability_score": 47,
    "completed_count": 5,
    "failed_count": 1,
    "abandoned_count": 0,
    "status_tier": "NOVICE"
  }
}
```

---

## Business Logic Rules

### Contract Generation
- **Frequency:** 2-3 contracts per hub per day (run during economy tick)
- **Types:** TRANSPORT (50%) or SUPPLY (50%) based on demand signals
- **Reward Formula:**
  - TRANSPORT: `(commodity_base_value * quantity * 0.1) * distance_factor`
  - SUPPLY: `commodity_base_value * quantity * 0.15`
- **Risk Rating:**
  - LOW: <1000 units distance
  - MEDIUM: 1000-2000 units
  - HIGH: >2000 units

### Contract Acceptance
- **Requirements:**
  - Contract must be POSTED
  - Player reputation ≥ `reputation_min` (0-100)
  - Player active contracts < `active_contract_limit` (default 5)
- **Effects:**
  - Status changes to ACCEPTED
  - Deadline set to `posted_at + 7 days` (configurable)
  - Player UUID recorded

### Contract Completion
- **Requirements:**
  - Contract must be ACCEPTED by this player
  - Player must be at destination POI
  - Player must provide exact cargo quantities from manifest
- **Atomicity:** All-or-nothing; cargo removed + added + credit paid in single transaction
- **Effects:**
  - Cargo removed from ship
  - Cargo added to destination hub inventory
  - Contract marked COMPLETED
  - Reward credits added to player
  - Reputation increased by 2 (max 100)
  - Event recorded

### Contract Failure
- **Triggers:**
  - Accepted contract reaches `deadline_at` without completion
  - Detected by hourly ContractExpiryCommand
- **Effects:**
  - Status changes to FAILED
  - Failure reason recorded
  - Reputation decreased (base penalty 5 points)
  - Event recorded

### Contract Expiry
- **Triggers:**
  - Posted contract reaches `expires_at` without acceptance
  - Detected by hourly ContractExpiryCommand
- **Effects:**
  - Status changes to EXPIRED
  - No penalty applied (never accepted)
  - Event recorded

### Reputation Scoring
- **Base:** 50 (neutral)
- **Range:** 0-100
- **Improvements:** +2 per successful completion (max 100)
- **Penalties:**
  - Failure: -5 (cumulative)
  - Abandonment (accepted but abandoned): -8 (cumulative)
- **Application:** Reputation gates contract acceptance (min_reputation field)

---

## Integration Points

### Economy Tick
Add to `EconomyTickCommand::handle()`:

```php
// Step 3: Generate new contracts
$generationService = app(ContractGenerationService::class);
foreach ($galaxy->tradingHubs as $hub) {
    $generationService->generateContractsForHub($hub, 2);
}
```

### Scheduled Jobs
Create artisan command `ContractExpiryCommand` and add to schedule:

```php
// app/Console/Kernel.php
$schedule->command('contracts:expire')->hourly();
```

### Player Model
Add relationships (see Models section)

### TradingHub / PointOfInterest
Add:
```php
public function postedContracts(): HasMany {
    return $this->hasMany(Contract::class, 'bar_location_id')->posted();
}
```

---

## Testing Strategy

### Unit Tests

#### ContractService Tests
```php
- testCanAcceptValidContract()
- testCannotAcceptWithLowReputation()
- testCannotAcceptWhenActive_ExceededLimit()
- testAcceptContractUpdatesStatus()
- testCompleteContractRequiresCorrectCargo()
- testCompleteContractRemovesCargoFromShip()
- testCompleteContractAddsCargoToHub()
- testCompleteContractPaysReward()
- testCompleteContractIncrementsReputation()
```

#### ContractGenerationService Tests
```php
- testGenerateTransportContractCalculatesCorrectReward()
- testGenerateSupplyContractCalculatesCorrectReward()
- testGenerateContractsCreatesMultipleContracts()
- testGenerationConsidersSupplyDemand()
```

#### ReputationService Tests
```php
- testGetDefaultReputation50()
- testRecordSuccessIncrementsScore()
- testRecordSuccessMaxes100()
- testRecordFailureDecrementsScore()
- testRecordAbandonmentAppliesPenalty()
```

#### ContractExpiryService Tests
```php
- testExpireContractsPastExpiration()
- testFailContractsPastDeadline()
- testApplyReputationPenaltiesOnFailure()
```

### Feature Tests
```php
- testListContractsAtLocationEndpoint()
- testAcceptContractEndpoint()
- testDeliverCargoCompletes()
- testGetPlayerContractsFiltering()
- testGetReputationEndpoint()
- testContractLifecyclePostedToCompleted()
- testContractLifecyclePostedToExpired()
- testContractLifecycleAcceptedToFailed()
```

---

## Rollout Plan

### Day 1: Database & Models
1. Create migrations
2. Create models with relationships
3. Seed test data
4. Run migrations: `php artisan migrate`

### Day 2: Services
1. Implement ContractService
2. Implement ContractGenerationService
3. Implement ReputationService
4. Implement ContractExpiryService
5. Add service bindings to `AppServiceProvider`
6. Unit tests for all services

### Day 3: API & Integration
1. Create ContractController
2. Register routes in `routes/api.php`
3. Create ContractExpiryCommand
4. Add command to scheduler
5. Integrate with EconomyTickCommand
6. Feature tests for all endpoints

### Day 4: Polish & Testing
1. Error handling & validation refinement
2. Integration tests (end-to-end contract lifecycle)
3. Load testing (contract generation at scale)
4. Documentation

### Day 5: Deployment
1. Code review
2. Staging validation
3. Production deployment
4. Monitor for errors

---

## Success Criteria

- [x] All migrations run cleanly
- [x] All models have relationships and work correctly
- [x] ContractService handles all lifecycle states
- [x] ReputationService correctly scores players
- [x] API endpoints validate input and return proper errors
- [x] Contracts expire and fail correctly via scheduled command
- [x] Full end-to-end test: POSTED → ACCEPTED → COMPLETED with rewards
- [x] Reputation gates contract acceptance
- [x] Active contract limits enforced
- [x] Cargo validation prevents wrong-item delivery
- [x] All tests passing (unit + feature)

---

## Configuration (config/contracts.php - to create)

```php
return [
    'generation' => [
        'contracts_per_hub_per_day' => 2,
        'transport_weight' => 0.5,
        'supply_weight' => 0.5,
    ],

    'rewards' => [
        'transport_multiplier' => 0.1, // 10% of cargo value
        'supply_multiplier' => 0.15,   // 15% of cargo value
        'distance_factor_base' => 1000, // units
    ],

    'deadlines' => [
        'posted_days' => 3,    // POSTED contracts expire after 3 days
        'completion_days' => 7, // ACCEPTED contracts must complete in 7 days
    ],

    'reputation' => [
        'default' => 50,           // New players start at 50
        'min' => 0,
        'max' => 100,
        'success_bonus' => 2,
        'failure_penalty' => 5,
        'abandon_penalty' => 8,
    ],

    'limits' => [
        'active_contracts_per_player' => 5,
        'contract_acceptance_cooldown_minutes' => 0, // Future: prevent spam
    ],
];
```

