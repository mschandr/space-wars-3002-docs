# Code Quality TODO List - PHPMD Analysis

**Generated:** 2026-03-05 21:45:24
**Total High-Impact Issues:** 247

## Priority Breakdown

- **CyclomaticComplexity**: 80 issues
- **NPathComplexity**: 51 issues
- **ExcessiveClassComplexity**: 23 issues
- **ExcessiveMethodLength**: 56 issues
- **CouplingBetweenObjects**: 22 issues
- **TooManyMethods**: 4 issues
- **TooManyPublicMethods**: 8 issues
- **ExcessiveClassLength**: 3 issues

## Detailed TODO List (Sorted by Impact)

### CyclomaticComplexity (80 issues)

- [ ] **1.** app/Console/Commands/AssignMineralProductionCommand.php:17
  - **Class:** AssignMineralProductionCommand::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **2.** app/Console/Commands/CartographyGenerateShopsCommand.php:19
  - **Class:** CartographyGenerateShopsCommand::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **3.** app/Console/Commands/ClassifyIceGiants.php:34
  - **Class:** ClassifyIceGiants::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **4.** app/Console/Commands/EconomyTickCommand.php:65
  - **Class:** EconomyTickCommand::​displayResults()
  - **Issue:** The method displayResults() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **5.** app/Console/Commands/GalaxyCreateMirror.php:23
  - **Class:** GalaxyCreateMirror::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **6.** app/Console/Commands/GalaxyDistributePirateBands.php:23
  - **Class:** GalaxyDistributePirateBands::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **7.** app/Console/Commands/GalaxyGenerateGates.php:23
  - **Class:** GalaxyGenerateGates::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **8.** app/Console/Commands/GalaxyInitialize.php:60
  - **Class:** GalaxyInitialize::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **9.** app/Console/Commands/GalaxyViewCommand.php:374
  - **Class:** GalaxyViewCommand::​handleTradingHub()
  - **Issue:** The method handleTradingHub() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **10.** app/Console/Commands/GalaxyViewCommand.php:425
  - **Class:** GalaxyViewCommand::​charToIndex()
  - **Issue:** The method charToIndex() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **11.** app/Console/Commands/InitializePlayerCommand.php:34
  - **Class:** InitializePlayerCommand::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **12.** app/Console/Commands/KnowledgeHydrateCommand.php:22
  - **Class:** KnowledgeHydrateCommand::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 18. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **13.** app/Console/Commands/PlayerCommand.php:224
  - **Class:** PlayerCommand::​renderStarSystem()
  - **Issue:** The method renderStarSystem() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **14.** app/Console/Commands/PlayerCommand.php:359
  - **Class:** PlayerCommand::​renderSpatialCanvas()
  - **Issue:** The method renderSpatialCanvas() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **15.** app/Console/Commands/PlayerInterfaceCommand.php:462
  - **Class:** PlayerInterfaceCommand::​getSystemViewColumn()
  - **Issue:** The method getSystemViewColumn() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **16.** app/Console/Commands/PlayerInterfaceCommand.php:772
  - **Class:** PlayerInterfaceCommand::​showTravelInterface()
  - **Issue:** The method showTravelInterface() has a Cyclomatic Complexity of 17. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **17.** app/Console/Commands/PlayerInterfaceCommand.php:911
  - **Class:** PlayerInterfaceCommand::​executeTravel()
  - **Issue:** The method executeTravel() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **18.** app/Console/Commands/PlayerInterfaceCommand.php:1252
  - **Class:** PlayerInterfaceCommand::​showCoordinateJump()
  - **Issue:** The method showCoordinateJump() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **19.** app/Console/Commands/PlayerInterfaceCommand.php:1409
  - **Class:** PlayerInterfaceCommand::​showScanResults()
  - **Issue:** The method showScanResults() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **20.** app/Console/Commands/PlayerInterfaceCommand.php:1495
  - **Class:** PlayerInterfaceCommand::​findNearestTradingHub()
  - **Issue:** The method findNearestTradingHub() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **21.** app/Console/Commands/TradingHubGenerateCommand.php:22
  - **Class:** TradingHubGenerateCommand::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **22.** app/Console/Shops/ComponentShopHandler.php:16
  - **Class:** ComponentShopHandler::​show()
  - **Issue:** The method show() has a Cyclomatic Complexity of 15. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **23.** app/Console/Shops/MineralTradingHandler.php:18
  - **Class:** MineralTradingHandler::​show()
  - **Issue:** The method show() has a Cyclomatic Complexity of 23. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **24.** app/Console/Shops/MineralTradingHandler.php:167
  - **Class:** MineralTradingHandler::​buyMineralFlow()
  - **Issue:** The method buyMineralFlow() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **25.** app/Console/Shops/PlansShopHandler.php:14
  - **Class:** PlansShopHandler::​show()
  - **Issue:** The method show() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **26.** app/Console/Shops/RepairShopHandler.php:14
  - **Class:** RepairShopHandler::​show()
  - **Issue:** The method show() has a Cyclomatic Complexity of 21. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **27.** app/Console/Shops/ShipShopHandler.php:25
  - **Class:** ShipShopHandler::​show()
  - **Issue:** The method show() has a Cyclomatic Complexity of 17. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **28.** app/Generators/Points/PoissonDisk.php:22
  - **Class:** PoissonDisk::​sample()
  - **Issue:** The method sample() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **29.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:54
  - **Class:** StarSystemResponseBuilder::​build()
  - **Issue:** The method build() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **30.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:178
  - **Class:** StarSystemResponseBuilder::​buildWarpGateData()
  - **Issue:** The method buildWarpGateData() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **31.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:278
  - **Class:** StarSystemResponseBuilder::​buildBodyData()
  - **Issue:** The method buildBodyData() has a Cyclomatic Complexity of 33. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **32.** app/Http/Controllers/Api/FacilitiesController.php:125
  - **Class:** FacilitiesController::​buildFacilitiesResponse()
  - **Issue:** The method buildFacilitiesResponse() has a Cyclomatic Complexity of 19. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **33.** app/Http/Controllers/Api/FacilitiesController.php:360
  - **Class:** FacilitiesController::​buildAvailableActions()
  - **Issue:** The method buildAvailableActions() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **34.** app/Http/Controllers/Api/GalaxyController.php:468
  - **Class:** GalaxyController::​join()
  - **Issue:** The method join() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **35.** app/Http/Controllers/Api/GalaxySettingsController.php:31
  - **Class:** GalaxySettingsController::​update()
  - **Issue:** The method update() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **36.** app/Http/Controllers/Api/LocationController.php:250
  - **Class:** LocationController::​getFacilitiesInfo()
  - **Issue:** The method getFacilitiesInfo() has a Cyclomatic Complexity of 20. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **37.** app/Http/Controllers/Api/NavigationController.php:83
  - **Class:** NavigationController::​getNearbySystems()
  - **Issue:** The method getNearbySystems() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **38.** app/Http/Controllers/Api/NavigationController.php:322
  - **Class:** NavigationController::​getLocalBodies()
  - **Issue:** The method getLocalBodies() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **39.** app/Http/Controllers/Api/PlayerKnowledgeMapController.php:29
  - **Class:** PlayerKnowledgeMapController::​index()
  - **Issue:** The method index() has a Cyclomatic Complexity of 27. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **40.** app/Http/Controllers/Api/ScanController.php:30
  - **Class:** ScanController::​scanSystem()
  - **Issue:** The method scanSystem() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **41.** app/Http/Controllers/Api/ShipShopController.php:177
  - **Class:** ShipShopController::​purchaseShip()
  - **Issue:** The method purchaseShip() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **42.** app/Http/Controllers/Api/TravelCalculationController.php:54
  - **Class:** TravelCalculationController::​calculateFuelCost()
  - **Issue:** The method calculateFuelCost() has a Cyclomatic Complexity of 15. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **43.** app/Http/Resources/ShipResource.php:213
  - **Class:** ShipResource::​getSpecialFeatures()
  - **Issue:** The method getSpecialFeatures() has a Cyclomatic Complexity of 17. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **44.** app/Jobs/CompleteGalaxyCreationJob.php:42
  - **Class:** CompleteGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **45.** app/Jobs/CompleteTieredGalaxyCreationJob.php:43
  - **Class:** CompleteTieredGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **46.** app/Services/ColonyCombatService.php:21
  - **Class:** ColonyCombatService::​initiateColonyAttack()
  - **Issue:** The method initiateColonyAttack() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **47.** app/Services/ColonyCombatService.php:148
  - **Class:** ColonyCombatService::​resolveColonyCombat()
  - **Issue:** The method resolveColonyCombat() has a Cyclomatic Complexity of 19. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **48.** app/Services/ColonyCycleService.php:66
  - **Class:** ColonyCycleService::​processColonyCycle()
  - **Issue:** The method processColonyCycle() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **49.** app/Services/CombatResolutionService.php:24
  - **Class:** CombatResolutionService::​resolveCombat()
  - **Issue:** The method resolveCombat() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **50.** app/Services/CustomsService.php:31
  - **Class:** CustomsService::​performCheck()
  - **Issue:** The method performCheck() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **51.** app/Services/GalaxyCreationService.php:71
  - **Class:** GalaxyCreationService::​createGalaxy()
  - **Issue:** The method createGalaxy() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **52.** app/Services/GalaxyGeneration/Generators/PlanetarySystemGenerator.php:536
  - **Class:** PlanetarySystemGenerator::​getMoonType()
  - **Issue:** The method getMoonType() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **53.** app/Services/GalaxyGeneration/Generators/WarpGateNetworkGenerator.php:292
  - **Class:** WarpGateNetworkGenerator::​collectGatePairsWithGlobalDedup()
  - **Issue:** The method collectGatePairsWithGlobalDedup() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **54.** app/Services/GalaxyGeneration/Support/SpatialIndex.php:64
  - **Class:** SpatialIndex::​findNeighbors()
  - **Issue:** The method findNeighbors() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **55.** app/Services/MerchantCommentaryService.php:521
  - **Class:** MerchantCommentaryService::​scoreShipSpecialty()
  - **Issue:** The method scoreShipSpecialty() has a Cyclomatic Complexity of 19. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **56.** app/Services/OuterSystemGenerator.php:103
  - **Class:** OuterSystemGenerator::​generatePlanetarySystems()
  - **Issue:** The method generatePlanetarySystems() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **57.** app/Services/OuterSystemGenerator.php:196
  - **Class:** OuterSystemGenerator::​generatePlanetDataForStar()
  - **Issue:** The method generatePlanetDataForStar() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **58.** app/Services/OuterSystemGenerator.php:392
  - **Class:** OuterSystemGenerator::​generateDormantGates()
  - **Issue:** The method generateDormantGates() has a Cyclomatic Complexity of 14. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **59.** app/Services/OuterSystemGenerator.php:502
  - **Class:** OuterSystemGenerator::​generateOuterPoints()
  - **Issue:** The method generateOuterPoints() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **60.** app/Services/PlayerDeathService.php:118
  - **Class:** PlayerDeathService::​determineRespawnLocation()
  - **Issue:** The method determineRespawnLocation() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **61.** app/Services/PlayerKnowledgeService.php:93
  - **Class:** PlayerKnowledgeService::​grantBulkKnowledge()
  - **Issue:** The method grantBulkKnowledge() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **62.** app/Services/PlayerKnowledgeService.php:332
  - **Class:** PlayerKnowledgeService::​getKnowledgeMap()
  - **Issue:** The method getKnowledgeMap() has a Cyclomatic Complexity of 14. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **63.** app/Services/PlayerSpawnService.php:414
  - **Class:** PlayerSpawnService::​ensureRichStarSystem()
  - **Issue:** The method ensureRichStarSystem() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **64.** app/Services/PvPCombatService.php:19
  - **Class:** PvPCombatService::​createChallenge()
  - **Issue:** The method createChallenge() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **65.** app/Services/PvPCombatService.php:190
  - **Class:** PvPCombatService::​resolvePvPCombat()
  - **Issue:** The method resolvePvPCombat() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **66.** app/Services/SalvageYardService.php:69
  - **Class:** SalvageYardService::​purchaseComponent()
  - **Issue:** The method purchaseComponent() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **67.** app/Services/ShipPurchaseService.php:280
  - **Class:** ShipPurchaseService::​getSpecialFeatures()
  - **Issue:** The method getSpecialFeatures() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **68.** app/Services/SystemPopulationService.php:389
  - **Class:** SystemPopulationService::​addMiningFacilities()
  - **Issue:** The method addMiningFacilities() has a Cyclomatic Complexity of 15. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **69.** app/Services/SystemPopulationService.php:627
  - **Class:** SystemPopulationService::​populateTerrestrialPlanet()
  - **Issue:** The method populateTerrestrialPlanet() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **70.** app/Services/SystemPopulationService.php:1408
  - **Class:** SystemPopulationService::​calculateHabitability()
  - **Issue:** The method calculateHabitability() has a Cyclomatic Complexity of 11. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **71.** app/Services/SystemScanService.php:178
  - **Class:** SystemScanService::​getFilteredSystemData()
  - **Issue:** The method getFilteredSystemData() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **72.** app/Services/TeamCombatService.php:20
  - **Class:** TeamCombatService::​inviteAlly()
  - **Issue:** The method inviteAlly() has a Cyclomatic Complexity of 15. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **73.** app/Services/TeamCombatService.php:168
  - **Class:** TeamCombatService::​acceptTeamChallenge()
  - **Issue:** The method acceptTeamChallenge() has a Cyclomatic Complexity of 16. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **74.** app/Services/TeamCombatService.php:259
  - **Class:** TeamCombatService::​resolveTeamCombat()
  - **Issue:** The method resolveTeamCombat() has a Cyclomatic Complexity of 23. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **75.** app/Services/TieredGalaxyCreationService.php:42
  - **Class:** TieredGalaxyCreationService::​createTieredGalaxy()
  - **Issue:** The method createTieredGalaxy() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **76.** app/Services/Trading/CommodityAccessService.php:24
  - **Class:** CommodityAccessService::​filterForPlayer()
  - **Issue:** The method filterForPlayer() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **77.** app/Services/Trading/TradingHubGenerator.php:102
  - **Class:** TradingHubGenerator::​batchStockHubs()
  - **Issue:** The method batchStockHubs() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **78.** app/Services/TravelService.php:66
  - **Class:** TravelService::​executeTravel()
  - **Issue:** The method executeTravel() has a Cyclomatic Complexity of 12. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **79.** app/Services/TravelService.php:232
  - **Class:** TravelService::​canDirectJump()
  - **Issue:** The method canDirectJump() has a Cyclomatic Complexity of 10. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

