# Internal API Reference

> **Added**: 2026-03-13
> **Source**: Phase 2 vendor dialogue generator integration

This document covers internal and admin API endpoints that are **not part of the public gameplay API**. These endpoints are used by backend services and operators, not by game clients.

---

## Table of Contents

1. [Internal Dialogue Generator API](#internal-dialogue-generator-api)
   - [Authentication](#authentication)
   - [Get Pending Vendors](#get-pending-vendors)
   - [Update Generation Status](#update-generation-status)
   - [Submit Dialogue Lines](#submit-dialogue-lines)
2. [Admin Vendor Dialogue API](#admin-vendor-dialogue-api)
   - [List Pending Dialogue](#list-pending-dialogue)
   - [Force Regenerate Dialogue](#force-regenerate-dialogue)
   - [Inspect Vendor Dialogue](#inspect-vendor-dialogue)
3. [Artisan Commands](#artisan-commands)
4. [Shared Enums](#shared-enums)
5. [Generation Matrix](#generation-matrix)
6. [Validation Rules](#validation-rules)

---

## Internal Dialogue Generator API

**Base prefix**: `/api/internal`

These endpoints are used exclusively by the Go dialogue generator service. They are protected by a shared bearer token (`DIALOGUE_GENERATOR_TOKEN` env var), **not** by Laravel Sanctum.

### Authentication

```
Authorization: Bearer <DIALOGUE_GENERATOR_TOKEN>
```

If the token is not configured, all internal endpoints return `503`. If the token is wrong, they return `401`.

Internal endpoints return plain JSON (not the `BaseApiController` envelope format used by gameplay endpoints).

---

### Get Pending Vendors

Returns all `galaxy_vendor_profiles` with `dialogue_generation_status` of `pending` or `failed`, with the profile data the Go generator needs to build prompts.

**Endpoint**: `GET /api/internal/vendor-dialogue/pending`

**Auth**: Internal token

#### Success Response (200 OK)

```json
{
  "count": 2,
  "vendors": [
    {
      "id": 42,
      "uuid": "53568459-6bb0-4540-b032-918138ced0be",
      "service_type": "salvage_yard",
      "criminality": 0.27,
      "dialogue_generation_version": 3,
      "profile": {
        "archetype": "opportunist",
        "personality": {
          "honesty": 0.45,
          "greed": 0.55,
          "charm": 0.44,
          "risk_tolerance": 0.76,
          "ego_drive": 0.62,
          "empathy": 0.31,
          "curiosity": 0.58
        },
        "markup_base": 0.2152
      }
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Internal DB ID (used in subsequent requests) |
| `uuid` | string | GalaxyVendorProfile UUID (used in subsequent requests) |
| `service_type` | string | `trading_hub`, `salvage_yard`, `shipyard`, or `market` |
| `criminality` | float | 0.0–1.0. Influences tone (low = cleaner, high = sharper/dubious) |
| `dialogue_generation_version` | integer | Current version. Lines stored with older versions are stale. |
| `profile.archetype` | string | VendorArchetype enum value (see [Vendor Archetypes](vendors.md#vendor-archetypes)) |
| `profile.personality` | object | Seven trait floats (0.0–1.0). See personality mapping below. |
| `profile.markup_base` | float | Price posture: high = defensive/premium language, low/negative = bargain language |

#### Personality → Prompt Mapping

| Trait | Prompt Influence |
|-------|-----------------|
| `honesty` | Bluntness / trustworthiness |
| `greed` | Harder sell / price defensiveness |
| `charm` | Smoothness / friendliness |
| `risk_tolerance` | Willingness to hype questionable goods |
| `ego_drive` | Confidence / ambition |
| `empathy` | Warmth toward the player |
| `curiosity` | Interest in what the player is carrying or doing |

---

### Update Generation Status

Go calls this to signal state transitions as it processes each vendor.

**Endpoint**: `PATCH /api/internal/vendor-dialogue/{vendorUuid}/status`

**Auth**: Internal token

#### Status Transitions

| From | To | Meaning |
|------|----|---------|
| `pending` | `generating` | Go has begun processing this vendor |
| `generating` | `complete` | Go finished successfully; `generated_at` is set |
| `generating` | `failed` | Go gave up after retries |

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | Yes | `generating`, `complete`, or `failed` |
| `generated_at` | string (ISO 8601) | No | Timestamp of completion (used when `status=complete`, defaults to `now()`) |

```json
{ "status": "generating" }
```

```json
{ "status": "complete", "generated_at": "2026-03-13T14:30:00Z" }
```

```json
{ "status": "failed" }
```

#### Success Response (200 OK)

```json
{ "ok": true, "status": "complete" }
```

#### Error Responses

| Code | Condition |
|------|-----------|
| 404 | Vendor UUID not found |
| 422 | Invalid status value |

---

### Submit Dialogue Lines

Go posts generated lines for a specific scope (one combination of line_type + bucket + transaction_context + inventory_context). PHP validates the lines and stores them using a **replace strategy** — existing rows for the exact scope are deleted before insertion.

**Endpoint**: `POST /api/internal/vendor-dialogue/{vendorUuid}/lines`

**Auth**: Internal token

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `line_type` | string | Yes | See [Shared Enums](#shared-enums) |
| `interaction_bucket` | string | Yes | See [Shared Enums](#shared-enums) |
| `transaction_context` | string | Yes | See [Shared Enums](#shared-enums) |
| `inventory_context` | string | Yes | See [Shared Enums](#shared-enums) |
| `generation_version` | integer | Yes | Must match the vendor's current `dialogue_generation_version` |
| `lines` | array of strings | Yes | 1–20 generated dialogue lines |

```json
{
  "line_type": "greeting",
  "interaction_bucket": "first_visit",
  "transaction_context": "neutral",
  "inventory_context": "none",
  "generation_version": 3,
  "lines": [
    "What do you need, stranger?",
    "Don't get lost in here. I've seen it happen.",
    "You've got the look of someone who wants something."
  ]
}
```

#### Success Response (200 OK)

```json
{
  "ok": true,
  "stored": 3,
  "duplicates_dropped": 0
}
```

#### Validation Failure Response (422)

If any line fails PHP-side validation, the entire submission is rejected. Go must fix the offending lines and resubmit.

```json
{
  "error": "Validation failed — fix lines and resubmit",
  "code": "DIALOGUE_VALIDATION_FAILED",
  "failures": [
    {
      "line": "Here are the lines you requested:",
      "reason": "Contains meta-commentary"
    },
    {
      "line": "Hi.",
      "reason": "Too short (1 words, min 6)"
    }
  ]
}
```

#### Error Responses

| Code | Condition |
|------|-----------|
| 404 | Vendor UUID not found |
| 422 | Validation error (see above) |

#### Replace Strategy

Before inserting new rows, all existing `vendor_dialogue` rows matching the exact scope `(galaxy_vendor_profile_id, line_type, interaction_bucket, transaction_context, inventory_context)` are deleted within a single DB transaction. This ensures clean versioning without stale lines accumulating.

---

## Admin Vendor Dialogue API

**Base prefix**: `/api/admin`

**Auth**: Laravel Sanctum (`auth:sanctum`). These endpoints require a logged-in user with admin access.

These endpoints give operators visibility into the dialogue generation pipeline and allow manual intervention.

---

### List Pending Dialogue

Returns all vendors currently needing dialogue generation (`status = pending` or `failed`).

**Endpoint**: `GET /api/admin/vendors/dialogue/pending`

**Auth**: Sanctum

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "count": 3,
    "vendors": [
      {
        "id": 42,
        "uuid": "53568459-6bb0-4540-b032-918138ced0be",
        "galaxy_name": "Andromeda Reach",
        "poi_name": "Kovac's Emporium",
        "vendor_name": "Kovac's Trading Post",
        "vendor_archetype": "opportunist",
        "service_type": "trading_hub",
        "status": "pending",
        "version": 1
      }
    ]
  }
}
```

---

### Force Regenerate Dialogue

Marks a specific vendor as `pending` (increments `dialogue_generation_version`, clears `dialogue_generated_at`). The Go generator will pick it up on its next poll.

**Endpoint**: `POST /api/admin/vendors/{uuid}/dialogue/regenerate`

**Auth**: Sanctum

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `uuid` | string | GalaxyVendorProfile UUID |

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "message": "Dialogue regeneration scheduled",
    "vendor_uuid": "53568459-6bb0-4540-b032-918138ced0be",
    "status": "pending",
    "version": 4
  }
}
```

#### Error Responses

| Code | Condition |
|------|-----------|
| 404 | Vendor not found |

---

### Inspect Vendor Dialogue

Returns all stored dialogue lines for a vendor, grouped by line type and interaction bucket. Useful for debugging generation output.

**Endpoint**: `GET /api/admin/vendors/{uuid}/dialogue`

**Auth**: Sanctum

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `uuid` | string | GalaxyVendorProfile UUID |

#### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "vendor": {
      "uuid": "53568459-6bb0-4540-b032-918138ced0be",
      "galaxy_name": "Andromeda Reach",
      "poi_name": "Kovac's Emporium",
      "profile_name": "Kovac's Trading Post",
      "service_type": "trading_hub",
      "criminality": 0.27
    },
    "generation": {
      "status": "complete",
      "version": 3,
      "generated_at": "2026-03-13T14:30:00+00:00"
    },
    "dialogue_by_type_bucket": {
      "greeting": {
        "first_visit": [
          { "id": 101, "text": "What do you need, stranger?", "weight": "1.0000", "inventory_context": "none", "generation_version": 3 }
        ],
        "repeat_customer": [
          { "id": 104, "text": "Back again? Either I impress you or concern you.", "weight": "1.0000", "inventory_context": "none", "generation_version": 3 }
        ]
      },
      "farewell": {
        "repeat_customer": [
          { "id": 120, "text": "Don't spend it all in one place.", "weight": "1.0000", "inventory_context": "none", "generation_version": 3 }
        ]
      }
    }
  }
}
```

#### Error Responses

| Code | Condition |
|------|-----------|
| 404 | Vendor not found |

---

## Artisan Commands

These commands are run by operators on the server. They are not exposed via HTTP.

### `vendor:dialogue-queue`

Mark vendors as pending for the Go generator to pick up.

```bash
# Queue all pending + failed vendors (across all galaxies)
php artisan vendor:dialogue-queue

# Scope to a specific galaxy
php artisan vendor:dialogue-queue --galaxy=<galaxyUuid>

# Re-queue all vendors including already-complete ones
php artisan vendor:dialogue-queue --all

# Also re-queue vendors currently being generated (use with caution)
php artisan vendor:dialogue-queue --force
```

### `vendor:dialogue-status`

Show generation status and line coverage for vendors.

```bash
# All galaxies
php artisan vendor:dialogue-status

# Specific galaxy
php artisan vendor:dialogue-status <galaxyUuid>
```

Output includes:
- Summary table: count per status (`pending`, `generating`, `complete`, `failed`)
- Detail table: per vendor with galaxy, name, service type, status, version, line count vs expected matrix size, and generation timestamp

---

## Shared Enums

Both the Go generator and PHP backend must use these exact string values. No ad-hoc strings are permitted.

### `line_type`

| Value | Description |
|-------|-------------|
| `greeting` | Opening line when player approaches vendor |
| `inventory_pitch` | Sales pitch for a category of item |
| `deal_accepted` | Vendor accepts a deal |
| `deal_rejected` | Vendor rejects a deal |
| `farewell` | Closing line when player leaves |

### `interaction_bucket`

Maps visit count to bucket at runtime (PHP side):

| Visit Count | Bucket |
|-------------|--------|
| 1 | `first_visit` |
| 2 | `second_visit` |
| 3 | `third_visit` |
| ≥ 4 | `repeat_customer` |

### `transaction_context`

| Value | When Used |
|-------|-----------|
| `neutral` | Greetings, farewells — no active transaction |
| `vendor_selling` | Vendor is selling to the player |
| `vendor_buying` | Vendor is buying from the player |

### `inventory_context`

| Value | Item Category |
|-------|---------------|
| `none` | No specific item (greetings, deal responses, farewells) |
| `ship` | Complete ship purchase |
| `shield_projector` | Shield component |
| `engine` | Engine component |
| `reactor` | Reactor/power component |
| `weapon` | Weapon component |
| `sensor_array` | Sensor component |
| `cargo_module` | Cargo expansion |
| `hull_plating` | Hull/armour component |
| `salvage_component` | Generic salvaged part |

---

## Generation Matrix

The canonical set of scopes the Go generator must produce per vendor. PHP uses this for coverage reporting in `vendor:dialogue-status`. Total: **18 scopes**.

| line_type | bucket | transaction_context | inventory_context |
|-----------|--------|---------------------|------------------|
| `greeting` | `first_visit` | `neutral` | `none` |
| `greeting` | `second_visit` | `neutral` | `none` |
| `greeting` | `third_visit` | `neutral` | `none` |
| `greeting` | `repeat_customer` | `neutral` | `none` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `ship` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `shield_projector` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `engine` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `reactor` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `weapon` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `sensor_array` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `cargo_module` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `hull_plating` |
| `inventory_pitch` | `repeat_customer` | `vendor_selling` | `salvage_component` |
| `deal_accepted` | `repeat_customer` | `vendor_selling` | `none` |
| `deal_accepted` | `repeat_customer` | `vendor_buying` | `none` |
| `deal_rejected` | `repeat_customer` | `vendor_selling` | `none` |
| `deal_rejected` | `repeat_customer` | `vendor_buying` | `none` |
| `farewell` | `repeat_customer` | `neutral` | `none` |

Recommended lines per scope: **5–10**.

---

## Validation Rules

PHP enforces these on submission via `DialogueValidationService`. Go must enforce the same rules before submitting.

| Rule | Detail |
|------|--------|
| Min words | 6 |
| Max words | 20 |
| Max characters | 255 |
| No control characters | ASCII `\x00–\x08`, `\x0B`, `\x0C`, `\x0E–\x1F`, `\x7F` |
| No meta-commentary | Must not contain: `"here are"`, `"here's"`, `"certainly"`, `"of course"`, `"as requested"`, `"I'll generate"`, `"sure, here"`, `"line N:"`, `"sure thing"`, `"no problem"`, `"I understand"`, `"I'd be happy"` |
| No exact item facts | Lines must not include specific condition percentages, defect names, or live item values — those belong in Phase 4 runtime composition |
| No within-batch duplicates | Same text string submitted twice in one request is rejected |

If **any** line in a submission fails, the entire request is rejected with `422`. Go must fix offending lines and retry the full scope.
