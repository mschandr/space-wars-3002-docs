# Vendor Profile System

## Overview

The Vendor Profile System implements a **Profile Pool Pattern** where vendor personas are global templates sampled per-galaxy to create unique instances. This enables diverse, consistent vendor personalities across the game while maintaining per-galaxy uniqueness.

**Last Updated**: March 11, 2026
**Status**: Production-Ready

---

## Architecture

### Profile Pool Pattern

```
┌─────────────────────────────────────┐
│   Global Vendor Profiles (Pool)     │
│   - Templates (29 base personalities)│
│   - Preserved across galaxy:flush    │
│   - Reusable across galaxies         │
└────────────────┬────────────────────┘
                 │
                 │ Sampled & Instantiated
                 ↓
┌─────────────────────────────────────┐
│  Galaxy-Specific Vendor Instances    │
│  - One per POI (trading hub)         │
│  - Dialogue generation state         │
│  - Flushed with galaxy              │
└─────────────────────────────────────┘
```

---

## Database Tables

### 1. `vendor_profiles` (Global Templates)

**Purpose**: Store reusable vendor persona templates

**Columns**:
- `id` - Primary key
- `uuid` - Unique identifier (route key)
- `name` - Vendor name (e.g., "Kovac's Trading Post")
- `archetype` - Type of vendor (see Archetypes below)
- `service_type` - enum('trading_hub', 'salvage_yard', 'shipyard', 'market')
- `criminality` - decimal(3,2) | Range 0.0-1.0 | Indicates black market involvement
- `personality` - JSON object with 7 traits (see Personality Traits below)
- `dialogue_pool` - JSON object with preset dialogue (see Dialogue below)
- `markup_base` - decimal(5,4) | Base markup for pricing

**Example Record**:
```json
{
  "id": 1,
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Kovac's Trading Post",
  "archetype": "hard_bargainer",
  "service_type": "trading_hub",
  "criminality": 0.35,
  "personality": {
    "honesty": 0.5,
    "greed": 0.8,
    "risk_tolerance": 0.6,
    "charm": 0.5,
    "ego_drive": 0.8,
    "empathy": 0.3,
    "curiosity": 0.5
  },
  "dialogue_pool": {
    "greeting": ["Looking for a deal, eh?", "Welcome back..."],
    "deal_accepted": ["Good business today.", "Not bad, not bad."]
  },
  "markup_base": 0.10
}
```

---

### 2. `galaxy_vendor_profiles` (Per-Galaxy Instances)

**Purpose**: Store galaxy-specific vendor instance data and dialogue generation state

**Columns**:
- `id` - Primary key
- `uuid` - Unique identifier (route key)
- `galaxy_id` - FK → galaxies
- `poi_id` - FK → points_of_interest (trading hub location)
- `vendor_profile_id` - FK → vendor_profiles (references template)
- `trading_post_id` - FK → trading_posts (nullable, for fallback dialogue)
- `service_type` - enum | Copied from template, can be modified per galaxy
- `criminality` - decimal(3,2) | Copied from template ± 5%, can vary per galaxy
- `dialogue_generation_status` - enum('pending', 'generating', 'complete', 'failed')
- `dialogue_generation_version` - unsigned int | Tracks dialogue iterations
- `dialogue_generated_at` - timestamp | When AI generated dialogue
- `created_at`, `updated_at` - Standard timestamps

**Constraints**:
- Unique constraint on `[galaxy_id, poi_id]` - One vendor per trading hub per galaxy

**Example Record**:
```json
{
  "id": 1,
  "uuid": "660f9511-f40c-52e5-b827-557766551111",
  "galaxy_id": 10,
  "poi_id": 256,
  "vendor_profile_id": 1,
  "trading_post_id": 12,
  "service_type": "trading_hub",
  "criminality": 0.32,
  "dialogue_generation_status": "complete",
  "dialogue_generation_version": 1,
  "dialogue_generated_at": "2026-03-11 14:30:00"
}
```

