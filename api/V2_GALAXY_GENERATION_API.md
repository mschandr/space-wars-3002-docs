# V2 Galaxy Generation API

**Version**: 1.0
**Status**: Active
**Date**: 2026-03-10

## Overview

The V2 Galaxy Generation API provides discrete, resumable, phase-based galaxy creation with fine-grained control over the generation process. Unlike the original monolithic approach, V2 allows you to:

- Create empty galaxy structure separately from population
- Generate stars, warp lanes, and systems independently
- Create players and supernodes at any time
- Resume interrupted generation without duplication
- Monitor progress and timing per phase

All V2 endpoints require authentication (Sanctum token).

---

## Endpoints

### 1. Create Empty Galaxy Structure

**POST** `/api/v2/galaxies/create`

Creates an empty galaxy with sector grid. No stars, gates, or players yet.

**Authentication**: Required (Sanctum)

**Request Body**:
```json
{
  "name": "Andromeda Prime",
  "tier": "large"
}
```

**Parameters**:
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | No | Galaxy name. Auto-generated if not provided. |
| `tier` | string | No | Size tier: `small`, `medium`, `large`, `massive`. Default: `large`. Only admins can choose non-default. |

**Response** (201 Created):
```json
{
  "success": true,
  "data": {
    "galaxy": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Andromeda Prime",
      "width": 2500,
      "height": 2500,
      "status": "initializing",
      "tier": "large",
      "next_phase": "stars"
    },
    "progress": {
      "phase": 1,
      "total_phases": 6,
      "description": "Galaxy structure created. Next: populate stars."
    },
    "next_steps": {
      "endpoint": "POST /api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/populate",
      "body": {
        "phase": "stars"
      }
    }
  },
  "message": "Galaxy structure created"
}
```

**Error Responses**:
- `403`: Non-admin attempted custom size tier
- `422`: Invalid size tier provided
- `500`: Creation failed

---

### 2. Populate Galaxy Phase

**POST** `/api/v2/galaxies/{uuid}/populate`

Execute a specific generation phase or all phases sequentially.

**Authentication**: Required (Sanctum)

**Path Parameters**:
| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `uuid` | string | Yes | Galaxy ID (UUID format) |

**Request Body**:
```json
{
  "phase": "stars",
  "user_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

**Phases**:
| Phase | Description | Duration (est.) | Dependencies |
|-------|-------------|-----------------|--------------|
| `stars` | Generate core + outer region stars | 2-4s | None |
| `gates` | Create warp lane network with distance-based distribution | 3-6s | Requires: `stars` |
| `player` | Create player in galaxy | 50-100ms | Requires: `user_id` parameter |
| `supernode` | Create player's starting location with all services | 100-300ms | Requires: `player` |
| `systems` | Populate remaining systems (planets, minerals, defenses) | 3-5s | Requires: `stars` |
| `all` | Execute all phases (stars → gates → player → supernode → systems) | 8-17s | Requires: `user_id` for player creation |

**Parameters**:
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `phase` | string | Yes | Phase name (see table above) |
| `user_id` | string | No | User UUID. Required if phase is `player`, `supernode`, or `all`. |

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "phase": "stars",
    "result": {
      "count": 3000,
      "description": "Stars generated (core + outer regions)"
    },
    "timing": {
      "elapsed_ms": 2850,
      "estimated_total_ms": 15000
    },
    "progress": {
      "current_phase": "stars",
      "next_phase": "gates"
    },
    "next_step": {
      "endpoint": "POST /api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/populate",
      "body": {
        "phase": "gates"
      }
    }
  }
}
```

**Error Responses**:
- `404`: Galaxy not found
- `422`: Invalid phase or missing required parameters
- `500`: Population failed

---

### 3. Get Galaxy Status

**GET** `/api/v2/galaxies/{uuid}/status`

Get current population status of a galaxy.

**Authentication**: Required (Sanctum)

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "galaxy": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Andromeda Prime",
      "status": "initializing",
      "dimensions": {
        "width": 2500,
        "height": 2500
      }
    },
    "population": {
      "points_of_interest": 3000,
      "warp_gates": 1547,
      "sectors": 100,
      "players": 1
    },
    "progress": {
      "structure_created": true,
      "stars_populated": true,
      "gates_created": true,
      "players_created": true
    }
  }
}
```

---

### 4. Get Available Size Tiers

**GET** `/api/v2/galaxies/tiers`

Get information about available galaxy size tiers.

**Authentication**: Required (Sanctum)

**Response** (200 OK):
```json
{
  "success": true,
  "data": {
    "default_tier": "large",
    "admin_only_custom_tiers": true,
    "tiers": [
      {
        "name": "small",
        "width": 500,
        "height": 500,
        "core_stars": 100,
        "outer_stars": 400,
        "total_stars": 500,
        "estimated_time_seconds": 5
      },
      {
        "name": "medium",
        "width": 1500,
        "height": 1500,
        "core_stars": 300,
        "outer_stars": 1200,
        "total_stars": 1500,
        "estimated_time_seconds": 10
      },
      {
        "name": "large",
        "width": 2500,
        "height": 2500,
        "core_stars": 500,
        "outer_stars": 2000,
        "total_stars": 2500,
        "estimated_time_seconds": 15
      },
      {
        "name": "massive",
        "width": 5000,
        "height": 5000,
        "core_stars": 1000,
        "outer_stars": 4000,
        "total_stars": 5000,
        "estimated_time_seconds": 25
      }
    ]
  }
}
```

---

## Complete Workflow Example

### Step 1: Create Galaxy Structure
```bash
curl -X POST http://localhost:8000/api/v2/galaxies/create \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Andromeda Prime",
    "tier": "large"
  }'
