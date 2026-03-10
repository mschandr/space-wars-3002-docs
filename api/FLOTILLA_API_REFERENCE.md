# Flotilla & Fleet Mechanics - API Reference

**Version:** 1.0
**Status:** Production Ready
**Last Updated:** March 6, 2026
**Implementation Date:** March 6, 2026

---

## Base URL

```
https://api.space-wars-3002.com/api
```

All requests require authentication via Bearer token (Sanctum).

---

## Authentication

All endpoints require the `Authorization: Bearer {token}` header.

```bash
curl -H "Authorization: Bearer your_token_here" \
  https://api.space-wars-3002.com/api/players/{uuid}/flotilla
```

---

## Error Handling

All endpoints return standardized error responses:

### 422 Validation Error
```json
{
  "error": "Description of validation error",
  "code": "VALIDATION_ERROR"
}
```

### 404 Not Found
```json
{
  "message": "Flotilla not found or player has no active flotilla"
}
```

### 403 Forbidden
```json
{
  "error": "Unauthorized"
}
```

---

## Endpoints

### 1. Create Flotilla

Creates a new flotilla with designated flagship.

**Endpoint:** `POST /players/{uuid}/flotilla`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Request Body:**
```json
{
  "flagship_ship_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Alpha Squadron"
}
```

**Response (201 Created):**
```json
{
  "message": "Flotilla created successfully",
  "flotilla": {
    "id": 1,
    "uuid": "550e8400-e29b-41d4-a716-446655440001",
    "name": "Alpha Squadron",
    "player_id": 1,
    "flagship": {
      "id": 100,
      "uuid": "550e8400-e29b-41d4-a716-446655440002",
      "name": "Flagship",
      "hull": 100,
      "max_hull": 100,
      "current_fuel": 50,
      "max_fuel": 100
    },
    "ships": [
      {
        "id": 100,
        "uuid": "550e8400-e29b-41d4-a716-446655440002",
        "name": "Flagship",
        "is_flagship": true,
        "hull": 100,
        "max_hull": 100,
        "current_fuel": 50,
        "max_fuel": 100,
        "warp_drive": 3,
        "cargo_hold": 500,
        "current_cargo": 0,
        "weapons": 5
      }
    ],
    "formation_stats": {
      "ship_count": 1,
      "is_full": false,
      "total_hull": 100,
      "weakest_ship_hull": 100,
      "slowest_warp_drive": 3,
      "total_cargo_hold": 500,
      "available_cargo_space": 500,
      "total_weapon_damage": 50
    },
    "location": {
      "poi_id": 1,
      "poi_name": "Trade Station Alpha",
      "at_same_location": true
    },
    "created_at": "2026-03-06T12:00:00Z",
    "updated_at": "2026-03-06T12:00:00Z"
  }
}
```

**Error Responses:**
- 422: Flagship ship not found, ship already in flotilla, player already has flotilla
- 400: Invalid request body

**Example:**
```bash
curl -X POST https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla \
  -H "Authorization: Bearer token" \
  -H "Content-Type: application/json" \
  -d '{
    "flagship_ship_id": "550e8400-e29b-41d4-a716-446655440002",
    "name": "Alpha Squadron"
  }'
```

---

### 2. Get Flotilla Status

Retrieves current flotilla information for player.

**Endpoint:** `GET /players/{uuid}/flotilla`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Response (200 OK):**
```json
{
  "id": 1,
  "uuid": "550e8400-e29b-41d4-a716-446655440001",
  "name": "Alpha Squadron",
  "player_id": 1,
  "flagship": {
    "id": 100,
    "uuid": "550e8400-e29b-41d4-a716-446655440002",
    "name": "Flagship",
    "hull": 85,
    "max_hull": 100,
    "current_fuel": 45,
    "max_fuel": 100
  },
  "ships": [
    {
      "id": 100,
      "uuid": "550e8400-e29b-41d4-a716-446655440002",
      "name": "Flagship",
      "is_flagship": true,
      "hull": 85,
      "max_hull": 100,
      "current_fuel": 45,
      "max_fuel": 100,
      "warp_drive": 3,
      "cargo_hold": 500,
      "current_cargo": 150,
      "weapons": 5
    },
    {
      "id": 101,
      "uuid": "550e8400-e29b-41d4-a716-446655440003",
      "name": "Escort",
      "is_flagship": false,
      "hull": 75,
      "max_hull": 80,
      "current_fuel": 40,
      "max_fuel": 100,
      "warp_drive": 2,
      "cargo_hold": 300,
      "current_cargo": 100,
      "weapons": 3
    }
  ],
  "formation_stats": {
    "ship_count": 2,
    "is_full": false,
    "total_hull": 160,
    "weakest_ship_hull": 75,
    "slowest_warp_drive": 2,
    "total_cargo_hold": 800,
    "available_cargo_space": 550,
    "total_weapon_damage": 80
  },
  "location": {
    "poi_id": 1,
    "poi_name": "Trade Station Alpha",
    "at_same_location": true
  },
  "created_at": "2026-03-06T12:00:00Z",
  "updated_at": "2026-03-06T12:00:00Z"
}
```

