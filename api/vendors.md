# Vendor Profiles & Negotiation API Reference

Complete API documentation for the vendor profile and negotiation system in Space Wars 3002.

**Authentication**: All endpoints require authentication via Laravel Sanctum (Bearer token).

**Base URL**: `/api`

---

## Table of Contents

1. [Overview](#overview)
2. [Vendor Endpoints](#vendor-endpoints)
   - [Get Vendor Profile](#get-vendor-profile)
   - [Negotiate Price](#negotiate-price)
3. [Vendor Impact on Existing Endpoints](#vendor-impact-on-existing-endpoints)
   - [Trading (Buy/Sell Minerals)](#trading-buysell-minerals)
   - [Salvage Yard Purchases](#salvage-yard-purchases)
   - [Ship Shop Purchases](#ship-shop-purchases)
   - [Plans Shop Purchases](#plans-shop-purchases)
   - [Ship Repairs](#ship-repairs)
   - [Crew Hiring (Pay Negotiation)](#crew-hiring-pay-negotiation)
4. [Data Types](#data-types)
   - [Vendor Archetypes](#vendor-archetypes)
   - [Negotiation Outcomes](#negotiation-outcomes)

---

## Overview

Every trading hub in the galaxy has a unique vendor with a distinct personality that affects all commerce at that location. Vendor personalities are defined by four traits (greed, business acumen, luck, apathy) that drive pricing, negotiation difficulty, and markup decay.

**Key concepts:**
- **Vendor markup**: Vendors add a greed-based markup to all prices at their trading hub. Greedy vendors charge more.
- **Relationship escalation**: Each time a player buys at the asking price, the vendor's markup escalates for that player. Greedy vendors escalate faster.
- **Markup decay**: Markup decreases over time (driven by vendor apathy) and through spending milestones (goodwill).
- **Active negotiation**: Players can propose a lower price. Success depends on crew stats vs. vendor resistance.
- **Precursor protection**: Attempting to negotiate on precursor items results in a permanent markup penalty.
- **Lockout**: Predatory proposals (< 20% of base price) lock the player out of the vendor for 24-48 hours.
- **Vendor dialogue**: Optional LLM-powered dialogue gives each vendor a unique voice. Falls back to static commentary when unavailable.

---

## Vendor Endpoints

### Get Vendor Profile

Get the vendor profile, relationship state, and greeting for a trading hub.

**Endpoint**: `GET /api/trading-hubs/{uuid}/vendor`

**Auth Required**: Yes

#### Request Parameters

| Parameter | Type   | Required | Location | Description        |
|-----------|--------|----------|----------|--------------------|
| uuid      | string | Yes      | Path     | Trading hub UUID   |

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "Vex Korrath",
    "archetype": "opportunist",
    "archetype_label": "Opportunist",
    "relationship": {
      "markup_escalation_pct": 3.5,
      "goodwill": 0.10,
      "interaction_count": 12,
      "is_locked_out": false,
      "locked_until": null
    },
    "greeting": "Another customer, another opportunity..."
  }
}
```

**Notes:**
- The `relationship` and `greeting` fields are only included when the request is authenticated with a player.
- `markup_escalation_pct` is the current extra markup percentage applied to this player's prices at this vendor (e.g., `3.5` means 3.5% extra).
- `goodwill` ranges from -1.0 to 1.0. Positive goodwill is earned through spending milestones.
- `greeting` is an LLM-generated or static greeting from the vendor, personalized by archetype.
- Raw trait values (greed, business_acumen, etc.) are intentionally **not** exposed to players.

#### Error Responses

| Code | Condition                          |
|------|------------------------------------|
| 404  | Trading hub not found              |
| 404  | No vendor at this trading hub      |

---

### Negotiate Price

Attempt to negotiate a lower price with a vendor.

**Endpoint**: `POST /api/players/{uuid}/vendors/{vendorUuid}/negotiate`

**Auth Required**: Yes (player must belong to authenticated user)

#### Request Body

| Parameter      | Type    | Required | Description                                             |
|----------------|---------|----------|---------------------------------------------------------|
| context        | string  | Yes      | Shop type: `salvage`, `ship`, `plan`, `mineral`, `repair` |
| item_id        | string  | No       | Optional identifier for the specific item               |
| proposed_price | numeric | Yes      | The price the player wants to pay                       |
| base_price     | numeric | Yes      | The original base price before vendor markup (min: 0)   |
| is_precursor   | boolean | No       | Whether the item is a precursor artifact (default: false)|

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "outcome": "success",
    "outcome_label": "Success",
    "final_price": 850.00,
    "discount_applied": 0.1500,
    "vendor_reaction": "Fine, you win this one.",
    "markup_penalty": null,
    "locked_until": null
  }
}
```

#### Outcome Variations

**Successful negotiation** (`outcome: "success"`):
```json
{
  "outcome": "success",
  "outcome_label": "Success",
  "final_price": 850.00,
  "discount_applied": 0.1500,
  "vendor_reaction": "Deal. Don't tell anyone what you paid.",
  "markup_penalty": null,
  "locked_until": null
}
```

**Partial success** (`outcome: "partial"`):
```json
{
  "outcome": "partial",
  "outcome_label": "Partial Success",
  "final_price": 925.00,
  "discount_applied": 0.0750,
  "vendor_reaction": "Meet me halfway, and we're done.",
  "markup_penalty": null,
  "locked_until": null
}
```

**Failed negotiation** (`outcome: "fail"`):
```json
{
  "outcome": "fail",
  "outcome_label": "Failed",
  "final_price": 1000.00,
  "discount_applied": 0.0,
  "vendor_reaction": "The price stands. Take it or leave it.",
  "markup_penalty": null,
  "locked_until": null
}
```

**Precursor insult** (`outcome: "insulted"`):
```json
{
  "outcome": "insulted",
  "outcome_label": "Vendor Insulted",
  "final_price": 1000.00,
  "discount_applied": 0.0,
  "vendor_reaction": "That artifact is priceless. Your insult, however, will cost you.",
  "markup_penalty": 0.0312,
  "locked_until": null
}
```

**Lockout** (`outcome: "lockout"`):
```json
{
  "outcome": "lockout",
  "outcome_label": "Locked Out",
  "final_price": 1000.00,
  "discount_applied": 0.0,
  "vendor_reaction": "Get out. Come back when you've learned some respect.",
  "markup_penalty": null,
  "locked_until": "2026-02-28T14:30:00+00:00"
}
```

#### Error Responses

| Code | Condition                          |
|------|------------------------------------|
| 404  | Player not found                   |
| 404  | Vendor not found                   |
| 422  | Validation error (missing fields)  |

---

## Vendor Impact on Existing Endpoints

The vendor system modifies pricing across all shop types at a trading hub. Below is how each existing endpoint is affected.

### Trading (Buy/Sell Minerals)

**Endpoint**: `POST /api/trading-hubs/{uuid}/buy`

Mineral buy prices now include vendor markup. The vendor at the trading hub applies their greed-based markup plus any relationship escalation on top of the inventory's `sell_price`.

**Price formula:**
```
vendor_markup   = greed * 0.15
escalation      = player's markup_escalation at this vendor
final_buy_price = sell_price * (1 + vendor_markup) * (1 + escalation)
```

Sell prices (player selling to vendor) are not affected by vendor markup.

---

### Salvage Yard Purchases

**Endpoint**: `POST /api/players/{uuid}/salvage-yard/purchase`

Component prices at salvage yards include vendor markup. The vendor markup is applied **before** crew negotiation discounts (multiplicative stacking):

```
quoted_price = base_price * (1 + vendor_markup) * (1 + escalation)
crew_discount = best crew member's negotiation discount
final_price  = quoted_price * (1 - crew_discount)
```

---

### Ship Shop Purchases

**Endpoint**: `POST /api/players/{uuid}/ship-shop/purchase`

Ship purchase prices include vendor markup from the vendor at the player's current trading hub.

---

### Plans Shop Purchases

**Endpoint**: `POST /api/players/{uuid}/plans/purchase`

Upgrade plan prices include vendor markup from the vendor at the specified trading hub.

---

### Ship Repairs

**Endpoints**:
- `GET /api/ships/{uuid}/repair-estimate`
- `POST /api/ships/{uuid}/repair-hull`
- `POST /api/ships/{uuid}/repair-components`
- `POST /api/ships/{uuid}/repair-all`

Repair costs are adjusted by vendor greed. The hull repair rate per point is:

```
vendor_rate = base_rate * (1 + greed * max_repair_markup)
```

Where `base_rate` = 10 credits/point and `max_repair_markup` = 0.50 (50%).

The repair estimate response now includes vendor pricing information:

```json
{
  "success": true,
  "data": {
    "hull_damage": 45,
    "hull_repair_cost": 585,
    "needs_hull_repair": true,
    "downgraded_components": [],
    "component_repair_cost": 0,
    "needs_component_repair": false,
    "total_repair_cost": 585,
    "can_negotiate": true,
    "vendor_rate": 13.0,
    "base_rate": 10,
    "vendor_markup_pct": 30.0
  }
}
```

---

### Crew Hiring (Pay Negotiation)

**Endpoint**: `POST /api/players/{uuid}/crew/hire`

An optional `proposed_pay_rate` field allows players to negotiate crew pay during hiring. Each crew candidate acts as their own "vendor" — their stats drive the negotiation.

#### Additional Request Parameter

| Parameter         | Type    | Required | Description                              |
|-------------------|---------|----------|------------------------------------------|
| proposed_pay_rate | numeric | No       | Proposed pay rate (0.01 - role max)      |

#### Negotiation Roll

```
candidate_resistance = experience * (1 - (morale - 0.5) * 0.3)
player_power         = player_level / 20
luck_swing           = (candidate.luck - 0.5) * 0.2
roll                 = player_power - candidate_resistance + luck_swing + random(-0.10, 0.10)
```

**Outcomes:**
- **roll > 0.30** (`accepted`): Candidate accepts the proposed pay rate
- **roll > 0.00** (`counter`): Candidate counters at the average of proposed and rolled rates
- **roll <= 0.00** (`rejected`): Candidate insists on their rolled rate

**Walk-away**: If `proposed_pay_rate < 0.01`, the candidate refuses and the hire fails with error code `CREW_WALKAWAY`.

#### Response with Negotiation

When `proposed_pay_rate` is provided, the response includes additional fields:

```json
{
  "success": true,
  "data": {
    "uuid": "a1b2c3d4-...",
    "role": "pilot",
    "name": "Kira Solenn",
    "luck": 0.65,
    "risk": 0.42,
    "cool": 0.78,
    "knowledge": 0.55,
    "pay_rate": 0.025,
    "morale": 0.70,
    "experience": 0.0,
    "negotiation_outcome": "accepted",
    "original_pay_rate": 0.035,
    "final_pay_rate": 0.025
  }
}
```

---

## Data Types

### Vendor Archetypes

Each vendor is assigned an archetype based on their dominant personality trait(s). Archetypes determine greeting flavor text and general behavior patterns.

| Archetype       | Value            | Dominant Traits                  | Behavior                                    |
|-----------------|------------------|----------------------------------|---------------------------------------------|
| Opportunist     | `opportunist`    | High greed                       | Aggressive markup, escalates fast            |
| Professional    | `professional`   | High business acumen             | Fair baseline, tough negotiator              |
| Lucky Fence     | `lucky_fence`    | High luck                        | Erratic pricing, occasionally great deals    |
| Indifferent     | `indifferent`    | High apathy                      | Barely marks up, barely reacts               |
| Shark           | `shark`          | High greed + high business acumen| Max markup, hardest to negotiate             |
| Pushover        | `pushover`       | Low business acumen + high apathy| Easy to negotiate, doesn't escalate          |
| Paranoid        | `paranoid`       | High business acumen + low apathy| Remembers everything, slow to forgive        |
| Gambler         | `gambler`        | High luck + low business acumen  | Wild outcome swings                          |
| Honest Dealer   | `honest_dealer`  | Balanced (all within 0.3-0.7)    | Moderate markup, fair negotiation            |

### Negotiation Outcomes

| Outcome   | Value       | Description                                              |
|-----------|-------------|----------------------------------------------------------|
| Success   | `success`   | Player gets their proposed price (floored at 50% of base)|
| Partial   | `partial`   | Price splits the difference between quoted and proposed   |
| Fail      | `fail`      | Price unchanged, no penalty                              |
| Insulted  | `insulted`  | Precursor negotiation attempt, permanent markup penalty   |
| Lockout   | `lockout`   | Predatory proposal, vendor locked for 24-48 hours        |
