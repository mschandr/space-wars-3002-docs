# Vendor Dialogue System

**Version**: 1.0
**Date**: 2026-03-10
**Status**: Active

## Overview

The Vendor Dialogue System provides dynamic, context-aware dialogue for NPC vendors. Each vendor can have multiple dialogue lines for different situations, with branching based on:

- **Line Type**: What the vendor is saying (greeting, pitch, acceptance, rejection, farewell)
- **Interaction Bucket**: The player's history with the vendor (first visit, repeat customer, etc.)
- **Inventory Context**: What the vendor is currently pitching (weapons, minerals, contraband, etc.)

This system moves away from hardcoded dialogue and enables rich, repeatable vendor interactions with personality and variety.

---

## Schema

```sql
CREATE TABLE vendor_dialogue (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vendor_profile_id BIGINT UNSIGNED NOT NULL,
    line_type ENUM('greeting', 'inventory_pitch', 'deal_accepted', 'deal_rejected', 'farewell'),
    interaction_bucket ENUM('first_visit', 'second_visit', 'third_visit', 'repeat_customer'),
    inventory_context VARCHAR(64) NULL,
    line_text VARCHAR(255) NOT NULL,
    weight DECIMAL(5,4) NOT NULL DEFAULT 1.0000,
    generation_version INT UNSIGNED NOT NULL DEFAULT 1,
    created_at TIMESTAMP NULL DEFAULT NULL,
    updated_at TIMESTAMP NULL DEFAULT NULL,
    INDEX idx_vendor_line_type (vendor_profile_id, line_type),
    INDEX idx_vendor_lookup (vendor_profile_id, line_type, interaction_bucket),
    INDEX idx_inventory_context (vendor_profile_id, inventory_context),
    FOREIGN KEY (vendor_profile_id) REFERENCES vendor_profiles(id) ON DELETE CASCADE
);
```

---

## Data Model

### VendorDialogue Model

```php
class VendorDialogue extends Model
{
    protected $table = 'vendor_dialogue';

    protected $fillable = [
        'vendor_profile_id',
        'line_type',
        'interaction_bucket',
        'inventory_context',
        'line_text',
        'weight',
        'generation_version',
    ];

    protected $casts = [
        'line_type' => DialogueLineType::class,
        'interaction_bucket' => InteractionBucket::class,
        'weight' => 'decimal:4',
    ];
}
```

**Methods**:
- `getDialogue(vendorId, lineType, bucket, context?)` - Get weighted random dialogue
- `getVendorDialogue(vendorId, lineType?, bucket?)` - Get all matching dialogue
- `forVendor($id)` - Scope: filter by vendor
- `byLineType(DialogueLineType)` - Scope: filter by line type
- `byBucket(InteractionBucket)` - Scope: filter by interaction bucket

### Enums

#### DialogueLineType
```php
enum DialogueLineType: string
{
    case GREETING = 'greeting';              // Initial greeting
    case INVENTORY_PITCH = 'inventory_pitch'; // Promoting items
    case DEAL_ACCEPTED = 'deal_accepted';    // Trade acceptance
    case DEAL_REJECTED = 'deal_rejected';    // Trade rejection
    case FAREWELL = 'farewell';              // Goodbye
}
```

#### InteractionBucket
```php
enum InteractionBucket: string
{
    case FIRST_VISIT = 'first_visit';          // First meeting (0 prior visits)
    case SECOND_VISIT = 'second_visit';        // Second meeting (1 prior visit)
    case THIRD_VISIT = 'third_visit';          // Third meeting (2 prior visits)
    case REPEAT_CUSTOMER = 'repeat_customer';  // Regular customer (3+ visits)
}
```

---

## Usage Examples

### Get a greeting line for a new customer

```php
$dialogue = VendorDialogue::getDialogue(
    vendorProfileId: $vendor->id,
    lineType: DialogueLineType::GREETING,
    bucket: InteractionBucket::FIRST_VISIT
);

echo $dialogue->line_text; // "Welcome, stranger! First time in port?"
```

### Get an inventory pitch for weapons

```php
$pitch = VendorDialogue::getDialogue(
    vendorProfileId: $vendor->id,
    lineType: DialogueLineType::INVENTORY_PITCH,
    bucket: InteractionBucket::REPEAT_CUSTOMER,
    inventoryContext: 'weapons'
);

echo $pitch->line_text; // "You know what you need? Better armaments..."
```

### Get all dialogue for a vendor

```php
$allDialogue = VendorDialogue::getVendorDialogue($vendor->id);
// Returns all dialogue lines for this vendor, grouped by type/bucket
```

### Create dialogue with factory

```php
VendorDialogue::factory()
    ->greeting()
    ->firstVisit()
    ->create([
        'vendor_profile_id' => $vendor->id,
        'line_text' => 'Welcome, friend! I haven\'t seen you around before.',
    ]);
```

---

## Weighted Random Selection

Dialogue lines have a **weight** field (0.0001 - 9.9999) for probability distribution.

**How it works**:
1. All matching lines are fetched (e.g., all GREETING lines for FIRST_VISIT)
2. Total weight is summed
3. Random number between 0 and total weight is generated
4. Line is selected based on cumulative weight

**Example**:
```
Line A: weight 1.0 (50% chance)
Line B: weight 1.0 (50% chance)
Line C: weight 2.0 (not selected if total is only A+B)

Total: 2.0
Random: 1.3
→ Select Line B (1.0 < 1.3 <= 2.0)
```

