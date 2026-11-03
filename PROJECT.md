# Grow a Sushi Empire

Roblox place **Grow a Sushi Empire** (`placeId` 94763502612420). Players fish at a shared Japanese seaside town, stock a personal restaurant on one of six plots, and sell sushi for coins. Progression is level, restaurant tier, one rod that levels from 1 to 50, and three personal upgrades. All purchases are checked on the server. Player progress is saved in DataStores.

Tune numbers in `ReplicatedStorage.EconomyConfig`. `GameConfig`, `RestaurantTiers`, and `RodConfig` read those values at runtime and overwrite their own starting numbers.

## Scripts

| Path | Role |
|---|---|
| `ServerScriptService.FishingServer` | Starts `PlayerData` and `PlotService`. Fishing, selling, bait, upgrades, restaurant upgrades, styles, rods, and conveyor auto-sell. |
| `ServerScriptService.PlayerData` | Load, save, leaderstats, inventory, XP, tier, style, rod. |
| `ServerScriptService.PlotService` | Assigns a free plot, clones the tier model onto the terrain, applies the style, sets the name sign. |
| `ServerScriptService.DayNight` | 10-minute day. Clear day, warm dusk, lighter night. `NightLantern` lights. |
| `ServerScriptService.Townsfolk` | 12 server-owned villagers. Routines walk the road graph, then a short pathfinding hop to a door. |
| `ServerScriptService.TownGraph` | Adjacency table for the road waypoints. `Townsfolk` runs A* on this, not on door nodes. |
| `ServerScriptService.TownAmbience` | Slowly orbits the coastal seagull models. |
| `ServerScriptService.RestaurantVisit` | BindableEvent. `Townsfolk` fires it with the plot owner when a villager eats at that restaurant. |
| `ServerScriptService.Leaderboard` | Ordered DataStores and the plaza board. |
| `ServerScriptService.MarketIdle` | Sways shop clerks tagged `MarketNPC`, using each part's `Home` CFrame. |
| `ServerStorage.TownsfolkTemplate` | R15 villager rig cloned by `Townsfolk`. |
| `ServerStorage.DecorTemplates` | Master copies: `StreetLamp`, `StoneLantern`, `Bench`, `Fence`, `Pine`, `Banner`, `Sakura`, `GableRoof`, `CurvedRoof`. |
| `StarterGui.GameHud.HudClient` | HUD, shops, catch card, catch effects, conveyor sale line. |
| `StarterPlayer.StarterPlayerScripts.SprintClient` | Sprint, stamina, dust. |
| `StarterPlayer.StarterPlayerScripts.FishingRodClient` | Rod hold, cast, bob, and reel. Shows fishing prompts only while the rod is equipped. Opens a zone gate once the player's level is high enough. |
| `ReplicatedStorage.GameConfig` | Fish pools, rarity colors, XP, upgrade formulas. |
| `ReplicatedStorage.EconomyConfig` | Live prices, odds, costs, auto-sell speed. |
| `ReplicatedStorage.RestaurantTiers` | Five tiers and three cosmetic styles. |
| `ReplicatedStorage.RodConfig` | One rod, levels 1–50, evolution looks, and the coin/XP curves. |
| `ReplicatedStorage.FishConfig` | Per-fish decal ids. |
| `ReplicatedStorage.FishingZones` | Four fishing zones: level, fish, rarity odds, XP multiplier. |

Remotes in `ReplicatedStorage.Remotes`:

- Events: `CatchFish`, `SellAll`, `DataUpdate`, `ClientFX`
- Functions: `BuyUpgrade`, `UpgradeRestaurant`, `SetStyle`, `BuyRod`, `EnchantRod`

`FishingServer` creates `BuyRod` and `EnchantRod` if they are missing.

## Multiplayer plots

`Workspace.Plots` holds `Plot1` through `Plot6`, sorted by name. On join, `PlotService` waits for saved data, then claims the first plot whose `Occupied` attribute is false. It sets `Occupied` and `OwnerUserId`, and the sign reads `{DisplayName}'s Sushi`. On leave the restaurant is cleared, the sign returns to `Empty Plot`, and the attributes reset.