---

### 3. `vendor_dialogue` (Generated Dialogue Lines)

**Purpose**: Store AI-generated dialogue lines for specific vendor instances

**Columns**:
- `id` - Primary key
- `galaxy_vendor_profile_id` - FK → galaxy_vendor_profiles (which instance)
- `line_type` - enum('greeting', 'inventory_pitch', 'deal_accepted', 'deal_rejected', 'farewell')
- `interaction_bucket` - enum('first_visit', 'second_visit', 'third_visit', 'repeat_customer')
- `inventory_context` - string(64) nullable | Category context (e.g., "weapons", "minerals")
- `line_text` - string(255) | The actual dialogue line
- `weight` - decimal(5,4) | Probability weight for selection (default 1.0)
- `generation_version` - unsigned int | Which version of AI generated this
- `created_at`, `updated_at` - Standard timestamps

**Indexes**:
- `idx_vendor_line_type`: `[galaxy_vendor_profile_id, line_type]`
- `idx_vendor_lookup`: `[galaxy_vendor_profile_id, line_type, interaction_bucket]`
- `idx_inventory_context`: `[galaxy_vendor_profile_id, inventory_context]`

**Example Records**:
```json
[
  {
    "id": 1,
    "galaxy_vendor_profile_id": 1,
    "line_type": "greeting",
    "interaction_bucket": "first_visit",
    "inventory_context": null,
    "line_text": "Welcome to my trading post, friend. I'm Kovac.",
    "weight": 1.0,
    "generation_version": 1
  },
  {
    "id": 2,
    "galaxy_vendor_profile_id": 1,
    "line_type": "inventory_pitch",
    "interaction_bucket": "repeat_customer",
    "inventory_context": "weapons",
    "line_text": "Got some quality weaponry in stock today.",
    "weight": 1.0,
    "generation_version": 1
  }
]
```

---

### 4. `player_vendor_relationships` (Player Reputation)

**Purpose**: Track player relationship with vendor templates (cross-galaxy)

**Columns**:
- `player_id` - FK → players
- `vendor_profile_id` - FK → vendor_profiles (template-level)
- `goodwill` - integer | Cumulative positive interactions
- `shady_dealings` - integer | Cumulative illegal activities
- `visit_count` - integer | Total visits across all galaxies
- `markup_modifier` - decimal(5,4) | Personal price adjustment
- `is_locked_out` - boolean | Whether vendor refuses service
- `last_interaction_at` - timestamp | Last trade timestamp

**Unique Constraint**: `[player_id, vendor_profile_id]`

---

## Personality Traits

All vendor profiles have **7 personality traits** (scale 0.0-1.0):

| Trait | Meaning | Low (0.0-0.3) | Mid (0.3-0.7) | High (0.7-1.0) |
|-------|---------|---------------|---------------|-----------------|
| **honesty** | Trust & fairness | Deceptive | Balanced | Trustworthy |
| **greed** | Desire for profit | Altruistic | Balanced | Profit-driven |
| **risk_tolerance** | Willingness to take chances | Cautious | Moderate | Reckless |
| **charm** | Persuasiveness | Cold | Personable | Highly charismatic |
| **ego_drive** | Ambition & confidence | Humble | Self-assured | Arrogant |
| **empathy** | Care for others | Callous | Empathetic | Compassionate |
| **curiosity** | Desire to learn | Incurious | Interested | Intensely curious |

---

## Vendor Archetypes

Each archetype has predefined trait values and behavior characteristics:

### 1. Honest Broker
- **Traits**: High honesty (0.9), low greed (0.2), high empathy (0.8)
- **Markup**: 0.0% (fair prices)
- **Criminality**: 0.1 (very low)
- **Dialogue**: Welcoming, fair-minded, builds trust
- **Use Case**: Legitimate trading hub operator

