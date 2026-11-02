# Grow a Sushi Empire

Roblox place **Grow a Sushi Empire** (`placeId` 94763502612420). Players fish at a shared Japanese seaside town, stock a personal restaurant on one of six plots, and sell sushi for coins. Progression is level, restaurant tier, rod tier, and three personal upgrades. All purchases are checked on the server. Player progress is saved in DataStores.

Tune numbers in `ReplicatedStorage.EconomyConfig`. `GameConfig`, `RestaurantTiers`, and `RodConfig` read those values at runtime and overwrite their own starting numbers.

## Scripts

| Path | Role |
|---|---|
| `ServerScriptService.FishingServer` | Starts `PlayerData` and `PlotService`. Fishing, selling, bait, upgrades, restaurant upgrades, styles, rods, and conveyor auto-sell. |
| `ServerScriptService.PlayerData` | Load, save, leaderstats, inventory, XP, tier, style, rod. |
| `ServerScriptService.PlotService` | Assigns a free plot, clones the tier model, applies the style, sets the name sign. |
| `ServerScriptService.DayNight` | 10-minute day, lanterns, lighting. |
| `ServerScriptService.Leaderboard` | Ordered DataStores and the plaza board. |
| `ServerScriptService.MarketIdle` | Sways shop clerks tagged `MarketNPC`. |
| `StarterGui.GameHud.HudClient` | HUD, shops, catch card, catch effects. |
| `StarterPlayer.StarterPlayerScripts.SprintClient` | Sprint, stamina, dust. |
| `ReplicatedStorage.GameConfig` | Fish pools, rarity colors, XP, upgrade formulas. |
| `ReplicatedStorage.EconomyConfig` | Live prices, odds, costs. |
| `ReplicatedStorage.RestaurantTiers` | Five tiers and three cosmetic styles. |
| `ReplicatedStorage.RodConfig` | Five rods and enchant math. |
| `ReplicatedStorage.FishConfig` | Per-fish decal ids. |

Remotes in `ReplicatedStorage.Remotes`:

- Events: `CatchFish`, `SellAll`, `DataUpdate`, `ClientFX`
- Functions: `BuyUpgrade`, `UpgradeRestaurant`, `SetStyle`, `BuyRod`, `EnchantRod`

`FishingServer` creates `BuyRod` and `EnchantRod` if they are missing.

## Multiplayer plots

`Workspace.Plots` holds `Plot1` through `Plot6`, sorted by name. On join, `PlotService` waits for saved data, then claims the first plot whose `Occupied` attribute is false. It sets `Occupied` and `OwnerUserId`, and the sign reads `{DisplayName}'s Sushi`. On leave the restaurant is cleared, the sign returns to `Empty Plot`, and the attributes reset.

Each plot has a `Pad`, fenced border, `PlotSpawn`, `Sign` (`OwnerLabel`), `PlotDock`, and `RestaurantHolder`. The restaurant is a clone of `ServerStorage.RestaurantModels.Tier1`–`Tier5`, named `Restaurant`, sat on the pad, then recolored by the player's style. `StyleRole` attributes (`Wood`, `Accent`, `Roof`, `Floor`, `Wall`, `Neon`) pick the palette. PointLights use the Neon color.

Conveyors are child folders named `Conveyor1` and up. Each has `Waypoints` and `Plates`. Sushi plates only spawn on the owner's conveyors. The owner's sell prompt is the only one that sells at that restaurant. The plaza sell counter sells the same inventory.

The plaza, main pier, and fishing spots are shared.

## Levels and XP

Saved fields: `Level` (starts at 1) and `XP`. Leaderstats show **Coins** and **Level**.

XP to the next level is `floor(100 * level ^ 1.5)`.

Catch XP: Common 10, Uncommon 18, Rare 35, Epic 70, Legendary 140, Mythic 300. Frozen adds 5, Golden 15, Rainbow 40.

Sell XP is `max(1, floor(coins earned * 0.25))`, including tier and rod sell bonuses. A level-up fires `ClientFX` `"LevelUp"`, which shows a popup and plays `ReplicatedStorage.Assets.LevelUp`.

## Restaurant tiers

Names, sizes, and conveyor counts live on `RestaurantTiers.Tiers`. Costs, required level, sell multiplier, and auto-sell interval are replaced from `EconomyConfig.Restaurant`.