- [ ] **80.** app/Services/WarpGate/IncrementalWarpGateGenerator.php:124
  - **Class:** IncrementalWarpGateGenerator::​collectGatePairs()
  - **Issue:** The method collectGatePairs() has a Cyclomatic Complexity of 13. The configured cyclomatic complexity threshold is 10.
  - **Priority:** Level 3

### NPathComplexity (51 issues)

- [ ] **81.** app/Console/Commands/AssignMineralProductionCommand.php:17
  - **Class:** AssignMineralProductionCommand::​handle()
  - **Issue:** The method handle() has an NPath complexity of 336. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **82.** app/Console/Commands/EconomyTickCommand.php:65
  - **Class:** EconomyTickCommand::​displayResults()
  - **Issue:** The method displayResults() has an NPath complexity of 1080. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **83.** app/Console/Commands/GalaxyCreateMirror.php:23
  - **Class:** GalaxyCreateMirror::​handle()
  - **Issue:** The method handle() has an NPath complexity of 768. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **84.** app/Console/Commands/GalaxyDistributePirateBands.php:23
  - **Class:** GalaxyDistributePirateBands::​handle()
  - **Issue:** The method handle() has an NPath complexity of 240. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **85.** app/Console/Commands/GalaxyGenerateGates.php:23
  - **Class:** GalaxyGenerateGates::​handle()
  - **Issue:** The method handle() has an NPath complexity of 240. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **86.** app/Console/Commands/GalaxyInitialize.php:60
  - **Class:** GalaxyInitialize::​handle()
  - **Issue:** The method handle() has an NPath complexity of 768. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **87.** app/Console/Commands/InitializePlayerCommand.php:34
  - **Class:** InitializePlayerCommand::​handle()
  - **Issue:** The method handle() has an NPath complexity of 384. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **88.** app/Console/Commands/KnowledgeHydrateCommand.php:22
  - **Class:** KnowledgeHydrateCommand::​handle()
  - **Issue:** The method handle() has an NPath complexity of 7580. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **89.** app/Console/Commands/PlayerInterfaceCommand.php:772
  - **Class:** PlayerInterfaceCommand::​showTravelInterface()
  - **Issue:** The method showTravelInterface() has an NPath complexity of 816. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **90.** app/Console/Commands/PlayerInterfaceCommand.php:1252
  - **Class:** PlayerInterfaceCommand::​showCoordinateJump()
  - **Issue:** The method showCoordinateJump() has an NPath complexity of 576. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **91.** app/Console/Commands/PlayerInterfaceCommand.php:1495
  - **Class:** PlayerInterfaceCommand::​findNearestTradingHub()
  - **Issue:** The method findNearestTradingHub() has an NPath complexity of 300. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **92.** app/Console/Commands/TradingHubGenerateCommand.php:22
  - **Class:** TradingHubGenerateCommand::​handle()
  - **Issue:** The method handle() has an NPath complexity of 3072. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **93.** app/Console/Shops/ComponentShopHandler.php:16
  - **Class:** ComponentShopHandler::​show()
  - **Issue:** The method show() has an NPath complexity of 445. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **94.** app/Console/Shops/MineralTradingHandler.php:18
  - **Class:** MineralTradingHandler::​show()
  - **Issue:** The method show() has an NPath complexity of 1441. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **95.** app/Console/Shops/MineralTradingHandler.php:167
  - **Class:** MineralTradingHandler::​buyMineralFlow()
  - **Issue:** The method buyMineralFlow() has an NPath complexity of 768. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **96.** app/Console/Shops/RepairShopHandler.php:14
  - **Class:** RepairShopHandler::​show()
  - **Issue:** The method show() has an NPath complexity of 5521. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **97.** app/Console/Shops/ShipShopHandler.php:25
  - **Class:** ShipShopHandler::​show()
  - **Issue:** The method show() has an NPath complexity of 6177. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **98.** app/Generators/Points/PoissonDisk.php:22
  - **Class:** PoissonDisk::​sample()
  - **Issue:** The method sample() has an NPath complexity of 496. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **99.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:54
  - **Class:** StarSystemResponseBuilder::​build()
  - **Issue:** The method build() has an NPath complexity of 2592. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **100.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:278
  - **Class:** StarSystemResponseBuilder::​buildBodyData()
  - **Issue:** The method buildBodyData() has an NPath complexity of 645120. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **101.** app/Http/Controllers/Api/FacilitiesController.php:125
  - **Class:** FacilitiesController::​buildFacilitiesResponse()
  - **Issue:** The method buildFacilitiesResponse() has an NPath complexity of 666. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **102.** app/Http/Controllers/Api/GalaxyController.php:468
  - **Class:** GalaxyController::​join()
  - **Issue:** The method join() has an NPath complexity of 1728. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **103.** app/Http/Controllers/Api/GalaxySettingsController.php:31
  - **Class:** GalaxySettingsController::​update()
  - **Issue:** The method update() has an NPath complexity of 576. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **104.** app/Http/Controllers/Api/LocationController.php:250
  - **Class:** LocationController::​getFacilitiesInfo()
  - **Issue:** The method getFacilitiesInfo() has an NPath complexity of 54432. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **105.** app/Http/Controllers/Api/NavigationController.php:83
  - **Class:** NavigationController::​getNearbySystems()
  - **Issue:** The method getNearbySystems() has an NPath complexity of 2880. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **106.** app/Http/Controllers/Api/PlayerKnowledgeMapController.php:29
  - **Class:** PlayerKnowledgeMapController::​index()
  - **Issue:** The method index() has an NPath complexity of 741120. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **107.** app/Http/Controllers/Api/ShipShopController.php:177
  - **Class:** ShipShopController::​purchaseShip()
  - **Issue:** The method purchaseShip() has an NPath complexity of 384. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **108.** app/Http/Controllers/Api/TradingTransactionController.php:147
  - **Class:** TradingTransactionController::​sellMinerals()
  - **Issue:** The method sellMinerals() has an NPath complexity of 256. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **109.** app/Http/Controllers/Api/TravelCalculationController.php:54
  - **Class:** TravelCalculationController::​calculateFuelCost()
  - **Issue:** The method calculateFuelCost() has an NPath complexity of 2880. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **110.** app/Jobs/CompleteGalaxyCreationJob.php:42
  - **Class:** CompleteGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has an NPath complexity of 242. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **111.** app/Jobs/CompleteTieredGalaxyCreationJob.php:43
  - **Class:** CompleteTieredGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has an NPath complexity of 362. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **112.** app/Services/ColonyCombatService.php:21
  - **Class:** ColonyCombatService::​initiateColonyAttack()
  - **Issue:** The method initiateColonyAttack() has an NPath complexity of 720. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **113.** app/Services/ColonyCombatService.php:148
  - **Class:** ColonyCombatService::​resolveColonyCombat()
  - **Issue:** The method resolveColonyCombat() has an NPath complexity of 8480. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **114.** app/Services/ColonyCycleService.php:66
  - **Class:** ColonyCycleService::​processColonyCycle()
  - **Issue:** The method processColonyCycle() has an NPath complexity of 320. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **115.** app/Services/GalaxyCreationService.php:71
  - **Class:** GalaxyCreationService::​createGalaxy()
  - **Issue:** The method createGalaxy() has an NPath complexity of 1452. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **116.** app/Services/GalaxyGeneration/Generators/WarpGateNetworkGenerator.php:292
  - **Class:** WarpGateNetworkGenerator::​collectGatePairsWithGlobalDedup()
  - **Issue:** The method collectGatePairsWithGlobalDedup() has an NPath complexity of 521. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **117.** app/Services/GalaxyGeneration/Support/SpatialIndex.php:64
  - **Class:** SpatialIndex::​findNeighbors()
  - **Issue:** The method findNeighbors() has an NPath complexity of 400. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **118.** app/Services/MerchantCommentaryService.php:521
  - **Class:** MerchantCommentaryService::​scoreShipSpecialty()
  - **Issue:** The method scoreShipSpecialty() has an NPath complexity of 18432. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **119.** app/Services/OuterSystemGenerator.php:103
  - **Class:** OuterSystemGenerator::​generatePlanetarySystems()
  - **Issue:** The method generatePlanetarySystems() has an NPath complexity of 320. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **120.** app/Services/OuterSystemGenerator.php:392
  - **Class:** OuterSystemGenerator::​generateDormantGates()
  - **Issue:** The method generateDormantGates() has an NPath complexity of 264. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **121.** app/Services/PlayerKnowledgeService.php:93
  - **Class:** PlayerKnowledgeService::​grantBulkKnowledge()
  - **Issue:** The method grantBulkKnowledge() has an NPath complexity of 222. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **122.** app/Services/PlayerKnowledgeService.php:332
  - **Class:** PlayerKnowledgeService::​getKnowledgeMap()
  - **Issue:** The method getKnowledgeMap() has an NPath complexity of 640. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **123.** app/Services/PvPCombatService.php:19
  - **Class:** PvPCombatService::​createChallenge()
  - **Issue:** The method createChallenge() has an NPath complexity of 240. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **124.** app/Services/PvPCombatService.php:190
  - **Class:** PvPCombatService::​resolvePvPCombat()
  - **Issue:** The method resolvePvPCombat() has an NPath complexity of 336. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **125.** app/Services/SalvageYardService.php:69
  - **Class:** SalvageYardService::​purchaseComponent()
  - **Issue:** The method purchaseComponent() has an NPath complexity of 864. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **126.** app/Services/ShipPurchaseService.php:280
  - **Class:** ShipPurchaseService::​getSpecialFeatures()
  - **Issue:** The method getSpecialFeatures() has an NPath complexity of 288. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **127.** app/Services/SystemScanService.php:178
  - **Class:** SystemScanService::​getFilteredSystemData()
  - **Issue:** The method getFilteredSystemData() has an NPath complexity of 512. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **128.** app/Services/TeamCombatService.php:20
  - **Class:** TeamCombatService::​inviteAlly()
  - **Issue:** The method inviteAlly() has an NPath complexity of 5184. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **129.** app/Services/TeamCombatService.php:168
  - **Class:** TeamCombatService::​acceptTeamChallenge()
  - **Issue:** The method acceptTeamChallenge() has an NPath complexity of 2880. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **130.** app/Services/TeamCombatService.php:259
  - **Class:** TeamCombatService::​resolveTeamCombat()
  - **Issue:** The method resolveTeamCombat() has an NPath complexity of 217088. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