Each plot has a `Pad`, fenced border, `PlotSpawn`, `Sign` (`OwnerLabel`), `PlotDock`, and `RestaurantHolder`. The restaurant is a clone of `ServerStorage.RestaurantModels.Tier1`–`Tier5`, named `Restaurant`. After it is placed on the plot, a raycast finds the terrain under the pad and the model is shifted so its bounding-box bottom sits on that surface (about y=6). It is then recolored by the player's style. `StyleRole` attributes (`Wood`, `Accent`, `Roof`, `Floor`, `Wall`, `Neon`) pick the palette. PointLights use the Neon color.

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
| 1 | Sushi Stall | 1 | 0 | 1.00x | 22s | 1 | Tier1 |
| 2 | Sushi Bar | 8 | 5,000 | 1.35x | 16s | 2 | Tier2 |
| 3 | Conveyor Restaurant | 16 | 50,000 | 1.80x | 11s | 3 | Tier3 |
| 4 | Omakase Lounge | 28 | 400,000 | 2.40x | 8s | 3 | Tier4 |
| 5 | Sushi Empire Tower | 40 | 3,000,000 | 3.20x | 5s | 4 | Tier5 |

The Restaurant button calls `UpgradeRestaurant`. The server checks level and coins, charges the player, clears plates, and respawns the model. Styles are cosmetic: Traditional (level 1), Modern (8), Neon (18). `SetStyle` checks the level, saves `Style`, and recolors the current model.

Auto-sell runs in `FishingServer`. `EconomyConfig.AutoSell` sets `fishPerTick` to 1 and `autoSellInterval` to 22, 16, 11, 8, 5 seconds, the same numbers copied onto `Restaurant.passiveInterval`. That is about 55% slower than the old 14 / 10 / 7 / 5 / 3.2 drain. Each tick removes that many oldest inventory fish and one plate each, pays `fish value * tier sell * (1 + rod sell bonus)`, and grants sell XP. Plates take 1.8 seconds between conveyor waypoints. The HUD line under the XP bar reads `Conveyor sells 1 every Ns`, and for 2.5 seconds after a sale it reads `Sold NAME +coins`.

## Economy config

`ReplicatedStorage.EconomyConfig` is the source of truth.

Fish sell values (about 40% of the original list in `GameConfig.FishByRarity`; `GameConfig` copies these over on require):

| Rarity | Fish (value) |
|---|---|
| Common | Sardine 3, Mackerel 4, Anchovy 3, Smelt 4, Shrimp 3 |
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

Rod leveling uses `EconomyConfig.RodProgress`: `baseCost` 40, `growth` 1.3, `maxLevel` 50, `xpBase` 24. The coin cost to leave level L is `floor(40 * 1.3 ^ L)`. Fish XP to the next level is `floor(24 * 1.3 ^ L)`, and 0 at level 50.

## Fishing rods

`ServerStorage.FishingRodTemplate` is one Tool (`IsFishingRod = true`). The Handle is the short butt. Shaft, Tip, and Reel are welded to it and run along the Handle's local +Y, so the tip is at local y=5.45. `Grip` is a 70.2° tilt on X (`CFrame.Angles(math.rad(70.2), 0, 0)`). On an R15 character, the default tool hold and the Hold animation both put the right hand in front of the body and the tip about 7 studs forward and 1.9 studs above the hand, idle and walking. Each player gets one clone named `FishingRod`.

`ReplicatedStorage.Assets.FishingAnims` holds KeyframeSequences `Hold`, `Cast`, `Bob`, and `Reel` (Action priority). Hold matches that forward grip. Cast pulls the tip up and back, then extends it forward toward the water, then returns to Hold. Bob is a small lift. Reel draws the tip in and returns to Hold. `FishingRodClient` registers each clip when `KeyframeSequenceProvider` allows it and plays it on the owning client. If registration is unavailable, the same poses are written onto the arm Motor6D or AnimationConstraint. Equip loops Hold. Pressing E plays Cast, then Bob. The server then sends Bite and Reel. Reel plays once; the catch result does not start a second reel.

A cast requires the rod equipped, a fishing spot within 18 studs, the zone's level, and the cooldown. The server raycasts outward from the spot, skipping the dock, and accepts only terrain water. A beam runs from the rod tip to a red bobber that arcs onto that water, bobs, shows a `!` billboard on the bite, then tweens back to the player. Invisible `Workspace.Baseplate` has `CanQuery` false so those rays reach the water instead of the baseplate top.

There is one rod, level 1–50. Stats blend linearly with `alpha = (level - 1) / 49`:

| Stat | Level 1 | Level 50 |
|---|---|---|
| Cast speed | 1.00 | 0.52 |
| Luck | 0 | 5 |
| Crit | 3% | 20% |
| Mutation | 0% | 9% |
| Sell bonus | 0% | 30% |

Every 5 levels the rod evolves. Higher tiers glow; from Coral up they emit particles; Dragon, Celestial, and Mythic Koi also leave a trail.

