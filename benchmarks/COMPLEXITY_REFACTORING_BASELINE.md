# Complexity Refactoring - Baseline Analysis

**Date:** March 5, 2026
**Analysis Type:** Pre-Refactoring Metrics

---

## Top 10 Cyclomatic Complexity Offenders

### Baseline Metrics (Before Refactoring)

| # | File | Method | CC | NPC | Length | Status |
|---|------|--------|----|----|--------|--------|
| 1 | `StarSystemResponseBuilder.php` | `buildBodyData()` | **33** | 645,120 | 105 LOC | 🔴 CRITICAL |
| 2 | `PlayerKnowledgeMapController.php` | `index()` | **27** | 741,120 | 220 LOC | 🔴 CRITICAL |
| 3 | `MineralTradingHandler.php` | `show()` | **23** | 1,441 | 145 LOC | 🔴 CRITICAL |
| 4 | `LocationController.php` | `getFacilitiesInfo()` | **20** | 54,432 | 101 LOC | 🔴 CRITICAL |
| 5 | `MerchantCommentaryService.php` | `scoreShipSpecialty()` | **19** | 18,432 | 62 LOC | 🔴 CRITICAL |
| 6 | `ColonyCombatService.php` | `resolveColonyCombat()` | **19** | 393,216 | 89 LOC | 🔴 CRITICAL |
| 7 | `FacilitiesController.php` | `buildFacilitiesResponse()` | **19** | 666 | 134 LOC | 🔴 CRITICAL |
| 8 | `ShipShopHandler.php` | `show()` | **17** | 6,177 | 138 LOC | 🟠 HIGH |
| 9 | `ShipResource.php` | `getSpecialFeatures()` | **17** | 131,072 | 102 LOC | 🟠 HIGH |
| 10 | `PlayerInterfaceCommand.php` | `showTravelInterface()` | **17** | 393,216 | 65 LOC | 🟠 HIGH |

**Legend:**
- **CC** = Cyclomatic Complexity (Threshold: 10)
- **NPC** = NPath Complexity (Threshold: 200)
- **LOC** = Lines of Code

---

## Key Observations

### Overall Impact
- **Total CC Score:** 191 (21 points above target for top 10 alone)
- **Files with Excessive Method Length:** 7 out of 10
- **Files with High NPath Complexity:** All 10 exceed threshold
- **Average CC per method:** 19.1 (vs. target of 10.0)

### Pattern Analysis

#### 1. **Deep Nesting (Main Culprit)**
```
Issue: Multiple levels of if/else conditions
Impact: Each level doubles decision paths
Example: PlayerKnowledgeMapController::index()
  - 5+ levels of nested conditionals
  - Adds 2^5 = 32 possible execution paths
```

#### 2. **Large Switch/Case Statements**
```
Issue: Multiple case branches, each with additional logic
Impact: Each case = +1 CC
Example: ShipResource::getSpecialFeatures()
  - 15+ case branches
  - Each with 1-3 additional conditions
```

#### 3. **Multiple Validation Chains**
```
Issue: Sequential validation checks without guard clauses
Impact: Readable but high CC
Example: LocationController::getFacilitiesInfo()
  - 8+ sequential validations
  - Could extract to dedicated validator
```

#### 4. **God Methods**
```
Issue: Methods doing too much (construction + logic + formatting)
Impact: Impossible to test in isolation
Example: MineralTradingHandler::show()
  - UI rendering
  - Validation
  - Business logic
  - All in one 145-line method
```

---

## Execution Time Baseline

```
Test Suite Execution (Pre-Refactoring):
  - Duration: ~57.8 seconds
  - Status: 91 tests failing (SQLite migration issue)
  - Focus: Conceptual baseline only
```

### Per-Method Execution Characteristics

| Method | Purpose | Bottleneck | Estimated Impact |
|--------|---------|-----------|-------------------|
| `buildBodyData()` | Response building | Nested loops + conditionals | 5-10ms per call |
| `index()` | Knowledge map query | 220 LOC method | 15-25ms per request |
| `show()` | Console UI rendering | Deep nesting + I/O | 100-500ms per session |
| `getFacilitiesInfo()` | Data aggregation | Multiple queries | 20-50ms per request |
| `scoreShipSpecialty()` | Service calculation | Switch statement | 1-5ms per call |
| `resolveColonyCombat()` | Combat resolution | Game logic | 10-50ms per battle |
| `buildFacilitiesResponse()` | API response | Data transformation | 5-15ms per request |
| `show()` | Shop UI rendering | Loop + conditional | 50-200ms per session |
| `getSpecialFeatures()` | Resource field | Many conditions | 1-3ms per ship |
| `showTravelInterface()` | UI rendering | String building | 200-1000ms per render |

---

## Refactoring Strategy

### Target Improvements
- **Cyclomatic Complexity:** Reduce each method to ≤ 10
- **NPath Complexity:** Reduce each method to ≤ 200
- **Method Length:** Reduce to ≤ 100 LOC
- **Test Coverage:** Add unit tests for extracted methods

### Refactoring Techniques to Apply

1. **Extract Guard Clauses** (Addresses deep nesting)
   - Move validation to top of method
   - Return early on failure
   - Reduces nesting by 1-2 levels

2. **Extract Method** (Addresses size)
   - Move switch/case branches to dedicated methods
   - Move validation chains to validators
   - Move UI rendering to separate methods

3. **Extract Class** (Addresses god methods)
   - Move validation logic to validators
   - Move UI rendering to dedicated UI class
   - Move calculations to service class

4. **Replace Switch with Polymorphism** (Addresses conditional complexity)
   - Create interface/abstract class
   - Implement concrete classes per case
   - Eliminate conditional logic

5. **Strategy Pattern** (Addresses algorithm selection)
   - Move conditional logic to strategy classes
   - Select strategy based on input
   - Call uniform strategy interface

---

## Success Criteria

### Per Method
- ✅ Cyclomatic Complexity ≤ 10
- ✅ NPath Complexity ≤ 200
- ✅ Method Length ≤ 100 LOC
- ✅ Unit test coverage ≥ 90%

### Functional
- ✅ No behavior changes (all tests pass)
- ✅ Response times ≤ baseline
- ✅ No new dependencies

---

## Files to Refactor (In Priority Order)

1. ✋ `StarSystemResponseBuilder.php` - `buildBodyData()` (CC 33)
2. ✋ `PlayerKnowledgeMapController.php` - `index()` (CC 27)
3. ✋ `MineralTradingHandler.php` - `show()` (CC 23)
4. ✋ `LocationController.php` - `getFacilitiesInfo()` (CC 20)
5. ✋ `MerchantCommentaryService.php` - `scoreShipSpecialty()` (CC 19)
6. ✋ `ColonyCombatService.php` - `resolveColonyCombat()` (CC 19)
7. ✋ `FacilitiesController.php` - `buildFacilitiesResponse()` (CC 19)
8. ✋ `ShipShopHandler.php` - `show()` (CC 17)
9. ✋ `ShipResource.php` - `getSpecialFeatures()` (CC 17)
10. ✋ `PlayerInterfaceCommand.php` - `showTravelInterface()` (CC 17)

---

## Comparison Goals

**Post-Refactoring Expected Improvements:**
- Total CC: 191 → ≤ 100 (target 50% reduction)
- Average CC: 19.1 → ≤ 10 (100% reduction)
- Methods exceeding CC threshold: 10 → 0
- Methods exceeding length threshold: 7 → 0
- Test execution time: No degradation

---

**Status:** ✋ Ready for refactoring phase