| Tier | Name | Level | Coins | Sell | Auto-sell | Conveyors | Model |
|---|---|---|---|---|---|---|---|
| 1 | Sushi Stall | 1 | 0 | 1.00x | 14s | 1 | Tier1 |
| 2 | Sushi Bar | 8 | 5,000 | 1.35x | 10s | 2 | Tier2 |
| 3 | Conveyor Restaurant | 16 | 50,000 | 1.80x | 7s | 3 | Tier3 |
| 4 | Omakase Lounge | 28 | 400,000 | 2.40x | 5s | 3 | Tier4 |
| 5 | Sushi Empire Tower | 40 | 3,000,000 | 3.20x | 3.2s | 4 | Tier5 |

The Restaurant button calls `UpgradeRestaurant`. The server checks level and coins, charges the player, clears plates, and respawns the model. Styles are cosmetic: Traditional (level 1), Modern (8), Neon (18). `SetStyle` checks the level, saves `Style`, and recolors the current model.

Auto-sell runs in `FishingServer`. Every `passiveInterval` it removes the oldest inventory fish and one plate, pays `fish value * tier sell * (1 + rod sell bonus)`, and grants sell XP.

## Economy config

`ReplicatedStorage.EconomyConfig` is the source of truth.

Fish sell values (about 40% of the original list in `GameConfig.FishByRarity`; `GameConfig` copies these over on require):

| Rarity | Fish (value) |
|---|---|
| Common | Sardine 3, Mackerel 4, Anchovy 3, Smelt 4 |
| Uncommon | Salmon 9, Tuna 10, Yellowtail 10, Sea Bream 8 |
| Rare | Eel 22, Octopus 20, Scallop 19, Sea Urchin 24 |
| Epic | Bluefin Tuna 56, King Crab 52, Giant Prawn 48 |
| Legendary | Golden Koi 160, Dragon Fish 180 |
| Mythic | Leviathan Tuna 600, Celestial Koi 800 |

Base rarity odds: Common 60%, Uncommon 25%, Rare 10%, Epic 4%, Legendary 0.9%, Mythic 0.1%. `GameConfig.LuckTables` is unused. Live rolls use `GameConfig.GetLuckTable`, which shifts weight off Common as luck rises.

Mutations, first match wins: Rainbow x20 at 0.1%, Golden x5 at 1%, Frozen x2 at 5%. Rod mutation chance multiplies those odds by `(1 + bonus * 8)`.

Personal upgrades, max level 60, cost `floor(baseCost * 1.35 ^ level)`:

| Id | Name | Base cost | Effect |
|---|---|---|---|
| FasterCast | Faster Cast | 45 | Cast time and cooldown shrink with `0.955 ^ level` |
| BiggerInventory | Bigger Inventory | 70 | Slots start at 20 and grow by a diminishing +4 per level |
| LuckierRod | Luckier Rod | 100 | Luck bonus, diminishing from +0.35 |

Lucky Bait is 40 coins, lasts 180 seconds, and adds 1 luck. It is not saved.

## Fishing rods

`ServerStorage.FishingRodTemplate` is a Tool (`IsFishingRod = true`) with Handle, Tip, and Reel. Each player gets one clone named `FishingRod`. Color, material, and glow follow the equipped rod and its enchant. Fishing requires the tool equipped and a fishing spot within 18 studs.

`RodConfig` stats before enchant. Costs come from `EconomyConfig.RodCosts`. The shop only sells the next rod in order.

| Order | Id | Name | Cost | Cast speed | Luck | Crit | Mutation | Sell bonus |
|---|---|---|---|---|---|---|---|---|
| 1 | Bamboo | Bamboo Rod | 0 | 1.00 | 0 | 3% | 0% | 0% |
| 2 | Fiberglass | Fiberglass Rod | 2,500 | 0.90 | 0.8 | 6% | 1.5% | 5% |
| 3 | Carbon | Carbon Rod | 20,000 | 0.78 | 1.8 | 10% | 3% | 10% |
| 4 | Golden | Golden Rod | 160,000 | 0.66 | 3.2 | 14% | 5% | 18% |
| 5 | MythicKoi | Mythic Koi Rod | 1,200,000 | 0.52 | 5.0 | 20% | 9% | 30% |

