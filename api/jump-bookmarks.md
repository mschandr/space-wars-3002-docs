# Jump Bookmarks API

API documentation for player jump bookmark endpoints in Space Wars 3002. Bookmarks allow players to save named coordinates for quick navigation reference, optionally linked to a known Point of Interest.

## Base URL
All endpoints are prefixed with `/api/players/{uuid}/jump-bookmarks`

## Response Structure
All API responses follow the standardized format provided by `BaseApiController`.

### Success Response Format
```json
{
  "success": true,
  "data": { /* endpoint-specific data */ },
  "message": "Success message",
  "meta": {
    "timestamp": "2026-02-27T12:00:00+00:00",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Error Response Format
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Error description",
    "details": null
  },
  "meta": {
    "timestamp": "2026-02-27T12:00:00+00:00",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## Configuration

Bookmark limits are configured in `config/game_config.php`:

| Key | Default | Description |
|-----|---------|-------------|
| `navigation.max_jump_bookmarks` | 25 | Maximum bookmarks per player |

---

## Data Model

### PlayerJumpBookmark

| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| id | integer | No | Auto-increment primary key |
| uuid | string (UUID) | No | Unique public identifier |
| player_id | integer (FK) | No | Owning player. Cascade-deletes when player is removed |
| poi_id | integer (FK) | Yes | Linked Point of Interest. Set to null if the POI is deleted |
| name | string (max 100) | No | Player-chosen label for the bookmark |
| x | integer | No | X coordinate |
| y | integer | No | Y coordinate |
| notes | text | Yes | Optional freeform notes |
| created_at | datetime | No | Creation timestamp |
| updated_at | datetime | No | Last update timestamp |

**Key behaviours:**
- A player can have multiple bookmarks for the same POI (different names/notes).
- If a linked POI is deleted, the bookmark survives with coordinates intact (`nullOnDelete`).
- All bookmarks are deleted when their owning player is deleted (`cascadeOnDelete`).
- Bookmarks are cleared during `galaxy:flush`.

---

## Endpoints

### 1. List Bookmarks

**GET** `/api/players/{uuid}/jump-bookmarks`

Returns all jump bookmarks for the player, ordered by creation date (oldest first). Each bookmark eager-loads its linked POI (if any).

**Authentication Required:** Yes

#### Request Parameters

| Parameter | Type | In | Required | Description |
|-----------|------|----|----------|-------------|
| uuid | string (UUID) | path | Yes | Player's unique identifier |

#### Request Example
```
GET /api/players/550e8400-e29b-41d4-a716-446655440000/jump-bookmarks
```

#### Success Response (200 OK)
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "player_id": 42,
      "poi_id": 100,
      "name": "Rich Mining Belt",
      "x": 145,
      "y": 230,
      "notes": "Titanium deposits, bring large cargo",
      "created_at": "2026-02-27T10:00:00.000000Z",
      "updated_at": "2026-02-27T10:00:00.000000Z",
      "poi": {
        "id": 100,
        "uuid": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
        "name": "Asteroid Belt Sigma-7",
        "x": 145,
        "y": 230
      }
    },
    {
      "id": 2,
      "uuid": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "player_id": 42,
      "poi_id": null,
      "name": "Empty space rendezvous",
      "x": 50,
      "y": 75,
      "notes": null,
      "created_at": "2026-02-27T11:00:00.000000Z",
      "updated_at": "2026-02-27T11:00:00.000000Z",
      "poi": null
    }
  ],
  "message": "",
  "meta": {
    "timestamp": "2026-02-27T12:00:00+00:00",
    "request_id": "..."
  }
}
```

#### Error Responses

| Status | Code | Condition |
|--------|------|-----------|
| 404 | NOT_FOUND | Player UUID not found |

---

### 2. Create Bookmark

**POST** `/api/players/{uuid}/jump-bookmarks`

Creates a new jump bookmark for the player. Enforces the per-player bookmark limit from config.