- [ ] **131.** app/Services/TravelService.php:66
  - **Class:** TravelService::​executeTravel()
  - **Issue:** The method executeTravel() has an NPath complexity of 864. The configured NPath complexity threshold is 200.
  - **Priority:** Level 3

### ExcessiveClassComplexity (23 issues)

- [ ] **132.** app/Console/Commands/GalaxyExpandCommand.php:15
  - **Class:** GalaxyExpandCommand
  - **Issue:** The class GalaxyExpandCommand has an overall complexity of 50 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **133.** app/Console/Commands/GalaxyFlushCommand.php:8
  - **Class:** GalaxyFlushCommand
  - **Issue:** The class GalaxyFlushCommand has an overall complexity of 57 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **134.** app/Console/Commands/GalaxyViewCommand.php:17
  - **Class:** GalaxyViewCommand
  - **Issue:** The class GalaxyViewCommand has an overall complexity of 55 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **135.** app/Console/Commands/PlayerCommand.php:21
  - **Class:** PlayerCommand
  - **Issue:** The class PlayerCommand has an overall complexity of 137 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **136.** app/Console/Commands/PlayerInterfaceCommand.php:20
  - **Class:** PlayerInterfaceCommand
  - **Issue:** The class PlayerInterfaceCommand has an overall complexity of 154 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **137.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:20
  - **Class:** StarSystemResponseBuilder
  - **Issue:** The class StarSystemResponseBuilder has an overall complexity of 95 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **138.** app/Http/Controllers/Api/FacilitiesController.php:21
  - **Class:** FacilitiesController
  - **Issue:** The class FacilitiesController has an overall complexity of 51 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **139.** app/Http/Controllers/Api/LocationController.php:21
  - **Class:** LocationController
  - **Issue:** The class LocationController has an overall complexity of 60 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **140.** app/Http/Controllers/Api/NavigationController.php:18
  - **Class:** NavigationController
  - **Issue:** The class NavigationController has an overall complexity of 58 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **141.** app/Models/Player.php:15
  - **Class:** Player
  - **Issue:** The class Player has an overall complexity of 55 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **142.** app/Models/PlayerShip.php:14
  - **Class:** PlayerShip
  - **Issue:** The class PlayerShip has an overall complexity of 74 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **143.** app/Models/PointOfInterest.php:26
  - **Class:** PointOfInterest
  - **Issue:** The class PointOfInterest has an overall complexity of 56 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **144.** app/Services/BarRumorService.php:18
  - **Class:** BarRumorService
  - **Issue:** The class BarRumorService has an overall complexity of 51 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **145.** app/Services/GalaxyGeneration/Generators/PlanetarySystemGenerator.php:27
  - **Class:** PlanetarySystemGenerator
  - **Issue:** The class PlanetarySystemGenerator has an overall complexity of 52 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **146.** app/Services/MerchantCommentaryService.php:12
  - **Class:** MerchantCommentaryService
  - **Issue:** The class MerchantCommentaryService has an overall complexity of 64 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **147.** app/Services/OuterSystemGenerator.php:29
  - **Class:** OuterSystemGenerator
  - **Issue:** The class OuterSystemGenerator has an overall complexity of 77 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **148.** app/Services/PlayerKnowledgeService.php:21
  - **Class:** PlayerKnowledgeService
  - **Issue:** The class PlayerKnowledgeService has an overall complexity of 56 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **149.** app/Services/PlayerSpawnService.php:19
  - **Class:** PlayerSpawnService
  - **Issue:** The class PlayerSpawnService has an overall complexity of 69 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **150.** app/Services/SalvageYardService.php:25
  - **Class:** SalvageYardService
  - **Issue:** The class SalvageYardService has an overall complexity of 61 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **151.** app/Services/SystemPopulationService.php:16
  - **Class:** SystemPopulationService
  - **Issue:** The class SystemPopulationService has an overall complexity of 147 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **152.** app/Services/SystemScanService.php:19
  - **Class:** SystemScanService
  - **Issue:** The class SystemScanService has an overall complexity of 95 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **153.** app/Services/TeamCombatService.php:11
  - **Class:** TeamCombatService
  - **Issue:** The class TeamCombatService has an overall complexity of 68 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