| Level | Id | Name | Look |
|---|---|---|---|
| 1 | Bamboo | Bamboo Rod | Wood, tan |
| 5 | Wooden | Wooden Rod | Wood, dark brown |
| 10 | Fiberglass | Fiberglass Rod | SmoothPlastic, blue |
| 15 | Carbon | Carbon Rod | Metal, near black |
| 20 | Steel | Steel Rod | Metal, glow |
| 25 | Coral | Coral Rod | SmoothPlastic, glow, particles |
| 30 | Golden | Golden Rod | Metal, glow, particles |
| 35 | Crystal | Crystal Rod | Glass, glow, particles |
| 40 | Dragon | Dragon Rod | Neon, glow, particles, trail |
| 45 | Celestial | Celestial Rod | Neon, glow, particles, trail |
| 50 | MythicKoi | Mythic Koi Rod | Neon, glow, particles, trail |

The Rod Shop spends coins for one level (`BuyRod`). `EnchantRod` calls the same level-up. Catch XP also fills `RodXP` and levels the rod. When the evolution id changes, the server fires `ClientFX` `"RodEvolved"` and a particle burst. The HUD popup reads `Your rod evolved into {name}!`.

Old saves migrate once (`RodSchema` is not 2). Bamboo starts at 1, Fiberglass at 10, Carbon at 15, Golden at 30, MythicKoi at 50, plus `floor(old enchant / 5)`, clamped to 1–50. Then `RodSchema` is 2, `RodId` is the evolution id, and `RodXP` is 0.

A crit (rod crit chance) catches 2 fish, or 3 if a second 35% roll hits, limited by inventory space. The HUD shows `CRITICAL CATCH xN!` and plays `Assets.CritCatch`.

Cast time and cooldown also scale with Faster Cast and the rod's cast speed. Floors are 0.25s cast and 0.4s cooldown.

## Shops and town

Shared world:

- `Workspace.Plaza` — cobblestone floor (under the terrain), `Fountain`, five market stalls. The old `Path1`–`Path6`, `PierPath`, and `SushiFountain` are gone. Visible roads are `Workspace.Town.Roads`. A downward ray through the plaza center (36 studs around the origin) found no floating parts.
- `Workspace.Plaza.Fountain` — stone basin 20 studs across and 3 studs tall, bottom flush at y=6, with four seats on the rim. A 10.6-stud center column holds a koi. Glass water, a spray emitter, and four soft `NightLantern` lights.
- `Workspace.Map.MainPier` and `Workspace.Map.FishingSpots` — the village dock is `WaterDock` on the pier, from z=626 (sand) to z=668 (about 40 studs over terrain water). All twelve Village Dock spots stand on that end (`Pier1`–`Pier6` at z=650, `Plot1Spot`–`Plot6Spot` at z=658). Each prompt says `Press E to Fish`, key E, no hold, and is clickable for touch. The client turns the prompts on only while the rod is equipped.
- `Workspace.Map.Interact` — invisible prompt hosts: `BaitShop`, `UpgradeShop`, `RodShop`, `PlazaSell`, standing a few studs in front of the shop doors. `OldFishingSpot` and the old `SellCounter` prompts are disabled
- `Workspace.Decor` — torii, one sakura model cloned along the town, boats, rocks, and bushes. Road lamps and benches live under `Workspace.Town`
- `Workspace.Town` — fishing town built on the terrain (surface about y=6). `StreamingEnabled` is on
- `Workspace.Boundaries` — invisible walls and a safety floor
- `Workspace.SpawnLocation` — plaza spawn

Terrain is a flat island, grass inside about 520 studs and sand beach outside that, with water past the shore. The wedges between the six plots stay open except for roads.

`Workspace.Town` folders:

| Folder | What is there |
|---|---|
| Roads | Cobblestone ring (radius 128, width 14) around the plaza, plus one straight spoke to each plot, the main dock, the town street (130°), and the shrine (190°). Round junction discs are the only joint. A short entrance path runs from the ring to each shop door. `Spoke_DockLink` continues the dock spoke to the pier bridge. Tops sit about y=6.06, CanCollide off, so nothing overlaps or z-fights. |
| Bridges | Wood ramp from the dock road onto the main pier, about z=524 to z=556. |
| Houses | 10 houses (Hana, Kenji, Aoi, Sora, Yuki, Haru, Ren, Mio, Emi, Jun), 25–35 studs tall, doors about 8 studs, varied tiled roofs. Along the street at 130° between plots 2 and 3. |
| Market | 6 fish stalls on that street. |
| TeaHouse / RamenShop | Tea house and ramen shop at the plaza end of the street. |
| Shrine | Grass hill near (-75, -423) with a shrine, torii, and steps. |
| Lighthouse | Striped coastal tower near (510, 166). Beacon is a `NightLantern`. |
| Harbor | Dock, boathouse, and four boats east of the main pier. |
| Warehouses | Two warehouses and crates inland of the dock. |
| Street | Clones of `DecorTemplates.StreetLamp`, about every 20 studs, alternating sides of the roads. Benches and banners use the same templates. Anchored. Small decor has CanCollide off. Lamp heads are tagged `NightLantern`. |
| Gardens | Stone lanterns (shrine and houses), flower beds, bamboo, and bonsai, all sitting on the terrain. Pines on the coast, and short fences behind the houses. Cherry trees in town are the one sakura template. |
| Mood | Koi pond and bridge in the plaza, paper lanterns over the market (the light is tagged `NightLantern`, not the paper), noren on shop doors, cats, a mossy stone wall, and dock crates, barrels, a net, and a drying rack. Seagulls orbit the coast. |
| Signs | Signposts for Market, Harbor, Shrine, and Lighthouse. |
| Waypoints | 70 invisible parts: junctions, spoke stops, plot ends, shop doors, house doors, tea, ramen, the fish market, the dock, and the shrine. Door parts set `Approach` to a road node, plus `InsideX` / `InsideZ` for house doors. `Home`, `Dest`, `Weight`, `Kind`, and `Plot` mark how `Townsfolk` may use them. |
| Routes | Empty. Old eat-seats were removed so villagers cannot target an empty plot. |
| Townsfolk | Runtime clones. Not saved in edit mode. |

| Shop | Prompt | What it does |
|---|---|---|
| Bait Shop | Buy Bait | Spends 40 coins immediately. No UI. |
| Upgrade Shop | Upgrades | Opens the upgrade panel. Buys go through `BuyUpgrade`. |
| Rod Shop | Rods | Opens the rod panel. `BuyRod` and `EnchantRod` both buy the next rod level. |
| Sell Counter | Sell All | Sells the whole inventory, same payout as the plot counter. |
| Leaderboard | none | SurfaceGuis `CoinsGui` and `LevelGui`. |

Two-part roofs are rebuilt from those templates. `GableRoof` is the wedge peak on the houses, tea house, ramen shop, and shrine: the halves meet at a ridge and slope out to the eaves, with about 1.5 studs of overhang. `CurvedRoof` is the shallow slate roof on the five stalls and every restaurant tier, with the same peak plus a short upturned eave. Flat single-part roofs (Kenji, Yuki, Mio, Jun) keep their slab, and the ridge cap sits on that slab. Every ridge, including the five restaurant templates, is level (world up) and its long axis follows the roof's ridge. A ray on each slope confirms the high side is the ridge. None needed to be flipped. The five stalls sit in the gaps around the ring, facing the plaza, with the floor bottom at y=6. Each has a blocky `Clerk` tagged `MarketNPC` (Head, Torso, arms, legs). `Home` CFrames match those grounded parts, so `MarketIdle` sways the arms and head in place instead of pulling them back through the floor. `Workspace.Map.Interact.GeneratedChef` is a mesh prop and is not tagged, so it does not animate.

`DayNight` runs a 600-second full day from clock 16.2. Night is before 6.4 or after 17.7, and that is when `NightLantern` parts turn Neon and their lights switch on. Dusk (16:00–18:36 and 5:24–7:18) keeps a warm tint. Day is clear: atmosphere density 0.08, haze 0.1, fog end 9000, so the whole island reads from above. Dusk is density 0.14 and haze 0.28. Night is density 0.16 and haze 0.22, not the old heavy fog.

`Townsfolk` clones `ServerStorage.TownsfolkTemplate` into 12 villagers (Aiko, Daichi, Chiyo, Goro, Botan, Isamu, Kaede, Makoto, Naomi, Riku, Saki, Taro). The server owns them. Each has a house-door home. The loop leaves home, walks to 1–3 destinations, goes home, and waits inside. Destinations are the shops, fish market, tea house, ramen shop, shrine, dock, and a player restaurant. A restaurant is eligible only when that plot is occupied and a `Restaurant` model exists. Weight is `2 + tier * 3`. Other stops use the waypoint `Weight` (shops and the fish market 5, tea and ramen 4, shrine and dock 3). One villager per destination; a full stop is skipped. Walk speed is 12–16, start delays are staggered, and a small lateral offset keeps people off the same line. Saki and Taro share a house with Aiko and Daichi and stand a few studs aside (`HomeSlot`).

