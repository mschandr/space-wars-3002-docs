# Vendor Dialogue Generation - AI Implementation Guide

**For**: Go AI/Text Generation Service
**Last Updated**: March 11, 2026

---

## Quick Start

The dialogue generation service receives vendor data with personality traits and generates 70-100+ unique dialogue lines across different contexts. Lines are stored in `vendor_dialogue` table and served to players during gameplay.

---

## Input JSON Format

```json
{
  "request_id": "unique-request-identifier",
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
  }
}
```

---

## Output JSON Format

```json
{
  "request_id": "unique-request-identifier",
  "status": "success",
  "vendor_uuid": "660f9511-f40c-52e5-b827-557766551111",
  "dialogue_lines": [
    {
      "line_type": "greeting",
      "interaction_bucket": "first_visit",
      "inventory_context": null,
      "line_text": "Welcome to my trading post, friend. I'm Kovac.",
      "weight": 1.0
    },
    {
      "line_type": "greeting",
      "interaction_bucket": "repeat_customer",
      "inventory_context": null,
      "line_text": "Kovac! Good to see a familiar face. What can I get you?",
      "weight": 1.0
    },
    {
      "line_type": "inventory_pitch",
      "interaction_bucket": "first_visit",
      "inventory_context": "weapons",
      "line_text": "Got some quality weaponry. Prime stock.",
      "weight": 1.0
    }
    // ... more lines
  ],
  "line_count": 87,
  "generation_time_seconds": 4.2
}
```

---

## Personality Trait Guide

### Understanding the Traits

Each trait is a **float 0.0-1.0** describing a personality dimension:

#### honesty (0.0 = deceptive, 1.0 = completely honest)
- **0.0-0.3**: Lies easily, manipulative, cannot be trusted
- **0.3-0.7**: Generally honest with flexibility, some gray areas
- **0.7-1.0**: Truthful, transparent, keeps promises

**Apply To**:
- Claims about product quality/value
- Pricing justification
- Promises to customers
- Risk disclosure

**Example (0.1)**: "Best gear in the sector, no questions." (obviously lying)
**Example (0.9)**: "It's used, honestly. But well-maintained. Fair price for its condition."

---

#### greed (0.0 = altruistic, 1.0 = extremely greedy)
- **0.0-0.3**: Cares more about value than profit, may undersell
- **0.3-0.7**: Wants profit but balanced with other concerns
- **0.7-1.0**: Obsessed with maximum profit, exploitative

**Apply To**:
- Eagerness about sales
- Pricing aggressiveness
- Willingness to upsell
- Value of money discussion

**Example (0.2)**: "Take it. I just want to help travelers."
**Example (0.9)**: "That? Rare stock. Not cheap. But you need it, right?"

---

#### risk_tolerance (0.0 = cautious, 1.0 = reckless)
- **0.0-0.3**: Avoids risk, cautious, conservative
- **0.3-0.7**: Balanced approach, careful but not paranoid
- **0.7-1.0**: Embraces danger, reckless, thrill-seeker

**Apply To**:
- Willingness to deal with criminals
- Engagement in dangerous trades
- Attitude toward law enforcement
- Black market involvement

**Example (0.1)**: "I keep my business legitimate. No risks."
**Example (0.9)**: "Black market contacts? Sure. I've dealt with worse."

---

#### charm (0.0 = cold/rude, 1.0 = charismatic)
- **0.0-0.3**: Gruff, blunt, off-putting, unfriendly
- **0.3-0.7**: Personable, friendly, engaging
- **0.7-1.0**: Charismatic, magnetic, instantly likable

**Apply To**:
- Warmth of greetings
- Humor and wit usage
- Building rapport
- Making customers feel special

**Example (0.1)**: "Yeah, what do you want?"
**Example (0.9)**: "Ah, welcome back, friend! Good to see you again. Let me find something special for you."

---

