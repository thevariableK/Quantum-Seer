# Rendering creature models/portraits from stored records, and creature data, in WoW: Forever (1.60.1, Interface 16001)

Research date: 2026-11-14. Labels: **[confirmed]** = seen in primary source (Blizzard UI source dump or Blizzard data via wago.tools); **[reported]** = from a secondary source or wiki text not checked against the client; **[unknown]** = not found.

Primary source used heavily: Gethe/wow-ui-source `forever` branch, HEAD commit `bebcb6bf` "1.60.1 (70938)" dated 2026-11-13 (shallow-cloned, read only) — https://github.com/Gethe/wow-ui-source/tree/forever . Paths below are relative to `Interface/AddOns/` in that branch.

Key context: Forever is **not** the Classic Era (1.15.x) client. It runs a mainline-family UI with a "Camelot" game-type override layer. `WOW_PROJECT_CAMELOT = 18; WOW_PROJECT_ID = WOW_PROJECT_CAMELOT` is in `Blizzard_ProjectConstants/Camelot/`. Toc files load `[Family]` (Mainline) files with `[Game]` (Camelot) overrides, e.g. `Blizzard_FrameXMLBase.toc`: `[Game]\Constants.lua [AllowLoadGameType camelot]`. Its API docs carry Midnight-era (12.x) "SecretArguments" predicates. So the model/creature API surface is essentially retail 12.x, **not** Classic Era 1.15.

## 1. Model widget APIs (PlayerModel / CinematicModel / ModelScene): availability and cache behavior

### Takeaway
Every API in question exists in Forever 1.60.1: SetCreature, SetDisplayInfo, SetUnit, SetPortraitZoom, SetCamDistanceScale, GetDisplayInfo, ClearModel, GetModelFileID, the OnModelLoaded handler, and the ModelScene actor's SetModelByCreatureDisplayID. The safe way to re-render a card is from a stored **creatureDisplayID** (`SetDisplayInfo` or `actor:SetModelByCreatureDisplayID`). `SetCreature(npcID)` depends on the client's creature cache and has historically drawn a blank model for NPCs the player hasn't encountered.

