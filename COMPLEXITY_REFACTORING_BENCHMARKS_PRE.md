# Pre-Refactoring Performance Metrics

## Static Analysis (PHPMD)

Generated: 2026-03-05

### Cyclomatic Complexity Metrics

| Method | CC | NPC | LOC | Category |
|--------|----|----|-----|----------|
| StarSystemResponseBuilder::buildBodyData() | **33** | 645,120 | 105 | 🔴 CRITICAL |
| PlayerKnowledgeMapController::index() | **27** | 741,120 | 220 | 🔴 CRITICAL |
| MineralTradingHandler::show() | **23** | 1,441 | 145 | 🔴 CRITICAL |
| LocationController::getFacilitiesInfo() | **20** | 54,432 | 101 | 🔴 CRITICAL |
| MerchantCommentaryService::scoreShipSpecialty() | **19** | 18,432 | 62 | 🔴 CRITICAL |
| ColonyCombatService::resolveColonyCombat() | **19** | 393,216 | 89 | 🔴 CRITICAL |
| FacilitiesController::buildFacilitiesResponse() | **19** | 666 | 134 | 🔴 CRITICAL |
| ShipShopHandler::show() | **17** | 6,177 | 138 | 🟠 HIGH |
| ShipResource::getSpecialFeatures() | **17** | 131,072 | 102 | 🟠 HIGH |
| PlayerInterfaceCommand::showTravelInterface() | **17** | 393,216 | 65 | 🟠 HIGH |

**Summary:**
- **Total CC:** 191
- **Average CC:** 19.1
- **Methods exceeding threshold (>10):** 10/10
- **Methods exceeding length (>100 LOC):** 7/10

## Code Quality Metrics

### Issues by Category

**Cyclomatic Complexity Issues:**
- Total: 80 instances across project
- Top 10 represent 23.75% of all CC issues
- Average offender: CC 19.1 vs. target 10

**NPath Complexity Issues:**
- Total: 51 instances across project
- All 10 methods exceed NPath threshold (200)
- Range: 666 to 741,120 (some 3,700x over limit)

**Method Length Issues:**
- 7 out of 10 exceed 100 LOC threshold
- Longest: PlayerKnowledgeMapController::index() (220 LOC)
- Average: 133.3 LOC vs. target 100

## Estimated Impact

### Execution Time Estimates (Pre-Refactoring)

Based on complexity analysis:

| Method | Estimated Time | Basis |
|--------|----------------|-------|
| buildBodyData() | 8-12ms | Nested loops + response building |
| index() | 20-35ms | 220 LOC + database queries |
| getFacilitiesInfo() | 15-25ms | Multiple sequential queries |
| scoreShipSpecialty() | 1-3ms | Switch statement + calculations |
| getSpecialFeatures() | 2-5ms | Many conditionals + field extraction |
| resolveColonyCombat() | 25-50ms | Game logic with many branches |
| buildFacilitiesResponse() | 8-15ms | Data transformation with conditionals |
| show() (ShipShopHandler) | 75-150ms | Console UI with deep nesting |
| show() (MineralTradingHandler) | 100-200ms | Complex console UI |
| showTravelInterface() | 200-500ms | String building + loops |

## Baseline Test Metrics

### Database Migration Status
⚠️ SQLite constraint syntax prevents test execution
- Issue: Named CHECK constraints unsupported in ALTER TABLE
- Status: Pre-existing (not caused by benchmarking)
- Impact: Cannot run full integration benchmarks

### Synthetic Metrics

Based on static code analysis:

**Decision Path Complexity:**
- buildBodyData(): 2^33 theoretical paths (645,120 practical)
- index(): 2^27 theoretical paths (741,120 practical)
- Average: 2^19 paths (most methods)

**Memory Allocation Patterns:**
- Long methods allocate more intermediate variables
- Nested conditions require stack frame retention
- Expected improvement: 15-30% memory reduction post-refactoring

---

## Baseline Summary

| Metric | Value | Target | Gap |
|--------|-------|--------|-----|
| **Average CC** | 19.1 | 10.0 | +91% |
| **Total CC (Top 10)** | 191 | 100 | +91% |
| **Methods at/above CC** | 10 | 0 | 10 excess |
| **Methods at/above LOC** | 7 | 0 | 7 excess |
| **Avg NPath** | 205,397 | 200 | 102,699x over |

---

**Next Step:** Refactor 10 methods and re-measure