### 2. Hard Bargainer
- **Traits**: Medium honesty (0.5), high greed (0.8), high ego_drive (0.8)
- **Markup**: 10% (aggressive)
- **Criminality**: 0.3 (moderate)
- **Dialogue**: Tough negotiator, profits-focused
- **Use Case**: Shrewd merchant willing to haggle

### 3. Fence
- **Traits**: Low honesty (0.2), high greed (0.7), low empathy (0.2)
- **Markup**: 5% (discrete)
- **Criminality**: 0.5 (high)
- **Dialogue**: Discrete, secretive, "no questions asked"
- **Use Case**: Black market goods dealer

### 4. Corporate Agent
- **Traits**: Medium honesty (0.6), medium greed (0.6), high curiosity (0.7)
- **Markup**: 5% (structured)
- **Criminality**: 0.15 (low)
- **Dialogue**: Professional, detail-oriented
- **Use Case**: Corporate trading post representative

### 5. Explorer Outfitter
- **Traits**: High honesty (0.7), low greed (0.4), very high curiosity (0.9)
- **Markup**: -5% (discount!)
- **Criminality**: 0.2 (low)
- **Dialogue**: Enthusiastic about exploration, helpful
- **Use Case**: Supplies for expeditions and discovery

### 6. Pirate Contact
- **Traits**: Low honesty (0.3), high greed (0.8), very high ego_drive (0.9)
- **Markup**: 20% (steep)
- **Criminality**: 0.6 (high)
- **Dialogue**: Dangerous, ambitious, intimidating
- **Use Case**: Contacts in pirate/military circles

### 7. Black Market Dealer
- **Traits**: Very low honesty (0.1), very high greed (0.9), low empathy (0.1)
- **Markup**: 15% (premium)
- **Criminality**: 0.9 (very high)
- **Dialogue**: Cryptic, dangerous, no ethics
- **Use Case**: Illegal goods (requires player reputation)

### 8. Gruff Mechanic
- **Traits**: High honesty (0.8), low greed (0.4), high curiosity (0.8)
- **Markup**: 0.0% (fair)
- **Criminality**: 0.1 (very low)
- **Dialogue**: Technical, no-nonsense, honest
- **Use Case**: Ship repairs and upgrades

### 9. Socialite
- **Traits**: Low honesty (0.4), medium greed (0.5), very high charm (0.9)
- **Markup**: 8% (variable)
- **Criminality**: 0.25 (low)
- **Dialogue**: Charming, gossipy, relationship-focused
- **Use Case**: Social hub operator, information broker

---

## Seeding Vendor Profiles

### Process Overview

1. **Clear existing data** - Remove old profiles
2. **Create 29 global templates** - 3-4 per archetype
3. **Seed each profile** with:
   - Unique name (from predefined pool)
   - Archetype (determines trait values)
   - Random service type
   - Criminality (base ± random variation)
   - Personality traits (archetype-specific values)
   - Dialogue pool (preset dialogue by archetype)
   - Markup base (archetype-specific)

### Running the Seeder

```bash
# Full reseed (clears + recreates)
php artisan db:seed --class=VendorProfileSeeder

# Or as part of complete test data seeding
php artisan seed:test-data
```

### Seeder Code Reference

**File**: `database/seeders/VendorProfileSeeder.php`

Key methods:
- `run()` - Main seeding logic
- `vendorNameForArchetype()` - Selects unique names
- `baseCriminalityForArchetype()` - Archetype base criminality
- `personalityForArchetype()` - Returns trait value for archetype + trait
- `generateDialoguePool()` - Creates archetype-specific dialogue presets

---

## AI Dialogue Generation (Go AI Instructions)

### Overview

The dialogue generation system uses an AI to create natural, contextual dialogue lines for vendor profiles. The AI reads personality traits and generates authentic character-specific dialogue.

### Input Data for AI

When calling the dialogue generation AI, provide:

```json
{
  "vendor_profile": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Kovac's Trading Post",
    "archetype": "hard_bargainer",
    "service_type": "trading_hub",
    "criminality": 0.35,
    "personality": {
      "honesty": 0.5,
      "greed": 0.8,
      "risk_tolerance": 0.6,
      "charm": 0.5,
      "ego_drive": 0.8,
      "empathy": 0.3,
      "curiosity": 0.5
    }
  },
  "galaxy_vendor_profile": {
    "uuid": "660f9511-f40c-52e5-b827-557766551111",
    "galaxy_id": 10,
    "poi_id": 256,
    "dialogue_generation_version": 1
  },
  "generation_request": {
    "line_types": ["greeting", "inventory_pitch", "deal_accepted", "deal_rejected", "farewell"],
    "interaction_buckets": ["first_visit", "second_visit", "third_visit", "repeat_customer"],
    "inventory_contexts": [null, "weapons", "minerals", "rare_goods", "contraband"],
    "lines_per_combination": 2
  }
}
```

### Personality-Based Generation Guidelines

#### For HIGH HONESTY (0.7-1.0)
- Use truthful, straightforward language
- Avoid deception or manipulation
- Express trustworthiness
- Build confidence in the customer
- **Example**: "I guarantee fair prices on everything we sell."

#### For HIGH GREED (0.7-1.0)
- Focus on profit and deals
- Express desire for maximum return
- Push for upsells
- Show excitement about expensive transactions
- **Example**: "These weapons? Prime stock. Worth every credit."

#### For HIGH RISK_TOLERANCE (0.7-1.0)
- Willingness to deal with danger
- Less concern about consequences
- Adventurous attitude
- **Example**: "Black market contacts? Sure, I can arrange that."

#### For HIGH CHARM (0.7-1.0)
- Personable, engaging language
- Build rapport quickly
- Use humor and warmth
- Make customers feel special
- **Example**: "I remember you! Come back for the good stuff?"

#### For HIGH EGO_DRIVE (0.7-1.0)
- Emphasize personal accomplishment
- Assert confidence in abilities
- Self-promotion
- Demand respect
- **Example**: "I'm the best supplier in this sector, and everyone knows it."

#### For HIGH EMPATHY (0.7-1.0)
- Express care for customer wellbeing
- Offer help and support
- Show understanding of needs
- Build emotional connection
- **Example**: "I understand you're just starting out. Let me help you get a fair deal."

#### For HIGH CURIOSITY (0.7-1.0)
- Ask questions about the customer
- Express interest in gossip/news
- Eager to learn and explore
- Reference discoveries and findings
- **Example**: "So what brings you to this sector? Got any interesting stories?"

### Dialogue Line Types

Generate dialogue for these interaction contexts:

1. **greeting** - First encounter or general greeting
   - Used for initial contact
   - Should be archetype-appropriate
   - Set the tone for the vendor

2. **inventory_pitch** - Selling current inventory
   - Context-specific (weapons, minerals, etc.)
   - Highlight vendor strengths
   - Create desire for products

3. **deal_accepted** - Player accepts the trade
   - Express satisfaction/relief/pride
   - Match personality (gloating vs genuine joy)
   - Thank customer appropriately

4. **deal_rejected** - Player refuses the trade
   - Show disappointment/anger/acceptance
   - May try to persuade (high charm)
   - Maintain character

5. **farewell** - Goodbye/safe travels
   - Send-off dialogue
   - Encourage return visits
   - Leave impression

### Interaction Buckets (Customer Familiarity)

Generate different dialogue based on how many times player has visited:

1. **first_visit** - New customer
   - Introduce yourself
   - Build initial trust or danger
   - Set expectations

2. **second_visit** - Returning customer
   - Show recognition
   - Building relationship
   - Offer incentives for loyalty

3. **third_visit** - Regular customer
   - Treat as familiar
   - Special offers/warnings
   - Establish patterns

4. **repeat_customer** - Long-term customer (4+ visits)
   - Deep familiarity
   - Personalized dialogue
   - Show true personality
   - May reference previous interactions