Movement is A* over `TownGraph`, then `Humanoid:MoveTo` between road waypoints. `PathfindingService` is only the last hop from the road to a door. At a restaurant the villager stands at the sell counter, plays animation `507770239` with a bowl, and `RestaurantVisit` fires with the owner. The visit does not pay coins yet. Idle is `507766666`, walk is `507777826`. If `MoveTo` does not finish within 8 seconds, the villager is teleported to the nearest road waypoint.

Night is the same clock as `DayNight` (before 6.4 or after 17.7). Goro, Botan, and Saki keep walking and carry a lantern. The others stay home.

## UI

`StarterGui.GameHud` (`HudClient`):

- Top: coins, inventory `count/cap`, level, XP bar, then the conveyor line (`Conveyor sells 1 every Ns`, or `Sold NAME +coins` for 2.5 seconds)
- Catch card: rarity, name, mutation, value, and a fish picture (`FishConfig` image, or a colored ViewportFrame fish if `imageId` is empty)
- Toast, level-up popup, crit popup, mythic announcement
- Upgrade shop rows: Faster Cast, Bigger Inventory, Luckier Rod, with next-stat preview and cost
- Restaurant panel: current tier, next tier, upgrade button, style buttons
- Rod panel is created in code: current level and name, the five stats, fish XP toward the next level, the next level's stats and coin cost, and the next evolution preview. The button reads `Level Up` plus the cost, or `MAX` at level 50.

`ClientFX` kinds: `Cast`, `Bite`, `Reel`, `Sell`, `OpenShop`, `Toast`, `LevelUp`, `RodEvolved`, `Crit`, `OpenRodShop`, `PassiveSell`, `CatchFX`.

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

Still placeholders: Anchovy, Smelt, Shrimp, Tuna, Sea Bream, Scallop, Sea Urchin, Leviathan Tuna, Celestial Koi.

## Fishing zones

`ReplicatedStorage.FishingZones` picks the fish. All twenty-one spots live in `Workspace.Map.FishingSpots`. Terrain water starts near radius 628 (and z=628 on the main pier) at y=-1. Each dock starts on the sand just before that line and runs about 40 studs out over the water, with posts down to about y=-7. A sign shows the name and level. A `LockGate` blocks the dock until `FishingRodClient` sees `leaderstats.Level` at or above the requirement. The server still rejects the catch with `{name} unlocks at level {n}.` Catch XP is multiplied by the zone's `XpMultiplier`. Odds are filtered to rarities that zone actually has, then luck nudges the higher ones.

| Zone | Level | XP | Where | Odds | Fish |
|---|---|---|---|---|---|
| Village Dock | 1 | 1.0x | Main pier `WaterDock`, z=626 to z=668. Twelve spots at the end. Sign near (10, 6.4, 530). | Common 100% | Anchovy, Sardine, Smelt, Mackerel, Shrimp |
| Coral Reef | 8 | 1.6x | Dock along the reef heading, radius about 621–663. Three spots at the end. Pink coral on the rails. | Uncommon 68%, Rare 32% | Salmon, Sea Bream, Scallop, Octopus, Yellowtail |
| Deep Sea Pier | 18 | 2.4x | Same length on the opposite heading. Dark slate deck and a blue light on the outer post. | Uncommon 18%, Rare 42%, Epic 40% | Tuna, Bluefin Tuna, Eel, Sea Urchin, King Crab |
| Mystic Lagoon | 30 | 3.6x | Same length toward the lagoon. Pale deck, mist at the end. | Legendary 70%, Mythic 30% | Golden Koi, Dragon Fish, Leviathan Tuna, Celestial Koi |

## Saving

`PlayerData` uses DataStore `SushiEmpire_PlayerData_v1`, key `Player_{userId}`. Autosave every 60 seconds, plus save on leave and `BindToClose`. Loads and saves retry 3 times with backoff. A failed load starts from defaults.

Saved fields: `Coins`, `Inventory` (name, rarity, mutation, value), `Upgrades`, `Level`, `XP`, `RestaurantTier`, `Style`, `RodId`, `RodLevel`, `RodXP`, `RodSchema` (2), `RodLevels`.

Leaderboards use ordered stores `SushiEmpire_TopCoins_v1` and `SushiEmpire_TopLevel_v1`, refreshed about every 20 seconds, top 8.