**Authentication Required:** Yes (must be the player's owner)

#### Request Parameters

| Parameter | Type | In | Required | Description |
|-----------|------|----|----------|-------------|
| uuid | string (UUID) | path | Yes | Player's unique identifier |
| name | string | body | Yes | Bookmark label (max 100 characters) |
| x | integer | body | Yes | X coordinate |
| y | integer | body | Yes | Y coordinate |
| poi_id | integer | body | No | ID of a Point of Interest to link |
| notes | string | body | No | Freeform notes (max 500 characters) |

#### Request Example
```
POST /api/players/550e8400-e29b-41d4-a716-446655440000/jump-bookmarks
Content-Type: application/json

{
  "name": "Secret Mining Spot",
  "x": 200,
  "y": 310,
  "poi_id": 100,
  "notes": "Found rare exotic matter here"
}
```

#### Success Response (201 Created)
```json
{
  "success": true,
  "data": {
    "id": 3,
    "uuid": "d4e5f6a7-b8c9-0123-defa-234567890123",
    "player_id": 42,
    "poi_id": 100,
    "name": "Secret Mining Spot",
    "x": 200,
    "y": 310,
    "notes": "Found rare exotic matter here",
    "created_at": "2026-02-27T12:30:00.000000Z",
    "updated_at": "2026-02-27T12:30:00.000000Z",
    "poi": {
      "id": 100,
      "uuid": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "name": "Asteroid Belt Sigma-7",
      "x": 200,
      "y": 310
    }
  },
  "message": "Bookmark created",
  "meta": {
    "timestamp": "2026-02-27T12:30:00+00:00",
    "request_id": "..."
  }
}
```

#### Error Responses

| Status | Code | Condition |
|--------|------|-----------|
| 400 | BOOKMARK_LIMIT_REACHED | Player already has the maximum number of bookmarks (default: 25) |
| 403 | - | Authenticated user does not own this player |
| 404 | NOT_FOUND | Player UUID not found |
| 422 | VALIDATION_ERROR | Invalid input (missing name, non-integer coordinates, etc.) |

#### Limit Error Example (400)
```json
{
  "success": false,
  "error": {
    "code": "BOOKMARK_LIMIT_REACHED",
    "message": "Maximum of 25 bookmarks reached. Delete an existing bookmark first.",
    "details": null
  },
  "meta": {
    "timestamp": "2026-02-27T12:30:00+00:00",
    "request_id": "..."
  }
}
```

---

### 3. Update Bookmark

**PATCH** `/api/players/{uuid}/jump-bookmarks/{bookmarkUuid}`

Updates the name and/or notes of an existing bookmark. Coordinates and linked POI are immutable after creation.

**Authentication Required:** Yes (must be the player's owner)

#### Request Parameters

| Parameter | Type | In | Required | Description |
|-----------|------|----|----------|-------------|
| uuid | string (UUID) | path | Yes | Player's unique identifier |
| bookmarkUuid | string (UUID) | path | Yes | Bookmark's unique identifier |
| name | string | body | No | New bookmark label (max 100 characters) |
| notes | string or null | body | No | New notes (max 500 characters), or null to clear |

#### Request Example
```
PATCH /api/players/550e8400-e29b-41d4-a716-446655440000/jump-bookmarks/d4e5f6a7-b8c9-0123-defa-234567890123
Content-Type: application/json

{
  "name": "Exotic Matter Hotspot",
  "notes": "Confirmed 3x exotic matter spawns"
}
```

#### Success Response (200 OK)
```json
{
  "success": true,
  "data": {
    "id": 3,
    "uuid": "d4e5f6a7-b8c9-0123-defa-234567890123",
    "player_id": 42,
    "poi_id": 100,
    "name": "Exotic Matter Hotspot",
    "x": 200,
    "y": 310,
    "notes": "Confirmed 3x exotic matter spawns",
    "created_at": "2026-02-27T12:30:00.000000Z",
    "updated_at": "2026-02-27T13:00:00.000000Z",
    "poi": {
      "id": 100,
      "uuid": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "name": "Asteroid Belt Sigma-7",
      "x": 200,
      "y": 310
    }
  },
  "message": "Bookmark updated",
  "meta": {
    "timestamp": "2026-02-27T13:00:00+00:00",
    "request_id": "..."
  }
}
```

#### Error Responses

| Status | Code | Condition |
|--------|------|-----------|
| 403 | - | Authenticated user does not own this player |
| 404 | NOT_FOUND | Player UUID or bookmark UUID not found, or bookmark does not belong to this player |
| 422 | VALIDATION_ERROR | Invalid input (name too long, notes too long) |

---

### 4. Delete Bookmark

**DELETE** `/api/players/{uuid}/jump-bookmarks/{bookmarkUuid}`

Permanently deletes a bookmark.

**Authentication Required:** Yes (must be the player's owner)

#### Request Parameters

| Parameter | Type | In | Required | Description |
|-----------|------|----|----------|-------------|
| uuid | string (UUID) | path | Yes | Player's unique identifier |
| bookmarkUuid | string (UUID) | path | Yes | Bookmark's unique identifier |

#### Request Example
```
DELETE /api/players/550e8400-e29b-41d4-a716-446655440000/jump-bookmarks/d4e5f6a7-b8c9-0123-defa-234567890123
```

#### Success Response (200 OK)
```json
{
  "success": true,
  "data": null,
  "message": "Bookmark deleted",
  "meta": {
    "timestamp": "2026-02-27T13:15:00+00:00",
    "request_id": "..."
  }
}
```

#### Error Responses

| Status | Code | Condition |
|--------|------|-----------|
| 403 | - | Authenticated user does not own this player |
| 404 | NOT_FOUND | Player UUID or bookmark UUID not found, or bookmark does not belong to this player |