### Inventory Contexts

Generate specialized dialogue when trading specific commodity types:

- `null` - Generic dialogue (always available as fallback)
- `"weapons"` - Combat gear, weapons systems
- `"minerals"` - Raw resources, ore, refined materials
- `"rare_goods"` - Unique items, collectibles
- `"contraband"` - Illegal goods (for high criminality vendors)

### Generation Guidelines

**DO:**
- ✅ Match personality traits consistently
- ✅ Vary dialogue within each context
- ✅ Use natural, conversational language
- ✅ Reference the vendor's archetype
- ✅ Keep lines under 255 characters
- ✅ Use first-person for vendor voice
- ✅ Consider interaction bucket (familiar vs stranger)
- ✅ Balance formality with personality

**DON'T:**
- ❌ Break character based on personality
- ❌ Use out-of-universe references
- ❌ Include metadata in dialogue
- ❌ Make lines longer than 255 chars
- ❌ Use technical game language
- ❌ Repeat exact lines across buckets
- ❌ Ignore inventory context
- ❌ Be generic (low personality)

---

## Storing Generated Dialogue

### Batch Insert Process

```php
// After AI generates dialogue lines, insert into database:
$dialogueLines = []; // From AI generation

foreach ($generatedLines as $line) {
    $dialogueLines[] = [
        'galaxy_vendor_profile_id' => $galaxyVendorProfile->id,
        'line_type' => $line['line_type'],
        'interaction_bucket' => $line['interaction_bucket'],
        'inventory_context' => $line['inventory_context'],
        'line_text' => $line['line_text'],
        'weight' => $line['weight'] ?? 1.0,
        'generation_version' => $galaxyVendorProfile->dialogue_generation_version,
        'created_at' => now(),
        'updated_at' => now(),
    ];
}

// Batch insert
VendorDialogue::insert($dialogueLines);

// Update generation status
$galaxyVendorProfile->update([
    'dialogue_generation_status' => 'complete',
    'dialogue_generated_at' => now(),
]);
```

### Updating Generation Status

**Pending Generation**:
```php
$galaxyVendorProfile->update([
    'dialogue_generation_status' => 'pending',
    'dialogue_generation_version' => 1,
    'dialogue_generated_at' => null,
]);
```

**In Progress**:
```php
$galaxyVendorProfile->update([
    'dialogue_generation_status' => 'generating',
]);
```

**Complete**:
```php
$galaxyVendorProfile->update([
    'dialogue_generation_status' => 'complete',
    'dialogue_generated_at' => now(),
]);
```

**Failed**:
```php
$galaxyVendorProfile->update([
    'dialogue_generation_status' => 'failed',
]);
```

### Regenerating Dialogue

To regenerate dialogue (new version):

```php
// Mark as stale
$galaxyVendorProfile->markDialogueStale();
// This sets: status='pending', version++, generated_at=null

// Clear old dialogue
VendorDialogue::where('galaxy_vendor_profile_id', $galaxyVendorProfile->id)
    ->where('generation_version', '<', $galaxyVendorProfile->dialogue_generation_version)
    ->delete();

// Queue for regeneration
// (AI picks up pending vendors)
```

---

## Accessing Vendor Data in Code

### Get Vendor Profile (Template)

```php
// By UUID
$vendor = VendorProfile::where('uuid', $uuid)->first();

// By ID
$vendor = VendorProfile::find($id);

// All templates
$vendors = VendorProfile::all();
```

### Get Galaxy Vendor Instance

```php
// By UUID
$instance = GalaxyVendorProfile::where('uuid', $uuid)->first();

// For a specific POI in a galaxy
$instance = GalaxyVendorProfile::where('galaxy_id', $galaxyId)
    ->where('poi_id', $poiId)
    ->first();

// Instances needing dialogue generation
$pending = GalaxyVendorProfile::needingDialogueGeneration()
    ->where('galaxy_id', $galaxyId)
    ->get();

// With relationships
$instance = GalaxyVendorProfile::with([
    'galaxy',
    'pointOfInterest',
    'vendorProfile',
    'dialogueLines',
])->find($id);
```