**Error Responses:**
- 404: Player has no active flotilla
- 403: Unauthorized

**Example:**
```bash
curl https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla \
  -H "Authorization: Bearer token"
```

---

### 3. Add Ship to Flotilla

Adds an existing ship to the player's flotilla.

**Endpoint:** `POST /players/{uuid}/flotilla/add-ship`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Request Body:**
```json
{
  "ship_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

**Response (200 OK):**
```json
{
  "message": "Ship added to flotilla",
  "flotilla": {
    // Full flotilla object (same as GET /players/{uuid}/flotilla)
  }
}
```

**Error Responses:**
- 422: Flotilla full (max 4 ships), ship not at same location, ship already in flotilla
- 404: Player flotilla not found, ship not found

**Example:**
```bash
curl -X POST https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla/add-ship \
  -H "Authorization: Bearer token" \
  -H "Content-Type: application/json" \
  -d '{
    "ship_id": "550e8400-e29b-41d4-a716-446655440003"
  }'
```

---

### 4. Remove Ship from Flotilla

Removes a ship from the flotilla. Cannot remove flagship.

**Endpoint:** `POST /players/{uuid}/flotilla/remove-ship`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Request Body:**
```json
{
  "ship_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

**Response (200 OK):**
```json
{
  "message": "Ship removed from flotilla",
  "flotilla": {
    // Full flotilla object
  }
}
```

**Error Responses:**
- 422: Cannot remove flagship, ship not in flotilla
- 404: Fleet not found, ship not found

**Example:**
```bash
curl -X POST https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla/remove-ship \
  -H "Authorization: Bearer token" \
  -H "Content-Type: application/json" \
  -d '{
    "ship_id": "550e8400-e29b-41d4-a716-446655440003"
  }'
```

---

### 5. Change Flagship

Designates a different ship as the flotilla's flagship.

**Endpoint:** `POST /players/{uuid}/flotilla/set-flagship`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Request Body:**
```json
{
  "ship_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

**Response (200 OK):**
```json
{
  "message": "Flagship changed successfully",
  "flotilla": {
    // Full flotilla object with new flagship
  }
}
```

**Error Responses:**
- 422: Ship not in flotilla
- 404: Flotilla not found, ship not found

**Example:**
```bash
curl -X POST https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla/set-flagship \
  -H "Authorization: Bearer token" \
  -H "Content-Type: application/json" \
  -d '{
    "ship_id": "550e8400-e29b-41d4-a716-446655440003"
  }'
```

---

### 6. Dissolve Flotilla

Completely dissolves the flotilla, releasing all ships to independent status.

**Endpoint:** `DELETE /players/{uuid}/flotilla`
**Created:** March 6, 2026
**Last Updated:** March 6, 2026

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| uuid | string (UUID) | Yes | Player UUID |

**Response (200 OK):**
```json
{
  "message": "Flotilla 'Alpha Squadron' dissolved. All ships are now independent."
}
```

**Error Responses:**
- 404: Player has no active flotilla

**Example:**
```bash
curl -X DELETE https://api.space-wars-3002.com/api/players/550e8400-e29b-41d4-a716-446655440000/flotilla \
  -H "Authorization: Bearer token"