- [ ] **154.** app/Services/Trading/TradingHubGenerator.php:14
  - **Class:** TradingHubGenerator
  - **Issue:** The class TradingHubGenerator has an overall complexity of 57 which is very high. The configured complexity threshold is 50.
  - **Priority:** Level 3

### ExcessiveMethodLength (56 issues)

- [ ] **155.** app/Console/Commands/AssignMineralProductionCommand.php:17
  - **Class:** AssignMineralProductionCommand::​handle()
  - **Issue:** The method handle() has 128 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **156.** app/Console/Commands/BenchmarkLeaderboardCommand.php:30
  - **Class:** BenchmarkLeaderboardCommand::​handle()
  - **Issue:** The method handle() has 120 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **157.** app/Console/Commands/BenchmarkNavigationCommand.php:26
  - **Class:** BenchmarkNavigationCommand::​handle()
  - **Issue:** The method handle() has 150 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **158.** app/Console/Commands/CartographyGenerateShopsCommand.php:19
  - **Class:** CartographyGenerateShopsCommand::​handle()
  - **Issue:** The method handle() has 103 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **159.** app/Console/Commands/ClassifyIceGiants.php:34
  - **Class:** ClassifyIceGiants::​handle()
  - **Issue:** The method handle() has 104 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **160.** app/Console/Commands/GalaxyCreateMirror.php:23
  - **Class:** GalaxyCreateMirror::​handle()
  - **Issue:** The method handle() has 145 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **161.** app/Console/Commands/GalaxyDistributePirateBands.php:23
  - **Class:** GalaxyDistributePirateBands::​handle()
  - **Issue:** The method handle() has 116 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **162.** app/Console/Commands/GalaxyFlushCommand.php:232
  - **Class:** GalaxyFlushCommand::​getRowCount()
  - **Issue:** The method getRowCount() has 142 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **163.** app/Console/Commands/GalaxyGenerateGates.php:23
  - **Class:** GalaxyGenerateGates::​handle()
  - **Issue:** The method handle() has 128 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **164.** app/Console/Commands/GalaxyInitialize.php:60
  - **Class:** GalaxyInitialize::​handle()
  - **Issue:** The method handle() has 195 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **165.** app/Console/Commands/KnowledgeHydrateCommand.php:22
  - **Class:** KnowledgeHydrateCommand::​handle()
  - **Issue:** The method handle() has 148 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **166.** app/Console/Commands/PlayerInterfaceCommand.php:620
  - **Class:** PlayerInterfaceCommand::​renderControls()
  - **Issue:** The method renderControls() has 102 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **167.** app/Console/Commands/PlayerInterfaceCommand.php:772
  - **Class:** PlayerInterfaceCommand::​showTravelInterface()
  - **Issue:** The method showTravelInterface() has 124 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **168.** app/Console/Commands/PlayerInterfaceCommand.php:911
  - **Class:** PlayerInterfaceCommand::​executeTravel()
  - **Issue:** The method executeTravel() has 117 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **169.** app/Console/Commands/PlayerInterfaceCommand.php:1029
  - **Class:** PlayerInterfaceCommand::​handleNoShip()
  - **Issue:** The method handleNoShip() has 113 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **170.** app/Console/Commands/PlayerInterfaceCommand.php:1143
  - **Class:** PlayerInterfaceCommand::​showStarMap()
  - **Issue:** The method showStarMap() has 108 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **171.** app/Console/Commands/PlayerInterfaceCommand.php:1252
  - **Class:** PlayerInterfaceCommand::​showCoordinateJump()
  - **Issue:** The method showCoordinateJump() has 156 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **172.** app/Console/Commands/PlayerInterfaceCommand.php:1495
  - **Class:** PlayerInterfaceCommand::​findNearestTradingHub()
  - **Issue:** The method findNearestTradingHub() has 119 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **173.** app/Console/Commands/TradingHubGenerateCommand.php:22
  - **Class:** TradingHubGenerateCommand::​handle()
  - **Issue:** The method handle() has 142 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **174.** app/Console/Shops/MineralTradingHandler.php:18
  - **Class:** MineralTradingHandler::​show()
  - **Issue:** The method show() has 145 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **175.** app/Console/Shops/MineralTradingHandler.php:167
  - **Class:** MineralTradingHandler::​buyMineralFlow()
  - **Issue:** The method buyMineralFlow() has 135 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **176.** app/Console/Shops/MineralTradingHandler.php:306
  - **Class:** MineralTradingHandler::​sellMineralFlow()
  - **Issue:** The method sellMineralFlow() has 126 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **177.** app/Console/Shops/PirateEncounterHandler.php:46
  - **Class:** PirateEncounterHandler::​showEncounterScreen()
  - **Issue:** The method showEncounterScreen() has 101 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **178.** app/Console/Shops/RepairShopHandler.php:14
  - **Class:** RepairShopHandler::​show()
  - **Issue:** The method show() has 150 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **179.** app/Console/Shops/ShipShopHandler.php:25
  - **Class:** ShipShopHandler::​show()
  - **Issue:** The method show() has 138 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **180.** app/Console/Shops/ShipShopHandler.php:201
  - **Class:** ShipShopHandler::​purchaseShip()
  - **Issue:** The method purchaseShip() has 130 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **181.** app/Http/Controllers/Api/Builders/StarSystemResponseBuilder.php:278
  - **Class:** StarSystemResponseBuilder::​buildBodyData()
  - **Issue:** The method buildBodyData() has 105 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **182.** app/Http/Controllers/Api/FacilitiesController.php:125
  - **Class:** FacilitiesController::​buildFacilitiesResponse()
  - **Issue:** The method buildFacilitiesResponse() has 134 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **183.** app/Http/Controllers/Api/GalaxyController.php:468
  - **Class:** GalaxyController::​join()
  - **Issue:** The method join() has 124 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **184.** app/Http/Controllers/Api/LocationController.php:250
  - **Class:** LocationController::​getFacilitiesInfo()
  - **Issue:** The method getFacilitiesInfo() has 101 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **185.** app/Http/Controllers/Api/NavigationController.php:83
  - **Class:** NavigationController::​getNearbySystems()
  - **Issue:** The method getNearbySystems() has 139 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **186.** app/Http/Controllers/Api/NavigationController.php:322
  - **Class:** NavigationController::​getLocalBodies()
  - **Issue:** The method getLocalBodies() has 120 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **187.** app/Http/Controllers/Api/PlayerKnowledgeMapController.php:29
  - **Class:** PlayerKnowledgeMapController::​index()
  - **Issue:** The method index() has 220 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **188.** app/Http/Controllers/Api/SectorMapController.php:22
  - **Class:** SectorMapController::​index()
  - **Issue:** The method index() has 112 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **189.** app/Http/Controllers/Api/ShipShopController.php:177
  - **Class:** ShipShopController::​purchaseShip()
  - **Issue:** The method purchaseShip() has 104 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **190.** app/Http/Controllers/Api/TravelCalculationController.php:54
  - **Class:** TravelCalculationController::​calculateFuelCost()
  - **Issue:** The method calculateFuelCost() has 141 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **191.** app/Http/Resources/ShipResource.php:15
  - **Class:** ShipResource::​toArray()
  - **Issue:** The method toArray() has 132 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **192.** app/Http/Resources/ShipResource.php:213
  - **Class:** ShipResource::​getSpecialFeatures()
  - **Issue:** The method getSpecialFeatures() has 102 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **193.** app/Jobs/CompleteGalaxyCreationJob.php:42
  - **Class:** CompleteGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has 115 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **194.** app/Jobs/CompleteTieredGalaxyCreationJob.php:43
  - **Class:** CompleteTieredGalaxyCreationJob::​handle()
  - **Issue:** The method handle() has 144 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **195.** app/Services/ColonyCombatService.php:148
  - **Class:** ColonyCombatService::​resolveColonyCombat()
  - **Issue:** The method resolveColonyCombat() has 170 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **196.** app/Services/CombatResolutionService.php:24
  - **Class:** CombatResolutionService::​resolveCombat()
  - **Issue:** The method resolveCombat() has 135 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **197.** app/Services/Economy/ConstructionService.php:33
  - **Class:** ConstructionService::​build()
  - **Issue:** The method build() has 138 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **198.** app/Services/GalaxyCreationService.php:71
  - **Class:** GalaxyCreationService::​createGalaxy()
  - **Issue:** The method createGalaxy() has 259 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **199.** app/Services/GalaxyCreationService.php:601
  - **Class:** GalaxyCreationService::​createGalaxyAsync()
  - **Issue:** The method createGalaxyAsync() has 142 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **200.** app/Services/GalaxyGeneration/Generators/MirrorUniverseGenerator.php:63
  - **Class:** MirrorUniverseGenerator::​generate()
  - **Issue:** The method generate() has 117 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **201.** app/Services/GalaxyGeneration/Generators/MirrorUniverseGenerator.php:267
  - **Class:** MirrorUniverseGenerator::​generateSectorsForMirror()
  - **Issue:** The method generateSectorsForMirror() has 102 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **202.** app/Services/GalaxyGeneration/Generators/PlanetarySystemGenerator.php:89
  - **Class:** PlanetarySystemGenerator::​generate()
  - **Issue:** The method generate() has 101 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **203.** app/Services/OuterSystemGenerator.php:392
  - **Class:** OuterSystemGenerator::​generateDormantGates()
  - **Issue:** The method generateDormantGates() has 106 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **204.** app/Services/PlayerSpawnService.php:414
  - **Class:** PlayerSpawnService::​ensureRichStarSystem()
  - **Issue:** The method ensureRichStarSystem() has 103 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **205.** app/Services/PvPCombatService.php:190
  - **Class:** PvPCombatService::​resolvePvPCombat()
  - **Issue:** The method resolvePvPCombat() has 126 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **206.** app/Services/SalvageYardService.php:69
  - **Class:** SalvageYardService::​purchaseComponent()
  - **Issue:** The method purchaseComponent() has 133 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **207.** app/Services/SystemPopulationService.php:627
  - **Class:** SystemPopulationService::​populateTerrestrialPlanet()
  - **Issue:** The method populateTerrestrialPlanet() has 101 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **208.** app/Services/TeamCombatService.php:259
  - **Class:** TeamCombatService::​resolveTeamCombat()
  - **Issue:** The method resolveTeamCombat() has 170 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **209.** app/Services/TieredGalaxyCreationService.php:42
  - **Class:** TieredGalaxyCreationService::​createTieredGalaxy()
  - **Issue:** The method createTieredGalaxy() has 152 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