#### ego_drive (0.0 = humble, 1.0 = egotistical)
- **0.0-0.3**: Self-deprecating, servant-minded, humble
- **0.3-0.7**: Confident, proud of accomplishments
- **0.7-1.0**: Arrogant, self-centered, insists on respect

**Apply To**:
- Self-references in dialogue
- Confidence in abilities
- Demands for respect
- Personal accomplishments

**Example (0.1)**: "Just happy to help. Not much to me."
**Example (0.9)**: "I'm the best supplier in this galaxy. Everyone knows it."

---

#### empathy (0.0 = callous, 1.0 = highly empathetic)
- **0.0-0.3**: Doesn't care about others, cold, uncaring
- **0.3-0.7**: Understands others, shows moderate concern
- **0.7-1.0**: Deeply compassionate, prioritizes others' feelings

**Apply To**:
- Understanding customer needs
- Concern for customer wellbeing
- Emotional support
- Relationship building

**Example (0.1)**: "Not my problem if it's too expensive."
**Example (0.9)**: "I see you're just starting out. Let me give you something fair. I remember being where you are."

---

#### curiosity (0.0 = uninterested, 1.0 = intensely curious)
- **0.0-0.3**: Satisfied with status quo, uninterested
- **0.3-0.7**: Interested in learning, asks questions
- **0.7-1.0**: Driven to explore, asks many questions, always learning

**Apply To**:
- Questions asked of customer
- Interest in news/gossip
- References to discoveries
- Desire to learn

**Example (0.1)**: "Just buy something and go."
**Example (0.9)**: "So where are you from? What's it like? Got any stories about what you've seen out there?"

---

## Dialogue Line Types

Generate dialogue for these 5 interaction types:

### 1. greeting
**Purpose**: Initial or routine greeting
**Context**: Player arrives at vendor location
**Tone**: Set expectations for interaction

**Low Charm Version (0.2)**: "Yeah, what do you need?"
**High Charm Version (0.9)**: "Welcome, friend! Always happy to see new faces."

**Repeat Customer High Curiosity (0.8)**: "Back again! You've got to tell me what you've been up to."

---

### 2. inventory_pitch
**Purpose**: Describe available inventory
**Context**: When showing what's for sale
**Tone**: Sell the goods, highlight value

**Low Greed (0.2)**: "I've got some decent supplies if you're interested."
**High Greed (0.9)**: "I've got exclusive merchandise. Premium quality. Worth the premium price."

**Weapons Context, Honest**: "Quality weaponry. Well-maintained, reliable."
**Weapons Context, Dishonest**: "The best in the sector. Take my word for it."

---

### 3. deal_accepted
**Purpose**: Reaction to accepted trade
**Context**: Player agrees to buy/sell
**Tone**: Express satisfaction (or gloating)

**High Empathy**: "Great! Glad I could help you out."
**Low Empathy**: "Yeah, I got the better end of that deal."
**High Ego**: "Wise choice. You won't regret buying from me."

---

### 4. deal_rejected
**Purpose**: Reaction to refused trade
**Context**: Player declines to trade
**Tone**: Disappointment (varies by personality)

**High Empathy**: "No problem, friend. Come back when you're ready."
**Low Empathy**: "Your loss."
**High Charm/Curiosity**: "No? Interesting. What would make you interested?"

---

### 5. farewell
**Purpose**: Goodbye/send-off dialogue
**Context**: Player leaves vendor
**Tone**: Encourage return, professional send-off

**High Empathy**: "Safe travels, friend. Come back soon."
**Pirate-like**: "Stay sharp out there. Dangerous times."
**High Curiosity**: "Come back and tell me what you find!"

---

## Interaction Buckets

Player familiarity levels - generate different dialogue for each:

### first_visit (Player's 1st interaction)
- Introduce yourself
- Establish initial rapport or danger
- Set expectations
- No references to previous meetings

**Example**: "Welcome to my shop. I'm Kovac."