### Access Personality Traits

```php
$vendor = VendorProfile::first();

// Direct access
$ego = $vendor->personality['ego_drive'];  // 0.8

// Using helper method
$curiosity = $vendor->getPersonality('curiosity');  // 0.5

// All traits
$traits = $vendor->personality;
// Returns: ['honesty' => 0.9, 'greed' => 0.2, ...]
```

### Get Vendor Dialogue

```php
$instance = GalaxyVendorProfile::first();

// All dialogue for this vendor
$lines = $instance->dialogueLines()->get();

// Specific line type
$greetings = $instance->dialogueLines()
    ->where('line_type', 'greeting')
    ->get();

// By interaction bucket
$firstVisit = $instance->dialogueLines()
    ->where('interaction_bucket', 'first_visit')
    ->get();

// By inventory context
$weaponsLines = $instance->dialogueLines()
    ->where('inventory_context', 'weapons')
    ->get();
```

---

## Workflow Examples

### Complete Dialogue Generation Workflow

```
1. Galaxy Created
   ↓
2. Trading Hubs Created (Points of Interest)
   ↓
3. GalaxyVendorProfileSeeder
   - For each trading hub in galaxy
   - Pick random vendor profile from template pool
   - Create galaxy_vendor_profiles record
   - Set dialogue_generation_status = 'pending'
   ↓
4. Admin Triggers: GET /api/admin/vendors/dialogue/pending
   - Returns all vendors with dialogue_generation_status='pending'
   - Admin selects vendors or triggers automatic batch
   ↓
5. AI Dialogue Generation Service
   - Read vendor personality traits
   - Generate 5 line types × 4 buckets × 5 contexts = 100 lines
   - Return structured JSON
   ↓
6. Store in vendor_dialogue Table
   - Batch insert all lines
   - Set dialogue_generation_status = 'complete'
   - Record dialogue_generated_at timestamp
   ↓
7. Game Ready
   - Players interact with vendor
   - System selects dialogue based on:
     * Line type (what just happened)
     * Interaction bucket (how familiar)
     * Inventory context (what they're trading)
   - VendorDialogueService::getDialogueWithFallback() picks line
```

### Accessing Vendor Dialogue in Game

```php
// In a trade interaction:
$vendor = GalaxyVendorProfile::find($vendorId);
$player = Player::find($playerId);

// Get greeting for first-time visitor
$greeting = VendorDialogueService::getDialogueWithFallback(
    $vendor,
    'greeting',
    $player->id,
    $interactionCount = 1
);

// Later: deal_accepted for repeat customer selling weapons
$acceptance = VendorDialogueService::getDialogueWithFallback(
    $vendor,
    'deal_accepted',
    $player->id,
    $interactionCount = 5,
    $inventoryContext = 'weapons'
);

// Output to player
echo $vendor->name . ": " . $greeting;
```

---

## API Endpoints

### Get Pending Dialogue Generation

```
GET /api/admin/vendors/dialogue/pending
Authorization: Bearer {token}
```

**Response**:
```json
{
  "count": 15,
  "vendors": [
    {
      "id": 1,
      "uuid": "660f9511-f40c-52e5-b827-557766551111",
      "galaxy_name": "Andromeda",
      "poi_name": "Starlight Station",
      "vendor_name": "Kovac's Trading Post",
      "vendor_archetype": "hard_bargainer",
      "service_type": "trading_hub",
      "status": "pending",
      "version": 1
    }
  ]
}
```

### Regenerate Dialogue

```
POST /api/admin/vendors/{uuid}/dialogue/regenerate
Authorization: Bearer {token}
```

