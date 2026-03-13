# Space Wars 3002 – Go Dialogue Generator
## Technical Design Document for Text Generation Methodology

## Purpose

This document specifies how the Go dialogue generator should create, validate, and store vendor dialogue for Space Wars 3002.

It focuses on the text generation methodology, not on gameplay mechanics.

The generator must produce stable, context-aware flavour text for vendors without attempting to model live gameplay state or item-instance-specific facts.

---

# 1. Generator Mission

The Go service exists to generate vendor voice lines.

It must not generate:
- exact item-instance descriptions
- real-time negotiation outcomes
- dynamic live responses to player inputs
- freeform conversations

It must generate:
- greetings
- farewells
- deal accepted / rejected lines
- generic category-level inventory pitches
- transaction-aware flavour lines

All generation is asynchronous and offline.

---

# 2. Core Generation Principle

## Voice, not state

The generator is responsible for producing the NPC's voice and tone.

The PHP backend is responsible for combining that voice with live gameplay state.

### Example
The generator may store:

- This engine still has some fight left in it.
- Ugly casing, solid output.
- You can do worse than this hull.

The generator must not store:

- This engine is at 61 percent condition.
- This exact part came from a pirate skirmish yesterday.
- This manifold has the exact defect Natalia just found.

Those facts belong to runtime composition.

---

# 3. Input Data

The generator reads from vendor_profiles.

Relevant fields:

- id
- service_type
- criminality
- markup_base
- personality
- dialogue_generation_status
- dialogue_generation_version

## Example source row data

```json
{
  "service_type": "salvage_yard",
  "criminality": 0.27,
  "markup_base": 0.2152,
  "personality": {
    "honesty": 0.45,
    "greed": 0.55,
    "charm": 0.44,
    "risk_tolerance": 0.76
  }
}
```

---

# 4. Output Data

The generator writes to vendor_dialogue.

Each row should be keyed by:

- vendor_profile_id
- line_type
- interaction_bucket
- transaction_context
- inventory_context

The row should contain:
- line_text
- weight
- generation_version

---

# 5. Controlled Generation Dimensions

All generated dialogue must be scoped by a known combination of dimensions.

## 5.1 line_type
Allowed v1 values:
- greeting
- inventory_pitch
- deal_accepted
- deal_rejected
- farewell

## 5.2 interaction_bucket
Allowed values:
- first_visit
- second_visit
- third_visit
- repeat_customer

## 5.3 transaction_context
Allowed values:
- neutral
- vendor_selling
- vendor_buying

## 5.4 inventory_context
Canonical values:
- none
- ship
- shield_projector
- engine
- reactor
- weapon
- sensor_array
- cargo_module
- hull_plating
- salvage_component

The generator must only use canonical inventory contexts.

---

# 6. Generation Matrix

## Recommended v1 matrix

### Greetings
- greeting / first_visit / neutral / none
- greeting / second_visit / neutral / none
- greeting / third_visit / neutral / none
- greeting / repeat_customer / neutral / none

### Inventory pitches
- inventory_pitch / repeat_customer / vendor_selling / ship
- inventory_pitch / repeat_customer / vendor_selling / shield_projector
- inventory_pitch / repeat_customer / vendor_selling / engine
- inventory_pitch / repeat_customer / vendor_selling / salvage_component

### Buying reactions
- inventory_pitch / repeat_customer / vendor_buying / ship
- inventory_pitch / repeat_customer / vendor_buying / engine
- inventory_pitch / repeat_customer / vendor_buying / salvage_component

### Deal outcomes
- deal_accepted / repeat_customer / vendor_selling / none
- deal_accepted / repeat_customer / vendor_buying / none
- deal_rejected / repeat_customer / vendor_selling / none
- deal_rejected / repeat_customer / vendor_buying / none

### Farewells
- farewell / repeat_customer / neutral / none

This matrix gives good coverage without exploding content volume.

---

# 7. Prompt Builder Methodology

The generator should not send raw database blobs to the model and hope for good output.

It must convert source data into explicit prompt hints.

---

# 8. Prompt Signal Derivation

## 8.1 Service type mapping

### salvage_yard
Prompt traits:
- used goods
- practical tone
- rough edges allowed
- may reference salvage, repairs, wear

### shipyard
Prompt traits:
- polished
- more prideful
- stronger sales posture
- higher prestige language

### trading_hub
Prompt traits:
- general merchant language
- broad product framing
- less specialized tone

### market
Prompt traits:
- open commerce
- general trade language
- less technical specificity

---

## 8.2 Criminality mapping

Convert criminality to tone hints:

### low criminality
- cleaner language
- low menace
- low evasiveness

### medium criminality
- sharper tone
- more opportunistic
- some roughness permitted

### high criminality
- more predatory edge
- rougher, more dubious
- but still within allowed content rules

The prompt should describe the tone, not simply dump the number.

---

## 8.3 Markup mapping

### high markup
Prompt hints:
- defensive about pricing
- premium framing
- worth the price language

### low or negative markup
Prompt hints:
- casual bargain framing
- lower price defensiveness
- less pompous

---

## 8.4 Personality mapping

Use source personality fields to derive tone hints:

### honesty
Controls:
- bluntness
- transparency
- reduced exaggeration

### greed
Controls:
- price defensiveness
- harder sell
- scarcity framing

### charm
Controls:
- friendliness
- smoothness
- social ease

### risk_tolerance
Controls:
- willingness to sell questionable goods
- rough confidence
- used but usable framing

These should be transformed into readable instructions.

---

# 9. Prompt Structure

Each request should use:

1. system message
2. user message
3. low-variance output settings

---

## 9.1 System message template

You generate short in-universe dialogue for NPC vendors in a space exploration and trading game.
Output JSON only.
Each line must be one sentence.
Do not include explanations.
Do not include markdown.