### second_visit (2nd interaction)
- Show some recognition
- Building relationship starts
- May offer small incentives
- Can subtly reference first meeting

**Example**: "Good to see you again. You have good taste."

### third_visit (3rd interaction)
- Regular customer status
- More personalized
- Can reference patterns
- Show growing comfort

**Example**: "You're becoming a regular! I like that."

### repeat_customer (4+ interactions)
- Deep familiarity
- Personal dialogue
- Can be more informal
- Show true personality
- Reference shared history if applicable

**Example**: "My friend! Always good to see you. Got something special this time."

---

## Inventory Contexts

Generate specialized dialogue for specific goods:

### null (Generic, fallback)
- No specific commodity
- Works for all interactions
- Safe default
- **Always generate these for every line_type × bucket combination**

### "weapons"
- Combat gear, weapons systems, ammunition
- Tone: Functional, reliable
- Appeals to: Warriors, mercenaries
- Example: "Got quality armaments."

### "minerals"
- Raw resources, ore, refined materials
- Tone: Value-focused, practical
- Appeals to: Miners, builders, traders
- Example: "Prime grade minerals fresh from the belt."

### "rare_goods"
- Unique items, collectibles, antiques
- Tone: Exclusive, valuable
- Appeals to: Collectors, explorers
- Example: "Rare finds that few vendors carry."

### "contraband"
- Illegal goods, restricted items
- Tone: Discrete, careful (for high criminality)
- Appeals to: Outlaws, risk-takers
- **Only generate if criminality >= 0.5**
- Example: "I handle... specialty items. Discretely." (low honesty)

---

## Generation Requirements

### Quantities to Generate

For each vendor, generate:
- **5 line_types** × **4 interaction_buckets** × **5 inventory_contexts** = **100 combinations**
- Generate **2-3 lines per combination** for variety
- Total: **200-300 dialogue lines per vendor**

### Quality Requirements

**DO Generate**:
- ✅ Archetype-specific dialogue (match archetype exactly)
- ✅ Personality-aligned responses (use all 7 traits)
- ✅ Context-appropriate language (match inventory_context)
- ✅ Varied phrasing (not repetitive)
- ✅ Lines under 255 characters
- ✅ First-person vendor voice
- ✅ Natural conversation style
- ✅ Bucket progression (strangers → friends)

**DON'T Generate**:
- ❌ Out-of-universe references
- ❌ Metadata or system language
- ❌ Exact duplicate lines
- ❌ Lines over 255 characters
- ❌ Generic non-personality dialogue
- ❌ Inconsistent character voice
- ❌ Overly formal game-speak
- ❌ Contraband dialogue for low criminality vendors

---

## Examples by Archetype

### HONEST BROKER (honesty: 0.9, greed: 0.2, empathy: 0.8)

**Greeting, First Visit, null**:
"Welcome. I'm committed to fair prices and honest dealings."

**Inventory Pitch, Repeat Customer, weapons**:
"These weapons are solid. I've tested them myself. Good quality for a fair price."

**Deal Accepted, First Visit, null**:
"Excellent choice. I'm glad I could help you. Come back anytime."

**Deal Rejected, Repeat Customer, null**:
"No problem at all. I understand. Just let me know when you're ready."

**Farewell, Third Visit, null**:
"Safe travels, friend. I look forward to seeing you again."

---

### HARD BARGAINER (honesty: 0.5, greed: 0.8, ego_drive: 0.8)

**Greeting, First Visit, null**:
"Looking for a deal? You came to the right place."

**Inventory Pitch, Repeat Customer, weapons**:
"Prime merchandise. You know the quality I stock. Not cheap, but worth it."

**Deal Accepted, First Visit, null**:
"Good business today. You made the right call."

**Deal Rejected, Repeat Customer, null**:
"Your loss. But I'll be here when you're ready to negotiate."

**Farewell, Repeat Customer, null**:
"Don't stay away too long. I've got your interests in mind."

---