**Response**:
```json
{
  "message": "Dialogue regeneration scheduled",
  "vendor_uuid": "660f9511-f40c-52e5-b827-557766551111",
  "status": "pending",
  "version": 2
}
```

### Inspect Generated Dialogue

```
GET /api/admin/vendors/{uuid}/dialogue
Authorization: Bearer {token}
```

**Response**:
```json
{
  "vendor": {
    "uuid": "660f9511-f40c-52e5-b827-557766551111",
    "galaxy_name": "Andromeda",
    "poi_name": "Starlight Station",
    "profile_name": "Kovac's Trading Post",
    "service_type": "trading_hub",
    "criminality": 0.35
  },
  "generation": {
    "status": "complete",
    "version": 1,
    "generated_at": "2026-03-11T14:30:00Z",
    "line_count": 47
  },
  "dialogue_by_type_bucket": {
    "greeting": {
      "first_visit": [
        {
          "id": 1,
          "text": "Welcome, friend. I'm Kovac.",
          "weight": 1.0,
          "inventory_context": null,
          "generation_version": 1
        }
      ]
    }
  }
}
```

---

## Best Practices

### For Game Designers

1. **Personality Balance** - Ensure traits span 0.0-1.0 range for variety
2. **Archetype Consistency** - Keep vendors true to their type
3. **Dialogue Variety** - Generate multiple lines per context to avoid repetition
4. **Context Awareness** - Use inventory contexts for specificity
5. **Interaction Progression** - Make dialogue feel like relationship building

### For Developers

1. **Always Use Transactions** - When updating both vendor data and dialogue
2. **Lazy Load Relationships** - Use `->with()` to avoid N+1 queries
3. **Cache Dialogue Lookups** - Consider caching `dialogueLines()` queries
4. **Version Control** - Always track `dialogue_generation_version`
5. **Batch Operations** - Use `insert()` for many dialogue lines
6. **Error Handling** - Set status to 'failed' if generation breaks

### For AI Systems

1. **Read All Traits** - Use complete 7-trait personality profile
2. **Match Archetype** - Verify output matches expected archetype behavior
3. **Respect Boundaries** - Don't exceed 255 characters per line
4. **Natural Language** - Generate conversational, not robotic dialogue
5. **Context Sensitivity** - Adjust tone based on inventory_context
6. **Bucket Progression** - Make repeat_customer dialogue more personal
7. **Personality Consistency** - Same vendor should "sound like" themselves

---

## Troubleshooting

### Dialogue Not Appearing

1. Check `dialogue_generation_status` = 'complete'
2. Verify `galaxy_vendor_profile_id` FK in `vendor_dialogue`
3. Ensure line_type and interaction_bucket match expectations
4. Check `generateDialoguePool()` fallback in VendorDialogueService

### Personality Traits Missing

1. Verify vendor_profiles has 7-trait personality JSON
2. Check VendorProfileSeeder includes all traits
3. Ensure personalityForArchetype() returns all 7

### Dialogue Generation Stuck

1. Check `dialogue_generation_status` = 'generating'
2. Look for errors in logs
3. Manually set to 'pending' to retry
4. Increase `dialogue_generation_version` if retrying

---

## Summary Table

| Component | Location | Purpose |
|-----------|----------|---------|
| **Templates** | `vendor_profiles` | Global vendor personas |
| **Instances** | `galaxy_vendor_profiles` | Per-galaxy vendor state |
| **Dialogue** | `vendor_dialogue` | Generated dialogue lines |
| **Relationships** | `player_vendor_relationships` | Player reputation |
| **Seeder** | `database/seeders/VendorProfileSeeder.php` | Create 29 templates |
| **Service** | `app/Services/VendorDialogueService.php` | Select appropriate dialogue |
| **Controller** | `app/Http/Controllers/Admin/AdminVendorDialogueController.php` | Dialogue management API |

---

**Version**: 1.0
**Last Updated**: March 11, 2026
**Maintained By**: Development Team