### Cited Findings
- [confirmed] The PlayerModel (CharacterModelBase) methods in the Forever docs are ApplySpellVisualKit, CanSetUnit, FreezeAnimation, GetDisplayInfo, GetDoBlend, GetKeepModelOnHide, HasAnimation, PlayAnimKit, RefreshCamera, RefreshUnit, SetAnimation, SetBarberShopAlternateForm, **SetCamDistanceScale, SetCreature, SetDisplayInfo**, SetDoBlend, SetItem, SetItemAppearance, **SetKeepModelOnHide, SetPortraitZoom**, SetRotation, **SetUnit**, StopAnimKit and ZeroCachedCenterXY. — `Blizzard_APIDocumentationGenerated/FrameAPICharacterModelBaseDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] Signatures. `SetCreature(creatureID, displayID = 0)`. `SetDisplayInfo(displayID, mountDisplayID?)`. `GetDisplayInfo() -> displayID`. All three carry `SecretArguments = "AllowedWhenUntainted"`. — same file, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] The Model base (SimpleModelAPI) includes ClearModel, **GetModelFileID**, SetModel, SetCamera, SetCameraDistance, SetFacing, SetPosition, SetModelScale, SetDesaturation, SetPaused, SetSequence, SetLight, SetModelAlpha, SetParticlesEnabled, among 62 methods. — `SimpleModelAPIDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever); the same list is on [warcraft.wiki.gg UIOBJECT_PlayerModel](https://warcraft.wiki.gg/wiki/UIOBJECT_PlayerModel)
- [confirmed] **OnModelLoaded(self)**, "Invoked when the model is loaded", is listed as a PlayerModel script handler. — [warcraft.wiki.gg UIOBJECT_PlayerModel](https://warcraft.wiki.gg/wiki/UIOBJECT_PlayerModel). Forever's own Camelot PaperDollFrame uses the ModelScene equivalent, `actor:SetOnModelLoadedCallback(...)`. — `Blizzard_UIPanels_Game/Camelot/PaperDollFrame.lua:3496`
- [confirmed] CinematicModel exists, with EquipItem, InitializeCamera, InitializePanCamera, RefreshCamera, SetAnimOffset, SetCameraPosition, SetCameraTarget, SetCreatureData, SetFacingLeft, SetFadeTimes, SetHeightFactor, SetJumpInfo, SetPanDistance, SetSpellVisualKit, SetTargetDistance, StartPan, StopPan and UnequipItems. — `FrameAPICinematicModelDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] ModelScene actors have **SetModelByCreatureDisplayID(creatureDisplayID, useActivePlayerCustomizations=false) -> success**, plus SetModelByFileID, SetModelByPath, SetModelByUnit, SetModelByUnitCreatureDisplayID, ClearModel, GetModelFileID, GetModelUnitGUID, IsLoaded, Get/SetYaw, Pitch, Roll, Scale, Get/SetAnimation and SetDesaturation. C_ModelInfo also exists, with GetModelSceneInfoByID, GetModelSceneActorInfoByID, GetModelSceneActorDisplayInfoByID, GetModelSceneCameraInfoByID and Add/ClearActiveModelScene(Actor). — `FrameAPIModelSceneFrameActorBaseDocumentation.lua`, `UIModelInfoDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] warcraft.wiki.gg's PlayerModel:SetCreature page lists the API for Forever 1.60.1 (69913), Midnight 12.1.5, Mists Classic 5.5.4, TBC Classic 2.5.6 and **Vanilla Classic 1.15.9**. So it also exists in Classic Era. — [warcraft.wiki.gg API PlayerModel SetCreature](https://warcraft.wiki.gg/wiki/API_PlayerModel_SetCreature)
- [confirmed] Blizzard's own Forever-family code renders portraits from display IDs. The Vanilla quest frame calls `QuestNPCModel:SetDisplayInfo(portraitDisplayID, mountPortraitDisplayID)` (`Blizzard_UIPanels_Game/Vanilla/QuestFrame.lua:25`). Shared code calls `self:SetDisplayInfo(11686)` (`Blizzard_SharedMapDataProviders/WorldQuestDataProvider.lua:485`). Mainline TutorialFrame calls `TutorialNPCModel:SetCreature(displayNPC)`. — [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [reported] SetCreature "can display any creature whose data is cached by the client". — [wowprogramming SetCreature](https://wowprogramming.com/docs/widgets/PlayerModel/SetCreature.html)
- [reported] Developer report (2016): SetCreature "doesn't show the model any longer if you haven't seen the npc before... once it's in your cache it does work". — [WoWInterface thread](https://www.wowinterface.com/forums/showthread.php?p=314931)
- [reported] After patch 8.2, a `DressUpModel:SetCreature(npcid)` macro stopped working for unseen NPCs. The poster was told players "would need to encounter NPCs to build a cache of their display IDs". — [Blizzard forums: Display creature macro issue](https://us.forums.blizzard.com/en/wow/t/display-creature-macro-issue/325634)
- [reported] Models can fall back to a white, untextured "ghost" state after certain calls, or when a model is set by file path without textures. — [Blizzard forums: Rendering Cached Offline Player Models](https://us.forums.blizzard.com/en/wow/t/addon-api-rendering-cached-offline-player-models/2342284)

### Inferences
- Store `displayID` on the card, not only `npcID`. `SetDisplayInfo(displayID)` resolves through the client-side CreatureDisplayInfo DB2, which ships with the client (14k rows, see section 4), so it does not need a creature-cache entry. `SetCreature(npcID)` needs the server-sent creature record, which is cached once the player has met the NPC. For cards this matters less, because the warlock was next to the creature when draining it. But the WDB cache can be cleared, and cards might be shared between players.
- Capture the displayID at drain time, since there is no unit→displayID API (section 3). Two routes: (a) set a hidden PlayerModel with `SetUnit("target")` and then read `GetDisplayInfo()`, or (b) use a ModelScene actor's `SetModelByUnit`/`GetModelFileID` to store a file ID. Route (a) is plausible but untested here. Whether `GetDisplayInfo()` returns the creature display ID after SetUnit on an NPC is not documented, so verify it in game.
- Use `OnModelLoaded` / `SetOnModelLoadedCallback` to apply camera, zoom and facing after an asynchronous load. Use `SetPortraitZoom(1)` + `SetCamDistanceScale` for a head-shot "portrait" framing.

### Gaps
- [unknown] Whether Forever's `SetCreature` still draws a blank model for an uncached npcID. Only old retail reports exist; nothing Forever-specific.
- [unknown] Performance cost of many simultaneous model frames. No sources found. Inference: render only the visible page of cards, reuse a small pool of PlayerModel/ModelScene frames (pagination), and ClearModel on hide.
- [unknown] Exact behavior of `GetDisplayInfo()` after `SetUnit` on an NPC, and of `SetModelByUnitCreatureDisplayID`. Not documented beyond signatures.

## 2. Static portraits: SetPortraitTexture vs SetPortraitTextureFromCreatureDisplayID

### Takeaway
`SetPortraitTexture(texture, unit)` needs a live unit token. Forever, like Classic 1.13.2+ and retail, also has **`SetPortraitTextureFromCreatureDisplayID(texture, creatureDisplayID)`**, which renders a 2D portrait for any display ID without the creature being present. This is the cheapest way to show card thumbnails.

### Cited Findings
- [confirmed] `SetPortraitTexture(textureObject, unitToken, disableMasking=false)` and `SetPortraitTextureFromCreatureDisplayID(textureObject, creatureDisplayID)` are both in the Forever Unit API docs. — `Blizzard_APIDocumentationGenerated/UnitDocumentation.lua:572-592`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] The wiki lists SetPortraitTextureFromCreatureDisplayID for Forever 1.60.1 (69913), Vanilla 1.15.9, TBC 2.5.6, Mists 5.5.4 and retail 12.1.5. It was added in 8.0.1 (and in Classic 1.13.2). — [warcraft.wiki.gg API_SetPortraitTextureFromCreatureDisplayID](https://warcraft.wiki.gg/wiki/API_SetPortraitTextureFromCreatureDisplayID)
- [confirmed] Blizzard code using it: `Blizzard_StableUI/Standard/Blizzard_StableUI.lua:75` and `Blizzard_SharedMapDataProviders/EncounterJournalDataProvider.lua:103`. Note that Forever's own Camelot Stable UI instead uses `self:SetPortraitToUnit(...)` (`Blizzard_StableUI/Camelot/Blizzard_StableUI.lua:51`). — [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] The CreatureDisplayInfo DB2 in the Forever build has `PortraitCreatureDisplayInfoID` and `PortraitTextureFileDataID` columns. Some displays therefore have pre-baked portrait textures or alternate portrait displays. — [wago.tools CreatureDisplayInfo CSV, build 1.60.1.70938](https://wago.tools/db2/CreatureDisplayInfo/csv?build=1.60.1.70938)

### Inferences
- Card grid: use `SetPortraitTextureFromCreatureDisplayID` on plain textures, which is cheap. Card detail view: use one PlayerModel/ModelScene showing the 3D model.

### Gaps
- [unknown] Whether the portrait renders for display IDs whose model files are not yet streamed/downloaded on a Forever client, for example with a partial install.

## 3. Getting a displayID or npcID at runtime

### Takeaway
npcID: `C_CreatureInfo.GetCreatureID(guid)` exists in Forever, or parse field 6 of the GUID. displayID: Forever has **no** unit→creatureDisplayID API. `C_PlayerInfo.GetDisplayID` is for the player only. Two caveats: inside instances, creature identity (GUID/ID) is reportedly "secret" under Midnight rules, and the combat log is reportedly unavailable to addons.

### Cited Findings
- [confirmed] `C_CreatureInfo` in Forever has GetClassInfo, GetCreatureFamilyIDs, GetCreatureFamilyInfo, **GetCreatureID(creatureGUID) -> creatureID?**, GetCreatureTypeIDs, GetCreatureTypeInfo, GetFactionInfo and GetRaceInfo. — `CreatureInfoDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever). Blizzard uses it in `Blizzard_BoostTutorial/Blizzard_TutorialLogic.lua:1513`.
- [confirmed] Searching all Forever API docs for functions with "DisplayID" in the name found only C_PlayerInfo.GetDisplayID / GetNativeDisplayID (player), C_PetJournal.GetDisplayIDByIndex, C_MountJournal.GetAllCreatureDisplayIDsForMountID and a Delves companion function. No `UnitCreatureDisplayID`-style API exists. — `Blizzard_APIDocumentationGenerated/*`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] GUID format is `Creature-0-[serverID]-[instanceID]-[zoneUID]-[npcID]-[spawnUID]`, and the npcID is `select(6, strsplit("-", UnitGUID("target")))`. On **Forever 1.60.1 the server and spawn portions are obfuscated**, stable and unique within a session, with the format otherwise unchanged. — [warcraft.wiki.gg GUID](https://warcraft.wiki.gg/wiki/GUID)
- [confirmed] In Forever docs, UnitGUID, UnitCreatureType and UnitCreatureFamily are flagged `SecretWhenUnitIdentityRestricted = true`. UnitName is `SecretWhenUnitNameIdentityRestricted`. UnitClassification and UnitLevel only have `SecretArguments`. — `UnitDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] AddOnRestrictionType in Forever has Combat, Encounter, ChallengeMode, PvPMatch, Map and Chat. `C_RestrictedActions.GetAddOnRestrictionState` / `IsAddOnRestrictionActive` exist. COMBAT_LOG_EVENT_UNFILTERED is marked `HasRestrictions = true`. — `RestrictedActionsConstantsDocumentation.lua`, `CombatLogDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [reported] Under SecretWhenUnitIdentityRestricted, "Guarded APIs and events produce secret values when the unit isn't player-controlled or in the party/raid" (Patch 12.0.0). — [warcraft.wiki.gg Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)
- [reported] Forever: "inside an instance, creature unit names, GUIDs and IDs are Secret", and `COMBAT_LOG_EVENT` "is no longer available to addons at all". The site says the only Blizzard Forever statement is a Discord comment from 2026-09-15 that Midnight restrictions apply to Forever. Open-world behavior is not addressed. — [wowforevertierlist.com restrictions](https://wowforevertierlist.com/addons/restrictions); similar claims at [forevermeta.me](https://forevermeta.me/en/guides/wow-forever-addons-interface) (secondary, mixed quality)

### Inferences
- Capture at drain time, with the target live: read `UnitGUID("target")` → `C_CreatureInfo.GetCreatureID`, then `UnitName`, `UnitLevel`, `UnitClassification`, `UnitCreatureType` and `UnitCreatureFamily`. Also capture the display, either via a hidden model (`SetUnit` → `GetDisplayInfo`) or by looking up an offline table. Check `issecretvalue()`-style guards (12.x secret API) before storing. Inside dungeons the identity may be secret, so fall back to the name or an offline table.
- Shard detection cannot rely on CLEU. Use UNIT_SPELLCAST_CHANNEL_START/STOP for Drain Soul (spellIDs) together with the target's death (UNIT_HEALTH / target-changed) and the shard item count/BAG_UPDATE delta. This is outside this note's scope, but it constrains the capture design.

### Gaps
- [unknown] Whether creature GUID/ID is secret in the **open world** on Forever. Untested by sources.
- [unknown] Whether `PlayerModel:GetDisplayInfo()` after `SetUnit("target")` returns a usable creature display ID on Forever.

## 4. Offline data sources (npcID → displayID, type/family, classification, zone)

### Takeaway
wago.tools carries Forever builds as product **`wowf`** (latest 1.60.1.70938, 2026-11-13) with CSV DB2 exports. The client-side **Creature** DB2 holds only ~189 rows (pets/critters), though. Mapping every npcID to a displayID, classification and zone needs server-side data scraped from datamining sites such as Wowhead (which has /forever/ NPC pages), or capturing at runtime. A ready-made addon, "Creature Display DB", claims Forever support but is All Rights Reserved.

### Cited Findings
- [confirmed] The wago.tools builds API lists product `wowf` with versions 1.60.1.70938 (2026-11-13), 70888 (11-10), 70862 (11-07) and 70840 (11-07). `wow_classic_era` is separate (1.15.9.70576). — [wago.tools/api/builds](https://wago.tools/api/builds)
- [confirmed] DB2 CSV export works for the Forever build: `https://wago.tools/db2/<Table>/csv?build=1.60.1.70938`. Sizes: Creature 11,970 bytes / **189 data rows**; CreatureDisplayInfo 1.15 MB / **14,227 rows**; CreatureType 13 rows; CreatureFamily 27 rows. — [wago.tools Creature CSV](https://wago.tools/db2/Creature/csv?build=1.60.1.70938), [CreatureDisplayInfo CSV](https://wago.tools/db2/CreatureDisplayInfo/csv?build=1.60.1.70938)
- [confirmed] Creature DB2 columns: ID, Name_lang, NameAlt_lang, Title_lang, TitleAlt_lang, **Classification, CreatureType, CreatureFamily**, StartAnimState, **DisplayID_0..3, DisplayProbability_0..3**, AlwaysItem_0..2. The rows are pets/critters, e.g. 2671 Mechanical Squirrel and 7380 Siamese, not general mobs. — [wago.tools Creature CSV](https://wago.tools/db2/Creature/csv?build=1.60.1.70938)
- [confirmed] CreatureDisplayInfo columns include ID, ModelID, SoundID, SizeClass, CreatureModelScale, CreatureModelAlpha, BloodID, ExtendedDisplayInfoID, NPCSoundID, ParticleColorID, PortraitCreatureDisplayInfoID, PortraitTextureFileDataID, Flags, Gender and more. — [wago.tools CreatureDisplayInfo CSV](https://wago.tools/db2/CreatureDisplayInfo/csv?build=1.60.1.70938)
- [confirmed] Wowhead has Forever NPC pages, e.g. "Young Wolf - NPC - WoW Forever", npc=299, Elwynn Forest, with a "View in 3D" link. Plain curl was refused (HTTP 403), so scraping is gated. — [wowhead forever npc=299](https://www.wowhead.com/forever/npc=299/diseased-young-wolf)
- [reported] Warcraft Wiki says display IDs are in Wowhead NPC page source and on wow.tools/db/creatures.php (datamined). — summarized from [warcraft.wiki.gg CreatureDisplayID](https://warcraft.wiki.gg/wiki/CreatureDisplayID)
- [reported] The CurseForge addon "Creature Display DB" supports Retail and **Forever**, uses wago data only, exposes `GetDisplayIdByNpcId`, `GetDisplayIdsByNpcId`, `GetNpcIdByName`, `GetCreatureDisplayDataById` and others, was last updated 2026-10-13 (v2.1.0), and is licensed **All Rights Reserved**. — [CurseForge Creature Display DB](https://www.curseforge.com/wow/addons/creature-display-db)
- [reported] "Creatures & NPCs Database" (CurseForge) exports DB2 tables per build/locale for multiple flavors. — [CurseForge creatures-npcs-database](https://www.curseforge.com/wow/addons/creatures-npcs-database) (search snippet only)

### Inferences
- The client Creature DB2 is useless for general mobs, so wago.tools alone cannot produce npc→display mappings. Their source must be the server's creature query, which the client caches and which Wowhead datamines. Options are: (1) capture at drain time (most robust, zero licensing burden); (2) optionally depend on Creature Display DB as an optional dependency rather than copying its data, since it is ARR; (3) scrape Wowhead, which is legally and ethically dubious (its ToS bars scraping, though this was not verified here) and fragile.
- Size: CreatureDisplayInfo is 14k display IDs; vanilla-content mobs are a few thousand npcIDs. A `[npcID]=displayID` Lua table of about 5–10k entries is roughly 100–200 KB. That is acceptable, but it is unnecessary if displays are captured at runtime.

### Gaps
- [unknown] wago.tools / Wowhead terms on redistributing extracted data in an addon. Not fetched (wago.tools/db2 HTML returned 403 to WebFetch).
- [unknown] Whether a Forever-specific Wowhead NPC page exposes the displayId, classification and level in its source. The fetch summary showed none.

## 5. C_CreatureInfo type/family IDs and icons

### Takeaway
Available in Forever. GetCreatureTypeIDs/GetCreatureTypeInfo return `{id, name}`, without an icon. GetCreatureFamilyIDs/GetCreatureFamilyInfo return `{id, name, iconFile}`. Forever's family list includes non-vanilla additions such as Felguard, Incubus, Core Hound and Fox.

### Cited Findings
- [confirmed] Struct fields: `CreatureTypeInfo {id, name}`, `CreatureFamilyInfo {id, name, iconFile}`. — `CreatureInfoDocumentation.lua`, [wow-ui-source forever](https://github.com/Gethe/wow-ui-source/tree/forever)
- [confirmed] CreatureType rows (wowf 1.60.1.70938): 1 Beast, 2 Dragonkin, 3 Demon, 4 Elemental, 5 Giant, 6 Undead, 7 Humanoid, 8 Critter, 9 Mechanical, 10 Not specified, 11 Totem, 12 Non-combat Pet, 14 Wild Pet. — [wago.tools CreatureType CSV](https://wago.tools/db2/CreatureType/csv?build=1.60.1.70938)
- [confirmed] CreatureFamily rows (with IconFileID column): 1 Wolf (icon 132203), 2 Cat (132185), 3 Spider, 4 Bear, 5 Boar, 6 Crocolisk, 7 Carrion Bird, 8 Crab, 9 Gorilla, 11 Raptor, 12 Tallstrider, 15 Felhunter, 16 Voidwalker, 17 Succubus, 19 Doomguard, 20 Scorpid, 21 Turtle, 23 Imp, 24 Bat, 25 Hyena, 26 Bird of Prey, 27 Wind Serpent, 28 Remote Control, **302 Incubus, 310 Felguard, 312 Core Hound, 319 Fox**. — [wago.tools CreatureFamily CSV](https://wago.tools/db2/CreatureFamily/csv?build=1.60.1.70938)

### Inferences
- Card "type" badges: creature types have no API icon, so ship your own icon mapping, e.g. atlas or spell icons per type. Family icons come from `iconFile`.
- `UnitClassification` (normal/elite/rare/rareelite/worldboss/trivial/minus) gives the card rarity at capture time. It is not secret-flagged in the Forever docs.

### Gaps
- [unknown] Whether 302/310/312/319 are player-pet families in Forever, e.g. a warlock Felguard/Incubus or hunter Core Hound/Fox. They are in the data, but their gameplay role was not verified.

## 6. Game mechanics: when Drain Soul / Shadowburn yield a Soul Shard (Classic vs Forever)

### Takeaway
Classic Era: a shard is granted only when the target dies while drained (Drain Soul) or within 5 s (Shadowburn), **and** yields XP or honor. Forever changes this. Drain Soul can grant shards on damage against "non-trivial" targets, a reduced chance as of the 2026-11-02 notes, and a qualifying kill guarantees one. Shadowburn's window is 8 s. Soul bags remain, and a Forever "Felblood Soul Bag" is reported.

### Cited Findings
- [confirmed-wiki] Classic Drain Soul: 4 ranks, Rank 1 = 1120, Rank 4 = 11675. The shard is granted only if the target dies while drained and yields experience or honor. Patch 1.5.0 made honorable kills grant shards. — [warcraft.wiki.gg Drain Soul](https://warcraft.wiki.gg/wiki/Drain_Soul). The intermediate rank IDs 8288/8289 are from the task brief and were not verified on that page.
- [reported] Forever Drain Soul (wiki): level 10, 4 ranks, 85 Shadow over 15 s. "Non-trivial targets damaged... have a chance to grant a Soul Shard, and non-trivial targets it kills always grant one." Related talents: Fel Concentration, Improved Drains, Pandemic, Soul Harvest, Soul Siphon. — [warcraft.wiki.gg Drain Soul](https://warcraft.wiki.gg/wiki/Drain_Soul)
- [reported] The spellbook text extracted from Forever beta build 1.60.1.69893 reads: "Damaging a non-trivial target can generate a Soul Shard; a qualifying Drain Soul kill guarantees one." — [foreverwisp.com](https://www.foreverwisp.com/guides/wow-forever-warlock-beta-levelling-changes)
- [reported] The Blizzard "WoW Forever Beta Development Notes – Updated November 2", as quoted by a forum poster on 2026-11-03, say: "Drain Soul: This chance for this spell to grant a Soul Shard on dealing damage (not a kill) has been reduced significantly." — [Blizzard forums: Latest warlock changes](https://us.forums.blizzard.com/en/wow/t/latest-warlock-changes/2399400)
- [confirmed-wiki] Classic Shadowburn: Rank 1 = 17877, Rank 6 = 18871, costs a Soul Shard. "If the target dies within 5 sec... and yields experience or honor, the caster gains a Soul Shard" (1.10.0 restricted this to XP/honor targets). — [warcraft.wiki.gg Shadowburn](https://warcraft.wiki.gg/wiki/Shadowburn)
- [reported] Forever Shadowburn: same IDs 17877/18871, 30 yd, 65–74 Shadow. A non-trivial target dying within **8 s** grants a shard. Improved by "Shadow and Flame". — [warcraft.wiki.gg Shadowburn](https://warcraft.wiki.gg/wiki/Shadowburn)
- [reported] Player threads during the beta asked for shards to stack and for the bag "tax" to be removed. None confirms a change. — [Blizzard forums: Keep the Mechanic, Drop the Bag Tax](https://us.forums.blizzard.com/en/wow/t/warlock-soul-shards-keep-the-mechanic-drop-the-bag-tax/2368623)
- [reported] A Forever database lists a "Felblood Soul Bag" (30 slots, ilvl 65, req. 60, crafted) alongside the Core Felcloth Bag (28 slots). — [metaforge WoW Forever containers](https://metaforge.app/wow-forever/database/containers?quality=4). The item ID 279271 was **not** confirmed.

### Inferences
- In Forever a shard no longer implies a kill: damage-only shards exist. If cards should represent a "soul drained", trigger on a Drain Soul kill (the channel is active when the target dies, and the target was non-trivial / not grey), or on a shard-count increase within the Drain Soul/Shadowburn window, and dedupe by GUID. "Non-trivial" probably corresponds to XP-yielding (non-grey) mobs, but that is not confirmed.

### Gaps
- [unknown] An official Blizzard source for whether shards stack or soul bags changed in Forever, and the Felblood Soul Bag item ID.
- [unknown] Forever spell IDs for Drain Soul ranks 2–3 and Shadowburn ranks 2–5, and whether Forever added new spell IDs. Pull these from the wago.tools `wowf` SpellName table.
- [unknown] Exact definition of "non-trivial" in Forever. Probably not grey-level (the Nov 13 notes are said to include "zero XP for trivial dungeon mobs" per [timesaver.gg](https://timesaver.gg/blog/wow-forever-launch-patch-notes), which was not fetched).
