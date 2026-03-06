# Space Wars 3002 – Job Board and Contract System
Technical Design Document

Version: 1.0  
Status: Proposed  
Author: Architecture

---

# 1. Purpose

The Job Board system introduces contracts that allow players to perform work on behalf of factions, colonies, or other players.

Contracts provide structured tasks such as:

- hauling cargo
- supplying colonies
- escorting convoys
- bounty hunting
- exploration

The system supports the core gameplay pillars:

- player-driven logistics
- economic simulation
- emergent risk (piracy, escorts)
- early-game income opportunities

Each station bar hosts a **job board** where contracts can be discovered and accepted.

---

# 2. Feature Scope (Version 1)

Initial implementation supports:

- Transport contracts
- Supply contracts

Future expansions may add:

- Escort contracts
- Bounty contracts
- Exploration contracts
- Player-posted contracts
- Black market job boards
- Pirate trap contracts
- Contract insurance

---

# 3. System Overview

Each bar hosts a **localized contract board**.

Contracts originate from:

- system economic demand
- faction logistics requirements
- player-posted contracts (future phase)

Players can:

- browse contracts
- accept contracts
- complete contracts
- receive rewards

Contracts are stored in the database and follow a deterministic lifecycle.

---

# 4. Prerequisite Systems

The following systems must exist prior to implementing the job board.

## 4.1 Player Accounts

Players must exist with a stable identifier.

Required fields:
`player_id`
`credits`
`current_location_id`

Authentication and session management must already be implemented.

---

## 4.2 Locations

The game must support identifiable locations.

Examples:

- sector
- star system
- station
- colony

Each location must have a unique identifier.

`location_id`

Contracts reference these identifiers for origin and destination.

---

## 4.3 Commodity System

A commodity catalogue must exist.

Example table:

commodities
| field        |   type   |
|--------------|----------|
| commodity_id | int      |
| name         | string   |
| base_price   | float    |

---

## 4.4 Station / Colony Inventory

Stations and colonies must maintain inventories.

Example structure:

station_inventory
| field        |   type   |
|--------------|----------|
| location_id  | int      |
| commodity_id | int      |
| quantity     | int      |

---

## 4.5 Ship Cargo

Ships must store cargo.

Example table:

ship_cargo
| field        |   type   |
|--------------|----------|
| ship_id      | int      |
| commodity_id | int      |
| quantity     | int      |

Server-side logic must exist to:

- add cargo to ships
- remove cargo from ships
- validate quantities

---

## 4.6 Credits / Currency

Players must have a credit balance.

Example:
players
| field        |   type   |
|--------------|----------|
| player_id    | int      |
| credits      | float    |

Alternatively a transaction ledger may exist.

---

## 4.7 Time System

The game must support time tracking.

Contracts rely on timestamps:

`posted_at`
`expires_at`
`deadline_at`

This may be based on:

- real time
- server ticks
- turn system

---

## 4.8 Scheduled Jobs

The backend must support scheduled processing.

Tasks include:

- expiring contracts
- failing overdue contracts
- generating new contracts

---

# 5. Contract Types (Version 1)

## Transport Contracts

Move cargo between two locations.

Example:

Cargo: 400 Titanium
Origin: Vega Station
Destination: Tau Ceti Colony
Reward: 12,000 credits
Deadline: 18 hours

---

## Supply Contracts

Deliver materials required for colony infrastructure.

Example:

Deliver: 1,000 Food
Destination: Frontier Colony
Reward: 8,000 credits

---

# 6. Contract Lifecycle

Contracts follow a deterministic lifecycle.

+_____________+      +_____________+      +_____________+  
|   POSTED    |   →  |  ACCEPTED   |   →  |  COMPLETED  |
+-------------+      +-------------+      +-------------+

Failure states:

FAILED
EXPIRED
CANCELLED

---

# 7. State Definitions

POSTED

Contract is visible on the job board and available for acceptance.

ACCEPTED

A player has accepted the contract.

COMPLETED

Contract requirements were fulfilled.

FAILED

Deadline expired or cargo destroyed.

EXPIRED

Contract expired before acceptance.

CANCELLED

Issuer or system removed the contract.

---

# 8. Database Schema

## contracts