- [ ] **210.** app/Services/TravelService.php:66
  - **Class:** TravelService::​executeTravel()
  - **Issue:** The method executeTravel() has 122 lines of code. Current threshold is set to 100. Avoid really long methods.
  - **Priority:** Level 3

### CouplingBetweenObjects (22 issues)

- [ ] **211.** app/Console/Commands/BenchmarkLeaderboardCommand.php:20
  - **Class:** BenchmarkLeaderboardCommand
  - **Issue:** The class BenchmarkLeaderboardCommand has a coupling between objects value of 13. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **212.** app/Console/Commands/BenchmarkNavigationCommand.php:20
  - **Class:** BenchmarkNavigationCommand
  - **Issue:** The class BenchmarkNavigationCommand has a coupling between objects value of 13. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **213.** app/Console/Commands/GalaxyGeneratePoints.php:19
  - **Class:** GalaxyGeneratePoints
  - **Issue:** The class GalaxyGeneratePoints has a coupling between objects value of 13. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **214.** app/Console/Commands/GalaxyInitialize.php:27
  - **Class:** GalaxyInitialize
  - **Issue:** The class GalaxyInitialize has a coupling between objects value of 20. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **215.** app/Console/Commands/PlayerInterfaceCommand.php:20
  - **Class:** PlayerInterfaceCommand
  - **Issue:** The class PlayerInterfaceCommand has a coupling between objects value of 14. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **216.** app/Console/Commands/SeedTestData.php:15
  - **Class:** SeedTestData
  - **Issue:** The class SeedTestData has a coupling between objects value of 15. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **217.** app/Http/Controllers/Api/GalaxyController.php:22
  - **Class:** GalaxyController
  - **Issue:** The class GalaxyController has a coupling between objects value of 19. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **218.** app/Http/Controllers/Api/PlayerKnowledgeMapController.php:16
  - **Class:** PlayerKnowledgeMapController
  - **Issue:** The class PlayerKnowledgeMapController has a coupling between objects value of 14. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **219.** app/Http/Controllers/Api/TradingController.php:24
  - **Class:** TradingController
  - **Issue:** The class TradingController has a coupling between objects value of 15. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **220.** app/Jobs/CompleteTieredGalaxyCreationJob.php:29
  - **Class:** CompleteTieredGalaxyCreationJob
  - **Issue:** The class CompleteTieredGalaxyCreationJob has a coupling between objects value of 16. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **221.** app/Models/Galaxy.php:19
  - **Class:** Galaxy
  - **Issue:** The class Galaxy has a coupling between objects value of 17. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **222.** app/Models/Player.php:15
  - **Class:** Player
  - **Issue:** The class Player has a coupling between objects value of 19. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **223.** app/Models/PointOfInterest.php:26
  - **Class:** PointOfInterest
  - **Issue:** The class PointOfInterest has a coupling between objects value of 30. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **224.** app/Services/GalaxyCreationService.php:29
  - **Class:** GalaxyCreationService
  - **Issue:** The class GalaxyCreationService has a coupling between objects value of 29. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **225.** app/Services/GalaxyGeneration/GalaxyGenerationOrchestrator.php:38
  - **Class:** GalaxyGenerationOrchestrator
  - **Issue:** The class GalaxyGenerationOrchestrator has a coupling between objects value of 23. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **226.** app/Services/GalaxyGeneration/Generators/MirrorUniverseGenerator.php:38
  - **Class:** MirrorUniverseGenerator
  - **Issue:** The class MirrorUniverseGenerator has a coupling between objects value of 22. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **227.** app/Services/PirateEncounterService.php:13
  - **Class:** PirateEncounterService
  - **Issue:** The class PirateEncounterService has a coupling between objects value of 13. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **228.** app/Services/PlayerSpawnService.php:19
  - **Class:** PlayerSpawnService
  - **Issue:** The class PlayerSpawnService has a coupling between objects value of 18. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **229.** app/Services/SalvageYardService.php:25
  - **Class:** SalvageYardService
  - **Issue:** The class SalvageYardService has a coupling between objects value of 13. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **230.** app/Services/TieredGalaxyCreationService.php:26
  - **Class:** TieredGalaxyCreationService
  - **Issue:** The class TieredGalaxyCreationService has a coupling between objects value of 29. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **231.** app/Services/TradingService.php:28
  - **Class:** TradingService
  - **Issue:** The class TradingService has a coupling between objects value of 14. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

