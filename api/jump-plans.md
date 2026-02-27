# Jump Plans API Reference

Complete API documentation for multi-hop route planning in Space Wars 3002.

**Authentication**: All endpoints require authentication via Laravel Sanctum (Bearer token).

**Base URL**: `/api`

---

## Table of Contents

1. [Overview](#overview)
2. [Endpoints](#endpoints)
   - [Create Jump Plan](#create-jump-plan)
   - [Get Active Plan](#get-active-plan)
   - [Execute Next Hop](#execute-next-hop)
   - [Abandon Plan](#abandon-plan)
3. [Pathfinding Algorithm](#pathfinding-algorithm)
4. [Hop Types](#hop-types)
5. [Pirate Interruption](#pirate-interruption)
6. [Plan Statuses](#plan-statuses)

---

## Overview

Jump Plans provide multi-hop route planning from the player's current position to a target coordinate. The planner combines warp gate hops (cheap, fast) with coordinate jumps (expensive, flexible) to build an optimal route.

**Key concepts:**
- **One active plan at a time**: Players must abandon their current plan before creating a new one.
- **Step-by-step execution**: Each hop is executed individually via the `execute-next-hop` endpoint.
- **Pirate interruption**: Before each hop, the game checks for pirate encounters. If pirates appear, the plan is interrupted and must be resolved before continuing.
- **Fuel per hop**: Each hop has a pre-calculated fuel cost. The plan's `total_fuel_cost` is the sum of all hops.
- **Crew pay**: Crew members are paid after each hop (included in the response).

---

## Endpoints

### Create Jump Plan

Plan a multi-hop route from the player's current position to target coordinates.

**Endpoint**: `POST /api/players/{uuid}/jump-plans`

**Auth Required**: Yes

#### Request Body

| Parameter | Type    | Required | Description            |
|-----------|---------|----------|------------------------|
| target_x  | integer | Yes      | Target X coordinate    |
| target_y  | integer | Yes      | Target Y coordinate    |

#### Success Response (201 Created)

```json
{
  "success": true,
  "data": {
    "uuid": "plan-uuid-here",
    "target": [250, 180],
    "total_hops": 5,
    "total_fuel_cost": 142,
    "hops": [
      {
        "step": 1,
        "type": "gate",
        "from": [100, 100],
        "to": [65, 80],
        "gate_uuid": "gate-uuid-here",
        "poi_name": "Sigma Station",
        "distance": 41.23,
        "fuel_cost": 12
      },
      {
        "step": 2,
        "type": "coordinate",
        "from": [65, 80],
        "to": [110, 120],
        "gate_uuid": null,
        "poi_name": null,
        "distance": 56.57,
        "fuel_cost": 45
      }
    ],
    "status": "active"
  },
  "message": "Jump plan created"
}
```

#### Error Responses

| Code | Error Code       | Condition                                     |
|------|------------------|-----------------------------------------------|
| 400  | JUMP_PLAN_FAILED | No active ship, no location, or active plan exists |

---

### Get Active Plan

Get the player's currently active jump plan.

**Endpoint**: `GET /api/players/{uuid}/jump-plans/active`

**Auth Required**: Yes

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "uuid": "plan-uuid-here",
    "target": [250, 180],
    "status": "active",
    "current_step": 2,
    "total_hops": 5,
    "remaining_hops": 3,
    "total_fuel_cost": 142,
    "interrupted_by": null,
    "hops": [ ... ],
    "created_at": "2026-02-27T10:00:00+00:00"
  }
}
```

#### Error Response

| Code | Error Code | Condition           |
|------|------------|---------------------|
| 404  | NOT_FOUND  | No active jump plan |

---

### Execute Next Hop

Execute the next step in the active jump plan. Each call advances one hop.

**Endpoint**: `POST /api/players/{uuid}/jump-plans/{planUuid}/execute-next-hop`

**Auth Required**: Yes

Before executing the hop, the system checks for pirate encounters. If pirates are detected, the plan is interrupted and the encounter details are returned instead.

#### Success Response — Hop Completed (200 OK)

```json
{
  "success": true,
  "data": {
    "hop_result": {
      "success": true,
      "message": "Travel successful",
      "destination": "Alpha Station",
      "distance": 41.23,
      "fuel_cost": 12,
      "fuel_remaining": 88,
      "xp_earned": 206,
      "old_level": 3,
      "new_level": 3,
      "leveled_up": false,
      "crew_pay": {
        "total_deducted": 6.18,
        "breakdown": [ ... ]
      }
    },
    "crew_pay": {
      "total_deducted": 6.18,
      "breakdown": [ ... ]
    },
    "plan": {
      "uuid": "plan-uuid-here",
      "status": "active",
      "current_step": 3,
      "total_hops": 5,
      "remaining_hops": 2
    }
  },
  "message": "Hop completed"
}
```

#### Success Response — Plan Completed (200 OK)

When the last hop is executed, the plan status changes to `completed`:

```json
{
  "message": "Jump plan completed"
}
```

#### Success Response — Pirate Interruption (200 OK)

```json
{
  "success": true,
  "data": {
    "interrupted": true,
    "interrupted_by": "pirate",
    "pirate_band": {
      "uuid": "pirate-band-uuid",
      "name": "Shadow Reapers",
      "threat_level": "dangerous"
    },
    "plan": {
      "uuid": "plan-uuid-here",
      "status": "interrupted",
      "current_step": 2,
      "remaining_hops": 3,
      "interrupted_by": "pirate"
    }
  },
  "message": "Jump plan interrupted by pirates!"
}
```

#### Error Responses

| Code | Error Code     | Condition                |
|------|----------------|--------------------------|
| 400  | PLAN_NOT_ACTIVE| Plan is not active       |
| 400  | NO_ACTIVE_SHIP | No active ship           |
| 400  | HOP_FAILED     | Hop execution failed     |

---

### Abandon Plan

Abandon the active jump plan. The player remains at their current location.

**Endpoint**: `DELETE /api/players/{uuid}/jump-plans/{planUuid}`

**Auth Required**: Yes

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": null,
  "message": "Jump plan abandoned."
}
```

#### Error Responses

| Code | Error Code     | Condition          |
|------|----------------|--------------------|
| 400  | PLAN_NOT_ACTIVE| Plan is not active |
| 404  | NOT_FOUND      | Plan not found     |

---

## Pathfinding Algorithm

The `JumpPlannerService` uses a greedy best-first approach:

1. Start at the player's current position.
2. **Loop** while distance to target > 1 unit:
   a. Load active, non-hidden warp gates from the current POI (if at a POI).
   b. For each gate, check if the far end is at least 5 units closer to the target.
   c. If a gate is accepted, add a **gate hop** and move to the far end.
   d. Otherwise, add a **coordinate hop**: jump along the line toward the target, limited by the ship's max jump distance.
3. Return the ordered hop list.

**Max jump distance** depends on ship warp drive level:
```
max_distance = base_max_distance + distance_per_warp_level × (warp_level - 1)
```

Default: 5 units base + 5 units per level (level 1 = 5, level 5 = 25).

---

## Hop Types

| Type         | Description                              | Fuel Cost Formula                |
|--------------|------------------------------------------|----------------------------------|
| `gate`       | Travel through a warp gate               | Standard gate fuel cost          |
| `coordinate` | Direct jump to coordinates               | Direct jump cost (4x penalty)    |

Gate hops are significantly cheaper due to the fuel penalty on coordinate jumps.

---

## Pirate Interruption

Before each hop execution, `PirateEncounterService::checkSectorPirateEncounter()` is called. This checks for pirate bands in the sector and rolls a probability-based encounter.

If pirates are encountered:
1. The hop is **not executed** (player stays in place).
2. The plan status changes to `interrupted`.
3. The pirate band details are returned.
4. The player must resolve the encounter (fight, escape, or surrender) using the existing combat endpoints.
5. After resolution, the player can create a new plan to continue their journey.

---

## Plan Statuses

| Status        | Description                                      |
|---------------|--------------------------------------------------|
| `active`      | Plan is in progress, hops can be executed         |
| `completed`   | All hops executed successfully                    |
| `abandoned`   | Player manually abandoned the plan                |
| `interrupted` | Pirate encounter halted the plan mid-route        |