### BLACK MARKET DEALER (honesty: 0.1, greed: 0.9, empathy: 0.1)

**Greeting, First Visit, null**:
"You came to the right place. No questions asked."

**Inventory Pitch, Repeat Customer, contraband**:
"I've got what you need. Exclusive stock. The price reflects that."

**Deal Accepted, First Visit, null**:
"Smart. You won't find better elsewhere."

**Deal Rejected, First Visit, null**:
"Your mistake."

**Farewell, Repeat Customer, null**:
"Watch yourself out there. You owe me."

---

### EXPLORER OUTFITTER (honesty: 0.7, greed: 0.4, curiosity: 0.9)

**Greeting, First Visit, null**:
"Welcome! Are you heading out on an expedition? Tell me where!"

**Inventory Pitch, Repeat Customer, rare_goods**:
"You won't believe what I've sourced. Found some incredible pieces. Just arrived."

**Deal Accepted, Repeat Customer, null**:
"Excellent! This is going to serve you well out there."

**Deal Rejected, First Visit, null**:
"Interesting. What are you looking for instead?"

**Farewell, Third Visit, null**:
"Go explore those uncharted systems! Come back with stories."

---

## Error Handling

### What to Return on Failure

```json
{
  "request_id": "unique-request-identifier",
  "status": "error",
  "error_code": "insufficient_personality_data",
  "error_message": "Missing personality traits",
  "vendor_uuid": "660f9511-f40c-52e5-b827-557766551111"
}
```

### Common Errors

| Error | Cause | Action |
|-------|-------|--------|
| `invalid_personality` | Traits missing or out of range | Return error, request retry |
| `insufficient_personality_data` | Incomplete trait data | Return error, require all 7 traits |
| `invalid_archetype` | Unknown vendor archetype | Return error, verify archetype list |
| `generation_timeout` | Took too long | Return partial results or error |
| `invalid_output` | Generated lines too long | Trim and regenerate |

---

## Performance Notes

- **Target generation time**: 3-8 seconds per vendor (100-300 lines)
- **Batch processing**: Can handle multiple vendors per request
- **Timeout**: Set 60-second timeout per request
- **Rate limiting**: Expect 5-10 requests/minute during full galaxy generation

---

## Validation Checklist

Before returning dialogue to database, verify:

- [ ] All 5 line_types are present
- [ ] All 4 interaction_buckets are represented
- [ ] At least one `null` inventory_context per type×bucket
- [ ] All lines under 255 characters
- [ ] All lines are unique (no exact duplicates)
- [ ] First-person vendor voice (using "I", "me", "my")
- [ ] Archetype-appropriate language
- [ ] Personality traits reflected in word choice
- [ ] Proper JSON format
- [ ] Status is "success"

---

## Testing

Test generation with this simple vendor:

```json
{
  "request_id": "test-001",
  "vendor_profile": {
    "uuid": "test-550e8400",
    "name": "Test Vendor",
    "archetype": "honest_broker",
    "service_type": "trading_hub",
    "criminality": 0.1,
    "personality": {
      "honesty": 0.9,
      "greed": 0.2,
      "risk_tolerance": 0.3,
      "charm": 0.6,
      "ego_drive": 0.4,
      "empathy": 0.8,
      "curiosity": 0.5
    }
  },
  "galaxy_vendor_profile": {
    "uuid": "test-660f9511",
    "galaxy_id": 999,
    "poi_id": 9999,
    "dialogue_generation_version": 1
  }
}
```

**Expected output**: ~100-150 lines showing honest broker characteristics (honest, empathetic, fair).

---

## Integration Endpoint

**Service Name**: Vendor Dialogue Generation Service (Go AI)
**Protocol**: HTTP POST
**Endpoint**: `/generate-vendor-dialogue` (or equivalent)
**Request Timeout**: 60 seconds
**Response Format**: JSON (see Output Format above)

---

**Version**: 1.0
**Last Updated**: March 11, 2026
