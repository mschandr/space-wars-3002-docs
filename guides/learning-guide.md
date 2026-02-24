# Space Wars 3002 Frontend — Learning Guide

> A file-by-file, method-by-method walkthrough of this SvelteKit application for people learning **Svelte 5** and **JavaScript/TypeScript**.

---

## Table of Contents

1. [How This App Works (Big Picture)](#how-this-app-works-big-picture)
2. [Key Concepts You'll See Everywhere](#key-concepts-youll-see-everywhere)
3. [Configuration Files](#configuration-files)
4. [The API Client — `src/lib/api.ts`](#the-api-client--srclibapits)
5. [Authentication Store — `src/lib/auth.svelte.ts`](#authentication-store--srclibautsveltets)
6. [Galaxy Cache — `src/lib/galaxyCache.ts`](#galaxy-cache--srclibgalaxycachets)
7. [Player State Store — `src/lib/stores/playerState.svelte.ts`](#player-state-store--srclibstoresplayerstatesveltets)
8. [Game Types — `src/lib/types/scanning.ts`](#game-types--srclibtypesscanningts)
9. [Routes (Pages)](#routes-pages)
   - [Root Layout](#root-layout----layoutsvelte)
   - [Home Page](#home-page----pagesvelte)
   - [Login Page](#login-page--loginpagesvelte)
   - [Register Pages](#register-pages)
   - [Galaxy Selection](#galaxy-selection--galaxiespagesvelte)
   - [Gameplay Page](#gameplay-page--galaxiesuuidplaypagesvelte)
   - [Star Map Page](#star-map-page--galaxiesuuidplaymappagesvelte)
10. [Shared Components](#shared-components)
    - [AuthLayout.svelte](#authlayoutsvelte)
    - [SpaceBackdrop.svelte](#spacebackdropsvelte)
    - [SpaceLoader.svelte](#spaceloadersvelte)
    - [GalaxyCard.svelte](#galaxycardsvelte)
11. [Game UI Components](#game-ui-components)
    - [StatBar.svelte](#statbarsvelte)
    - [PlayerStats.svelte](#playerstatssvelte)
    - [SystemMenu.svelte](#systemmenusvelte)
    - [ActionPanel.svelte](#actionpanelsvelte)
    - [TradingPanel.svelte](#tradingpanelsvelte)
    - [WarpLoader.svelte](#warploadersvelte)
    - [SectorGrid.svelte](#sectorgridsvelte)
    - [Other Game Components](#other-game-components)
12. [Svelte 5 Runes Reference](#svelte-5-runes-reference)
13. [Common Patterns in This Codebase](#common-patterns-in-this-codebase)
14. [How Data Flows Through the App](#how-data-flows-through-the-app)

---

## How This App Works (Big Picture)

Space Wars 3002 is a **turn-based space trading & conquest game**. The frontend is built with **SvelteKit 2** (a framework that adds routing, server-side rendering, and more to Svelte) and **Svelte 5** (the latest version of Svelte with a new "runes" reactivity system).

**The user journey:**
1. **Register/Login** → Create an account or sign in
2. **Galaxy Selection** → Pick an existing galaxy or create a new one
3. **Gameplay** → Explore star systems, trade minerals, buy ships, warp between systems
4. **Star Map** → View the full galaxy from above with fog-of-war

**Tech stack at a glance:**
| Technology | Role |
|---|---|
| **Svelte 5** | UI framework (components + reactivity) |
| **SvelteKit 2** | Full-stack framework (routing, SSR, builds) |
| **TypeScript** | Type-safe JavaScript |
| **Tailwind CSS v4** | Utility-first CSS framework |
| **Vitest + Playwright** | Testing |

---

## Key Concepts You'll See Everywhere

### Svelte 5 Runes (New Reactivity)

Svelte 5 introduced "runes" — special functions that start with `$`. Here's what each one does:

| Rune | What It Does | Example |
|---|---|---|
| `$state` | Creates reactive state that triggers re-renders when changed | `let count = $state(0)` |
| `$derived` | Computes a value from other reactive values (recalculates automatically) | `const double = $derived(count * 2)` |
| `$effect` | Runs side effects when dependencies change (like `useEffect` in React) | `$effect(() => { console.log(count) })` |
| `$props()` | Receives props passed to a component | `let { name, age } = $props()` |

### TypeScript Interfaces

You'll see `interface` everywhere. These define the *shape* of data:

```typescript
// This says "a User object must have an id (number), name (string), and email (string)"
interface User {
  id: number;
  name: string;
  email: string;
}
```

### Async/Await

API calls use `async`/`await` to handle asynchronous operations:

```typescript
// Without async/await (harder to read):
fetch('/api/data').then(response => response.json()).then(data => console.log(data));

// With async/await (reads like synchronous code):
const response = await fetch('/api/data');
const data = await response.json();
console.log(data);
```

---

## Configuration Files

### `package.json`
**What it is:** The project's identity card. Lists dependencies, scripts, and metadata.

**Key sections:**
- `"scripts"` — Commands you can run:
  - `npm run dev` — Start the development server
  - `npm run build` — Build for production
  - `npm run test` — Run tests
  - `npm run check` — Check TypeScript types
  - `npm run lint` — Check code formatting
- `"devDependencies"` — Tools used during development (Svelte, Vite, etc.)

### `svelte.config.js`
**What it is:** Tells SvelteKit how to build and deploy the app.

**Key parts:**
- `adapter` — Determines where the app can be deployed (auto-detect)
- `preprocess` — Enables TypeScript and other pre-processors in `.svelte` files

### `vite.config.ts`
**What it is:** Configuration for Vite, the build tool that powers SvelteKit.

**Key parts:**
- `server` — Dev server settings (hostname, port, HMR)
- `test` — Vitest configuration with two test projects:
  - **Client tests** (`*.svelte.spec.ts`) — Run in a real browser via Playwright
  - **Server tests** (`*.spec.ts`) — Run in Node.js

### `tsconfig.json`
**What it is:** TypeScript compiler configuration.

**Key settings:**
- `"strict": true` — Enables all strict type checks
- `"paths": { "$lib": ["./src/lib"] }` — Lets you write `$lib/api` instead of `../../lib/api`

---

## The API Client — `src/lib/api.ts`

**Purpose:** This is the app's communication layer with the backend server. Every API call goes through this file.

**Size:** ~1500 lines (the largest file in the project)

### How It's Organized

The file has three parts:

#### Part 1: Type Definitions (Lines 1-1033)

These `interface` declarations describe every piece of data the API can send or receive. Think of them as contracts: "when I call this endpoint, I expect data shaped like this."

**Key types to understand:**

```typescript
// Every API response wraps data in this structure
interface ApiResponse<T> {
  success: boolean;          // Did the request work?
  data?: T;                  // The actual data (generic type T)
  error?: {                  // Error details (if success is false)
    code: string;            // Machine-readable code like "NOT_FOUND"
    message: string;         // Human-readable message
    details: Record<string, string[]> | null;  // Per-field errors
  };
  meta: {
    timestamp: string;       // When the response was generated
    request_id: string;      // Unique ID for debugging
  };
}
```

> **Learning note:** The `<T>` is a *generic type*. It means "T is a placeholder — it can be any type." When you call `request<User>(...)`, T becomes `User`, so `data` is typed as `User | undefined`.

**Other important types:**
| Type | What it represents |
|---|---|
| `User` | A logged-in user (id, name, email) |
| `AuthData` | Login response (user + access token) |
| `GalaxySummary` | Lightweight galaxy info for lists |
| `Galaxy` | Full galaxy object with dimensions |
| `PlayerData` | A player in a galaxy (call sign, credits, location) |
| `MyShipResponse` | The player's active ship stats |
| `CurrentLocationResponse` | What's at the player's current location |
| `HubInventoryItem` | A tradeable mineral with prices |
| `TravelResponse` | Result of warping to a new system |
| `FacilitiesResponse` | What services are available at current location |
| `CargoItem` | An item in the player's cargo hold |

#### Part 2: The Request Helper (Lines 1034-1074)

```typescript
async function request<T>(endpoint: string, options: RequestOptions = {}): Promise<ApiResponse<T>>
```

**What it does:** A reusable fetch wrapper that:
1. Reads the auth token from `localStorage`
2. Sets JSON headers + Authorization header
3. Creates an `AbortController` for timeouts (default 10 seconds)
4. Makes the fetch call
5. Returns the parsed JSON response
6. Handles timeout errors gracefully

**Why it exists:** Without this, every API call would repeat ~30 lines of boilerplate code. This is the [DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) in action.

> **Learning note:** `AbortController` is a browser API that lets you cancel a `fetch` request. The pattern is:
> ```typescript
> const controller = new AbortController();
> setTimeout(() => controller.abort(), 10000); // Cancel after 10s
> fetch(url, { signal: controller.signal });    // Pass the signal
> ```

#### Part 3: The API Object (Lines 1076-1508)

The `api` object is organized by resource:

```typescript
export const api = {
  auth: { ... },        // Login, register, logout, etc.
  galaxies: { ... },    // List, create, join galaxies
  npcs: { ... },        // NPC management
  players: { ... },     // Player actions (scan, travel, trade, etc.)
  ships: { ... },       // Ship catalog
  location: { ... },    // Current location details
  tradingHubs: { ... }, // Buy/sell minerals
  minerals: { ... },    // Mineral list
  sectors: { ... },     // Sector details
};
```

**Method breakdown by namespace:**

##### `api.auth` — Authentication
| Method | HTTP | Endpoint | What it does |
|---|---|---|---|
| `register(data)` | POST | `/auth/register` | Create a new account |
| `login(data)` | POST | `/auth/login` | Sign in, get a token |
| `logout()` | POST | `/auth/logout` | Invalidate the token |
| `me()` | GET | `/auth/me` | Get current user from token |
| `refresh()` | POST | `/auth/refresh` | Refresh an expiring token |

##### `api.galaxies` — Galaxy Management
| Method | HTTP | Endpoint | What it does |
|---|---|---|---|
| `list()` | GET | `/galaxies/list` | Lightweight list (my games + open games) |
| `listFull()` | GET | `/galaxies` | Full galaxy objects |
| `get(uuid)` | GET | `/galaxies/{uuid}` | Single galaxy details |
| `create(data)` | POST | `/galaxies/create` | Create galaxy (10-min timeout!) |
| `getMyPlayer(uuid)` | GET | `/galaxies/{uuid}/my-player` | Get player in this galaxy |
| `getMyShip(uuid)` | GET | `/galaxies/{uuid}/my-ship` | Get active ship stats |
| `join(uuid, data)` | POST | `/galaxies/{uuid}/join` | Join a galaxy (idempotent) |
| `getMap(uuid)` | GET | `/galaxies/{uuid}/map` | Get star map data |

> **Learning note:** The `list()` method doesn't use the shared `request()` helper — it has its own fetch logic with extra response normalization. This is because the API response format needed special handling (lines 1116-1166).

##### `api.players` — Player Actions
| Method | HTTP | Endpoint | What it does |
|---|---|---|---|
| `scanSystem(uuid, poi)` | POST | `/players/{uuid}/scan-system` | Scan a star system for info |
| `getSystemData(uuid, poi)` | GET | `/players/{uuid}/system-data/{poi}` | Get system details |
| `travel(uuid, dest)` | POST | `/players/{uuid}/travel` | Warp to another system |
| `getLocalBodies(uuid)` | GET | `/players/{uuid}/local-bodies` | Planets, moons, stations |
| `getCargo(uuid)` | GET | `/players/{uuid}/cargo` | What's in your cargo hold |
| `getFacilities(uuid)` | GET | `/players/{uuid}/facilities` | Available services |
| `purchaseShip(uuid, data)` | POST | `/players/{uuid}/ships/purchase` | Buy a ship |
| `switchShip(uuid, data)` | POST | `/players/{uuid}/ships/switch` | Change active ship |
| `getBulkScanLevels(uuid, pois)` | POST | `/players/{uuid}/bulk-scan-levels` | Batch scan level fetch |

##### `api.tradingHubs` — Trading
| Method | HTTP | Endpoint | What it does |
|---|---|---|---|
| `getInventory(hub)` | GET | `/trading-hubs/{hub}/inventory` | What the hub sells |
| `buy(hub, data)` | POST | `/trading-hubs/{hub}/buy` | Buy minerals |
| `sell(hub, data)` | POST | `/trading-hubs/{hub}/sell` | Sell minerals |

---

## Authentication Store — `src/lib/auth.svelte.ts`

**Purpose:** Manages who's logged in. This is a *reactive store* — when auth state changes, every component using it re-renders automatically.

**File extension note:** The `.svelte.ts` extension tells SvelteKit "this is a TypeScript file that uses Svelte runes (`$state`, `$derived`, etc.)."

### Architecture: The Factory Pattern

```typescript
function createAuthStore() {
  const state = $state<AuthState>({ ... });  // Private reactive state

  // Private helper functions
  function setAuth(user, token) { ... }
  function clearAuth() { ... }

  // Public methods
  async function initialize() { ... }
  async function login(email, password) { ... }
  async function register(name, email, password) { ... }
  async function logout() { ... }

  // Return public API with getter properties
  return {
    get user() { return state.user; },
    get isAuthenticated() { return state.isAuthenticated; },
    initialize, login, register, logout
  };
}

export const auth = createAuthStore();
```

> **Learning note:** This pattern uses JavaScript *closures*. The `state` variable is "closed over" — it's private and can't be accessed directly from outside. Only the returned getters and methods can touch it. This prevents accidental state corruption.

### Method-by-Method Breakdown

#### `initialize()` (Line 73)
**When called:** On app startup (in the root `+page.svelte`)

**What it does:**
1. Skip if already authenticated
2. Read token from `localStorage`
3. Filter out garbage tokens (`"undefined"`, `"null"`, very short strings)
4. Try the cached user first (avoids an API call)
5. If no cache, call `api.auth.me()` to validate the token
6. If validation fails, clear everything

**Why caching matters:** Without the cache, every page load would make an API call to `/auth/me`. The 5-minute cache (`AUTH_CACHE_TTL = 5 * 60 * 1000`) avoids this.

#### `login(email, password)` (Line 120)
**What it does:**
1. Call `api.auth.login()`
2. If successful, extract the `access_token` and user
3. Call `setAuth()` to save the token and update state
4. Return the response (caller handles navigation)

#### `register(name, email, password)` (Line 134)
**What it does:** Same pattern as login — calls the register API, saves auth if successful.

#### `logout()` (Line 147)
**What it does:** Calls the logout API (invalidates server-side token), then clears local state.

### Cache Helper Functions (Lines 18-47)

```typescript
function getCachedUser(): AuthCache | null   // Read from localStorage
function cacheUser(user: User): void         // Write to localStorage
function clearUserCache(): void              // Delete from localStorage
```

**Pattern:** Each function checks `typeof localStorage === 'undefined'` first. This is because SvelteKit can run code on the server (during SSR), where `localStorage` doesn't exist.

---

## Galaxy Cache — `src/lib/galaxyCache.ts`

**Purpose:** Caches the galaxy list in `localStorage` so the galaxy selection page loads instantly.

**Size:** 77 lines — the smallest module, and a great example of a focused utility.

### Exported Functions

#### `getCachedGalaxyList()` (Line 43)
```typescript
export function getCachedGalaxyList(): GalaxyListResponse | null
```
**What it does:** Returns the cached galaxy list, or `null` if no valid cache exists.

#### `cacheGalaxyList(data)` (Line 53)
```typescript
export function cacheGalaxyList(data: GalaxyListResponse): void
```
**What it does:** Saves the galaxy list to localStorage with a version number.

#### `clearGalaxyCache()` (Line 62)
```typescript
export function clearGalaxyCache(): void
```
**What it does:** Removes the cache entry from localStorage.

#### `getGalaxyFromCache(uuid)` (Line 67)
```typescript
export function getGalaxyFromCache(uuid: string): GalaxySummary | null
```
**What it does:** Searches both `my_games` and `open_games` arrays to find a galaxy by UUID.

### Version-Based Cache Invalidation

```typescript
const CACHE_VERSION = 3;
```

**How it works:** When the cache format changes, the version number is bumped. Old caches with a different version are ignored (line 22: `if (parsed.version === CACHE_VERSION)`).

> **Learning note:** This is simpler than time-based expiration (TTL). The auth cache uses TTL; the galaxy cache uses versioning. Both are valid strategies — choose based on your use case.

---

## Player State Store — `src/lib/stores/playerState.svelte.ts`

**Purpose:** The central nervous system of the game. Manages everything about the player: location, ship, cargo, credits, travel state, and more.

**Size:** ~990 lines — the second largest file.

### State Shape

The store manages a massive state object (lines 50-78):

```typescript
interface PlayerState {
  // Identity
  playerUuid: string | null;
  galaxyUuid: string | null;
  galaxyName: string | null;
  callSign: string | null;

  // Location
  currentSystem: SystemInfo | null;   // Where you are
  currentSector: SectorInfo | null;   // Which grid sector
  locationDetails: CurrentLocationResponse | null;  // What's here
  facilities: FacilitiesResponse | null;            // What services exist

  // Ship
  activeShip: ShipInfo | null;        // Currently flying
  ships: ShipInfo[];                  // All owned ships
  ship: ShipStats | null;             // Detailed ship stats

  // Economy
  credits: number;
  cargo: CargoItem[];
  cargoCapacity: number;
  cargoUsed: number;

  // Trading
  tradingHubInventory: HubInventoryItem[];
  currentTradingHubUuid: string | null;

  // Exploration
  scanLevels: Record<string, number>; // POI UUID → scan level
  level: number;
  experience: number;

  // Travel Animation
  isTraveling: boolean;
  travelDestination: string | null;
  travelStatus: string | null;

  // UI State
  isLoading: boolean;
  needsCreation: boolean;  // True if player doesn't exist in galaxy yet
  error: string | null;
}
```

### Method-by-Method Breakdown

#### `initialize(galaxyUuid)` (Line 489)
**When called:** When the player enters a galaxy (gameplay page mount)

**Step by step:**
1. Set loading state, clear errors
2. Try to get galaxy name from cache (instant display)
3. Call `api.galaxies.getMyPlayer(galaxyUuid)`
4. If player exists → call `setPlayerData()`, extract sector info, load ship
5. If error code is `NO_PLAYER_IN_GALAXY` → set `needsCreation = true`
6. Otherwise → show error message

> **Learning note:** The response format varies (`{ player: {...} }` vs direct player object). Line 510 handles both: `const playerData = data.player ?? data`. The `??` operator means "use the left side unless it's null/undefined."

#### `setPlayerData(player)` (Line 112)
**What it does:** Normalizes API response data into the store's format.

**Key transformations:**
- `player.active_ship` → `state.activeShip` (camelCase)
- `player.current_location` → `state.currentSystem` (restructured)
- `player.credits ?? 0` → `state.credits` (default to 0)

> **Learning note:** The `??` (nullish coalescing) operator is different from `||`. With `||`, the value `0` would be treated as falsy and replaced. With `??`, only `null` and `undefined` are replaced. So `player.credits ?? 0` correctly preserves a `credits` value of `0`.

#### `joinGalaxy(callSign)` (Line 559)
**What it does:** Sends a join request to enter a galaxy with a chosen call sign.

**Error handling:** Uses a `switch` statement to provide user-friendly error messages for each possible error code (`GALAXY_FULL`, `DUPLICATE_CALL_SIGN`, etc.).

#### `loadLocationDetails()` (Line 170)
**What it does:** Fetches detailed information about the current star system (services, gates, trading hubs).

#### `loadFacilities()` (Line 201)
**What it does:** Fetches what facilities (trading hubs, shipyards, bars, etc.) exist at the player's current location.

#### `loadMyShip()` (Line 314)
**What it does:** Gets detailed ship stats from the `/galaxies/{uuid}/my-ship` endpoint.

**Key transformation (lines 326-343):**
```typescript
state.ship = {
  uuid: shipData.uuid,
  name: shipData.name,
  hull: { current: shipData.hull, max: shipData.max_hull },
  shield: { current: shipData.shields, max: shipData.max_shields },
  fuel: { current: shipData.current_fuel, max: shipData.max_fuel, regenRate: shipData.fuel_regen_rate },
  // ... more fields
};
```

> **Learning note:** The API uses `snake_case` (common in Python/PHP backends), but JavaScript convention is `camelCase`. This mapping happens in the store so components don't have to think about it.

#### `loadCargo()` (Line 374)
**What it does:** Fetches the player's cargo manifest and updates `cargo`, `cargoCapacity`, and `cargoUsed`.

#### `buyMineral(hubUuid, mineralUuid, quantity)` (Line 419)
**What it does:**
1. Call `api.tradingHubs.buy()`
2. Update credits with `remaining_credits` from response
3. Reload cargo and hub inventory (to refresh quantities)
4. If the buy fails, throw an error (so the UI can show it)

#### `sellMineral(hubUuid, mineralUuid, quantity)` (Line 450)
**What it does:** Mirror of `buyMineral` but for selling.

#### `travel(destinationUuid, destinationName)` (Line 687)
**What it does:** The most complex method — handles warp travel with animations.

**Step by step:**
1. Set `isTraveling = true`, show "Initiating warp drive..."
2. Call `api.players.travel()`
3. **If status is "generating":** The destination system doesn't exist yet! Poll every 2 seconds for up to 2 minutes until it's ready.
4. Update `currentSystem`, `currentSector`, fuel
5. Clear `locationDetails` and `facilities` (they belong to the old location)
6. Enforce minimum 2-second display time for the warp animation
7. Set `isTraveling = false` in `finally` block (always runs)

> **Learning note:** The `finally` block (line 848) is crucial. Even if the travel request throws an error, the `isTraveling` flag gets reset. Without `finally`, a failed travel could leave the UI stuck in "traveling" mode forever.

#### `scanSystem(poiUuid, force)` (Line 651)
**What it does:** Scans a system to discover features. Caches the scan level locally.

#### `loadBulkScanLevels(poiUuids)` (Line 667)
**What it does:** Fetches scan levels for many systems at once (used by the star map). More efficient than calling `scanSystem` individually.

#### `reset()` (Line 855)
**What it does:** Resets every field back to its default value. Called when leaving a galaxy.

---

## Game Types — `src/lib/types/scanning.ts`

**Purpose:** TypeScript type definitions for the scanning/exploration system and star map.

### Key Types

#### `MapSystemData` (Line 78)
Used by the star map to render each system:
```typescript
interface MapSystemData {
  uuid: string;
  name: string;
  type: string;
  position: { x: number; y: number };
  scan_level: number;           // 0-9, determines visibility
  has_warp_gate: boolean;       // Can you warp here?
  is_inhabited: boolean;        // Has settlements?
  is_hazardous: boolean;        // Dangerous?
}
```

#### `SCAN_COLORS` (Line 127)
A constant object mapping scan levels (0-9) to visual styles:
```typescript
export const SCAN_COLORS: Record<number, { color: string; opacity: number }> = {
  0: { color: '#1a1a2e', opacity: 0.2 },  // Unscanned — nearly invisible
  // ...
  9: { color: '#ff6600', opacity: 1.0 },   // Precursor secrets — bright orange
};
```

> **Learning note:** `Record<number, ...>` is a TypeScript utility type that says "an object whose keys are numbers and values are `{ color, opacity }`." It's like a dictionary/map.

---

## Routes (Pages)

### How SvelteKit Routing Works

SvelteKit uses **file-based routing**. The folder structure under `src/routes/` maps directly to URLs:

```
src/routes/
├── +page.svelte              →  /
├── login/+page.svelte        →  /login
├── register/
│   ├── +page.svelte          →  /register
│   └── confirm/+page.svelte  →  /register/confirm
└── galaxies/
    ├── +page.svelte          →  /galaxies
    └── [uuid]/play/
        ├── +page.svelte      →  /galaxies/abc-123/play
        └── map/+page.svelte  →  /galaxies/abc-123/play/map
```

- `+page.svelte` = The page component for that route
- `+page.ts` = Data loading that runs before the page renders
- `+layout.svelte` = Wraps all pages at that level and below
- `[uuid]` = Dynamic parameter (the actual UUID is captured as `params.uuid`)

---

### Root Layout — `+layout.svelte`

**File:** `src/routes/+layout.svelte` (15 lines)

**What it does:** Wraps every page in the app.

```svelte
<script lang="ts">
  import '../app.css';                        // Load global styles (Tailwind)
  import SpaceBackdrop from '$lib/components/SpaceBackdrop.svelte';
  let { children } = $props();               // Svelte 5: receive child content
</script>

<svelte:head>
  <link rel="icon" href={favicon} />          <!-- Set the browser tab icon -->
</svelte:head>

<SpaceBackdrop />                             <!-- Animated space background -->
{@render children?.()}                        <!-- Render the current page -->
```

> **Learning note:** `{@render children?.()}` is Svelte 5 syntax for rendering child content. The `?.()` is optional chaining — it safely calls the function even if `children` is undefined.

---

### Home Page — `+page.svelte`

**File:** `src/routes/+page.svelte` (137 lines)

**What it does:** The landing page. Shows different content based on auth state.

**Key concepts demonstrated:**
- **Lifecycle:** `onMount()` runs `auth.initialize()` when the component first appears
- **Conditional rendering:** `{#if}` / `{:else if}` / `{:else}` blocks
- **Scoped styles:** The `<style>` block only affects this component

**Methods:**

| Method | Line | What it does |
|---|---|---|
| `handleLogout()` | 11 | Calls `auth.logout()` then navigates to `/login` |

**Template structure:**
```
if auth.isLoading     → "Loading..."
else if authenticated → Welcome message + "Continue" button + "Logout" button
else                  → "Login" button + "Register" button
```

> **Learning note:** `goto()` from `$app/navigation` is SvelteKit's programmatic navigation. It's like `window.location.href = '...'` but works with SvelteKit's client-side routing (no full page reload).

---

### Login Page — `login/+page.svelte`

**File:** `src/routes/login/+page.svelte` (~206 lines)

**State variables:**
```typescript
let email = $state('');           // Form input
let password = $state('');        // Form input
let rememberMe = $state(false);   // Checkbox
let error = $state('');           // Top-level error message
let fieldErrors = $state<Record<string, string[]>>({});  // Per-field errors
let isSubmitting = $state(false); // Disable button while submitting
```

**Methods:**

| Method | What it does |
|---|---|
| `handleSubmit()` | Validates, calls `auth.login()`, redirects to `/galaxies` on success |

**Form handling flow:**
1. User fills in email + password
2. Clicks "Login" (triggers `handleSubmit`)
3. Set `isSubmitting = true` (disables button, shows spinner)
4. Call `auth.login(email, password)`
5. If success → `goto('/galaxies')`
6. If error → Show error message, extract field errors from `response.error.details`
7. Set `isSubmitting = false`

> **Learning note:** `fieldErrors` uses `Record<string, string[]>` — an object where keys are field names (`"email"`, `"password"`) and values are arrays of error messages. This allows showing multiple validation errors per field.

---

### Register Pages

**File:** `src/routes/register/+page.svelte` (~215 lines)

Similar to the login page but with additional fields (`name`, `passwordConfirmation`). Includes client-side password match validation.

**File:** `src/routes/register/confirm/+page.svelte` (74 lines)

A simple success page shown after registration. No interactivity — just a "Registration Complete!" message with a link to login.

---

### Galaxy Selection — `galaxies/+page.svelte`

**File:** `src/routes/galaxies/+page.svelte` (~833 lines)

**Purpose:** The main hub — lists galaxies, lets you create new ones.

**State variables:**
```typescript
// Galaxy lists
let myGames = $state<GalaxySummary[]>([]);
let openGames = $state<GalaxySummary[]>([]);
let isLoading = $state(true);

// Create modal
let showCreateModal = $state(false);
let createForm = $state({ size_tier: 'small', game_mode: 'multiplayer', name: '' });
let isCreating = $state(false);
let creationStats = $state<GalaxyCreationStatistics | null>(null);
```

**Key methods:**

| Method | What it does |
|---|---|
| `loadGalaxies()` | Fetches galaxy list from API, updates cache |
| `handleCreateGalaxy()` | Creates a new galaxy (10-min timeout!), shows stats |
| `handleEnterGalaxy(galaxy)` | Navigates to `/galaxies/{uuid}/play` |
| `handleViewMap(galaxy)` | Navigates to `/galaxies/{uuid}/play/map` |

**Lifecycle (onMount):**
1. Load cached galaxies immediately (no loading spinner!)
2. Fetch fresh data from API in background
3. Start 10-second refresh interval

> **Learning note:** The "cache-then-network" pattern gives instant UI feedback. Users see stale data immediately, then it updates silently when fresh data arrives. This is a common UX optimization.

**Galaxy Creation Flow:**
1. User clicks "Create Galaxy" → modal opens
2. Choose size (Small/Medium/Large) and mode (Solo/Multiplayer)
3. Click "Create" → `SpaceLoader` animation appears
4. Wait up to 10 minutes for generation
5. Show creation statistics (admin only)
6. Navigate to the new galaxy

---

### Gameplay Page — `galaxies/[uuid]/play/+page.svelte`

**File:** `src/routes/galaxies/[uuid]/play/+page.svelte` (~100+ lines)

**Purpose:** The main game interface where you interact with star systems.

**Data loading (`+page.ts`):**
```typescript
export const load: PageLoad = ({ params }) => ({
  galaxyUuid: params.uuid   // Extract UUID from URL
});
```

**Layout (3 panels):**
```
┌──────────┬────────────────────────────┬──────────────┐
│ Player   │     Action Panel           │   Sector     │
│ Stats    │  (context-dependent)       │   Grid       │
│          │                            │              │
│ (Ship    │  [Planets | Trading |      │  [2D grid    │
│  bars)   │   Warp | Shipyard | ...]   │   showing    │
│          │                            │   position]  │
├──────────┤                            │              │
│ System   │                            │              │
│ Menu     │                            │              │
└──────────┴────────────────────────────┴──────────────┘
```

**Key state:**
```typescript
let activeMenuItem = $state<MenuItemId | null>(null);  // Which service is selected
```

**Lifecycle:**
1. Check auth (redirect to login if not authenticated)
2. Initialize playerState with `galaxyUuid`
3. If player needs creation → show "Join Galaxy" form
4. Otherwise → load location details + facilities in parallel

> **Learning note:** `Promise.all()` runs multiple async operations simultaneously:
> ```typescript
> await Promise.all([
>   playerState.loadLocationDetails(),
>   playerState.loadFacilities()
> ]);
> // Both complete before continuing
> ```

---

### Star Map Page — `galaxies/[uuid]/play/map/+page.svelte`

**File:** `src/routes/galaxies/[uuid]/play/map/+page.svelte` (~150+ lines)

**Purpose:** Full galaxy view with all star systems, fog-of-war, and travel controls.

**State:**
```typescript
let systems = $state<MapSystemData[]>([]);
let selectedSystem = $state<string | null>(null);
let zoom = $state(1.0);                        // 0.5x to 3.0x
let filters = $state({ gates: true, inhabited: true, hazards: true });
```

**Lifecycle:**
1. Initialize player state if needed
2. Fetch map data via `api.galaxies.getMap(uuid)`
3. Normalize response format (handles multiple API formats)
4. Load scan levels for all systems via `playerState.loadBulkScanLevels()`

---

## Shared Components

### `AuthLayout.svelte`

**File:** `src/lib/components/AuthLayout.svelte` (80 lines)

**Purpose:** Wraps the login and register pages with a consistent look.

**Props:**
```typescript
let { children } = $props();  // Svelte 5 snippet (child content)
```

**What it renders:** A centered card with title bar, dark background, and glassmorphism effect.

> **Learning note:** In Svelte 5, child content is passed as a `Snippet`. The parent renders it with `{@render children()}`. This replaces Svelte 4's `<slot />` syntax.

---

### `SpaceBackdrop.svelte`

**File:** `src/lib/components/SpaceBackdrop.svelte` (107 lines)

**Purpose:** The animated starfield background visible behind every page.

**Props:**
```typescript
interface Props {
  showNebula?: boolean;   // Toggle nebula layer (default: true)
}
let { showNebula = true } = $props();
```

**How it works — Three Layers:**
1. **Nebula layer** — A background image (`/nursery.jpg.webp`) at 60% opacity
2. **Stars layer** — Tiny radial gradients repeated in a grid pattern, drifting slowly (120s animation)
3. **Dust layer** — Large colored ellipses drifting in the opposite direction (180s animation)

**CSS technique — Radial Gradient Stars:**
```css
background-image:
  radial-gradient(1px 1px at 20px 30px, white, transparent),
  radial-gradient(1px 1px at 40px 70px, rgba(255,255,255,0.8), transparent),
  /* ... more "stars" ... */;
background-size: 520px 180px;
background-repeat: repeat;
```

This creates a repeating pattern of bright dots that looks like stars. Each `radial-gradient` is one "star."

**Accessibility:**
```css
@media (prefers-reduced-motion: reduce) {
  .stars-layer, .dust-layer { animation: none; }
}
```

> **Learning note:** Always respect `prefers-reduced-motion`. Users with motion sensitivity or vestibular disorders set this OS preference. Removing animations for them is important accessibility practice.

---

### `SpaceLoader.svelte`

**File:** `src/lib/components/SpaceLoader.svelte` (202 lines)

**Purpose:** An animated galaxy loader shown during long operations (like galaxy creation).

**Props:**
```typescript
interface Props {
  message?: string;   // Custom message (default: "Loading...")
}
```

**Animation elements:** Core pulsing sphere, 3 orbiting planets, 12 twinkling stars, pulsing text.

---

### `GalaxyCard.svelte`

**File:** `src/lib/components/GalaxyCard.svelte` (209 lines)

**Purpose:** A card component for each galaxy in the selection list.

**Props:**
```typescript
interface Props {
  galaxy: GalaxySummary;
  onEnter: (galaxy: GalaxySummary) => void;
  onViewMap: (galaxy: GalaxySummary) => void;
}
```

**Interesting technique — Dynamic Colors:**
```typescript
function getGalaxyHue(name: string): number {
  let hash = 0;
  for (let i = 0; i < name.length; i++) {
    hash = name.charCodeAt(i) + ((hash << 5) - hash);
  }
  return Math.abs(hash) % 360;
}
```

This generates a consistent color for each galaxy based on its name. The same name always produces the same hue.

> **Learning note:** `charCodeAt()` gets the Unicode number for a character. The bit-shifting (`<< 5`) and subtraction create a hash. `% 360` constrains it to the HSL color wheel (0-360°). This is a common technique for generating deterministic colors from strings.

**Helper methods:**
| Method | What it does |
|---|---|
| `getSizeLabel(size)` | Capitalizes first letter: `"small"` → `"Small"` |
| `getModeLabel(mode)` | Maps `"single_player"` → `"Solo"`, etc. |
| `getGalaxyHue(name)` | Generates a unique color hue from the galaxy name |

---

## Game UI Components

### `StatBar.svelte`

**File:** `src/lib/components/game/StatBar.svelte` (102 lines)

**Purpose:** A reusable color-coded progress bar (used for hull, shield, fuel).

**Props:**
```typescript
interface Props {
  label: string;                                    // "Hull", "Shield", "Fuel"
  current: number;                                  // Current value
  max: number;                                      // Maximum value
  color: 'red' | 'blue' | 'orange' | 'green';     // Bar color
  suffix?: string;                                  // Optional text after numbers
}
```

**Key Svelte 5 features used:**
```typescript
const percentage = $derived(max > 0 ? Math.round((current / max) * 100) : 0);
const colors = $derived(colorClasses[color] || colorClasses.blue);
```

`$derived` recalculates automatically when `current`, `max`, or `color` change.

**How the bar animates:**
```css
.bar-fill {
  height: 100%;
  transition: width 0.3s ease-out;  /* Smooth width animation */
}
```

The width is set via inline style: `style="width: {percentage}%"`. When `percentage` changes, the CSS transition animates it smoothly.

> **Learning note:** The component uses both Tailwind utility classes (applied via `class="{colors.fill}"`) and scoped CSS. The `:global()` wrapper (line 76) is needed because Svelte scopes styles by default — `:global(.bg-red-900\/50)` makes the class available even though it's dynamically applied.

---

### `PlayerStats.svelte`

**File:** `src/lib/components/game/PlayerStats.svelte` (195 lines)

**Purpose:** Sidebar showing ship hull, shield, fuel bars plus navigation indicators.

**Props:**
```typescript
interface Props {
  hasShip: boolean;
  hull?: { current: number; max: number };
  shield?: { current: number; max: number; grade?: string };
  fuel?: { current: number; max: number };
  distance?: number;
  cooldown?: number;
  collision?: boolean;
  clamp?: boolean;
}
```

**Conditional rendering:**
```svelte
{#if hasShip && hull && shield && fuel}
  <!-- Show ship stats with StatBar components -->
{:else}
  <!-- Show "No Ship Assigned" message -->
{/if}
```

**Svelte class directives:**
```svelte
<span class="nav-indicator" class:active={collision} class:warning={collision}>
```

`class:active={collision}` adds the CSS class `active` when `collision` is truthy. This is Svelte's shorthand for conditional classes.

---

### `SystemMenu.svelte`

**File:** `src/lib/components/game/SystemMenu.svelte` (~120 lines)

**Purpose:** The service menu in the game — shows what you can interact with at the current location.

**Props:**
```typescript
interface Props {
  systemName: string;
  systemType: string;
  sector?: SectorData | null;
  availableServices?: string[];
  activeItem: MenuItemId | null;
  onSelect: (item: MenuItemId) => void;
}
```

**Interesting feature — Fuzzy Service Matching:**

The backend might call a service "trading_hub", "commerce_hub", "trade center", or many other variations. The menu normalizes these:

```typescript
// Simplified example of the pattern:
function matchService(serviceName: string): MenuItemId | null {
  const normalized = serviceName.toLowerCase().replace(/[_-]/g, ' ');
  if (normalized.includes('trading') || normalized.includes('commerce')) return 'trading_hub';
  if (normalized.includes('salvage') || normalized.includes('junkyard')) return 'salvage';
  // ... etc
}
```

> **Learning note:** This defensive approach handles API inconsistency gracefully. Rather than breaking when the backend changes a service name, the fuzzy matching adapts. In real projects, API contracts evolve — building resilience into the frontend saves debugging time.

---

### `ActionPanel.svelte`

**File:** `src/lib/components/game/ActionPanel.svelte` (~100+ lines)

**Purpose:** The main content area that changes based on which menu item is selected.

**Props:**
```typescript
interface Props {
  activeItem: MenuItemId | null;
  onAction: (action: string, targetUuid: string) => void;
  onTravelComplete?: () => void;
}
```

**How it works:** A large conditional block that renders different panels:

```svelte
{#if activeItem === 'planets'}
  <!-- Planet list -->
{:else if activeItem === 'trading_hub'}
  <TradingPanel hubUuid={...} />
{:else if activeItem === 'warp'}
  <!-- Warp gate list + travel buttons -->
{:else if activeItem === 'shipyard'}
  <!-- Ship catalog -->
{/if}
```

**Sub-components used:**
- `TradingPanel` — For buying/selling minerals
- `WarpLoader` — Overlay animation during warp travel

---

### `TradingPanel.svelte`

**File:** `src/lib/components/game/TradingPanel.svelte` (~80+ lines)

**Purpose:** Buy and sell minerals at trading hubs.

**Props:**
```typescript
interface Props {
  hubUuid: string;
  hubName?: string;
}
```

**How it works:**
1. On mount, loads trading hub inventory and player cargo
2. Displays two sections: "Available Goods" and "Your Cargo"
3. Buy/sell buttons call `playerState.buyMineral()` / `playerState.sellMineral()`
4. After each transaction, both inventory and cargo reload

**Sub-components:**
- `MineralRow` — Displays one mineral with price and buy button
- `CargoManifest` — Shows what's in your cargo hold
- `PriceDisplay` — Formatted credit amount

---

### `WarpLoader.svelte`

**File:** `src/lib/components/game/WarpLoader.svelte` (411 lines)

**Purpose:** An immersive full-screen warp animation shown during travel.

**Props:**
```typescript
interface Props {
  destinationName?: string;
  message?: string;
  status?: string;
}
```

**Animation elements (all CSS — no JavaScript animation):**
1. **100 streaking stars** — Random positions, each moving at different speeds
2. **5 warp tunnel rings** — Expanding concentric circles
3. **Warp gate SVG** — Spinning outer ring with rotating chevrons
4. **Event horizon** — Pulsing radial gradient at center
5. **Status text** — Flickering destination name and progress bar

> **Learning note:** All animations use CSS `@keyframes` — no `requestAnimationFrame` or JS timers. CSS animations are hardware-accelerated (run on the GPU), making them smooth even with 100+ animated elements. This is a performance best practice for visual effects.

---

### `SectorGrid.svelte`

**File:** `src/lib/components/game/SectorGrid.svelte` (~80+ lines)

**Purpose:** Shows the player's position on a 2D sector grid.

**Props:**
```typescript
interface Props {
  sector: SectorData | null;
  gridSize?: number;
  currentSystemUuid?: string;
  onSectorClick?: (x: number, y: number) => void;
}
```

**How the grid is built:**
```typescript
// Generate cells from gridSize
const cells = $derived(/* ... build a 2D array ... */);
```

The grid highlights the current sector and supports clicking to expand into a sector star map view.

---

### Other Game Components

| Component | Purpose |
|---|---|
| `SectorStarMap.svelte` | Detailed star map within a single sector |
| `StarMap.svelte` | SVG-based interactive full galaxy map |
| `SystemNode.svelte` | Individual star system node on the map |
| `MapFilters.svelte` | Zoom controls and toggle filters for the map |
| `MapLegend.svelte` | Legend explaining map symbols and colors |
| `MineralRow.svelte` | Single mineral in trading inventory |
| `CargoManifest.svelte` | Player's cargo items display |
| `PriceDisplay.svelte` | Formatted credit/price display |

---

## Svelte 5 Runes Reference

Here's every rune used in this codebase with real examples:

### `$state` — Reactive State

```typescript
// In a .svelte component:
let count = $state(0);           // Simple value
let items = $state<string[]>([]); // Typed array

// In a .svelte.ts store:
const state = $state<AuthState>({
  user: null,
  isAuthenticated: false
});
```

When you assign to `$state` variables, Svelte automatically re-renders affected parts of the template.

### `$derived` — Computed Values

```typescript
const percentage = $derived(max > 0 ? Math.round((current / max) * 100) : 0);
const colors = $derived(colorClasses[color] || colorClasses.blue);
```

`$derived` recalculates whenever its dependencies change. You can't assign to it — it's read-only.

### `$props()` — Component Props

```typescript
// Define props with destructuring:
let { label, current, max, color, suffix }: Props = $props();

// With defaults:
let { showNebula = true }: Props = $props();
```

### `$effect` — Side Effects

Used in some components to react to prop changes:
```typescript
$effect(() => {
  // This runs whenever hubUuid changes
  loadInventory(hubUuid);
});
```

---

## Common Patterns in This Codebase

### 1. The Store Factory Pattern

Both `auth.svelte.ts` and `playerState.svelte.ts` use this pattern:

```typescript
function createStore() {
  const state = $state({ ... });  // Private state

  function doThing() { ... }      // Methods that modify state

  return {
    get value() { return state.value; },  // Read-only getters
    doThing                                // Public methods
  };
}

export const store = createStore();        // Singleton instance
```

**Why:** Encapsulates state, prevents direct mutation, provides a clean API.

### 2. Guard Clauses

Many methods start by checking prerequisites:

```typescript
async function loadCargo() {
  if (!state.playerUuid) return null;  // Guard clause — exit early
  // ... rest of the method
}
```

**Why:** Prevents errors from missing data and keeps the happy path un-nested.

### 3. Optimistic Cache + Background Refresh

```typescript
// Show cached data immediately
const cached = getCachedGalaxyList();
if (cached) { myGames = cached.my_games; }

// Then refresh in the background
const fresh = await api.galaxies.list();
if (fresh.success) {
  myGames = fresh.data.my_games;
  cacheGalaxyList(fresh.data);
}
```

### 4. Error Handling with Typed Error Codes

```typescript
const response = await api.galaxies.join(uuid, data);
if (!response.success) {
  switch (response.error?.code) {
    case 'GALAXY_FULL': error = 'Galaxy is full'; break;
    case 'DUPLICATE_CALL_SIGN': error = 'Name taken'; break;
    default: error = response.error?.message || 'Unknown error';
  }
}
```

### 5. Parallel Data Loading

```typescript
await Promise.all([
  playerState.loadLocationDetails(),
  playerState.loadFacilities()
]);
```

### 6. SSR Safety Checks

```typescript
if (typeof localStorage === 'undefined') return null;
```

SvelteKit runs code on the server first (SSR). `localStorage` only exists in browsers, so every access is guarded.

---

## How Data Flows Through the App

### Authentication Flow
```
User fills login form
  → handleSubmit() calls auth.login(email, password)
    → api.auth.login() sends POST /auth/login
      → Server returns { user, access_token }
    → auth.setAuth() saves token to localStorage + caches user
  → goto('/galaxies') navigates to galaxy list
```

### Galaxy Join Flow
```
User clicks "Continue" on a galaxy card
  → Navigate to /galaxies/{uuid}/play
    → playerState.initialize(galaxyUuid)
      → api.galaxies.getMyPlayer(uuid)
        → If NO_PLAYER_IN_GALAXY → show join form
        → If success → setPlayerData() + loadMyShip()
          → Load location details + facilities in parallel
            → UI renders system menu + action panel
```

### Trading Flow
```
User selects "Trading Hub" from menu
  → TradingPanel loads:
    → loadTradingHubInventory(hubUuid) → shows available minerals
    → loadCargo() → shows current cargo
  → User clicks "Buy 5 units of Iron"
    → playerState.buyMineral(hubUuid, mineralUuid, 5)
      → api.tradingHubs.buy() sends POST
        → Server returns { remaining_credits, cargo_used }
      → Update credits
      → Reload cargo and inventory (quantities changed)
    → UI updates automatically (reactive state)
```

### Warp Travel Flow
```
User clicks warp gate destination
  → playerState.travel(destinationUuid, destinationName)
    → Set isTraveling = true (shows WarpLoader overlay)
    → api.players.travel() sends POST
    → If status = "generating":
      → Poll /current-system every 2 seconds
      → Update travel status messages
      → Wait for generation to complete
    → Update currentSystem, currentSector, fuel
    → Clear old locationDetails + facilities
    → Ensure minimum 2s animation display
    → Set isTraveling = false (hides WarpLoader)
  → Gameplay page reloads location + facilities
```

---

## Glossary

| Term | Meaning |
|---|---|
| **Rune** | A Svelte 5 reactive primitive (`$state`, `$derived`, `$effect`, `$props`) |
| **SSR** | Server-Side Rendering — SvelteKit renders pages on the server first |
| **POI** | Point of Interest — a star system or other scannable location |
| **UUID** | Universally Unique Identifier — a long random string like `a1b2c3d4-...` |
| **TTL** | Time To Live — how long a cache entry is considered valid |
| **Guard clause** | An early `return` that prevents running code when prerequisites aren't met |
| **Idempotent** | An operation that produces the same result whether run once or many times |
| **Closure** | A function that "remembers" variables from its surrounding scope |
| **Generic type** | A type with a placeholder (`<T>`) that gets filled in when used |
| **Snippet** | Svelte 5's way of passing renderable content to child components |