- [ ] **232.** app/Services/TravelService.php:18
  - **Class:** TravelService
  - **Issue:** The class TravelService has a coupling between objects value of 15. Consider to reduce the number of dependencies under 13.
  - **Priority:** Level 2

### TooManyMethods (4 issues)

- [ ] **233.** app/Console/Commands/PlayerCommand.php:21
  - **Class:** PlayerCommand
  - **Issue:** The class PlayerCommand has 33 non-getter- and setter-methods. Consider refactoring PlayerCommand to keep number of methods under 25.
  - **Priority:** Level 3

- [ ] **234.** app/Console/Commands/PlayerInterfaceCommand.php:20
  - **Class:** PlayerInterfaceCommand
  - **Issue:** The class PlayerInterfaceCommand has 27 non-getter- and setter-methods. Consider refactoring PlayerInterfaceCommand to keep number of methods under 25.
  - **Priority:** Level 3

- [ ] **235.** app/Models/PlayerShip.php:14
  - **Class:** PlayerShip
  - **Issue:** The class PlayerShip has 27 non-getter- and setter-methods. Consider refactoring PlayerShip to keep number of methods under 25.
  - **Priority:** Level 3

- [ ] **236.** app/Services/SystemPopulationService.php:16
  - **Class:** SystemPopulationService
  - **Issue:** The class SystemPopulationService has 35 non-getter- and setter-methods. Consider refactoring SystemPopulationService to keep number of methods under 25.
  - **Priority:** Level 3