---

## Interaction Buckets

The interaction bucket determines which dialogue a vendor uses based on player history:

| Bucket | Visits | Behavior |
|--------|--------|----------|
| `first_visit` | 0 | Player meeting vendor for first time. Cautious, formal greeting. |
| `second_visit` | 1 | Second meeting. Vendor recognizes player. Slightly more friendly. |
| `third_visit` | 2 | Third meeting. Vendor establishing rapport. More casual. |
| `repeat_customer` | 3+ | Regular customer. Vendor treats as familiar. Friendly, informal. |

**Implementation**:
```php
$interactionCount = $player->vendorInteractionCount($vendor->id);
$bucket = match (true) {
    $interactionCount == 0 => InteractionBucket::FIRST_VISIT,
    $interactionCount == 1 => InteractionBucket::SECOND_VISIT,
    $interactionCount == 2 => InteractionBucket::THIRD_VISIT,
    default => InteractionBucket::REPEAT_CUSTOMER,
};

$dialogue = VendorDialogue::getDialogue(
    $vendor->id,
    DialogueLineType::GREETING,
    $bucket
);
```

---

## Inventory Context

Vendors can pitch specific items using the `inventory_context` field:

**Valid contexts**:
- `weapons` - Combat equipment, weapons
- `minerals` - Raw materials, commodities
- `rare_goods` - High-value specialty items
- `contraband` - Illegal/restricted items
- (extensible via migrations)

**Example pitch**:
```
Context: minerals
Line: "Listen, I've got premium titanium just came in. The refinery's been dry."

Context: weapons
Line: "You look like you could use better firepower. I've got something special."
```

**NULL context**: Vendor can use generic pitches without specific inventory mention

---

## Generation Version

The `generation_version` field tracks when dialogue was created:

- `1` = Initial seeding (current)
- `2+` = Future updates/regeneration

**Use cases**:
- Update vendor dialogue without losing player history
- A/B test different dialogue pools
- Revert to older dialogue if new generation underperforms

```php
$currentDialogue = $vendor->dialogueLines()
    ->byVersion(1)  // Only use generation 1
    ->get();
```

---

## VendorProfile Integration

VendorProfile model includes:
```php
public function dialogueLines(): HasMany
{
    return $this->hasMany(VendorDialogue::class, 'vendor_profile_id');
}
```

**Usage**:
```php
$vendor->dialogueLines()->count();  // Total dialogue lines
$vendor->dialogueLines()
    ->byLineType(DialogueLineType::GREETING)
    ->get();  // All greeting lines
```

---

## Seeding

### Using Factory

```php
// Create 50 dialogue lines for a vendor
VendorDialogue::factory(50)
    ->for($vendor, 'vendorProfile')
    ->create();

// Create specific dialogue
VendorDialogue::factory()
    ->greeting()
    ->firstVisit()
    ->for($vendor)
    ->create([
        'line_text' => 'Welcome to my shop, friend!',
    ]);
```

### Bulk Seeding

```php
// Seed all vendors with dialogue
foreach (VendorProfile::all() as $vendor) {
    $vendor->dialogueLines()->createMany([
        [
            'line_type' => 'greeting',
            'interaction_bucket' => 'first_visit',
            'line_text' => 'Welcome to my shop!',
            'weight' => 1.0,
        ],
        // ... more lines
    ]);
}
```

---

## Performance Considerations

**Indexes**:
- `idx_vendor_line_type` - Fast lookup by vendor + line type
- `idx_vendor_lookup` - Optimal for the common query pattern (vendor + line + bucket)
- `idx_inventory_context` - For context-specific pitches

**Query examples**:
```php
// Fast (uses idx_vendor_lookup)
SELECT * FROM vendor_dialogue
WHERE vendor_profile_id = 1
  AND line_type = 'greeting'
  AND interaction_bucket = 'first_visit';

// Fast (uses idx_vendor_line_type)
SELECT * FROM vendor_dialogue
WHERE vendor_profile_id = 1
  AND line_type = 'inventory_pitch';

// Fast (uses idx_inventory_context)
SELECT * FROM vendor_dialogue
WHERE vendor_profile_id = 1
  AND inventory_context = 'weapons';
```

---

## Future Enhancements

### Dialogue Trees
- Add `parent_dialogue_id` for branching dialogue
- Player choices lead to different vendor responses

### Sentiment Tags
- Add `sentiment` enum (positive, neutral, negative)
- Vendor attitude reflects reputation with player

### Dynamic Substitution
- `{player_name}` → player name
- `{vendor_name}` → vendor name
- `{item_name}` → item being pitched

### Dialogue Events
- Log dialogue selections for analytics
- Track which lines are most effective
- Automatically weight popular dialogue higher

### Reputation-Based Dialogue
- Hostile dialogue for players with low reputation
- Friendly dialogue for high-reputation customers
- Neutral stance for new players

---

## Migration Notes

**Migration file**: `2026_03_10_153308_create_vendor_dialogue_table.php`

**Run migration**:
```bash
php artisan migrate
```

**Rollback**:
```bash
php artisan migrate:rollback
```

The migration includes:
- Complete vendor_dialogue table creation
- All indexes for optimal performance
- Foreign key constraint to vendor_profiles with CASCADE delete

---

## Related Models

- **VendorProfile** - The vendor using this dialogue
- **PlayerVendorRelationship** - Tracks player interaction count (to determine bucket)
- **VendorDialogue** - This table
