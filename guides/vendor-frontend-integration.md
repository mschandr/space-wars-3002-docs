# Vendor System — Frontend Integration Guide

> **Audience**: Frontend AI / FE developer implementing vendor interactions
> **Last Updated**: 2026-03-16
> **API Version**: V1 (Sanctum-authenticated)

This guide covers the complete vendor interaction model — how to move from a player arriving at a trading hub through dialogue, browsing, trading, and negotiating. All prices at every shop type (minerals, repairs, ships, plans, salvage) are affected by the vendor at the current trading hub.

---

## Table of Contents

1. [Mental Model](#mental-model)
2. [Hub Arrival Flow](#hub-arrival-flow)
3. [Endpoint Quick Reference](#endpoint-quick-reference)
4. [Step-by-Step Interaction Flows](#step-by-step-interaction-flows)
   - [Arriving at a Trading Hub](#arriving-at-a-trading-hub)
   - [Trading Minerals](#trading-minerals)
   - [Ship Repairs](#ship-repairs)
   - [Salvage Yard](#salvage-yard)
   - [Ship Shop](#ship-shop)
   - [Plans Shop](#plans-shop)
   - [Hiring Crew](#hiring-crew)
5. [Negotiation Flow](#negotiation-flow)
6. [Vendor Archetypes — UI Hints](#vendor-archetypes--ui-hints)
7. [Criminality Signal](#criminality-signal)
8. [Dialogue Status Handling](#dialogue-status-handling)
9. [Lockout State Management](#lockout-state-management)
10. [Price Formula Reference](#price-formula-reference)
11. [Key Implementation Notes](#key-implementation-notes)

---

## Mental Model

Every trading hub has exactly **one vendor** (a `GalaxyVendorProfile`). The vendor is:

- **Persistent** — they remember you (markup escalates with each un-negotiated purchase)
- **Unique per galaxy** — same archetype/name, but a separate relationship state per galaxy instance
- **Priced per archetype** — greedy vendors charge more; indifferent vendors barely mark up
- **Negotiable** — players can propose a lower price before any purchase

The vendor affects **all** shop types at the hub: minerals, repairs, ships, plans, and salvage.

### Key IDs

| Concept | UUID field | Used in |
|---------|-----------|---------|
| Trading hub | `trading_hub.uuid` | Buy/sell, repair, ship shop, plans shop, salvage yard |
| GalaxyVendorProfile | `vendor.uuid` | Dialogue endpoint, negotiate endpoint |
| Player | `player.uuid` | Cargo, crew, negotiate, dialogue |

---

## Hub Arrival Flow

When a player docks at a trading hub, the recommended sequence is:

```
1. GET /api/trading-hubs/{hubUuid}/vendor        → Load vendor profile + greeting
2. GET /api/players/{playerUuid}/vendors/{vendorUuid}/dialogue  → Fetch contextual greeting line
3. [Show vendor greeting in UI]
4. Player navigates to a shop area (trade/repair/ships/plans/salvage)
5. Load shop inventory for that area
6. Player selects item → optionally negotiate → confirm purchase
```

Step 2 is optional if you already have a greeting from step 1, but the dedicated dialogue endpoint gives a richer, context-aware line (first visit vs. returning customer).

---

## Endpoint Quick Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| Get vendor profile + relationship | `GET` | `/api/trading-hubs/{uuid}/vendor` |
| Get contextual greeting | `GET` | `/api/players/{playerUuid}/vendors/{vendorUuid}/dialogue` |
| Negotiate a price | `POST` | `/api/players/{uuid}/vendors/{vendorUuid}/negotiate` |
| Get trading hub inventory | `GET` | `/api/trading-hubs/{uuid}/inventory` |
| Buy minerals | `POST` | `/api/trading-hubs/{uuid}/buy` |
| Sell minerals | `POST` | `/api/trading-hubs/{uuid}/sell` |
| Repair estimate | `GET` | `/api/ships/{uuid}/repair-estimate` |
| Repair hull | `POST` | `/api/ships/{uuid}/repair-hull` |
| Repair components | `POST` | `/api/ships/{uuid}/repair-components` |
| Repair all | `POST` | `/api/ships/{uuid}/repair-all` |
| Ship shop inventory | `GET` | `/api/players/{uuid}/ship-shop` |
| Purchase ship | `POST` | `/api/players/{uuid}/ship-shop/purchase` |
| Salvage yard inventory | `GET` | `/api/players/{uuid}/salvage-yard` |
| Purchase salvage component | `POST` | `/api/players/{uuid}/salvage-yard/purchase` |
| Plans shop inventory | `GET` | `/api/players/{uuid}/plans` |
| Purchase plan | `POST` | `/api/players/{uuid}/plans/purchase` |
| List available crew | `GET` | `/api/players/{uuid}/crew/available` |
| Hire crew member | `POST` | `/api/players/{uuid}/crew/hire` |

---

## Step-by-Step Interaction Flows

### Arriving at a Trading Hub

#### 1. Load Vendor Profile

```
GET /api/trading-hubs/{hubUuid}/vendor
Authorization: Bearer {token}
```

**Response** (`200 OK`):
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

**What to do with this:**
- Store `vendor.uuid` — you'll need it for dialogue and negotiation calls
- Check `relationship.is_locked_out` — if `true`, disable all purchase actions and show lockout UI (see [Lockout State Management](#lockout-state-management))
- Display `archetype_label` as a subtitle or flavor badge (e.g. "⚠ Opportunist")
- `markup_escalation_pct` tells the player how much extra they're currently paying at this vendor

#### 2. Fetch Contextual Greeting

```
GET /api/players/{playerUuid}/vendors/{vendorUuid}/dialogue
Authorization: Bearer {token}
```

**Response** (`200 OK`):
```json
{
  "success": true,
  "data": {
    "vendor": {
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "service_type": "trading_hub",
      "criminality": 0.27
    },
    "dialogue": {
      "greeting": "Back again? Either I impress you or concern you."
    },
    "interaction_bucket": "repeat_customer",
    "dialogue_status": "complete"
  }
}
```

**What to do with this:**
- Display `dialogue.greeting` as the vendor's opening line in the interaction panel
- Use `interaction_bucket` to choose greeting animation variant (first visit = wider animation, returning customer = brief nod)
- Check `dialogue_status` — if `pending` or `failed`, a static fallback was used; no special UI needed unless you want to indicate AI dialogue isn't ready

---

### Trading Minerals

#### 1. Load Inventory

```
GET /api/trading-hubs/{hubUuid}/inventory
Authorization: Bearer {token}
```

Inventory items include vendor-adjusted `sell_price` (price player pays). The raw `base_price` and vendor markup are embedded.

#### 2. (Optional) Negotiate Before Buying

Before calling `/buy`, players may call negotiate with `context: "mineral"`. See [Negotiation Flow](#negotiation-flow).

If negotiation succeeds, use the returned `final_price` as the `price` field in the buy request (see negotiation notes below — the API automatically uses the negotiated price for the next transaction).

#### 3. Buy Minerals

```
POST /api/trading-hubs/{hubUuid}/buy
Authorization: Bearer {token}

{
  "mineral_uuid": "...",
  "quantity": 50
}
```

**Note:** Each un-negotiated buy increases the player's `markup_escalation_pct` at this vendor. Encourage negotiation for large purchases.

#### 4. Sell Minerals

```
POST /api/trading-hubs/{hubUuid}/sell
Authorization: Bearer {token}

{
  "mineral_uuid": "...",
  "quantity": 50
}
```

Sell prices are **not** affected by vendor markup. No negotiation applies to selling.

---

### Ship Repairs

#### 1. Get Repair Estimate

```
GET /api/ships/{shipUuid}/repair-estimate
```

Response includes vendor-adjusted cost:
```json
{
  "data": {
    "hull_damage": 45,
    "hull_repair_cost": 585,
    "total_repair_cost": 910,
    "can_negotiate": true,
    "vendor_rate": 13.0,
    "base_rate": 10,
    "vendor_markup_pct": 30.0
  }
}
```

Display `vendor_markup_pct` to the player so they understand why repairs cost more than base rate. If `can_negotiate` is `true`, offer a "Negotiate" button.

#### 2. (Optional) Negotiate Repair Price

```
POST /api/players/{playerUuid}/vendors/{vendorUuid}/negotiate

{
  "context": "repair",
  "proposed_price": 700,
  "base_price": 910,
  "is_precursor": false
}
```

#### 3. Perform Repair

```
POST /api/ships/{shipUuid}/repair-hull
POST /api/ships/{shipUuid}/repair-components
POST /api/ships/{shipUuid}/repair-all
```

No additional body needed. The server uses the current vendor-adjusted rate.

---

### Salvage Yard

The salvage yard sells used/scavenged ship components. Vendor markup stacks with crew negotiation discounts.

#### 1. Browse Salvage Inventory

```
GET /api/players/{playerUuid}/salvage-yard
Authorization: Bearer {token}
```

Each item includes `quoted_price` (vendor-adjusted) and `base_price`.

#### 2. (Optional) Negotiate

```
POST /api/players/{playerUuid}/vendors/{vendorUuid}/negotiate

{
  "context": "salvage",
  "item_id": "{salvageItemUuid}",
  "proposed_price": 800,
  "base_price": 1000
}
```

#### 3. Purchase

```
POST /api/players/{playerUuid}/salvage-yard/purchase

{
  "item_uuid": "...",
  "install_immediately": true
}
```

**Price calculation:**
```
quoted_price = base_price × (1 + vendor_markup) × (1 + escalation)
crew_discount = best crew member's negotiation discount
final_price  = quoted_price × (1 - crew_discount)
```

---

### Ship Shop

Sells ship blueprints. Vendor markup applies to all ship purchase prices.

#### 1. Browse Ships

```
GET /api/players/{playerUuid}/ship-shop
Authorization: Bearer {token}
```

#### 2. (Optional) Negotiate

```
POST /api/players/{playerUuid}/vendors/{vendorUuid}/negotiate

{
  "context": "ship",
  "item_id": "{shipUuid}",
  "proposed_price": 45000,
  "base_price": 50000
}
```

**Do not negotiate on precursor ships** (items with `is_precursor: true`) — this triggers the `insulted` outcome and adds a permanent markup penalty. Always check `is_precursor` before showing a negotiate button.

#### 3. Purchase

```
POST /api/players/{playerUuid}/ship-shop/purchase

{
  "ship_uuid": "..."
}
```

---

### Plans Shop

Sells rare upgrade plans. Vendor markup applies.

#### 1. Browse Plans

```
GET /api/players/{playerUuid}/plans
Authorization: Bearer {token}
```

#### 2. (Optional) Negotiate

```
POST /api/players/{playerUuid}/vendors/{vendorUuid}/negotiate

{
  "context": "plan",
  "item_id": "{planUuid}",
  "proposed_price": 9000,
  "base_price": 10000
}
```

#### 3. Purchase

```
POST /api/players/{playerUuid}/plans/purchase

{
  "plan_uuid": "..."
}
```

---

### Hiring Crew

Crew are hired from a pool available at the player's location. Each candidate's stats drive their own negotiation (they act as their own vendor).

#### 1. List Available Crew

```
GET /api/players/{playerUuid}/crew/available
Authorization: Bearer {token}
```

#### 2. Hire (with Optional Pay Negotiation)

```
POST /api/players/{playerUuid}/crew/hire

{
  "crew_uuid": "...",
  "proposed_pay_rate": 0.025
}
```

`proposed_pay_rate` is optional. If omitted, the candidate's rolled pay rate is used.

**Negotiation outcomes:**

| Outcome | Condition | Result |
|---------|-----------|--------|
| `accepted` | roll > 0.30 | Hired at your proposed rate |
| `counter` | roll > 0.00 | Hired at midpoint between proposed and rolled rate |
| `rejected` | roll ≤ 0.00 | Hired at candidate's rolled rate |
| Walk-away error | proposed < 0.01 | `CREW_WALKAWAY` — hire fails |

**Response includes:**
```json
{
  "data": {
    "uuid": "...",
    "negotiation_outcome": "counter",
    "original_pay_rate": 0.035,
    "final_pay_rate": 0.030
  }
}
```

---

## Negotiation Flow

Negotiation is an optional step before any purchase. The same endpoint handles all contexts.

```
POST /api/players/{playerUuid}/vendors/{vendorUuid}/negotiate
Authorization: Bearer {token}

{
  "context": "mineral" | "salvage" | "ship" | "plan" | "repair",
  "item_id": "{itemUuid}",         // optional, for specific item context
  "proposed_price": 850,
  "base_price": 1000,
  "is_precursor": false            // always pass explicitly for ships/plans
}
```

### Outcome Handling

| `outcome` | `final_price` | UI Action |
|-----------|--------------|-----------|
| `success` | Negotiated price | Show success toast, proceed to purchase with final_price |
| `partial` | Split price | Show partial success, offer to accept or decline |
| `fail` | Original quoted price | Show failure message, offer to buy at full price or leave |
| `insulted` | Original quoted price | Show insult warning + `markup_penalty` applied permanently |
| `lockout` | Original quoted price | Lock all vendor interactions until `locked_until` timestamp |

### Applying the Negotiated Price

After a successful or partial negotiation, the API **records the negotiated price internally**. Simply proceed with the normal purchase endpoint — no need to pass the `final_price` to the purchase call. The server uses the most recent negotiated price for the next transaction.

### When to Show Negotiate Button

Show negotiate when:
- `can_negotiate: true` in the repair estimate response
- The item is NOT a precursor (`is_precursor: false`)
- The player is NOT locked out (`relationship.is_locked_out: false`)

Hide or disable negotiate for:
- Precursor items (would trigger `insulted` outcome)
- Locked-out players
- Selling minerals (negotiation doesn't apply to sells)

---

## Vendor Archetypes — UI Hints

Use `archetype` to set appropriate visual tone and player expectations:

| Archetype | `archetype` value | Flavor | UI Hint |
|-----------|------------------|--------|---------|
| Opportunist | `opportunist` | Aggressive markup, escalates fast | Show warning badge; highlight escalation |
| Professional | `professional` | Fair baseline, tough negotiator | Neutral tone; negotiation harder |
| Lucky Fence | `lucky_fence` | Erratic pricing, occasional great deals | Show volatility indicator |
| Indifferent | `indifferent` | Barely marks up, barely reacts | Safe for large un-negotiated purchases |
| Shark | `shark` | Maximum markup, hardest to negotiate | Strong warning; always negotiate |
| Pushover | `pushover` | Easy negotiate, doesn't escalate | Encourage negotiation; easy deals |
| Paranoid | `paranoid` | Remembers everything, slow to forgive | Warn about escalation persistence |
| Gambler | `gambler` | Wild outcome swings | Prepare for both extreme and great outcomes |
| Honest Dealer | `honest_dealer` | Moderate markup, fair negotiation | Neutral; reliable reference vendor |

---

## Criminality Signal

`vendor.criminality` (0.0–1.0) from the dialogue endpoint:

| Range | Meaning | UI Treatment |
|-------|---------|--------------|
| 0.0–0.3 | Legitimate operation | No indicator |
| 0.3–0.6 | Slightly shady | Subtle indicator (e.g. amber dot) |
| 0.6–0.8 | Clearly criminal | Red indicator, player warned |
| ≥ 0.8 | Black market dealer | Full black market UI theme |

**Important**: Black market items are silently filtered from inventory responses for players below the access threshold — never show "access denied" or reference items the player cannot see. Items simply don't appear.

---

## Dialogue Status Handling

The `dialogue_status` field in the dialogue endpoint response:

| Status | Meaning | UI Behavior |
|--------|---------|-------------|
| `complete` | Pre-generated LLM line served | Display normally |
| `pending` | LLM generation not yet done; static fallback served | Display normally (fallback is still coherent) |
| `failed` | LLM generation failed; static fallback served | Display normally; optionally show subtle "offline" indicator |

The static fallback is always a coherent, serviceable line. You don't need to show the player that fallback was used unless you want to add a subtle "voice lines offline" badge in a settings/debug view.

---

## Lockout State Management

When `relationship.is_locked_out: true` or the negotiate endpoint returns `outcome: "lockout"`:

1. **Block all purchases** at this vendor (minerals, repairs, ships, plans, salvage)
2. **Show timer** until `locked_until` ISO timestamp
3. **Still allow browsing** — inventory endpoints still work; only purchase/negotiate actions are blocked
4. **Re-check lockout state** after the timestamp has passed before re-enabling purchase buttons

The vendor profile endpoint (`GET /api/trading-hubs/{uuid}/vendor`) always returns the current lockout state. Poll this or use the timestamp client-side to know when to re-enable.

```
locked_until: "2026-02-28T14:30:00+00:00"
```

Lockout duration is 24–48 hours (server-randomized). Do not attempt purchases during lockout — the API will return a `403` error with code `VENDOR_LOCKED_OUT`.

---

## Price Formula Reference

Quick reference for understanding how final prices are computed:

### Mineral Buy Price

```
final_price = inventory.sell_price × (1 + greed × 0.15) × (1 + markup_escalation_pct / 100)
```

### Mineral Sell Price

```
final_price = inventory.buy_price   // no vendor markup on sells
```

### Repair Cost

```
vendor_rate  = 10 × (1 + greed × 0.50)   // credits per hull point
hull_cost    = hull_damage × vendor_rate
```

### Salvage / Ship / Plans

```
quoted_price = base_price × (1 + vendor_markup) × (1 + escalation)
crew_discount = best assigned crew member's negotiation discount
final_price  = quoted_price × (1 - crew_discount)
```

### Crew Hire Negotiation Roll

```
candidate_resistance = experience × (1 − (morale − 0.5) × 0.3)
player_power         = player_level / 20
luck_swing           = (candidate.luck − 0.5) × 0.2
roll = player_power − candidate_resistance + luck_swing + random(−0.10, +0.10)
```

---

## Key Implementation Notes

1. **Always load vendor before showing any shop prices.** The vendor's markup affects every price displayed. Loading the hub inventory before loading the vendor means you may show incorrect prices briefly.

2. **Cache the vendor profile for the session at this hub.** The profile only changes when the player navigates away and returns. No need to re-fetch on every tab switch within the hub.

3. **`vendor.uuid` ≠ `trading_hub.uuid`.** The negotiate and dialogue endpoints use the `GalaxyVendorProfile` UUID from `GET /api/trading-hubs/{uuid}/vendor → data.uuid`. Store this separately from the hub UUID.

4. **Negotiation is one-shot per purchase.** A successful negotiation locks in the price for the immediate next purchase at that vendor. If the player browses away and comes back, the negotiated price may no longer apply.

5. **Don't show raw trait values.** The API intentionally withholds trait floats (greed, business_acumen, etc.) from players. Use `archetype` and `archetype_label` for player-facing flavor only.

6. **Precursor guard is mandatory.** Always pass `"is_precursor": false` explicitly when calling negotiate for ships and plans. If the item has a precursor flag in its inventory response, hide the negotiate button entirely — don't let the player trigger the `insulted` outcome accidentally.

7. **Markup escalation is per-player per-vendor.** `markup_escalation_pct` in the relationship object is specific to this player at this vendor. Different players at the same hub see different markups.

8. **Goodwill is earned, not spent.** The `goodwill` field (−1.0 to 1.0) tracks spending milestone bonuses. Positive goodwill reduces escalation decay speed. Negative goodwill is earned by consistently lowballing the vendor.

9. **Dialogue endpoint vendorUuid = GalaxyVendorProfile UUID.** This is a per-galaxy instance UUID, not the global template UUID. Use the UUID from the vendor profile response, not any other vendor identifier.

10. **The fallback chain always returns something.** The dialogue endpoint never returns a `404` or empty greeting if the vendor exists. If pre-generated lines are unavailable, a static service-type string is returned. This means you can always display `dialogue.greeting` unconditionally.