### TooManyPublicMethods (8 issues)

- [ ] **237.** app/Models/CommodityLedgerEntry.php:11
  - **Class:** CommodityLedgerEntry
  - **Issue:** The class CommodityLedgerEntry has 11 public methods. Consider refactoring CommodityLedgerEntry to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **238.** app/Models/Galaxy.php:19
  - **Class:** Galaxy
  - **Issue:** The class Galaxy has 19 public methods. Consider refactoring Galaxy to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **239.** app/Models/NpcShip.php:12
  - **Class:** NpcShip
  - **Issue:** The class NpcShip has 11 public methods. Consider refactoring NpcShip to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **240.** app/Models/Player.php:15
  - **Class:** Player
  - **Issue:** The class Player has 24 public methods. Consider refactoring Player to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **241.** app/Models/PlayerShip.php:14
  - **Class:** PlayerShip
  - **Issue:** The class PlayerShip has 26 public methods. Consider refactoring PlayerShip to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **242.** app/Models/PointOfInterest.php:26
  - **Class:** PointOfInterest
  - **Issue:** The class PointOfInterest has 25 public methods. Consider refactoring PointOfInterest to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **243.** app/Models/WarpGate.php:13
  - **Class:** WarpGate
  - **Issue:** The class WarpGate has 16 public methods. Consider refactoring WarpGate to keep number of public methods under 10.
  - **Priority:** Level 3

- [ ] **244.** app/Services/NotificationService.php:10
  - **Class:** NotificationService
  - **Issue:** The class NotificationService has 14 public methods. Consider refactoring NotificationService to keep number of public methods under 10.
  - **Priority:** Level 3

### ExcessiveClassLength (3 issues)

- [ ] **245.** app/Console/Commands/PlayerCommand.php:21
  - **Class:** PlayerCommand
  - **Issue:** The class PlayerCommand has 1195 lines of code. Current threshold is 1000. Avoid really long classes.
  - **Priority:** Level 3

- [ ] **246.** app/Console/Commands/PlayerInterfaceCommand.php:20
  - **Class:** PlayerInterfaceCommand
  - **Issue:** The class PlayerInterfaceCommand has 1695 lines of code. Current threshold is 1000. Avoid really long classes.
  - **Priority:** Level 3

- [ ] **247.** app/Services/SystemPopulationService.php:16
  - **Class:** SystemPopulationService
  - **Issue:** The class SystemPopulationService has 1529 lines of code. Current threshold is 1000. Avoid really long classes.
  - **Priority:** Level 3

