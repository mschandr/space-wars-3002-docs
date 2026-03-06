# Construction API Reference

This document covers all construction job, blueprint, and asynchronous crafting endpoints for Space Wars 3002.

## Table of Contents

- [Blueprints](#blueprints)
  - [GET /api/trading-hubs/{uuid}/blueprints](#get-apitrading-hubsuuidblueprints)
- [Construction Jobs](#construction-jobs)
  - [POST /api/trading-hubs/{uuid}/build](#post-apitrading-hubsuuidbuild)
  - [GET /api/players/{uuid}/construction-jobs](#get-apiplayersuuidconstruction-jobs)

---

## Blueprints

### GET /api/trading-hubs/{uuid}/blueprints

List all available blueprints for construction at a specific trading hub.

**Authentication:** Required (Sanctum)

**Description:** Returns all blueprints available for construction at the specified trading hub, including input requirements, build time, and feasibility status. Each blueprint includes a `can_build` flag indicating whether the hub currently has sufficient input commodities, and a `shortages` array detailing any missing resources.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uuid` | string (UUID) | Yes | UUID of the trading hub or its POI |

#### Success Response

**Status Code:** `200 OK`

**Response Body:**
```json
{
  "success": true,
  "data": [
    {
      "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "code": "FIGHTER_MK1",
      "name": "Fighter Ship MK1",
      "description": "A lightweight combat vessel designed for intercepting smaller targets",
      "type": "ship",
      "output_item_code": "ship_fighter_mk1",
      "build_time_seconds": 3600,
      "can_build": true,
      "inputs": [
        {
          "commodity_id": 1,
          "commodity": {
            "uuid": "550e8400-e29b-41d4-a716-446655440000",
            "code": "IRON",
            "name": "Iron Ore"
          },
          "qty_required": 100
        },
        {
          "commodity_id": 5,
          "commodity": {
            "uuid": "550e8400-e29b-41d4-a716-446655440005",
            "code": "PLATINUM",
            "name": "Platinum"
          },
          "qty_required": 50
        }
      ],
      "shortages": []
    },
    {
      "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d480",
      "code": "CARGO_EXPAND",
      "name": "Cargo Expansion Module",
      "description": "Increases ship cargo hold by 50 units",
      "type": "module",
      "output_item_code": "mod_cargo_expand",
      "build_time_seconds": 1800,
      "can_build": false,
      "inputs": [
        {
          "commodity_id": 2,
          "commodity": {
            "uuid": "550e8400-e29b-41d4-a716-446655440002",
            "code": "COPPER",
            "name": "Copper"
          },
          "qty_required": 75
        }
      ],
      "shortages": [
        {
          "commodity": {
            "uuid": "550e8400-e29b-41d4-a716-446655440002",
            "code": "COPPER",
            "name": "Copper"
          },
          "required": 75,
          "available": 30,
          "shortfall": 45
        }
      ]
    }
  ],
  "message": "Blueprints retrieved",
  "meta": {
    "timestamp": "2026-03-04T10:42:24Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `uuid` | string | UUID of the blueprint |
| `code` | string | Unique identifier for the blueprint (e.g., "FIGHTER_MK1") |
| `name` | string | Display name of the blueprint |
| `description` | string | Lore description of what the blueprint creates |
| `type` | string | Output type: "ship", "module", "upgrade", "part", etc. |
| `output_item_code` | string | Identifier for the item produced (used in future item delivery phase) |
| `build_time_seconds` | integer | Time in seconds to complete construction |
| `can_build` | boolean | Whether the hub currently has sufficient resources to build |
| `inputs` | array | List of input commodities required |
| `inputs[].commodity_id` | integer | Database ID of the required commodity |
| `inputs[].commodity` | object | Commodity details (uuid, code, name) |
| `inputs[].qty_required` | integer | Quantity of this commodity required per build unit |
| `shortages` | array | List of input commodities with insufficient stock |
| `shortages[].commodity` | object | Commodity details (uuid, code, name) |
| `shortages[].required` | integer | Total quantity required |
| `shortages[].available` | integer | Current quantity available at hub |
| `shortages[].shortfall` | integer | Difference (required - available) |

#### Error Responses

**404 Not Found:** Trading hub not found
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Trading hub not found"
  }
}
```

**401 Unauthorized:** Missing or invalid authentication token
```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Unauthorized access"
  }
}
```

#### Warnings & Caveats

- `can_build` is a snapshot at request time; hub inventory may change before construction starts
- `build_time_seconds` is fixed per blueprint and immutable
- Multiple instances of the same blueprint can be under construction simultaneously
- Future phases will implement item delivery (currently a stub—see ADR 0001)

---

## Construction Jobs

### POST /api/trading-hubs/{uuid}/build

Start a new construction job at a trading hub.

**Authentication:** Required (Sanctum)

**Description:** Initiates construction of a blueprint at the specified trading hub. The player's ship must be docked at the hub. All required input commodities are consumed immediately from the hub inventory via the ledger economy system. The job is created with a `completes_at` timestamp calculated as `now() + blueprint->build_time_seconds`. Job maturation is handled asynchronously by the economy tick system.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uuid` | string (UUID) | Yes | UUID of the trading hub or its POI |

#### Request Body

```json
{
  "player_uuid": "550e8400-e29b-41d4-a716-446655440010",
  "ship_uuid": "550e8400-e29b-41d4-a716-446655440011",
  "blueprint_uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "quantity": 1
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `player_uuid` | string (UUID) | Yes | UUID of the player initiating construction |
| `ship_uuid` | string (UUID) | Yes | UUID of the player's ship (must be at hub) |
| `blueprint_uuid` | string (UUID) | Yes | UUID of the blueprint to construct |
| `quantity` | integer | Yes | Number of units to build (min: 1) |

#### Success Response

**Status Code:** `201 Created`

**Response Body:**
```json
{
  "success": true,
  "data": {
    "job_uuid": "550e8400-e29b-41d4-a716-446655440020",
    "blueprint": {
      "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "name": "Fighter Ship MK1"
    },
    "quantity": 1,
    "started_at": "2026-03-04T10:42:24Z",
    "completes_at": "2026-03-04T11:42:24Z",
    "status": "PENDING"
  },
  "message": "Construction started",
  "meta": {
    "timestamp": "2026-03-04T10:42:24Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `job_uuid` | string | UUID of the created construction job |
| `blueprint.uuid` | string | UUID of the blueprint being built |
| `blueprint.name` | string | Display name of the blueprint |
| `quantity` | integer | Number of units being constructed |
| `started_at` | string (ISO 8601) | Timestamp when construction started |
| `completes_at` | string (ISO 8601) | Timestamp when construction will complete |
| `status` | string | Job status: "PENDING" (will mature during next economy tick) |

#### Error Responses

**400 Bad Request:** Insufficient resources
```json
{
  "success": false,
  "error": {
    "code": "CONSTRUCTION_FAILED",
    "message": "Insufficient resources for construction",
    "details": {
      "shortages": [
        {
          "commodity": {
            "uuid": "550e8400-e29b-41d4-a716-446655440002",
            "code": "COPPER",
            "name": "Copper"
          },
          "required_per_unit": 50,
          "total_required": 50,
          "available": 30,
          "shortfall": 20
        }
      ]
    }
  }
}
```

**400 Bad Request:** Ship not at hub
```json
{
  "success": false,
  "error": {
    "code": "ERROR",
    "message": "Ship is not at this trading hub"
  }
}
```

**404 Not Found:** Resource not found
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Player not found or does not belong to player"
  }
}
```

**422 Validation Error:** Invalid input
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The given data was invalid",
    "errors": {
      "quantity": ["The quantity field must be at least 1."]
    }
  }
}
```

#### Ledger Integration

When construction succeeds:
1. Input commodities are consumed from hub inventory
2. A negative-delta `CommodityLedgerEntry` is created for each input commodity
3. Entries are linked with a correlation ID for audit trail
4. `ConstructionJob` is created with a snapshot of inputs in `inputs_consumed` JSON

This ensures:
- Conservation: Total commodities in galaxy are immutable
- Auditability: Every construction is traceable in ledger
- Atomicity: All inputs consumed in single transaction

#### Warnings & Caveats

- Construction consumes resources **immediately**; if the build is never completed (e.g., due to error), resources are permanently consumed
- Job status is "PENDING" until the economy tick processes it
- Player is responsible for monitoring `completes_at` to retrieve the item (future phases)
- Multiple builds can run in parallel at the same hub or same player

---

### GET /api/players/{uuid}/construction-jobs

List all construction jobs for a player.

**Authentication:** Required (Sanctum)

**Description:** Returns paginated list of all construction jobs belonging to the player, including completed and failed jobs. Results are ordered by most recent first. Supports filtering by status.

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uuid` | string (UUID) | Yes | UUID of the player |
| `status` | string | No | Filter by job status: "PENDING", "COMPLETE", "FAILED" |
| `page` | integer | No | Page number (default: 1) |
| `per_page` | integer | No | Items per page (default: 20, max: 100) |

#### Success Response

**Status Code:** `200 OK`

**Response Body:**
```json
{
  "success": true,
  "data": {
    "jobs": [
      {
        "uuid": "550e8400-e29b-41d4-a716-446655440020",
        "blueprint": {
          "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
          "name": "Fighter Ship MK1"
        },
        "trading_hub": {
          "uuid": "550e8400-e29b-41d4-a716-446655440001",
          "name": "Hub Prime"
        },
        "quantity": 1,
        "status": "COMPLETE",
        "started_at": "2026-03-04T09:42:24Z",
        "completes_at": "2026-03-04T10:42:24Z",
        "completed_at": "2026-03-04T10:42:30Z",
        "time_remaining_seconds": 0,
        "output_item_code": "ship_fighter_mk1"
      },
      {
        "uuid": "550e8400-e29b-41d4-a716-446655440021",
        "blueprint": {
          "uuid": "f47ac10b-58cc-4372-a567-0e02b2c3d480",
          "name": "Cargo Expansion Module"
        },
        "trading_hub": {
          "uuid": "550e8400-e29b-41d4-a716-446655440001",
          "name": "Hub Prime"
        },
        "quantity": 2,
        "status": "PENDING",
        "started_at": "2026-03-04T10:50:00Z",
        "completes_at": "2026-03-04T11:20:00Z",
        "completed_at": null,
        "time_remaining_seconds": 1230,
        "output_item_code": "mod_cargo_expand"
      }
    ],
    "pagination": {
      "total": 42,
      "count": 2,
      "per_page": 20,
      "current_page": 1,
      "total_pages": 3
    }
  },
  "message": "Construction jobs retrieved",
  "meta": {
    "timestamp": "2026-03-04T10:42:24Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `jobs[].uuid` | string | UUID of the construction job |
| `jobs[].blueprint` | object | Blueprint details (uuid, name) |
| `jobs[].trading_hub` | object | Trading hub details (uuid, name) where build occurred |
| `jobs[].quantity` | integer | Number of units being constructed |
| `jobs[].status` | string | Job status: "PENDING", "COMPLETE", "FAILED" |
| `jobs[].started_at` | string (ISO 8601) | Timestamp when construction started |
| `jobs[].completes_at` | string (ISO 8601) | Timestamp when construction will/did complete |
| `jobs[].completed_at` | string (ISO 8601) or null | Timestamp when job was marked complete (null if pending) |
| `jobs[].time_remaining_seconds` | integer | Seconds until completion (0 if already complete) |
| `jobs[].output_item_code` | string | Identifier for the item produced (denormalized from blueprint) |
| `pagination.total` | integer | Total number of jobs across all pages |
| `pagination.count` | integer | Number of jobs on this page |
| `pagination.per_page` | integer | Items per page setting |
| `pagination.current_page` | integer | Current page number |
| `pagination.total_pages` | integer | Total number of pages |

#### Error Responses

**404 Not Found:** Player not found
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Player not found"
  }
}
```

**401 Unauthorized:** Missing or invalid authentication token
```json
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Unauthorized access"
  }
}
```

#### Query Examples

Filter by pending jobs only:
```
GET /api/players/550e8400-e29b-41d4-a716-446655440010/construction-jobs?status=PENDING
```

Get page 2 with 50 items per page:
```
GET /api/players/550e8400-e29b-41d4-a716-446655440010/construction-jobs?page=2&per_page=50
```

#### Warnings & Caveats

- `time_remaining_seconds` is calculated at request time; actual remaining time may differ
- Completed jobs are kept indefinitely (for history/audit trail)
- Status is "PENDING" after creation and until economy tick processes completion
- Item delivery is not yet implemented (Phase 3 stub)—completed jobs show `status=COMPLETE` but items are not yet available for pickup

---

## Construction Lifecycle

### Job States

1. **PENDING**: Job created, inputs consumed, awaiting maturation during economy tick
2. **COMPLETE**: Job matured (completes_at <= now()), item ready (future phases: available for pickup)
3. **FAILED**: Job failed during processing (rare; usually due to exception during economy tick)

### Job Maturation

Jobs transition from PENDING → COMPLETE during `php artisan economy:tick` execution:

```bash
# Manual economy tick (processes all matured construction jobs)
php artisan economy:tick

# Specific galaxy only
php artisan economy:tick --galaxy=<galaxy-uuid>

# Dry-run (shows what would happen, no DB writes)
php artisan economy:tick --dry-run
```

Output includes construction section:
```
Construction Jobs:
  Checked: 5
  Completed: 3
```

### Input Consumption Details

When a construction job is created:
- All input inventory rows are locked in `mineral_id ASC` order (prevents deadlocks)
- Each input commodity is deducted from hub inventory atomically
- A `CommodityLedgerEntry` is created for each input with:
  - `reason_code = 'CONSTRUCTION'`
  - `qty_delta = -qty_required` (negative = sink)
  - `actor_type = 'PLAYER'`
  - `actor_id = player_id`
  - `correlation_id` linking all inputs for audit trail
- Ledger entries are immutable (provides full economic history)

---

## Related Documentation

- [ADR 0001: Construction Output Deferral](../ADR/0001-construction-output-deferral.md) — Explains Phase 3 scope and deferred item delivery
- [Trading API](./trading.md) — Mineral trading and hub inventory
- [Economy Guide](../guides/ECONOMICS_GUIDE.md) — Ledger-backed economy system