```

Response includes galaxy UUID: `550e8400-e29b-41d4-a716-446655440000`

### Step 2: Populate Stars
```bash
curl -X POST http://localhost:8000/api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/populate \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "phase": "stars"
  }'
```

### Step 3: Create Warp Gates
```bash
curl -X POST http://localhost:8000/api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/populate \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "phase": "gates"
  }'
```

### Step 4: Complete All Remaining Phases
```bash
curl -X POST http://localhost:8000/api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/populate \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "phase": "all",
    "user_id": "550e8400-e29b-41d4-a716-446655440001"
  }'
```

### Step 5: Check Status
```bash
curl -X GET http://localhost:8000/api/v2/galaxies/550e8400-e29b-41d4-a716-446655440000/status \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Distance-Based Warp Gate Distribution

The V2 generation implements distance-based gate distribution, decreasing gate counts as you move away from the galactic core:

**Core Region** (inner 50% of galaxy):
- Max gates per system: 8
- Inhabited probability: 95%
- Type: Core (active) gates

**Middle Region** (50-75% from center):
- Max gates per system: 5
- Inhabited probability: 70%
- Type: Mix of core and dormant

**Outer Region** (75-100% from center):
- Max gates per system: 2
- Inhabited probability: 40%
- Type: Dormant gates, can be dead ends

This distribution creates naturally:
- Crowded hub regions near the core
- Sparser, more isolated outer systems
- Dead-end systems requiring coordinate travel

---

## Supernode Requirements

When creating a player's supernode, the system attempts to find a location matching these requirements:

**Strict Tier** (attempted first):
- Large/Giant star
- Shipyard, Salvage Yard, Trading Hub, Cartographer
- ≥2 habitable planets
- Gas giant with mining
- ≥1 habitable moon with mining
- Orbital defenses

**Standard Tier** (fallback):
- Large/Giant star
- Shipyard, Salvage Yard, Trading Hub, Cartographer
- ≥2 habitable planets
- Gas giant with mining
- Orbital defenses

**Minimal Tier** (final fallback):
- Large star
- Shipyard, Salvage Yard, Trading Hub
- ≥1 habitable planet

If no natural supernode matches any tier, a synthetic supernode is created at the galactic core.

---

## Admin Authorization

**Size Tier Selection**: Only administrators can specify custom size tiers (small, medium, massive). Non-admins receive a `403 Forbidden` error.

```bash
# Non-admin: only default 'large' allowed
curl -X POST http://localhost:8000/api/v2/galaxies/create \
  -H "Authorization: Bearer USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test", "tier": "massive"}' \
  # Returns: 403 Forbidden - Only administrators can specify custom galaxy sizes

# Admin: all tiers allowed
curl -X POST http://localhost:8000/api/v2/galaxies/create \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test", "tier": "massive"}' \
  # Returns: 201 Created
```

---

## Error Handling

All errors follow a consistent format:

```json
{
  "success": false,
  "error": {
    "message": "Error description",
    "code": "ERROR_CODE",
    "data": {
      "additional": "context"
    }
  },
  "status_code": 400
}
```

**Common Error Codes**:
| Code | Status | Meaning |
|------|--------|---------|
| `GALAXY_NOT_FOUND` | 404 | Galaxy UUID doesn't exist |
| `INVALID_TIER` | 422 | Invalid size tier provided |
| `UNAUTHORIZED_SIZE_TIER` | 403 | Non-admin attempted custom size |
| `CREATION_FAILED` | 500 | Galaxy creation failed |
| `POPULATION_FAILED` | 500 | Phase population failed |

---

## Performance Characteristics

| Phase | Queries | Time Range | Big O |
|-------|---------|-----------|-------|
| Create structure | 2 | 10-50ms | O(1) |
| Stars | 2 | 2-4s | O(n log n) |
| Warp gates | 3-5 | 3-6s | O(n log n) |
| Player | ~5 | 50-100ms | O(1) |
| Supernode | ~10 | 100-300ms | O(n) |
| Systems | 3-5 | 3-5s | O(n) |

**Total Time**: ~8-17 seconds for complete large galaxy

---

## Version History

### v1.0 (2026-03-10)
- Initial release
- 6 discrete phases: structure, stars, gates, player, supernode, systems
- Distance-based gate distribution
- Admin-only size tier selection
- Supernode requirement fallback chain