| field                     |   type      |
|---------------------------|-------------|
| id                        | int         |
| type                      | enum or int |
| status                    | enum or int |
| scope                     | enum or int |
| bar_location_id           | int         |
| issuer_type               | enum or int |
| issuer_id                 | int         |
| title                     | string      |
| description               | text        |
| origin_location_id        | int         |
| destination_location_id   | int         |
| cargo_manifest_json       | text        |
| reward_credits            | int         |
| posted_at                 | datetime    |
| expires_at                | datetime    | 
| deadline_at               | datetime    |
| risk_rating               | enum or int |
| reputation_min            | int         |
| accepted_by_player_id     | int         |
| accepted_at               | datetime    |
| completed_at              | datetime    |
| failed_at                 | datetime    |
| failure_reason            | text        |
| seed                      | int         |

Indexes recommended:
(bar_location_id, status)
(accepted_by_player_id, status)
(origin_location_id, destination_location_id)

---

## contract_events

Lifecycle event tracking.


| field                     |   type      |
|---------------------------|-------------|
| id                        | int         |
| contract_id               | int         |
| event_type                | enum or int |
| actor_type                | enum or int |
| actor_id                  | int         |
| payload_json              | text        |
| created_at                | datetime    |

---

## player_contract_reputation (optional)


| field                     |   type      |
|---------------------------|-------------|
| player_id                 | int         |
| reliability_score         | int (0-100) |
| completed_count           | int         |
| failed_count              | int         |
| abandoned_count           | int         |
| last_updated              | datetime    |

---

# 9. Contract Acceptance

Players accept contracts through the API.

Validation checks:

- contract status must be POSTED
- player must meet reputation requirements
- player must not exceed active contract limits

After acceptance:
status = ACCEPTED
accepted_by_player_id = player_id
accepted_at = datetime

---

# 10. Contract Completion

For transport or supply contracts:

1. Player arrives at destination location.
2. Player performs delivery action.
3. System validates cargo requirements.
4. Cargo removed from ship.
5. Cargo added to station inventory.
6. Contract marked complete.
7. Reward paid to player.

---

# 11. Reward Distribution

Upon completion:

player_credits += reward_credits

If a ledger system exists, record a transaction entry.

Operations should be executed within a database transaction.

---

# 12. Contract Generation

Contracts are generated from economic conditions.

Examples:

### Colony shortages

Food deficit → food delivery contract

### Construction demand

Defense satellite project → titanium supply contract

### Market imbalance 

Mineral shortage → transport contracts

If no economic signals exist, contracts may be randomly generated.

---

# 13. Job Board UI

Bars expose a job board interface.

Example display:

Station Bar – Job Board

[Transport]
Deliver 200 Titanium
Destination: Tau Ceti
Reward: 12,000 CR

[Supply]
Deliver 500 Food
Destination: Frontier Colony
Reward: 8,500 CR

Available filters:

- contract type
- distance
- reward
- risk
- reputation requirement

---

# 14. Abuse Prevention

Basic safeguards include:

- limit active contracts per player
- enforce deadlines
- apply reputation penalties for failures
- expire unused contracts

Optional safeguards:

- contract posting fees
- acceptance cooldowns
- reputation gating

---

# 15. Implementation Phases

## Phase 1

Implement:

- transport contracts
- supply contracts
- job board UI
- contract acceptance
- delivery completion

---

## Phase 2

Add:

- escort contracts
- bounty contracts
- exploration contracts

---

## Phase 3

Add:

- player-posted contracts
- escrow or bond system
- black market job boards
- smuggling contracts

---

# 16. Future Enhancements

Potential future systems include:

- pirate ambush contracts
- dynamic risk estimation
- convoy simulation
- contract insurance
- faction reputation
- long-haul logistics contracts
- player-run shipping corporations

---

# 17. Minimal Viable Implementation

The smallest working system includes:

- contract database table
- contract listing endpoint
- contract acceptance endpoint
- delivery validation
- credit reward payment
- contract expiry job

This allows players to:

enter bar → view jobs → accept job → deliver cargo → get paid

All additional complexity can be layered on later.

---

# 18. Architectural Guidelines

Business logic should reside in service classes.

Example services:

ContractService
ContractGenerationService
ReputationService
InventoryService

Controllers should only coordinate request handling.

Database mutations must occur through services to maintain consistency.

---

# 19. Summary

The job board system acts as a **labor marketplace for the galactic economy**.

It connects:

- colonies needing supplies
- traders moving cargo
- combat pilots protecting convoys
- explorers surveying frontier space

This system supports both **player-driven gameplay** and **simulation-driven economic activity**.

enter bar → view jobs → accept job → perform task → receive reward

The initial implementation focuses on simplicity while leaving room for future expansion.

---


