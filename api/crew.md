# Crew System API Reference

Complete API documentation for the company crew system in Space Wars 3002.

**Authentication**: All endpoints require authentication via Laravel Sanctum (Bearer token).

**Base URL**: `/api`

---

## Table of Contents

1. [Overview](#overview)
2. [Crew Endpoints](#crew-endpoints)
   - [List Crew](#list-crew)
   - [Hire Crew Member](#hire-crew-member)
   - [Get Crew Member](#get-crew-member)
   - [Fire Crew Member](#fire-crew-member)
   - [Assign to Ship](#assign-to-ship)
   - [Unassign from Ship](#unassign-from-ship)
3. [Ship Persona](#ship-persona)
   - [Get Ship Persona](#get-ship-persona)
4. [Crew Roles](#crew-roles)
5. [Pay Mechanics](#pay-mechanics)
6. [Negotiation Discounts](#negotiation-discounts)

---

## Overview

Players are company owners who employ crew members. Crew members are hired into the company roster and can be assigned to ships. Each crew member has a role, randomized stats, and a pay rate.

**Key concepts:**
- **Company roster**: All hired crew, both assigned and benched. Capped at `max_crew` (default 10).
- **Assigned vs benched**: Crew assigned to a ship contribute to its persona and earn pay per hop. Unassigned crew are benched.
- **Ship persona**: Aggregate personality derived from assigned crew stats (risk, cool, luck, knowledge domains).
- **Crew pay**: Deducted from player credits after each travel hop, proportional to XP earned.
- **Negotiation discounts**: Crew luck + experience reduces purchase prices and increases swap sale values at salvage yards.

---

## Crew Endpoints

### List Crew

Get all crew members in the player's company roster.

**Endpoint**: `GET /api/players/{uuid}/crew`

**Auth Required**: Yes (player must belong to authenticated user)

#### Request Parameters

| Parameter | Type   | Required | Location | Description  |
|-----------|--------|----------|----------|--------------|
| uuid      | string | Yes      | Path     | Player UUID  |

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "crew": [
      {
        "uuid": "a1b2c3d4-...",
        "role": "pilot",
        "role_label": "Pilot",
        "name": "Brex Olin",
        "luck": 0.78,
        "risk": 0.45,
        "cool": 0.62,
        "knowledge": 0.55,
        "pay_rate": 0.03,
        "morale": 0.70,
        "experience": 0.12,
        "status": "active",
        "hired_at": "2026-02-27T10:00:00+00:00",
        "last_active_at": "2026-02-27T12:30:00+00:00",
        "assigned_ship": {
          "uuid": "e5f6a7b8-...",
          "name": "Stardust Runner"
        },
        "negotiation_discount": 0.0094
      }
    ],
    "max_crew": 10,
    "total": 3
  }
}
```

---

### Hire Crew Member

Hire a new randomly generated crew member into the company roster.

**Endpoint**: `POST /api/players/{uuid}/crew/hire`

**Auth Required**: Yes

#### Request Body

| Parameter | Type   | Required | Description                                                   |
|-----------|--------|----------|---------------------------------------------------------------|
| role      | string | Yes      | `pilot`, `weapons_officer`, `science_officer`, `chief_engineer` |

#### Success Response (201 Created)

Returns the newly hired crew member (same format as list items above).

#### Error Responses

| Code | Error Code           | Condition                            |
|------|----------------------|--------------------------------------|
| 422  | CREW_ROSTER_FULL     | Roster at max capacity               |
| 422  | INSUFFICIENT_CREDITS | Not enough credits for hiring fee    |

**Hiring cost**: Configured in `game_config.crew.hire_cost` (default: 500 credits).

**Stat generation**: All stats (luck, risk, cool, knowledge) are rolled randomly between 0.00 and 1.00. Pay rate is rolled between 0.01 and the role's max pay rate.

---

### Get Crew Member

Get detailed information about a specific crew member.

**Endpoint**: `GET /api/players/{uuid}/crew/{crewUuid}`

**Auth Required**: Yes

#### Request Parameters

| Parameter | Type   | Required | Location | Description      |
|-----------|--------|----------|----------|------------------|
| uuid      | string | Yes      | Path     | Player UUID      |
| crewUuid  | string | Yes      | Path     | Crew member UUID |

#### Success Response (200 OK)

Returns a single crew member object (same format as list items).

---

### Fire Crew Member

Fire a crew member from the company roster.

**Endpoint**: `DELETE /api/players/{uuid}/crew/{crewUuid}/fire`

**Auth Required**: Yes

The crew member's status is set to `fired` and they are unassigned from any ship. If they were assigned, the ship's persona is recalculated.

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": null,
  "message": "Crew member Brex Olin has been fired."
}
```

---

### Assign to Ship

Assign a crew member to one of the player's ships.

**Endpoint**: `POST /api/players/{uuid}/crew/{crewUuid}/assign`

**Auth Required**: Yes

#### Request Body

| Parameter | Type   | Required | Description      |
|-----------|--------|----------|------------------|
| ship_uuid | string | Yes      | Target ship UUID |

The ship's persona is recalculated after assignment.

#### Error Responses

| Code | Error Code      | Condition                |
|------|-----------------|--------------------------|
| 400  | CREW_NOT_ACTIVE | Crew member is not active |
| 404  | NOT_FOUND       | Ship not found            |

#### Success Response (200 OK)

Returns the updated crew member object with `assigned_ship` populated.

---

### Unassign from Ship

Move a crew member from their assigned ship to the bench.

**Endpoint**: `POST /api/players/{uuid}/crew/{crewUuid}/unassign`

**Auth Required**: Yes

The ship's persona is recalculated after unassignment.

#### Error Responses

| Code | Error Code       | Condition                       |
|------|------------------|---------------------------------|
| 400  | CREW_NOT_ASSIGNED| Crew member not on any ship     |

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": null,
  "message": "Brex Olin moved to bench."
}
```

---

## Ship Persona

### Get Ship Persona

Get the aggregate personality profile for a ship based on its assigned crew.

**Endpoint**: `GET /api/ships/{uuid}/persona`

**Auth Required**: Yes (ship must belong to authenticated user)

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "persona": {
      "risk_factor": 0.52,
      "cool_factor": 0.68,
      "luck_factor": 0.41,
      "system_knowledge": 1.10,
      "weapon_knowledge": 0.75,
      "sensor_knowledge": 0.88,
      "engine_knowledge": 0.62,
      "persona_label": "Steady Professionals"
    },
    "crew_count": 3
  }
}
```

**Persona label examples**: "Reckless Hotshots", "Ice-Cold Aces", "Bold Daredevils", "Lucky Charmers", "Green Crew"

---

## Crew Roles

| Role              | Value              | Max Pay Rate | Negotiates For         | Max Discount |
|-------------------|--------------------|-------------|------------------------|--------------|
| Pilot             | `pilot`            | 5%          | All components (general)| 2%           |
| Weapons Officer   | `weapons_officer`  | 1%          | Weapon slots           | 5%           |
| Science Officer   | `science_officer`  | 2%          | Sensor array slots     | 5%           |
| Chief Engineer    | `chief_engineer`   | 10%         | Engine, Reactor, Cargo | 5%           |

---

## Pay Mechanics

Crew are paid after each travel hop (warp gate or coordinate jump). Pay is deducted from the player's credits.

**Per-hop pay formula:**
```
base_pay = xp_earned × crew.pay_rate
combat_bonus = xp_earned × 0.15 (weapons officers only, if combat occurred)
total_pay = base_pay + combat_bonus
```

The `crew_pay` object is included in travel responses:

```json
{
  "crew_pay": {
    "total_deducted": 12.50,
    "breakdown": [
      {
        "crew_uuid": "a1b2c3d4-...",
        "name": "Brex Olin",
        "role": "pilot",
        "base_pay": 7.50,
        "combat_bonus": 0,
        "total_pay": 7.50
      }
    ]
  }
}
```

Crew members also gain +0.001 experience per hop (capped at 1.0).

---

## Negotiation Discounts

Crew members provide discounts at salvage yards based on their role, luck, and experience.

**Discount formula:**
```
discount = role_max_discount × crew.luck × crew.experience
```

Discounts apply in two places:
1. **Purchase discount**: Reduces the price of components being bought.
2. **Swap sale bonus**: Increases the sell percentage when replacing a core component (added on top of the base 50% sell rate).

The best applicable crew member's discount is used (not stacked).