---

## 9.2 User message template

Generate {N} {line_type} lines for a {service_type} vendor.

Context:
- interaction_bucket: {interaction_bucket}
- transaction_context: {transaction_context}
- inventory_context: {inventory_context}
- criminality_tone: {criminality_tone}
- markup_tone: {markup_tone}

Vendor voice:
- honesty_hint: {honesty_hint}
- greed_hint: {greed_hint}
- charm_hint: {charm_hint}
- risk_hint: {risk_hint}

Rules:
- one sentence per line
- {min_words} to {max_words} words
- keep lines in character
- do not mention exact live item condition percentages
- do not invent exact live defects
- return JSON: {"lines":["..."]}

This structure is stable and explicit.

---

# 10. Line-Type-Specific Rules

## 10.1 Greetings
Should:
- fit interaction bucket
- optionally acknowledge repeat visits
- not mention exact products unless asked
- establish vendor tone

## 10.2 Inventory pitches
Should:
- reference category-level goods
- sound like sales language
- avoid exact item-instance facts
- differ for vendor_selling vs vendor_buying

### vendor_selling
Should sound:
- boastful
- persuasive
- minimizing flaws

### vendor_buying
Should sound:
- skeptical
- appraising
- value-lowering

## 10.3 Deal accepted
Should:
- reflect tone
- optionally acknowledge professional respect or annoyance
- be short

## 10.4 Deal rejected
Should:
- refuse without becoming freeform argument
- remain in vendor tone
- avoid exact mechanical explanations

## 10.5 Farewells
Should:
- close interaction cleanly
- optionally hint at future business
- fit vendor tone

---

# 11. Suggested Batch Sizes

Per request, generate small sets.

Recommended:
- greetings: 15 lines
- inventory pitches: 10 lines
- deal accepted/rejected: 10 lines
- farewells: 10 lines

This keeps output manageable and validation cheap.

---

# 12. LLM Parameters

Recommended defaults for llama.cpp:

- temperature: 0.3 - 0.5
- top_p: 0.6 - 0.8
- max_tokens: 300 - 500
- stream: false

Do not run high temperatures. Small local models become sloppy quickly.

---

# 13. Validation Rules

The generator must reject poor output.

## Required validation

- response parses as JSON
- object contains lines
- lines is an array
- line count meets minimum threshold
- no empty lines
- exact duplicates removed
- each line:
  - at least 6 words
  - at most 20 words
  - at most 255 characters
- no obvious meta text
- no malformed control characters

## Fail examples
Reject if line resembles:
- Here are your lines:
- Certainly, here is the JSON
- code block junk
- multi-sentence rambles

---

# 14. Normalization

Before dedupe or insert:

- trim whitespace
- collapse repeated spaces
- strip accidental numbering prefixes
- normalize terminal punctuation if necessary

Use normalized text for dedupe hashing.

---

# 15. Retry Strategy

Retry only when useful.

## Retry conditions
- malformed JSON
- line count too low
- lines mostly invalid
- timeout
- repeated meta output

## Retry limits
- max 2 retries

## Retry adjustments
Second attempt should:
- lower temperature
- repeat JSON-only instruction
- reduce requested line count if needed

If still bad:
- fail the bucket
- mark vendor failed in current run

---

# 16. Storage Strategy

Generation should replace old rows by scope.

Delete existing rows for:

- vendor_profile_id
- line_type
- interaction_bucket
- transaction_context
- inventory_context

Then insert new rows for the current generation_version.

This avoids duplicates and ambiguity.

---

# 17. Logging Strategy

Log:
- vendor ID
- line type
- interaction bucket
- transaction context
- inventory context
- prompt hash
- line count inserted
- retry count
- final status

Do not log raw prompts/responses in production except at debug level.

---

# 18. Hashing and Versioning

Use:
- generation_version from vendor_profiles
- optional prompt hash for diagnostics

A prompt template change or model change should justify incrementing generation version.

---

# 19. Worker Design

Because the machine is limited:
- use 1 or 2 workers max
- small batches
- sequential bucket generation per vendor

Suggested:
- WORKER_COUNT=1
- BATCH_SIZE=10 vendors

Optimize for stability, not throughput.

---

# 20. Hard Rule About Live Item Facts

The generator must never store lines that depend on exact item-instance values.

Do not generate lines like:
- 61 percent condition
- coolant instability on manifold A3
- came from pirate captain X

Those values move and change.

The stored line should remain useful even when the item changes.

---

# 21. Example Good Stored Lines

### Engine selling pitch
- This engine still has some fight left in it.
- Ugly casing, strong output.
- Not pretty, but it will move a ship.

### Engine buying reaction
- I've seen worse.
- Depends what's wrong with it.
- If the core is clean, we can talk.

These work with runtime clause composition later.

---

# 22. Example Bad Stored Lines

- This engine is at 61 percent condition.
- This exact manifold is unstable.
- This was salvaged from a battle yesterday.

These are not stable.

---

# 23. Final Generator Workflow

load vendor
derive prompt hints
iterate generation matrix
build prompt for each scope
call llama.cpp
parse JSON
validate lines
delete old rows for that scope
insert fresh rows
mark vendor complete

---

# 24. Deliverables for Go AI

The Go implementation should include:

- config loader
- MariaDB repository layer
- vendor loader
- prompt builder
- llama client
- validation package
- generator orchestrator
- logging
- retry handling

---

# 25. Final Summary

The Go generator must create stable flavour text for vendors.

It must generate:
- context-scoped
- category-scoped
- transaction-aware
- item-instance-agnostic lines

The generator succeeds if the PHP backend can later combine those lines with live item facts and produce believable dialogue without ever calling the LLM during gameplay.