Enchant is per rod, max 25. Cost is `base 180 * 2.4 ^ (order - 1)`, then `* 1.35 ^ level`. Gains approach a cap (cast speed down to 35% of the rod's scale reduction, +2.5 luck, +8% crit capped at 45%, +6% mutation, +12% sell). Glow turns on at enchant 10 even if the rod itself does not glow.

A crit (rod crit chance) catches 2 fish, or 3 if a second 35% roll hits, limited by inventory space. The HUD shows `CRITICAL CATCH xN!` and plays `Assets.CritCatch`.

Cast time and cooldown also scale with Faster Cast and the rod's cast speed. Floors are 0.25s cast and 0.4s cooldown.

## Shops and town

Shared world:

- `Workspace.Plaza` — cobblestone floor, paths, sushi fountain, five market stalls
- `Workspace.Map.MainPier` and `Workspace.Map.FishingSpots` — pier spots `Pier1`–`Pier6` and plot spots `Plot1Spot`–`Plot6Spot`, each with a `Fish` prompt
- `Workspace.Map.Interact` — invisible prompt hosts: `BaitShop`, `UpgradeShop`, `RodShop`, `PlazaSell`. `OldFishingSpot` and the old `SellCounter` prompts are disabled
- `Workspace.Decor` — torii, sakura, boats, benches, rocks, bushes, lantern posts tagged `NightLantern`
- `Workspace.Boundaries` — invisible walls and a safety floor
- `Workspace.SpawnLocation` — plaza spawn

| Shop | Prompt | What it does |
|---|---|---|
| Bait Shop | Buy Bait | Spends 40 coins immediately. No UI. |
| Upgrade Shop | Upgrades | Opens the upgrade panel. Buys go through `BuyUpgrade`. |
| Rod Shop | Rods | Opens the rod panel. `BuyRod` and `EnchantRod`. |
| Sell Counter | Sell All | Sells the whole inventory, same payout as the plot counter. |
| Leaderboard | none | SurfaceGuis `CoinsGui` and `LevelGui`. |

Each stall has a blocky `Clerk` model tagged `MarketNPC` (Head, Torso, arms, legs, each with a `Home` CFrame). `MarketIdle` sways the arms and head. `Workspace.Map.Interact.GeneratedChef` is a mesh prop and is not tagged, so it does not animate.

`DayNight` runs a 600-second full day from clock 16.2. Night is before 6.4 or after 17.7. Tagged lanterns turn on, become Neon, and the atmosphere thickens. Lighting uses Future, atmosphere, bloom, color correction, and sun rays.

## UI

`StarterGui.GameHud` (`HudClient`):

- Top: coins, inventory `count/cap`, level, XP bar
- Catch card: rarity, name, mutation, value, and a fish picture (`FishConfig` image, or a colored ViewportFrame fish if `imageId` is empty)
- Toast, level-up popup, crit popup, mythic announcement
- Upgrade shop rows: Faster Cast, Bigger Inventory, Luckier Rod, with next-stat preview and cost
- Restaurant panel: current tier, next tier, upgrade button, style buttons
- Rod panel is created in code: stats, enchant, buy next rod

`ClientFX` kinds: `Cast`, `Sell`, `OpenShop`, `Toast`, `LevelUp`, `Crit`, `OpenRodShop`, `PassiveSell`, `CatchFX`.

Catch bursts scale by rarity (white, green, blue, purple plus a light, gold, rainbow). Mythic flashes the screen and announces the catch to everyone. Legendary and Mythic shake the catcher's camera. Mutations add a second burst.

`SprintClient` builds `SprintGui`: walk 16, sprint 26 while Left Shift is held (or the touch Sprint button), stamina 100 draining at 22/s and regenerating at 18/s, FOV +10, foot dust.

Sounds in `ReplicatedStorage.Assets`: `CoinCollect`, `WaterSplash`, `LevelUp`, `CritCatch`.

## Fish images

`FishConfig.imageId`. Empty ids use the ViewportFrame fish.

| Fish | Image |
|---|---|
| Sardine | `rbxassetid://128508991462347` |
| Mackerel | `rbxassetid://81710946102806` |
| Salmon | `rbxassetid://104514540483933` |
| Yellowtail | `rbxassetid://138312589245611` |
| Eel | `rbxassetid://93583371888506` |
| Octopus | `rbxassetid://137317154185968` |
| Bluefin Tuna | `rbxassetid://131012871240669` |
| King Crab | `rbxassetid://121598303907063` |
| Giant Prawn | `rbxassetid://123502790750621` (shrimp icon) |
| Golden Koi | `rbxassetid://121824233452904` |
| Dragon Fish | `rbxassetid://86456472670126` |

Still placeholders: Anchovy, Smelt, Tuna, Sea Bream, Scallop, Sea Urchin, Leviathan Tuna, Celestial Koi.

## Saving

`PlayerData` uses DataStore `SushiEmpire_PlayerData_v1`, key `Player_{userId}`. Autosave every 60 seconds, plus save on leave and `BindToClose`. Loads and saves retry 3 times with backoff. A failed load starts from defaults.

Saved fields: `Coins`, `Inventory` (name, rarity, mutation, value), `Upgrades`, `Level`, `XP`, `RestaurantTier`, `Style`, `RodId`, `RodLevels`.

Leaderboards use ordered stores `SushiEmpire_TopCoins_v1` and `SushiEmpire_TopLevel_v1`, refreshed about every 20 seconds, top 8.