```

---

## Combat Integration

### Modified Endpoint: Get Combat Preview

Now shows whether player will engage as flotilla or single ship.

**Endpoint:** `GET /players/{uuid}/combat/preview`
**Created:** (Existing endpoint)
**Flotilla Support Added:** March 6, 2026
**Last Updated:** March 6, 2026

**Response includes:**
```json
{
  "combat_readiness": {
    "has_flotilla": true,
    "flotilla_ship_count": 2,
    "will_engage_as_flotilla": true,
    "flotilla_details": {
      "name": "Alpha Squadron",
      "ship_count": 2,
      "total_hull": 160,
      "total_weapons": 8,
      "slowest_speed": 2
    }
  },
  "combat_preview": {
    "combat_type": "flotilla",
    "flotilla": {
      "name": "Alpha Squadron",
      "ship_count": 2,
      "total_hull": 160,
      "total_weapons": 8,
      "formation_strength": 85
    },
    "note": "All ships engage together. Slowest ship determines movement speed."
  }
}
```

---

### Modified Endpoint: Engage Combat

Automatically routes to flotilla or single-ship combat.

**Endpoint:** `POST /players/{uuid}/combat/engage`
**Created:** (Existing endpoint)
**Flotilla Support Added:** March 6, 2026
**Last Updated:** March 6, 2026

**Response includes:**
```json
{
  "combat_type": "flotilla",
  "victory": true,
  "rounds": [
    {
      "round": 1,
      "player_damage_dealt": 150,
      "pirate_damage_taken": 45,
      "pirate_ships_remaining": 2,
      "player_ships_remaining": 2
    }
  ],
  "xp_earned": 600,
  "player_hull_remaining": 140,
  "salvage": {
    "available_cargo": "See salvage options",
    "available_components": "See salvage options",
    "pirate_loot": "Random percentage"
  },
  "message": "Victory! All pirate ships destroyed. Your flotilla emerges victorious."
}
```

---

### Modified Endpoint: Attempt Escape

Flotillas use slowest ship's speed for escape chance.

**Endpoint:** `POST /players/{uuid}/combat/escape`
**Created:** (Existing endpoint)
**Flotilla Support Added:** March 6, 2026
**Last Updated:** March 6, 2026

**Response includes:**
```json
{
  "escape_type": "flotilla",
  "escaped": true,
  "escape_chance": 65,
  "message": "Your flotilla jumped to hyperspace! Escaped from pirates."
}
```

---

### Modified Endpoint: Surrender

All ships in flotilla surrender together, losing 70% cargo.

**Endpoint:** `POST /players/{uuid}/combat/surrender`
**Created:** (Existing endpoint)
**Flotilla Support Added:** March 6, 2026
**Last Updated:** March 6, 2026

**Response includes:**
```json
{
  "surrender_type": "flotilla",
  "surrendered": true,
  "total_cargo_lost": 250,
  "message": "Your flotilla surrendered. Pirates took 250 units of cargo from all ships."
}
```

---

## Data Models

### Flotilla Object

```json
{
  "id": 1,
  "uuid": "550e8400-e29b-41d4-a716-446655440001",
  "name": "Alpha Squadron",
  "player_id": 1,
  "flagship": {
    "id": 100,
    "uuid": "550e8400-e29b-41d4-a716-446655440002",
    "name": "Flagship",
    "hull": 85,
    "max_hull": 100,
    "current_fuel": 45,
    "max_fuel": 100
  },
  "ships": [
    {
      "id": 100,
      "uuid": "550e8400-e29b-41d4-a716-446655440002",
      "name": "Flagship",
      "is_flagship": true,
      "hull": 85,
      "max_hull": 100,
      "current_fuel": 45,
      "max_fuel": 100,
      "warp_drive": 3,
      "cargo_hold": 500,
      "current_cargo": 150,
      "weapons": 5
    }
  ],
  "formation_stats": {
    "ship_count": 1,
    "is_full": false,
    "total_hull": 100,
    "weakest_ship_hull": 100,
    "slowest_warp_drive": 3,
    "total_cargo_hold": 500,
    "available_cargo_space": 350,
    "total_weapon_damage": 50
  },
  "location": {
    "poi_id": 1,
    "poi_name": "Trade Station Alpha",
    "at_same_location": true
  },
  "created_at": "2026-03-06T12:00:00Z",
  "updated_at": "2026-03-06T12:00:00Z"
}
```

### Ship Object (in Flotilla)

```json
{
  "id": 100,
  "uuid": "550e8400-e29b-41d4-a716-446655440002",
  "name": "Flagship",
  "is_flagship": true,
  "hull": 85,
  "max_hull": 100,
  "current_fuel": 45,
  "max_fuel": 100,
  "warp_drive": 3,
  "cargo_hold": 500,
  "current_cargo": 150,
  "weapons": 5
}
```

---

## Rate Limiting

No specific rate limits for flotilla endpoints. Standard API rate limits apply.

---

## Deprecations

None. All endpoints are stable and production-ready.

---

## Changelog

### Version 1.0 (March 6, 2026)
**Status:** Production Ready
**Implementation Date:** March 6, 2026

**New Endpoints (6):**
- `POST /players/{uuid}/flotilla` - Create flotilla (March 6, 2026)
- `GET /players/{uuid}/flotilla` - Get flotilla status (March 6, 2026)
- `POST /players/{uuid}/flotilla/add-ship` - Add ship (March 6, 2026)
- `POST /players/{uuid}/flotilla/remove-ship` - Remove ship (March 6, 2026)
- `POST /players/{uuid}/flotilla/set-flagship` - Change flagship (March 6, 2026)
- `DELETE /players/{uuid}/flotilla` - Dissolve flotilla (March 6, 2026)

**Modified Endpoints (4):**
- `GET /players/{uuid}/combat/preview` - Flotilla support added (March 6, 2026)
- `POST /players/{uuid}/combat/engage` - Flotilla combat routing (March 6, 2026)
- `POST /players/{uuid}/combat/escape` - Flotilla escape logic (March 6, 2026)
- `POST /players/{uuid}/combat/surrender` - Flotilla surrender (March 6, 2026)

**Features:**
- Flotilla creation and management (6 new endpoints)
- Multi-ship movement with fuel penalties
- Combined-arms combat system
- XOR salvage choice (cargo OR components)
- Pirate loot recovery
- Full API documentation with examples

---

## Support

For issues or questions regarding the Flotilla API, please refer to:
- Technical documentation: `/docs/guides/FLOTILLA_TESTING_MANUAL.md`
- Implementation summary: `/docs/guides/FLOTILLA_PHASE_1_IMPLEMENTATION_SUMMARY.md`

