# Space Wars 3002 - Frontend Implementation Plan

**Version:** 1.1
**Created:** 2026-02-05
**Updated:** 2026-02-05
**Based on:** API_WORKFLOW_GUIDE.md, API_DOCUMENTATION.md

This document outlines the complete implementation plan for the Space Wars 3002 frontend, organized into logical phases that build upon each other.

---

## Table of Contents

1. [Current State Assessment](#current-state-assessment)
2. [Phase 1: Core Trading Loop](#phase-1-core-trading-loop)
3. [Phase 2: Travel System](#phase-2-travel-system)
4. [Phase 3: Ship Management](#phase-3-ship-management)
5. [Phase 4: Combat System](#phase-4-combat-system)
6. [Phase 5: Exploration & Scanning](#phase-5-exploration--scanning)
7. [Phase 6: Star Charts & Cartography](#phase-6-star-charts--cartography)
8. [Phase 7: Colony Management](#phase-7-colony-management)
9. [Phase 8: Precursor Ship Hunt](#phase-8-precursor-ship-hunt)
10. [Phase 9: Leaderboards & Victory](#phase-9-leaderboards--victory)
11. [Phase 10: Mirror Universe](#phase-10-mirror-universe)
12. [Phase 11: Notifications](#phase-11-notifications)
13. [Phase 12: NPC Interactions](#phase-12-npc-interactions)
14. [Testing Infrastructure](#testing-infrastructure)
15. [Technical Notes](#technical-notes)

---

## Current State Assessment

### Already Implemented

- Authentication (login, register, logout, token refresh)
- Galaxy listing, creation, and joining
- Basic player state management
- Star map view with fog-of-war
- In-system view with menu navigation
- Local bodies display (planets, moons, asteroids)
- Ship catalog viewing
- Basic player stats display
- Sector grid visualization

### Not Yet Implemented

- Trading (buy/sell minerals)
- Travel system (warp gates, coordinate jumps)
- Combat system (pirates, PvP)
- Ship upgrades and component installation
- Colony management
- Star charts / Cartography
- Precursor ship hunt
- Mirror Universe
- Leaderboards and victory progress
- Notifications
- NPC interactions

---

## Phase 1: Core Trading Loop

**Goal:** Enable players to buy and sell minerals at trading hubs.

**Priority:** High - This is the primary income source for new players.

### 1.1 API Integration

Add to `src/lib/api.ts`:

```typescript
tradingHubs: {
  // Get list of nearby trading hubs
  async list(playerUuid: string, radius?: number)

  // Get trading hub details
  async get(hubUuid: string)

  // Get hub inventory with buy/sell prices
  async getInventory(hubUuid: string)

  // Purchase minerals
  async buy(hubUuid: string, playerUuid: string, mineralUuid: string, quantity: number)

  // Sell minerals
  async sell(hubUuid: string, playerUuid: string, mineralUuid: string, quantity: number)
}

minerals: {
  // Get all minerals in the game
  async list()
}

players: {
  // Get player cargo manifest
  async getCargo(playerUuid: string)
}
```

### 1.2 New Components

| Component              | Location                   | Description                             |
| ---------------------- | -------------------------- | --------------------------------------- |
| `TradingPanel.svelte`  | `src/lib/components/game/` | Main trading interface                  |
| `MineralRow.svelte`    | `src/lib/components/game/` | Single mineral with buy/sell controls   |
| `CargoManifest.svelte` | `src/lib/components/game/` | Player's current cargo display          |
| `PriceDisplay.svelte`  | `src/lib/components/game/` | Price formatting with profit indicators |

### 1.3 State Management

Add to `playerState.svelte.ts`:

- `cargo: CargoItem[]` - Current cargo manifest
- `loadCargo()` - Fetch cargo from API
- `buyMineral(hubUuid, mineralUuid, quantity)` - Execute purchase
- `sellMineral(hubUuid, mineralUuid, quantity)` - Execute sale

### 1.4 UI Integration

- Update `ActionPanel.svelte` "Trading Hub" tab to show `TradingPanel`
- Display current credits prominently
- Show cargo capacity (used/available)
- Enable buy/sell buttons based on affordability and cargo space

### 1.5 Acceptance Criteria

- [ ] Player can view minerals available at current trading hub
- [ ] Player can see buy/sell prices for each mineral
- [ ] Player can purchase minerals (deducts credits, adds to cargo)
- [ ] Player can sell minerals (adds credits, removes from cargo)
- [ ] Cargo capacity is enforced
- [ ] Insufficient credits shows error
- [ ] Price differences between hubs are visible

### 1.6 Testing Methodology

#### Unit Tests (`src/lib/stores/playerState.svelte.spec.ts`)

| Test Case                                    | Description                         | Expected Result                    |
| -------------------------------------------- | ----------------------------------- | ---------------------------------- |
| `loadCargo returns cargo items`              | Call loadCargo with mock API        | Cargo state populated correctly    |
| `buyMineral updates credits and cargo`       | Execute buy with sufficient credits | Credits deducted, cargo increased  |
| `buyMineral fails with insufficient credits` | Execute buy without credits         | Error thrown, state unchanged      |
| `buyMineral fails with full cargo`           | Execute buy with no cargo space     | Error thrown, state unchanged      |
| `sellMineral updates credits and cargo`      | Execute sell with cargo             | Credits increased, cargo reduced   |
| `sellMineral fails with no cargo`            | Execute sell without item           | Error thrown, state unchanged      |
| `cargo capacity calculation`                 | Add items to cargo                  | Correct available space calculated |

```typescript
// Example test
describe('Trading State', () => {
	it('should deduct credits when buying minerals', async () => {
		const mockApi = vi.fn().mockResolvedValue({
			success: true,
			data: { remaining_credits: 5000, cargo_used: 50 }
		});

		await playerState.buyMineral('hub-1', 'mineral-1', 10);

		expect(playerState.credits).toBe(5000);
		expect(playerState.cargo.length).toBeGreaterThan(0);
	});
});
```

#### Component Tests (`src/lib/components/game/TradingPanel.svelte.spec.ts`)

| Test Case                                       | Description                | Expected Result         |
| ----------------------------------------------- | -------------------------- | ----------------------- |
| `renders mineral list`                          | Mount with inventory data  | All minerals displayed  |
| `buy button disabled when insufficient credits` | Set credits below price    | Button disabled         |
| `buy button disabled when cargo full`           | Set cargo at capacity      | Button disabled         |
| `buy button triggers purchase`                  | Click buy with valid state | buyMineral called       |
| `sell button disabled when no cargo`            | Set empty cargo            | Button disabled         |
| `sell button triggers sale`                     | Click sell with cargo      | sellMineral called      |
| `quantity input validates range`                | Enter invalid quantity     | Input shows error       |
| `price display shows profit/loss`               | Compare buy/sell prices    | Correct indicator shown |

```typescript
// Example component test
describe('TradingPanel', () => {
	it('should disable buy button when player cannot afford', async () => {
		const { getByTestId } = render(TradingPanel, {
			props: {
				inventory: [{ mineral: { name: 'Titanium' }, buy_price: 100 }],
				playerCredits: 50
			}
		});

		const buyButton = getByTestId('buy-titanium');
		expect(buyButton).toBeDisabled();
	});
});
```

#### Integration Tests (`src/tests/integration/trading.spec.ts`)

| Test Case                   | Description                          | Expected Result               |
| --------------------------- | ------------------------------------ | ----------------------------- |
| `complete buy transaction`  | API call → State update → UI refresh | Full flow works               |
| `complete sell transaction` | API call → State update → UI refresh | Full flow works               |
| `API error handling`        | Simulate API failure                 | Error displayed, state intact |
| `concurrent transactions`   | Rapid buy/sell clicks                | No race conditions            |

#### E2E Tests (`e2e/trading.spec.ts`)

| Test Scenario              | Steps                                                                             | Expected Outcome                     |
| -------------------------- | --------------------------------------------------------------------------------- | ------------------------------------ |
| **Profitable Trade Route** | 1. Navigate to hub A<br>2. Buy Titanium<br>3. Travel to hub B<br>4. Sell Titanium | Player has more credits than started |
| **Insufficient Credits**   | 1. Try to buy expensive mineral<br>2. Verify error                                | Error message displayed              |
| **Full Cargo**             | 1. Fill cargo<br>2. Try to buy more                                               | Error message displayed              |

---

## Phase 2: Travel System

**Goal:** Enable players to move between star systems.

**Priority:** High - Required for trading profits and exploration.

### 2.1 API Integration

Add to `src/lib/api.ts`:

```typescript
warpGates: {
  // Get warp gates at a location
  async list(locationUuid: string)

  // Check for pirates on a gate route
  async checkPirates(gateUuid: string)
}

travel: {
  // Calculate fuel cost for travel
  async getFuelCost(distance: number, shipUuid: string)

  // Preview XP gain for travel
  async previewXp(distance: number, playerUuid: string)
}

players: {
  // Travel via warp gate
  async travelWarpGate(playerUuid: string, gateUuid: string)

  // Direct coordinate jump
  async travelCoordinate(playerUuid: string, targetX: number, targetY: number)

  // Direct jump to POI
  async travelDirectJump(playerUuid: string, targetPoiUuid: string)

  // Get nearby systems
  async getNearbySystems(playerUuid: string)
}
```

### 2.2 New Components

| Component               | Location                   | Description                          |
| ----------------------- | -------------------------- | ------------------------------------ |
| `WarpGatePanel.svelte`  | `src/lib/components/game/` | List of available warp gates         |
| `WarpGateRow.svelte`    | `src/lib/components/game/` | Single gate with destination info    |
| `TravelConfirm.svelte`  | `src/lib/components/game/` | Confirmation modal with fuel cost    |
| `PirateWarning.svelte`  | `src/lib/components/game/` | Warning when pirates detected        |
| `TravelResult.svelte`   | `src/lib/components/game/` | Travel outcome (XP gained, level up) |
| `CoordinateJump.svelte` | `src/lib/components/game/` | Direct coordinate input              |

### 2.3 State Management

Add to `playerState.svelte.ts`:

- `warpGates: WarpGate[]` - Available gates at current location
- `loadWarpGates()` - Fetch gates from API
- `travelViaGate(gateUuid)` - Execute warp travel
- `travelToCoordinate(x, y)` - Execute coordinate jump
- `checkForPirates(gateUuid)` - Scout gate for danger

### 2.4 UI Integration

- Update `ActionPanel.svelte` "Warp Gates" tab to show `WarpGatePanel`
- Add travel button to star map system selection
- Show fuel cost and current fuel
- Display pirate warnings with escape probability
- Handle pirate encounter responses (triggers combat phase)

### 2.5 Acceptance Criteria

- [ ] Player can see available warp gates at current location
- [ ] Player can see destination info for each gate
- [ ] Player can check for pirates before traveling
- [ ] Player can travel via warp gate (consumes fuel)
- [ ] Player can direct jump to coordinates (higher fuel cost)
- [ ] Travel result shows XP gained
- [ ] Level up is displayed when earned
- [ ] Insufficient fuel shows error
- [ ] Pirate encounters are handled (ties to Combat phase)

### 2.6 Testing Methodology

#### Unit Tests (`src/lib/stores/playerState.svelte.spec.ts`)

| Test Case                                | Description           | Expected Result        |
| ---------------------------------------- | --------------------- | ---------------------- |
| `loadWarpGates populates gates`          | Call with mock API    | Gates array populated  |
| `travelViaGate updates location`         | Execute travel        | Location state updated |
| `travelViaGate deducts fuel`             | Execute travel        | Fuel reduced correctly |
| `travelViaGate fails with no fuel`       | Travel without fuel   | Error thrown           |
| `travelViaGate handles pirate encounter` | API returns pirate    | Combat state triggered |
| `travelToCoordinate calculates fuel`     | Jump to coordinates   | Correct fuel consumed  |
| `checkForPirates returns threat level`   | Scout gate            | Pirate info returned   |
| `XP awarded on travel`                   | Complete travel       | XP increased           |
| `level up detection`                     | Travel with enough XP | Level up triggered     |

```typescript
describe('Travel State', () => {
	it('should update location after warp gate travel', async () => {
		const mockResponse = {
			success: true,
			data: {
				fuel_consumed: 5,
				xp_earned: 100,
				new_location: { uuid: 'poi-2', name: 'Alpha Centauri' }
			}
		};
		vi.mocked(api.players.travelWarpGate).mockResolvedValue(mockResponse);

		await playerState.travelViaGate('gate-1');

		expect(playerState.currentSystem?.uuid).toBe('poi-2');
		expect(playerState.ship.fuel.current).toBeLessThan(initialFuel);
	});
});
```

#### Component Tests

| Test Case                             | Description           | Expected Result        |
| ------------------------------------- | --------------------- | ---------------------- |
| `WarpGatePanel renders gates`         | Mount with gates data | All gates displayed    |
| `WarpGateRow shows destination`       | Render single gate    | Name, distance shown   |
| `WarpGateRow shows fuel cost`         | Render gate           | Fuel cost displayed    |
| `travel button disabled without fuel` | Low fuel state        | Button disabled        |
| `TravelConfirm shows costs`           | Open modal            | Fuel, XP preview shown |
| `PirateWarning shows threat`          | Pirates detected      | Warning displayed      |
| `TravelResult shows XP gained`        | After travel          | XP amount shown        |
| `TravelResult shows level up`         | Level gained          | Celebration displayed  |
| `CoordinateJump validates input`      | Invalid coords        | Error shown            |

#### Integration Tests

| Test Case               | Description                    | Expected Result  |
| ----------------------- | ------------------------------ | ---------------- |
| `warp gate travel flow` | Select gate → Confirm → Travel | Location changes |
| `coordinate jump flow`  | Enter coords → Confirm → Jump  | Location changes |
| `pirate encounter flow` | Travel → Pirate detected       | Combat triggered |
| `fuel regeneration`     | Wait → Check fuel              | Fuel increases   |

#### E2E Tests

| Test Scenario         | Steps                                                                | Expected Outcome            |
| --------------------- | -------------------------------------------------------------------- | --------------------------- |
| **Basic Warp Travel** | 1. Open warp gates<br>2. Select destination<br>3. Confirm travel     | Arrive at destination       |
| **Scout for Pirates** | 1. Check gate for pirates<br>2. See warning<br>3. Decide to travel   | Warning shown before travel |
| **Coordinate Jump**   | 1. Enter coordinates<br>2. Confirm high fuel cost<br>3. Execute jump | Arrive at coordinates       |
| **Insufficient Fuel** | 1. Deplete fuel<br>2. Try to travel                                  | Error prevents travel       |

---

## Phase 3: Ship Management

**Goal:** Enable ship purchases, upgrades, and component installation.

**Priority:** High - Required for progression.

### 3.1 API Integration

Add to `src/lib/api.ts`:

```typescript
ships: {
  // Get ship status
  async getStatus(shipUuid: string)

  // Get fuel status
  async getFuel(shipUuid: string)

  // Regenerate fuel
  async regenerateFuel(shipUuid: string)

  // Rename ship
  async rename(shipUuid: string, name: string)

  // Get upgrade options
  async getUpgradeOptions(shipUuid: string)

  // Execute upgrade
  async upgrade(shipUuid: string, component: string, levels?: number)

  // Get repair estimate
  async getRepairEstimate(shipUuid: string)

  // Repair ship
  async repair(shipUuid: string)
}

players: {
  // Purchase a new ship
  async purchaseShip(playerUuid: string, shipId: number, hubUuid: string, tradeIn?: boolean)

  // Switch active ship
  async switchShip(playerUuid: string, shipUuid: string)

  // Get player's fleet
  async getFleet(playerUuid: string)

  // Get ship components
  async getShipComponents(playerUuid: string)

  // Get salvage yard inventory
  async getSalvageYard(playerUuid: string)

  // Purchase and install component
  async purchaseComponent(playerUuid: string, inventoryId: number, slotIndex: number)

  // Uninstall component
  async uninstallComponent(playerUuid: string, componentId: number, sell?: boolean)
}
```

### 3.2 New Components

| Component                 | Location                   | Description                 |
| ------------------------- | -------------------------- | --------------------------- |
| `ShipyardPanel.svelte`    | `src/lib/components/game/` | Ship purchase interface     |
| `ShipCard.svelte`         | `src/lib/components/game/` | Ship display with stats     |
| `ShipUpgradePanel.svelte` | `src/lib/components/game/` | Upgrade interface           |
| `UpgradeRow.svelte`       | `src/lib/components/game/` | Single upgrade option       |
| `ComponentSlots.svelte`   | `src/lib/components/game/` | Weapon/utility slot display |
| `SalvageInventory.svelte` | `src/lib/components/game/` | Salvage yard components     |
| `ComponentCard.svelte`    | `src/lib/components/game/` | Component with effects      |
| `ShipRepairPanel.svelte`  | `src/lib/components/game/` | Repair interface            |
| `FleetManager.svelte`     | `src/lib/components/game/` | Multi-ship management       |

### 3.3 State Management

Add to `playerState.svelte.ts`:

- `fleet: Ship[]` - All owned ships
- `shipComponents: ComponentSlots` - Installed components
- `loadFleet()` - Fetch fleet from API
- `loadShipComponents()` - Fetch installed components
- `purchaseShip(...)` - Buy new ship
- `upgradeShip(component, levels)` - Upgrade component
- `installComponent(inventoryId, slotIndex)` - Install component

### 3.4 UI Integration

- Update `ActionPanel.svelte` "Salvage Yard" tab to show full interface
- Add shipyard access when at trading hub with shipyard
- Show upgrade options with costs
- Display component slots with empty/filled states
- Show trade-in value when purchasing new ship

### 3.5 Acceptance Criteria

- [ ] Player can view ships available at shipyard
- [ ] Player can purchase new ships (with optional trade-in)
- [ ] Player can switch between owned ships
- [ ] Player can view and execute upgrades
- [ ] Player can view salvage yard inventory
- [ ] Player can purchase and install components
- [ ] Player can uninstall and sell components
- [ ] Player can repair damaged ships
- [ ] Fuel regeneration is displayed

### 3.6 Testing Methodology

#### Unit Tests

| Test Case                        | Description       | Expected Result        |
| -------------------------------- | ----------------- | ---------------------- |
| `loadFleet returns ships`        | Fetch fleet       | Fleet array populated  |
| `purchaseShip deducts credits`   | Buy ship          | Credits reduced        |
| `purchaseShip with trade-in`     | Buy with trade-in | Net cost applied       |
| `switchShip updates active`      | Switch ship       | Active ship changed    |
| `upgradeShip increases stat`     | Upgrade sensors   | Sensor level increased |
| `upgradeShip deducts credits`    | Execute upgrade   | Credits reduced        |
| `installComponent fills slot`    | Install weapon    | Slot populated         |
| `uninstallComponent clears slot` | Uninstall         | Slot empty             |
| `uninstallComponent with sell`   | Sell component    | Credits gained         |
| `repair restores hull`           | Repair ship       | Hull at max            |

```typescript
describe('Ship Management State', () => {
	it('should apply trade-in value when purchasing ship', async () => {
		const initialCredits = 50000;
		const shipPrice = 35000;
		const tradeInValue = 5000;

		vi.mocked(api.players.purchaseShip).mockResolvedValue({
			success: true,
			data: { net_cost: 30000, remaining_credits: 20000 }
		});

		await playerState.purchaseShip(2, 'hub-1', true);

		expect(playerState.credits).toBe(20000);
	});
});
```

#### Component Tests

| Test Case                           | Description         | Expected Result      |
| ----------------------------------- | ------------------- | -------------------- |
| `ShipyardPanel shows ships`         | Render with catalog | All ships displayed  |
| `ShipCard shows stats`              | Render ship         | Stats visible        |
| `ShipCard shows price`              | Render ship         | Price displayed      |
| `purchase disabled without credits` | Low credits         | Button disabled      |
| `trade-in checkbox updates price`   | Toggle trade-in     | Net price changes    |
| `UpgradeRow shows current level`    | Render upgrade      | Level displayed      |
| `UpgradeRow shows cost`             | Render upgrade      | Cost shown           |
| `ComponentSlots shows empty`        | No components       | Empty slots shown    |
| `ComponentSlots shows installed`    | With components     | Components displayed |
| `ComponentCard shows effects`       | Render component    | Effects listed       |

#### Integration Tests

| Test Case                | Description                 | Expected Result   |
| ------------------------ | --------------------------- | ----------------- |
| `ship purchase flow`     | Select → Purchase → Switch  | New ship active   |
| `upgrade flow`           | View options → Purchase     | Stat increased    |
| `component installation` | Browse → Purchase → Install | Component in slot |
| `component sale`         | Uninstall → Sell            | Credits received  |
| `repair flow`            | View estimate → Repair      | Hull restored     |

#### E2E Tests

| Test Scenario           | Steps                                                   | Expected Outcome         |
| ----------------------- | ------------------------------------------------------- | ------------------------ |
| **Purchase First Ship** | 1. Visit shipyard<br>2. Select ship<br>3. Purchase      | New ship in fleet        |
| **Upgrade Ship**        | 1. Open upgrades<br>2. Upgrade sensors<br>3. Verify     | Sensor level increased   |
| **Install Component**   | 1. Visit salvage yard<br>2. Buy component<br>3. Install | Component in slot        |
| **Trade-in Ship**       | 1. Buy better ship with trade-in<br>2. Verify discount  | Correct credits deducted |

---

## Phase 4: Combat System

**Goal:** Handle pirate encounters and PvP combat.

**Priority:** High - Pirates are encountered during travel.

### 4.1 API Integration

Add to `src/lib/api.ts`:

```typescript
combat: {
  // Get combat preview
  async preview(playerUuid: string)

  // Engage in combat
  async engage(playerUuid: string, encounterUuid: string)

  // Attempt escape
  async escape(playerUuid: string)

  // Surrender
  async surrender(playerUuid: string)

  // Collect salvage after victory
  async collectSalvage(playerUuid: string)
}

pvp: {
  // Issue challenge
  async challenge(playerUuid: string, targetPlayerUuid: string, wager?: number)

  // List challenges
  async listChallenges(playerUuid: string)

  // Accept challenge
  async acceptChallenge(playerUuid: string, challengeUuid: string)

  // Decline challenge
  async declineChallenge(playerUuid: string, challengeUuid: string)
}
```

### 4.2 New Components

| Component               | Location                   | Description                    |
| ----------------------- | -------------------------- | ------------------------------ |
| `CombatScreen.svelte`   | `src/lib/components/game/` | Main combat interface          |
| `CombatPreview.svelte`  | `src/lib/components/game/` | Pre-combat strength comparison |
| `CombatActions.svelte`  | `src/lib/components/game/` | Fight/Flee/Surrender buttons   |
| `CombatResult.svelte`   | `src/lib/components/game/` | Victory/Defeat outcome         |
| `SalvageCollect.svelte` | `src/lib/components/game/` | Post-combat salvage            |
| `PvPChallenge.svelte`   | `src/lib/components/game/` | PvP challenge interface        |
| `ChallengeList.svelte`  | `src/lib/components/game/` | Incoming/outgoing challenges   |

### 4.3 State Management

Add to `playerState.svelte.ts`:

- `currentEncounter: CombatEncounter | null` - Active combat
- `combatResult: CombatResult | null` - Last combat outcome
- `engageCombat(encounterUuid)` - Fight pirates
- `attemptEscape()` - Try to flee
- `surrender()` - Give up cargo
- `collectSalvage()` - Claim victory rewards

### 4.4 UI Integration

- Combat screen appears when pirate encounter returned from travel
- Combat preview shows strength comparison
- Action buttons enabled/disabled based on state
- Victory shows salvage collection option
- Defeat shows losses and respawn option

### 4.5 Acceptance Criteria

- [ ] Combat preview shows player vs enemy strength
- [ ] Win probability and escape chance displayed
- [ ] Player can choose to fight, flee, or surrender
- [ ] Combat result shows damage taken, XP earned
- [ ] Victory allows salvage collection
- [ ] Defeat shows cargo/credits lost
- [ ] PvP challenges can be issued and received
- [ ] Ship destruction handled gracefully

### 4.6 Testing Methodology

#### Unit Tests

| Test Case                    | Description     | Expected Result                |
| ---------------------------- | --------------- | ------------------------------ |
| `engageCombat starts combat` | Trigger combat  | Combat state active            |
| `engageCombat victory`       | Win fight       | Victory result, XP gained      |
| `engageCombat defeat`        | Lose fight      | Defeat result, losses applied  |
| `attemptEscape success`      | Escape succeeds | Combat ends, fuel used         |
| `attemptEscape failure`      | Escape fails    | Combat continues, damage taken |
| `surrender loses cargo`      | Surrender       | Cargo cleared, ship intact     |
| `collectSalvage adds items`  | Collect salvage | Credits/cargo gained           |
| `ship destruction`           | Hull to zero    | Destroyed state triggered      |
| `PvP challenge creation`     | Issue challenge | Challenge created              |
| `PvP challenge response`     | Accept/decline  | Appropriate action taken       |

```typescript
describe('Combat State', () => {
	it('should apply damage and award XP on victory', async () => {
		vi.mocked(api.combat.engage).mockResolvedValue({
			success: true,
			data: {
				result: 'victory',
				damage_taken: 25,
				xp_earned: 500,
				salvage_available: { credits: 1000 }
			}
		});

		await playerState.engageCombat('enc-1');

		expect(playerState.combatResult?.result).toBe('victory');
		expect(playerState.ship.hull.current).toBeLessThan(initialHull);
	});
});
```

#### Component Tests

| Test Case                           | Description          | Expected Result          |
| ----------------------------------- | -------------------- | ------------------------ |
| `CombatScreen renders`              | Mount with encounter | Combat UI shown          |
| `CombatPreview shows comparison`    | Render preview       | Both strengths shown     |
| `CombatPreview shows probabilities` | Render preview       | Win/escape chances shown |
| `CombatActions fight button`        | Click fight          | engageCombat called      |
| `CombatActions flee button`         | Click flee           | attemptEscape called     |
| `CombatActions surrender button`    | Click surrender      | surrender called         |
| `CombatResult victory`              | Victory state        | XP and salvage shown     |
| `CombatResult defeat`               | Defeat state         | Losses shown             |
| `SalvageCollect lists items`        | Render salvage       | Items displayed          |
| `PvPChallenge form`                 | Render form          | Target and wager inputs  |

#### Integration Tests

| Test Case                      | Description                | Expected Result       |
| ------------------------------ | -------------------------- | --------------------- |
| `complete combat victory flow` | Fight → Win → Salvage      | Full cycle works      |
| `complete combat defeat flow`  | Fight → Lose → Respawn     | Full cycle works      |
| `escape combat flow`           | Flee → Success/Fail        | Appropriate outcome   |
| `surrender flow`               | Surrender → Losses         | Cargo lost, ship safe |
| `PvP challenge flow`           | Challenge → Accept → Fight | Combat occurs         |

#### E2E Tests

| Test Scenario     | Steps                                              | Expected Outcome             |
| ----------------- | -------------------------------------------------- | ---------------------------- |
| **Win Combat**    | 1. Encounter pirates<br>2. Choose fight<br>3. Win  | XP gained, salvage available |
| **Lose Combat**   | 1. Encounter strong pirates<br>2. Fight<br>3. Lose | Cargo/credits lost           |
| **Escape Combat** | 1. Encounter pirates<br>2. Flee                    | Escape with some damage      |
| **Surrender**     | 1. Encounter pirates<br>2. Surrender               | Lose cargo, keep ship        |
| **PvP Challenge** | 1. Challenge player<br>2. They accept<br>3. Battle | Winner gets wager            |

---

## Phase 5: Exploration & Scanning

**Goal:** Enable system scanning and discovery.

**Priority:** Medium - Enhances gameplay depth.

### 5.1 API Integration

Add to `src/lib/api.ts`:

```typescript
players: {
  // Scan current system
  async scanSystem(playerUuid: string, poiUuid?: string, force?: boolean)

  // Get scan results
  async getScanResults(playerUuid: string, poiUuid: string)

  // Get exploration log
  async getExplorationLog(playerUuid: string)

  // Get bulk scan levels (already implemented)
  async getBulkScanLevels(playerUuid: string, poiUuids: string[])

  // Scan local area (detailed POI scan)
  async scanLocal(playerUuid: string)
}
```

### 5.2 New Components

| Component                   | Location                   | Description                 |
| --------------------------- | -------------------------- | --------------------------- |
| `ScanButton.svelte`         | `src/lib/components/game/` | Initiate system scan        |
| `ScanResults.svelte`        | `src/lib/components/game/` | Display scan discoveries    |
| `ExplorationLog.svelte`     | `src/lib/components/game/` | History of explored systems |
| `DiscoveryCard.svelte`      | `src/lib/components/game/` | Individual discovery item   |
| `ScanLevelIndicator.svelte` | `src/lib/components/game/` | Visual scan level display   |

### 5.3 State Management

Add to `playerState.svelte.ts`:

- `scanResults: ScanResult | null` - Latest scan
- `explorationLog: ExplorationEntry[]` - Exploration history
- `scanCurrentSystem()` - Execute scan
- `loadExplorationLog()` - Fetch history

### 5.4 UI Integration

- Add scan button to system view
- Show scan level on star map nodes
- Display discoveries after scanning
- Exploration log accessible from menu

### 5.5 Acceptance Criteria

- [ ] Player can scan current system
- [ ] Scan reveals planets, resources, anomalies
- [ ] Scan level increases with better sensors
- [ ] Exploration log shows history
- [ ] Star map shows scan levels per system

### 5.6 Testing Methodology

#### Unit Tests

| Test Case                              | Description      | Expected Result      |
| -------------------------------------- | ---------------- | -------------------- |
| `scanCurrentSystem updates scan level` | Execute scan     | Scan level increases |
| `scanCurrentSystem reveals bodies`     | Execute scan     | Bodies discovered    |
| `loadExplorationLog fetches history`   | Load log         | Log populated        |
| `scan level affects visibility`        | Different levels | Correct info visible |
| `force scan re-scans system`           | Force flag true  | New scan executed    |

#### Component Tests

| Test Case                         | Description      | Expected Result    |
| --------------------------------- | ---------------- | ------------------ |
| `ScanButton shows scan action`    | Render button    | Scan available     |
| `ScanButton disabled during scan` | Scanning state   | Button disabled    |
| `ScanResults shows discoveries`   | After scan       | Discoveries listed |
| `ExplorationLog shows history`    | Render log       | Entries displayed  |
| `ScanLevelIndicator shows level`  | Render indicator | Level visible      |

#### Integration Tests

| Test Case                    | Description         | Expected Result   |
| ---------------------------- | ------------------- | ----------------- |
| `scan and view results flow` | Scan → View results | Discoveries shown |
| `exploration log updates`    | Scan → Check log    | New entry added   |

#### E2E Tests

| Test Scenario            | Steps                                                 | Expected Outcome |
| ------------------------ | ----------------------------------------------------- | ---------------- |
| **Scan New System**      | 1. Travel to new system<br>2. Scan<br>3. View results | Planets revealed |
| **View Exploration Log** | 1. Scan multiple systems<br>2. Open log               | All scans listed |

---

## Phase 6: Star Charts & Cartography

**Goal:** Enable star chart purchases and management.

**Priority:** Medium - Supports exploration gameplay.

### 6.1 API Integration

Add to `src/lib/api.ts`:

```typescript
players: {
  // Get player's star charts
  async getStarCharts(playerUuid: string)

  // Purchase star chart
  async purchaseStarChart(playerUuid: string, poiUuid: string, hubUuid: string)
}

tradingHubs: {
  // Get cartographer info
  async getCartographer(hubUuid: string)
}
```

### 6.2 New Components

| Component                  | Location                   | Description              |
| -------------------------- | -------------------------- | ------------------------ |
| `CartographerPanel.svelte` | `src/lib/components/game/` | Chart purchase interface |
| `ChartCard.svelte`         | `src/lib/components/game/` | Available chart display  |
| `StarChartList.svelte`     | `src/lib/components/game/` | Player's owned charts    |

### 6.3 UI Integration

- Add Cartographer tab when at hub with cartographer
- Show available charts with prices
- Indicate which systems player already has charts for
- Purchased charts reveal coordinates on map

### 6.4 Acceptance Criteria

- [ ] Player can view available star charts at cartographer
- [ ] Player can purchase star charts
- [ ] Purchased charts reveal system coordinates
- [ ] Player can view owned charts list
- [ ] Unknown systems show as "Unknown" without chart

### 6.5 Testing Methodology

#### Unit Tests

| Test Case                              | Description    | Expected Result        |
| -------------------------------------- | -------------- | ---------------------- |
| `getStarCharts returns charts`         | Fetch charts   | Charts array populated |
| `purchaseStarChart deducts credits`    | Buy chart      | Credits reduced        |
| `purchaseStarChart adds to collection` | Buy chart      | Chart in owned list    |
| `chart reveals coordinates`            | After purchase | Coords accessible      |

#### Component Tests

| Test Case                        | Description   | Expected Result         |
| -------------------------------- | ------------- | ----------------------- |
| `CartographerPanel shows charts` | Render panel  | Available charts listed |
| `ChartCard shows price`          | Render card   | Price displayed         |
| `ChartCard shows owned`          | Already owned | Owned indicator shown   |
| `purchase disabled if owned`     | Chart owned   | Button disabled         |
| `StarChartList shows collection` | Render list   | Owned charts shown      |

#### Integration Tests

| Test Case                | Description         | Expected Result           |
| ------------------------ | ------------------- | ------------------------- |
| `purchase chart flow`    | Browse → Purchase   | Chart added to collection |
| `map updates with chart` | Purchase → View map | Coordinates revealed      |

#### E2E Tests

| Test Scenario | Steps                                                     | Expected Outcome         |
| ------------- | --------------------------------------------------------- | ------------------------ |
| **Buy Chart** | 1. Visit cartographer<br>2. Purchase chart<br>3. View map | System coordinates shown |

---

## Phase 7: Colony Management

**Goal:** Enable colony establishment and management.

**Priority:** Medium - Victory path feature.

### 7.1 API Integration

Add to `src/lib/api.ts`:

```typescript
colonies: {
  // List player's colonies
  async list(playerUuid: string)

  // Get colony details
  async get(colonyUuid: string)

  // Establish new colony
  async establish(playerUuid: string, poiUuid: string, name: string)

  // Get colony production
  async getProduction(colonyUuid: string)

  // Upgrade colony
  async upgrade(colonyUuid: string)

  // List buildings
  async getBuildings(colonyUuid: string)

  // Construct building
  async constructBuilding(colonyUuid: string, buildingType: string)

  // Abandon colony
  async abandon(colonyUuid: string)
}
```

### 7.2 New Components

| Component                  | Location                   | Description                |
| -------------------------- | -------------------------- | -------------------------- |
| `ColonyList.svelte`        | `src/lib/components/game/` | Player's colonies overview |
| `ColonyDetails.svelte`     | `src/lib/components/game/` | Single colony management   |
| `ColonyProduction.svelte`  | `src/lib/components/game/` | Production stats           |
| `BuildingList.svelte`      | `src/lib/components/game/` | Colony buildings           |
| `BuildingConstruct.svelte` | `src/lib/components/game/` | Construction interface     |
| `ColonyEstablish.svelte`   | `src/lib/components/game/` | New colony creation        |

### 7.3 State Management

Add to `playerState.svelte.ts`:

- `colonies: Colony[]` - Player's colonies
- `loadColonies()` - Fetch colonies
- `establishColony(poiUuid, name)` - Create colony
- `upgradeColony(colonyUuid)` - Upgrade colony

### 7.4 UI Integration

- Add "Colonies" option to main menu
- Show colonizable indicator on habitable planets
- Colony management screen with tabs
- Building construction with costs

### 7.5 Acceptance Criteria

- [ ] Player can view owned colonies
- [ ] Player can establish colony on habitable planet
- [ ] Player can view colony production
- [ ] Player can construct buildings
- [ ] Player can upgrade colony development
- [ ] Player can abandon colony
- [ ] Colonists requirement enforced

### 7.6 Testing Methodology

#### Unit Tests

| Test Case                            | Description    | Expected Result    |
| ------------------------------------ | -------------- | ------------------ |
| `loadColonies returns colonies`      | Fetch colonies | Colonies populated |
| `establishColony creates colony`     | Establish      | New colony in list |
| `establishColony requires colonists` | No colonists   | Error thrown       |
| `upgradeColony increases level`      | Upgrade        | Level increased    |
| `constructBuilding adds building`    | Build          | Building in list   |
| `abandon removes colony`             | Abandon        | Colony removed     |

#### Component Tests

| Test Case                         | Description       | Expected Result         |
| --------------------------------- | ----------------- | ----------------------- |
| `ColonyList shows colonies`       | Render list       | Colonies displayed      |
| `ColonyDetails shows info`        | Render details    | Population, level shown |
| `ColonyProduction shows output`   | Render production | Credits/resources shown |
| `BuildingList shows buildings`    | Render buildings  | Buildings listed        |
| `BuildingConstruct shows options` | Render construct  | Options available       |
| `ColonyEstablish form`            | Render form       | Name input, confirm     |

#### Integration Tests

| Test Case                | Description             | Expected Result |
| ------------------------ | ----------------------- | --------------- |
| `establish colony flow`  | Find planet → Establish | Colony created  |
| `colony management flow` | View → Build → Upgrade  | Actions succeed |
| `abandon colony flow`    | Confirm → Abandon       | Colony removed  |

#### E2E Tests

| Test Scenario        | Steps                                                         | Expected Outcome |
| -------------------- | ------------------------------------------------------------- | ---------------- |
| **Establish Colony** | 1. Find habitable planet<br>2. Establish colony<br>3. Name it | Colony in list   |
| **Build Structure**  | 1. View colony<br>2. Construct building                       | Building added   |
| **Upgrade Colony**   | 1. View colony<br>2. Upgrade level                            | Level increased  |

---

## Phase 8: Precursor Ship Hunt

**Goal:** Enable the legendary ship hunt gameplay.

**Priority:** Medium - End-game content.

### 8.1 API Integration

Add to `src/lib/api.ts`:

```typescript
precursor: {
  // Check for rumors at current location
  async check(playerUuid: string)

  // Get free gossip
  async getGossip(playerUuid: string)

  // Pay bribe for rumor
  async bribe(playerUuid: string)

  // Get collected rumors
  async getRumors(playerUuid: string)
}
```

### 8.2 New Components

| Component                    | Location                   | Description                   |
| ---------------------------- | -------------------------- | ----------------------------- |
| `PrecursorHunt.svelte`       | `src/lib/components/game/` | Main hunt interface           |
| `RumorDisplay.svelte`        | `src/lib/components/game/` | Single rumor with coordinates |
| `RumorMap.svelte`            | `src/lib/components/game/` | Map overlay showing rumors    |
| `GossipDialog.svelte`        | `src/lib/components/game/` | NPC gossip conversation       |
| `TriangulationHelper.svelte` | `src/lib/components/game/` | Visual aid for triangulation  |

### 8.3 UI Integration

- Precursor hunt option at shipyards
- Gossip dialog with NPC
- Bribe option with cost
- Rumor collection with map overlay
- Triangulation visualization

### 8.4 Acceptance Criteria

- [ ] Player can check for rumors at shipyards
- [ ] Player can get free gossip
- [ ] Player can bribe for coordinates
- [ ] Collected rumors displayed on map
- [ ] Triangulation helper shows overlap area
- [ ] Finding Void Strider handled

### 8.5 Testing Methodology

#### Unit Tests

| Test Case                          | Description | Expected Result        |
| ---------------------------------- | ----------- | ---------------------- |
| `check returns rumor availability` | Check       | has_rumor boolean      |
| `getGossip returns text`           | Get gossip  | Gossip string returned |
| `bribe returns coordinates`        | Pay bribe   | Coords returned        |
| `bribe deducts credits`            | Pay bribe   | Credits reduced        |
| `getRumors returns all`            | Get rumors  | All rumors listed      |

#### Component Tests

| Test Case                        | Description        | Expected Result       |
| -------------------------------- | ------------------ | --------------------- |
| `PrecursorHunt shows option`     | Render at shipyard | Hunt option visible   |
| `GossipDialog shows text`        | Render dialog      | Gossip displayed      |
| `RumorDisplay shows coords`      | Render rumor       | Coordinates shown     |
| `RumorMap shows markers`         | Render map         | Rumor markers visible |
| `TriangulationHelper shows area` | Multiple rumors    | Overlap highlighted   |

#### Integration Tests

| Test Case               | Description          | Expected Result     |
| ----------------------- | -------------------- | ------------------- |
| `rumor collection flow` | Check → Bribe → View | Rumor in collection |
| `map overlay flow`      | Collect → View map   | Markers displayed   |

#### E2E Tests

| Test Scenario          | Steps                                              | Expected Outcome           |
| ---------------------- | -------------------------------------------------- | -------------------------- |
| **Collect Rumor**      | 1. Visit shipyard<br>2. Get gossip<br>3. Pay bribe | Rumor coordinates obtained |
| **View Triangulation** | 1. Collect 3+ rumors<br>2. View map                | Overlap area shown         |

---

## Phase 9: Leaderboards & Victory

**Goal:** Display rankings and victory progress.

**Priority:** Medium - Competitive feature.

### 9.1 API Integration

Add to `src/lib/api.ts`:

```typescript
leaderboards: {
  // Overall rankings
  async getOverall(galaxyUuid: string)

  // Combat rankings
  async getCombat(galaxyUuid: string)

  // Economic rankings
  async getEconomic(galaxyUuid: string)

  // Colonial rankings
  async getColonial(galaxyUuid: string)
}

galaxies: {
  // Get victory conditions
  async getVictoryConditions(galaxyUuid: string)
}

players: {
  // Get victory progress
  async getVictoryProgress(playerUuid: string)
}
```

### 9.2 New Components

| Component                  | Location                   | Description               |
| -------------------------- | -------------------------- | ------------------------- |
| `LeaderboardPanel.svelte`  | `src/lib/components/game/` | Tabbed leaderboards       |
| `LeaderboardTable.svelte`  | `src/lib/components/game/` | Rankings table            |
| `VictoryProgress.svelte`   | `src/lib/components/game/` | Player's victory progress |
| `VictoryConditions.svelte` | `src/lib/components/game/` | Victory path descriptions |

### 9.3 UI Integration

- Leaderboards accessible from main menu
- Victory progress in player stats
- Victory conditions displayed
- Ranking changes highlighted

### 9.4 Acceptance Criteria

- [ ] Player can view overall leaderboard
- [ ] Player can view category-specific leaderboards
- [ ] Victory conditions displayed
- [ ] Player's progress toward each victory shown
- [ ] Current rank displayed

### 9.5 Testing Methodology

#### Unit Tests

| Test Case                             | Description    | Expected Result     |
| ------------------------------------- | -------------- | ------------------- |
| `getOverall returns rankings`         | Fetch overall  | Rankings array      |
| `getCombat returns rankings`          | Fetch combat   | Rankings array      |
| `getVictoryProgress returns progress` | Fetch progress | Progress object     |
| `victory path calculation`            | Check progress | Correct percentages |

#### Component Tests

| Test Case                       | Description       | Expected Result       |
| ------------------------------- | ----------------- | --------------------- |
| `LeaderboardPanel tabs work`    | Click tabs        | Content changes       |
| `LeaderboardTable shows ranks`  | Render table      | Rankings displayed    |
| `VictoryProgress shows bars`    | Render progress   | Progress bars visible |
| `VictoryConditions shows paths` | Render conditions | All paths listed      |
| `player highlight in rankings`  | Current player    | Row highlighted       |

#### Integration Tests

| Test Case                       | Description   | Expected Result             |
| ------------------------------- | ------------- | --------------------------- |
| `leaderboard fetch and display` | Open → Load   | Rankings shown              |
| `victory progress updates`      | Trade → Check | Economic progress increases |

#### E2E Tests

| Test Scenario              | Steps                                                   | Expected Outcome        |
| -------------------------- | ------------------------------------------------------- | ----------------------- |
| **View Leaderboards**      | 1. Open leaderboards<br>2. Switch categories            | All categories viewable |
| **Check Victory Progress** | 1. View progress<br>2. Perform action<br>3. Check again | Progress updated        |

---

## Phase 10: Mirror Universe

**Goal:** Enable alternate dimension gameplay.

**Priority:** Low - Advanced feature.

### 10.1 API Integration

Add to `src/lib/api.ts`:

```typescript
mirror: {
  // Check access
  async checkAccess(playerUuid: string)

  // Enter mirror universe
  async enter(playerUuid: string)

  // Get mirror gate location
  async getGateLocation(galaxyUuid: string)
}
```

### 10.2 New Components

| Component                | Location                   | Description                 |
| ------------------------ | -------------------------- | --------------------------- |
| `MirrorAccess.svelte`    | `src/lib/components/game/` | Access check and entry      |
| `MirrorWarning.svelte`   | `src/lib/components/game/` | Danger warning              |
| `MirrorIndicator.svelte` | `src/lib/components/game/` | Current dimension indicator |

### 10.3 Acceptance Criteria

- [ ] Player can check mirror access requirements
- [ ] Player can enter mirror universe
- [ ] Visual indicator when in mirror dimension
- [ ] Enhanced rewards/risks displayed

### 10.4 Testing Methodology

#### Unit Tests

| Test Case                          | Description            | Expected Result         |
| ---------------------------------- | ---------------------- | ----------------------- |
| `checkAccess returns requirements` | Check access           | Requirements object     |
| `enter transitions dimension`      | Enter mirror           | Dimension state changed |
| `sensor level requirement`         | Check with low sensors | Access denied           |

#### Component Tests

| Test Case                         | Description | Expected Result        |
| --------------------------------- | ----------- | ---------------------- |
| `MirrorAccess shows requirements` | Render      | Requirements displayed |
| `MirrorWarning shows risks`       | Render      | Risks listed           |
| `MirrorIndicator shows dimension` | In mirror   | Indicator visible      |

#### E2E Tests

| Test Scenario    | Steps                                            | Expected Outcome  |
| ---------------- | ------------------------------------------------ | ----------------- |
| **Enter Mirror** | 1. Meet requirements<br>2. Find gate<br>3. Enter | Dimension changes |

---

## Phase 11: Notifications

**Goal:** Display in-game notifications.

**Priority:** Low - Quality of life feature.

### 11.1 API Integration

Add to `src/lib/api.ts`:

```typescript
notifications: {
  // List notifications
  async list(playerUuid: string)

  // Get unread count
  async getUnreadCount(playerUuid: string)

  // Mark as read
  async markRead(playerUuid: string, notificationId: number)

  // Mark all as read
  async markAllRead(playerUuid: string)

  // Delete notification
  async delete(playerUuid: string, notificationId: number)
}
```

### 11.2 New Components

| Component                  | Location              | Description                       |
| -------------------------- | --------------------- | --------------------------------- |
| `NotificationBell.svelte`  | `src/lib/components/` | Header notification indicator     |
| `NotificationList.svelte`  | `src/lib/components/` | Notification dropdown/panel       |
| `NotificationItem.svelte`  | `src/lib/components/` | Single notification               |
| `NotificationToast.svelte` | `src/lib/components/` | Toast popup for new notifications |

### 11.3 Acceptance Criteria

- [ ] Unread count displayed in header
- [ ] Notifications list accessible
- [ ] Individual notifications can be marked read
- [ ] Notifications can be deleted
- [ ] Toast notifications for important events

### 11.4 Testing Methodology

#### Unit Tests

| Test Case                      | Description | Expected Result      |
| ------------------------------ | ----------- | -------------------- |
| `list returns notifications`   | Fetch list  | Notifications array  |
| `getUnreadCount returns count` | Fetch count | Number returned      |
| `markRead updates status`      | Mark read   | Read status changed  |
| `markAllRead clears unread`    | Mark all    | Count becomes 0      |
| `delete removes notification`  | Delete      | Notification removed |

#### Component Tests

| Test Case                          | Description      | Expected Result     |
| ---------------------------------- | ---------------- | ------------------- |
| `NotificationBell shows count`     | Unread exist     | Count badge visible |
| `NotificationBell hidden if zero`  | No unread        | Badge hidden        |
| `NotificationList shows items`     | Render list      | Items displayed     |
| `NotificationItem shows content`   | Render item      | Content visible     |
| `NotificationToast appears`        | New notification | Toast shown         |
| `NotificationToast auto-dismisses` | Wait             | Toast disappears    |

#### Integration Tests

| Test Case                   | Description                | Expected Result |
| --------------------------- | -------------------------- | --------------- |
| `notification flow`         | Receive → View → Mark read | Count decreases |
| `toast on new notification` | Receive → Toast appears    | Toast visible   |

#### E2E Tests

| Test Scenario          | Steps                                | Expected Outcome       |
| ---------------------- | ------------------------------------ | ---------------------- |
| **View Notifications** | 1. Click bell<br>2. View list        | Notifications shown    |
| **Mark as Read**       | 1. Open notification<br>2. Mark read | Unread count decreases |

---

## Phase 12: NPC Interactions

**Goal:** Display and interact with NPCs.

**Priority:** Low - Single-player enhancement.

### 12.1 API Integration

Already partially implemented. Add:

```typescript
npcs: {
  // Get NPC details
  async get(npcUuid: string)
}
```

### 12.2 New Components

| Component               | Location                   | Description            |
| ----------------------- | -------------------------- | ---------------------- |
| `NpcList.svelte`        | `src/lib/components/game/` | NPCs in current system |
| `NpcCard.svelte`        | `src/lib/components/game/` | NPC info display       |
| `NpcInteraction.svelte` | `src/lib/components/game/` | Interaction options    |

### 12.3 Acceptance Criteria

- [ ] NPCs visible in current system
- [ ] NPC details displayed
- [ ] NPC archetype/difficulty shown

### 12.4 Testing Methodology

#### Unit Tests

| Test Case                     | Description | Expected Result     |
| ----------------------------- | ----------- | ------------------- |
| `get returns NPC details`     | Fetch NPC   | NPC object returned |
| `list returns NPCs in system` | Fetch list  | NPCs array          |

#### Component Tests

| Test Case                  | Description | Expected Result      |
| -------------------------- | ----------- | -------------------- |
| `NpcList shows NPCs`       | Render list | NPCs displayed       |
| `NpcCard shows archetype`  | Render card | Archetype shown      |
| `NpcCard shows difficulty` | Render card | Difficulty indicator |

#### E2E Tests

| Test Scenario | Steps                                     | Expected Outcome |
| ------------- | ----------------------------------------- | ---------------- |
| **View NPCs** | 1. Enter system with NPCs<br>2. View list | NPCs visible     |

---

## Testing Infrastructure

### Test Configuration

The project uses Vitest with separate configurations for client and server tests:

- **Client tests** (`*.svelte.spec.ts`): Run in browser via Playwright
- **Server tests** (`*.spec.ts`): Run in Node.js environment

### Test File Structure

```
src/
├── lib/
│   ├── stores/
│   │   ├── playerState.svelte.ts
│   │   └── playerState.svelte.spec.ts      # Unit tests
│   └── components/
│       └── game/
│           ├── TradingPanel.svelte
│           └── TradingPanel.svelte.spec.ts # Component tests
├── tests/
│   └── integration/
│       ├── trading.spec.ts                  # Integration tests
│       ├── travel.spec.ts
│       └── combat.spec.ts
e2e/
├── trading.spec.ts                          # E2E tests
├── travel.spec.ts
└── combat.spec.ts
```

### Mock Strategy

#### API Mocking

```typescript
// src/tests/mocks/api.ts
export const mockApi = {
	tradingHubs: {
		getInventory: vi.fn(),
		buy: vi.fn(),
		sell: vi.fn()
	}
	// ... other namespaces
};

// In tests
vi.mock('$lib/api', () => ({ api: mockApi }));
```

#### State Mocking

```typescript
// For component tests
const mockPlayerState = {
	credits: 10000,
	cargo: [],
	buyMineral: vi.fn(),
	sellMineral: vi.fn()
};

render(TradingPanel, {
	context: new Map([['playerState', mockPlayerState]])
});
```

### Test Commands

```bash
# Run all tests
npm run test

# Run unit tests in watch mode
npm run test:unit

# Run specific test file
npm run test:unit -- --run src/lib/stores/playerState.svelte.spec.ts

# Run E2E tests
npm run test:e2e

# Run tests with coverage
npm run test:coverage
```

### Coverage Requirements

| Category         | Minimum Coverage |
| ---------------- | ---------------- |
| State Management | 90%              |
| API Functions    | 85%              |
| UI Components    | 75%              |
| Integration      | 70%              |

### Continuous Integration

Tests should be run on:

- Every pull request
- Before merge to main
- On scheduled nightly builds

---

## Implementation Order Summary

| Phase | Name                      | Priority | Dependencies      |
| ----- | ------------------------- | -------- | ----------------- |
| 1     | Core Trading Loop         | High     | None              |
| 2     | Travel System             | High     | None              |
| 3     | Ship Management           | High     | Phase 1 (credits) |
| 4     | Combat System             | High     | Phase 2 (travel)  |
| 5     | Exploration & Scanning    | Medium   | Phase 2           |
| 6     | Star Charts & Cartography | Medium   | Phase 5           |
| 7     | Colony Management         | Medium   | Phase 2           |
| 8     | Precursor Ship Hunt       | Medium   | Phase 5, 6        |
| 9     | Leaderboards & Victory    | Medium   | Phases 1-7        |
| 10    | Mirror Universe           | Low      | Phase 5           |
| 11    | Notifications             | Low      | None              |
| 12    | NPC Interactions          | Low      | None              |

---

## Technical Notes

### API Request Pattern

All API calls should follow this pattern:

```typescript
const response = await api.namespace.method(params);
if (response.success && response.data) {
	// Handle success
} else {
	// Handle error: response.error?.message
}
```

### State Management Pattern

Follow the existing `playerState.svelte.ts` pattern using Svelte 5 runes:

```typescript
function createSomeState() {
  let state = $state<StateType>({ ... });

  return {
    get property() { return state.property; },
    async loadData() { ... },
    async performAction() { ... }
  };
}

export const someState = createSomeState();
```

### Error Handling

- Display user-friendly error messages
- Log detailed errors to console
- Handle network timeouts gracefully
- Provide retry options where appropriate

---

## Success Metrics

1. **Trading Loop** - Player can complete a profitable trade route
2. **Travel System** - Player can navigate between systems
3. **Combat** - Player can survive pirate encounters
4. **Progression** - Player can upgrade ship and buy new ship
5. **Exploration** - Player can discover new systems
6. **Victory** - Player can track progress toward victory conditions

---

## Risks and Mitigations

| Risk           | Impact | Mitigation                                |
| -------------- | ------ | ----------------------------------------- |
| API changes    | High   | Version API endpoints, document contracts |
| Complex state  | Medium | Keep state normalized, use derived states |
| Performance    | Medium | Lazy load components, cache API responses |
| UX complexity  | Medium | Progressive disclosure, guided tutorials  |
| Test flakiness | Medium | Use proper async handling, retry logic    |

---

## Appendix: File Structure

```
src/lib/
├── api.ts                    # API client (modify)
├── stores/
│   └── playerState.svelte.ts # Player state (modify)
├── types/
│   ├── trading.ts            # NEW: Trading types
│   ├── combat.ts             # NEW: Combat types
│   ├── colony.ts             # NEW: Colony types
│   └── ...
└── components/
    └── game/
        ├── trading/          # NEW: Trading components
        ├── travel/           # NEW: Travel components
        ├── combat/           # NEW: Combat components
        ├── ships/            # NEW: Ship management
        ├── exploration/      # NEW: Exploration components
        ├── colonies/         # NEW: Colony components
        └── ...

src/tests/
├── mocks/
│   ├── api.ts                # API mocks
│   └── state.ts              # State mocks
└── integration/
    ├── trading.spec.ts
    ├── travel.spec.ts
    └── combat.spec.ts

e2e/
├── trading.spec.ts
├── travel.spec.ts
├── combat.spec.ts
└── full-session.spec.ts      # Complete gameplay flow
```

---

_Document maintained by development team. Update as implementation progresses._
