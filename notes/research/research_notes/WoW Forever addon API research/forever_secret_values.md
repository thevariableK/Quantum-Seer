# WoW Forever (1.60.1) addon API restrictions and secret values: what a Warlock shard-source addon can read

Labels used below: **[CONFIRMED]** = from Blizzard's generated API docs on the `forever` branch of the UI source mirror, or a quoted Blizzard statement. **[REPORTED]** = community, wiki-paraphrase, or empirical findings from the local addons (not Blizzard-confirmed). **[UNKNOWN]** = not settled. **[MIDNIGHT-ONLY]** = documented for retail Midnight (12.x), not checked on Forever.

Primary local source: `Gethe/wow-ui-source` branch `forever`, HEAD `bebcb6bf309cf41b41bd74f8ca651e6a105b2676` ("1.60.1 (70938)", 2026-11-13). The previous commits are 70888 (2026-11-10) and 70862 (2026-11-07), and tag `1.60.1` exists. Files are under `Interface/AddOns/Blizzard_APIDocumentationGenerated/`. Cited below as "[forever API docs: <file>](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)". Note: the target build ~70450 is older than this HEAD. WarlockToolsForever pins source review to commit `943764493e…` (build 70338). Annotations could differ slightly between 70338/70450 and 70938.

## What is WoW: Forever?

### Takeaway
World of Warcraft: Forever is Blizzard's "Classic+" (level-60 cap, new content), announced at BlizzCon on 2026-09-12 and launched worldwide on 2026-11-04. It is not a Classic-engine client. It is a Mainline-family client (internal game type "Camelot") that shares Midnight's UI architecture and APIs, including the full secret-values and addon-restriction system.

### Cited Findings
- [CONFIRMED] Announced at BlizzCon on 2026-09-12. Worldwide launch on 2026-11-04. Included with a WoW subscription. Level 60 cap. — [Blizzard News](https://news.blizzard.com/en-us/article/24302093/carve-a-new-path-with-world-of-warcraft-forever); [Blizzard Watch](https://blizzardwatch.com/2026/09/12/world-warcraft-forever-long-speculated-classic-plus-unveiled-will-launch-worldwide-november-4-2026/); [TechSpot](https://www.techspot.com/news/113832-world-warcraft-forever-launches-november-4-new-race.html)
- [REPORTED] Beta access began around 2026-09-16/17. Raids are said to unlock on 2026-12-09. — [Hearthproof](https://hearthproof.com/systems/addons/); [Shacknews](https://shacknews.com/article/150712/world-of-warcraft-forever-classic-release-date-beta-sign-up)
- [CONFIRMED, Blizzard statement quoted on the wiki, 2026-09-15] "WoW Forever shares Mainline WoW's UI architecture, including the vast majority of APIs available in 12.1.5." Also "The addon disarmament changes introduced in Midnight, including secrets, will also be active in Forever," and "you can expect similar UI changes in the future to be introduced to both Standard and Forever." Forever and Midnight are two game types in the Mainline code family, internally "Camelot" and "Standard". — [warcraft.wiki.gg Patch 1.60.1/API changes/notes](https://warcraft.wiki.gg/wiki/Patch_1.60.1/API_changes/notes)
- [REPORTED] Wowhead reported Blizzard's confirmation, given via the WoW UI Discord, on 2026-09-16. Wowhead says WeakAuras "will not be returning for Forever". — [Warperia](https://warperia.com/wow-forever-inherits-midnights-addon-changes/)
- [CONFIRMED] TOC `## Interface: 16001`. Build 70291 (Beta 4) accepts "Forever" as an alias for "Camelot" in TOC suffixes. — [wiki notes](https://warcraft.wiki.gg/wiki/Patch_1.60.1/API_changes/notes); local `reference/WarlockToolsForever/WarlockToolsForever.toc:1` (`## Interface: 16001, 11509`)
- [REPORTED] The live build is 1.60.1.70888 (Nov 11). `WOW_PROJECT_ID` was 1 on beta build 69913 and 18 (`WOW_PROJECT_CAMELOT`) on 70338, so code should probe for APIs instead of branching on the project ID. — [Hearthproof](https://hearthproof.com/systems/addons/); [wowforeverguides](https://wowforeverguides.com/addons/developers)
- [CONFIRMED] Build 70425 (Beta 5): creature, vehicle, pet and game-object GUIDs have their server-ID and spawn-ID portions "replaced with obfuscated values to prevent comparison to determine layer". "They remain stable and unique within your own session, and the GUID string format is otherwise unchanged." — [wiki notes](https://warcraft.wiki.gg/wiki/Patch_1.60.1/API_changes/notes); [API UnitGUID](https://warcraft.wiki.gg/wiki/API_UnitGUID)

### Inferences
- The retail Midnight documentation (warcraft.wiki.gg Secret Values, Blizzard_APIDocumentationGenerated) largely applies directly. The `forever` branch docs are the authoritative source.
- Because GUIDs are obfuscated per session, a mob GUID is only usable within one login session. Shard-to-mob records saved across sessions should store the mob's name, NPC type and zone, not rely on the GUID.

### Gaps
- No Blizzard patch note lists Forever-specific secret behaviour. Blizzard reportedly has not published a Forever-specific list of secret values ([Hearthproof](https://hearthproof.com/systems/addons/)).
- It is unclear whether the creature-ID (NPC ID) part of the GUID is also obfuscated. The note names only the server and spawn IDs, which implies the NPC ID stays readable, but this is not verified.

## Which unit APIs return secret values, and when?

### Takeaway
Per the Forever docs, `UnitGUID`, `UnitCreatureType`, `UnitCreatureFamily`, `UnitName` (via a name-specific predicate) and `UnitTokenFromGUID` are secret **when the unit is not player-controlled and not in your party/raid**, which covers every hostile NPC target. `UnitHealth` is always tagged `SecretReturns`, and `UnitHealthMax` is secret for non-player-controlled units. `UnitLevel`, `UnitClassification`, `UnitIsDead` and `UnitExists` carry no secret-return tags. The docs do not say whether the identity predicates apply only under an active restriction (combat, encounter, etc.) or always. Empirically, WarlockToolsForever reports that target name and `UnitGUID` are readable in combat in the open world, while nameplate GUIDs are sometimes secret. Design for both outcomes.

### Cited Findings
Doc annotations on build 70938, from `UnitDocumentation.lua` — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated):

| API | Secret-return annotation (Forever docs) |
|---|---|
| `UnitGUID(unit)` | `SecretWhenUnitIdentityRestricted`. Arg type `UnitTokenPvPRestrictedForAddOns`. Returns WOWGUID (nilable). |
| `UnitName(unit)` | `SecretWhenUnitNameIdentityRestricted`. Returns `unitName`, `unitServer`. |
| `UnitNameUnmodified`, `UnitFullName` | `SecretWhenUnitIdentityRestricted` |
| `UnitLevel` | none (only `SecretArguments = "AllowedWhenUntainted"`) |
| `UnitEffectiveLevel` | none |
| `UnitClassification` | none |
| `UnitCreatureType` | `SecretWhenUnitIdentityRestricted` |
| `UnitCreatureFamily` | `SecretWhenUnitIdentityRestricted` |
| `UnitClass`, `UnitRace` | `SecretWhenUnitIdentityRestricted` |
| `UnitIsDead`, `UnitIsDeadOrGhost` | none |
| `UnitExists` | none |
| `UnitHealth` | `SecretReturns = true` (always secret-tagged). Arg type `UnitTokenPvPRestrictedForAddOns`. |
| `UnitHealthMax` | `SecretWhenUnitHealthMaxRestricted` |
| `UnitHealthPercent` | `SecretReturns`, `SecretWhenCurveSecret` |
| `UnitIsUnit` | `RequiresComparableUnitTokens`, `SecretWhenUnitComparisonRestricted` |
| `UnitTokenFromGUID` | `SecretWhenUnitIdentityRestricted` |
| `UnitCanAttack`, `UnitIsEnemy`, `UnitReaction`, `UnitIsTapDenied`, `UnitAffectingCombat`, `UnitIsPlayer` | none in the function header (per-return flags not exhaustively checked) |

- [CONFIRMED] Predicate definitions (`SecretPredicatesDocumentation.lua`, forever branch):
  - `SecretWhenUnitIdentityRestricted`: "Guarded APIs and events produce secret values when the unit isn't player-controlled or in the party/raid. For compound tokens (eg. 'boss1target'), results are secret if any unit in the chain fails this."
  - `SecretWhenUnitNameIdentityRestricted`: "…under regular unit identity secrecy rules, except in PvP when the queried unit is a player."
  - `SecretWhenUnitHealthMaxRestricted`: "…when the unit isn't player-controlled."
  - `SecretWhenUnitComparisonRestricted`: "…Comparisons involving compound unit tokens (eg. 'boss1target') are always secret. This restriction only applies when the player is on an addon-restricted map."
  - `SecretWhenInCombat`: "…when combat addon restrictions are in effect."
  - `SecretOnRestrictedMaps`: "…when the player is on an addon-restricted map such as a dungeon or raid."
  - `RequiresUnitIdentityAccess`: "This does not raise a blocked action error - instead, protected APIs will return no values."

  — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)
- [CONFIRMED] `SecretUtil.HasSecretRestrictions()`: "Returns true if this client build has secret value restrictions enabled. If false, all APIs that are tagged as potentially returning secrets will never do so." The same system also exposes `CanCompareUnitTokens`, `GetSpellCastSecrecy`, `GetSpellAuraSecrecy`, `GetSpellCooldownSecrecy`, `ShouldAurasBeSecret`, `ShouldCooldownsBeSecret`, `ShouldSpellAuraBeSecret` and `ShouldActionCooldownBeSecret`. — `SecretPredicateAPIDocumentation.lua`, [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)
- [REPORTED, wiki on Midnight] `UnitName` "returns secret values if querying a non-player or pet unit while in combat". This implies identity secrecy is gated on combat restrictions, not permanent. — [warcraft.wiki.gg Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)
- [REPORTED, empirical Forever] WarlockToolsForever's guidelines say "cast events, target name, and `UnitGUID` are readable in combat" for the player's casts on hostile mobs. They also say that target and focus GUIDs are readable, while nameplate GUIDs were sometimes secret (player report: "a dotted target you tabbed away from stayed dim beside its own plate"). A mob seen only via its nameplate stays unbound "until you target or focus it, or combat ends". — local `reference/WarlockToolsForever/AGENTS.md` lines ~386-424
- [REPORTED, empirical Forever] The same guidelines say enemy health is secret in combat. They pass it only to `StatusBar:SetMinMaxValues/SetValue`, `AbbreviateNumbers`, and `UnitHealthPercent(unit, true, CurveConstants.ScaleTo100)`. — local `reference/WarlockToolsForever/AGENTS.md` ~line 473
- [REPORTED] Player `UnitHealth` returned a secret number on beta build 69913. `type()` reports "number" but arithmetic errors. `UnitCanAttack` and `UnitExists` "may be secret booleans" (a "Warcraft Forever report"). — [wowforeverguides](https://wowforeverguides.com/addons/developers). This contradicts the docs, which show no secret tag on `UnitExists`. Defensive `issecretvalue` checks are still advisable.
- [REPORTED, empirical Forever] `UnitAffectingCombat("player")` is treated as possibly secret: "true or secret". — local `reference/WarlockToolsForever/Core.lua:42-56`
- [REPORTED, empirical Forever] ShardGrid wraps `UnitGUID("NPC")` (the trade partner) in `pcall` plus an `IsSecret` check. — local `reference/shard-grid/ShardGrid.lua:2628-2632`

### Inferences
- The doc predicates are worded unconditionally ("when the unit isn't player-controlled"). The actual trigger is probably an active addon restriction (the wiki says "while in combat", and `HasSecretRestrictions` is the global gate). Empirical reports that the target GUID is readable in combat in the open world suggest identity secrecy may not trigger for `target` in open-world combat on Forever, or may trigger only for some tokens or maps. This is the single most important thing to verify in the client: `/dump issecretvalue(UnitGUID("target"))`, `issecretvalue(UnitName("target"))` and `issecretvalue(UnitCreatureType("target"))` on a hostile NPC in combat, in the open world and in a dungeon.
- Values tagged with no secret predicate (`UnitLevel`, `UnitClassification`, `UnitIsDead`, `UnitExists`) should be readable for a hostile target at any time. That gives a fallback fingerprint (level plus classification) when the name or GUID is secret.
- Safe design: snapshot the target's GUID, name, level, classification and creature type **at cast start** (or out of combat when you first target the mob). Store them raw even if secret, and resolve them later.

### Gaps
- The exact runtime gating of `SecretWhenUnitIdentityRestricted` on Forever (combat only? restricted maps only? specific tokens?) is not documented. Local empirical evidence and the doc wording disagree.
- The meaning of `UnitTokenPvPRestrictedForAddOns` (the arg type on `UnitGUID` and `UnitHealth`) is not documented in the files checked. It presumably restricts some tokens in PvP.

## Spellcast event payloads: UNIT_SPELLCAST_CHANNEL_START/STOP and UNIT_SPELLCAST_SUCCEEDED

### Takeaway
All three events carry `SecretWhenUnitSpellCastRestricted`. Their payloads (castGUID, spellID, and interruptedBy on STOP) are secret only when the unit is not the player or pet, **unless the individual spell is flagged always or never secret**. For the player's own Drain Soul they should normally be readable, but WarlockToolsForever empirically saw a secret spellID on `UNIT_SPELLCAST_CHANNEL_STOP`. `C_Spell.GetSpellName` has no secret-return tag.

### Cited Findings
- [CONFIRMED] Payloads, from `UnitDocumentation.lua` (forever) — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated):
  - `UNIT_SPELLCAST_CHANNEL_START`: (`unitTarget`, `castGUID`, `spellID`, `castBarID` [NeverSecret]). Event tagged `SecretWhenUnitSpellCastRestricted`.
  - `UNIT_SPELLCAST_CHANNEL_STOP`: (`unitTarget`, `castGUID`, `spellID`, `interruptedBy` WOWGUID, `castBarID` [NeverSecret]). Same tag.
  - `UNIT_SPELLCAST_SUCCEEDED`: (`unitTarget`, `castGUID`, `spellID`, `castBarID` [NeverSecret]). Same tag.
- [CONFIRMED] Predicate text: "Guarded APIs and events produce secret values if the unit being queried for cast information is not the player or their pet. Individual spells may be flagged as never or always secret, which takes priority." The per-spell flag is queryable via `GetSpellCastSecrecy(spellIdentifier) -> SecrecyLevel`. — `SecretPredicatesDocumentation.lua` / `SecretPredicateAPIDocumentation.lua`
- [CONFIRMED] `C_Spell.GetSpellName(spellIdentifier)` has `MayReturnNothing`, `SecretArguments = "AllowedWhenTainted"`, and no secret-return tag. Note that passing a secret spellID is "allowed" but would presumably produce a secret result. — `SpellDocumentation.lua` (forever)
- [REPORTED, empirical Forever] WarlockToolsForever's `Shards.lua` sets `state.secretCasts = true` when a spell-cast spellID is secret and then refuses the creation window. Project context says this was observed on `UNIT_SPELLCAST_CHANNEL_STOP`. — local `reference/WarlockToolsForever/Shards.lua:179-190`
- [REPORTED, empirical Forever] WarlockToolsForever uses `UNIT_SPELLCAST_SENT` (target name) and then `UNIT_SPELLCAST_SUCCEEDED` to bind a cast to a mob GUID in combat. `Announce.lua` guards `castGUID` with `IsSecret` before using it as a table key. — local `reference/WarlockToolsForever/AGENTS.md` ~386-391; `Announce.lua:205,255,271`

### Inferences
- For Drain Soul and Shadowburn tracking, identify the spell from `UNIT_SPELLCAST_CHANNEL_START` or `UNIT_SPELLCAST_SUCCEEDED` (unit `"player"`), and fall back to `UnitChannelInfo("player")` or a "currently channeling Drain Soul" flag kept from START if STOP's spellID is secret. Never use a possibly-secret castGUID as a table key.
- `GetSpellCastSecrecy(<Drain Soul ID>)` in the client would tell whether Drain Soul is flagged always-secret.

### Gaps
- The cause of the secret spellID on CHANNEL_STOP for the player is unknown (a per-spell flag? a STOP-specific behaviour?). It is not reproduced in the docs.
- The `UNIT_SPELLCAST_SENT` payload secrecy (its target-name arg) was not checked in the docs.

## Is COMBAT_LOG_EVENT_UNFILTERED available? Replacements?

### Takeaway
No, not for addons on Forever. `COMBAT_LOG_EVENT_UNFILTERED` is `HasRestrictions` in the docs, and registering it from addon code raises `ADDON_ACTION_BLOCKED` (empirical). `GetCurrentEventInfo` exists only in internal or secure namespaces. The best replacements for kill attribution are the events **`PARTY_KILL` (attackerGUID, targetGUID)** and **`UNIT_DIED` (unitGUID)**, both present on Forever and tagged `SecretWhenUnitIdentityRestricted`, plus `C_DamageMeter` (which includes a `Deaths` type, with values secret in combat).

### Cited Findings
- [CONFIRMED] `COMBAT_LOG_EVENT_UNFILTERED` and `COMBAT_LOG_EVENT` are tagged `HasRestrictions = true`. `C_CombatLog.IsCombatLogRestricted() -> restricted` exists. `GetCurrentEventInfo` appears only in `C_CombatLogInternal` and `C_CombatLogSecure` (`CombatLogInternalDocumentation.lua`, `CombatLogSecureDocumentation.lua`), not in the public `CombatLogDocumentation.lua`. — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)
- [REPORTED, empirical Forever] "Never register `COMBAT_LOG_EVENT_UNFILTERED`: the client refuses it and raises `ADDON_ACTION_BLOCKED`". "On Forever the combat log is restricted at all times". WarlockToolsForever's code paths check `C_CombatLog.IsCombatLogRestricted` before using the combat log. — local `reference/WarlockToolsForever/AGENTS.md` lines ~400, ~575-580, ~1102
- [REPORTED] "Warcraft Forever reports that `CombatLogGetCurrentEventInfo` is unavailable to addons on Forever". Registering CLEU "throws an error on Midnight". — [wowforeverguides](https://wowforeverguides.com/addons/developers). Combat log "Not readable by addons at all" (BETA). — [Hearthproof](https://hearthproof.com/systems/addons/)
- [CONFIRMED] `PARTY_KILL` payload (`attackerGUID`, `targetGUID`) and `UNIT_DIED` payload (`unitGUID`) are both tagged `SecretWhenUnitIdentityRestricted` and `SynchronousEvent`. — `UnitDocumentation.lua` (forever)
- [CONFIRMED] The `C_DamageMeter` namespace exists. `DamageMeterType` enum values: DamageDone 0, Dps 1, HealingDone 2, Hps 3, Absorbs 4, Interrupts 5, Dispels 6, DamageTaken 7, AvoidableDamageTaken 8, Deaths 9, EnemyDamageTaken 10. Its functions are tagged `SecretWhenInCombat`. Events: `DAMAGE_METER_COMBAT_SESSION_UPDATED`, `DAMAGE_METER_CURRENT_SESSION_UPDATED`, `DAMAGE_METER_RESET`. Source fields include `sourceGUID` (ConditionalSecret), `name` (ConditionalSecret), and `classification` (NeverSecret). — `DamageMeterDocumentation.lua` / `DamageMeterConstantsDocumentation.lua` (forever)
- [REPORTED] Blizzard's built-in Damage Meter and Cooldown Manager ship on Forever. — [Warperia](https://warperia.com/wow-forever-inherits-midnights-addon-changes/); [wowforeverguides](https://wowforeverguides.com/addons/developers)

### Inferences
- Shard-source design: on `PARTY_KILL`, if `attackerGUID` is the player's GUID and both GUIDs are readable, that is a kill credit. Match `targetGUID` against the GUID snapshotted at Drain Soul or Shadowburn cast start, then look up the snapshotted name, level and type. If the payloads are secret, fall back to: the Drain Soul channel ended + `UnitIsDead("target")` true (no secret tag) + a new shard appeared in the bags (`BAG_UPDATE`) within the creation window → attribute to the snapshotted target.
- `C_DamageMeter` Deaths is player-focused and secret in combat, so it is not useful for mob kill attribution.

### Gaps
- Whether `PARTY_KILL` or `UNIT_DIED` fire with readable GUIDs for hostile NPCs in open-world combat on Forever is untested. By the predicate they would be secret for NPC targets when identity restriction is active.
- Whether `UNIT_DESTROYED` exists on Forever was not checked.

## What can you do with a secret value, and when does it become readable?

### Takeaway
Tainted (addon) code may store, pass, concatenate and `string.format` secrets and hand them to secret-accepting widget setters (`FontString:SetText`, `StatusBar:SetValue`). It may not compare them, do arithmetic, take `#`, use them as a table key, index them, call them, or test a secret boolean (each is an immediate Lua error). A secret value never becomes readable later. You must re-query the API after the restriction lifts (e.g., `PLAYER_REGEN_ENABLED` or `ADDON_RESTRICTION_STATE_CHANGED` → Inactive).

### Cited Findings
- [CONFIRMED-ish, wiki on Midnight] Allowed: storing in variables, upvalues or table values; passing to Lua functions; concatenating string or number secrets; `string.concat`/`format`/`join`; `type()` returns the real type; boolean tests on non-boolean secrets; passing to native APIs that accept secrets, like `StatusBar:SetValue`. Disallowed (immediate Lua error): arithmetic, comparisons, boolean tests on secret booleans, `#`, a secret as a table key, indexed access or assignment, calling. "When execution is not tainted secret values are effectively equivalent to regular values." — [warcraft.wiki.gg Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)
- [CONFIRMED-ish, wiki on Midnight] `FontString:SetText(secret)` applies a "Text" secret aspect, so `GetText` then returns secrets. Aspects clear only via `SetToDefaults`. Objects can become secret-anchored, and that propagates to children. Curves, ColorCurves and Durations let you display or compute with secrets without inspecting them. — [warcraft.wiki.gg Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)
- [CONFIRMED] Global helpers exist on Forever:
  - `issecretvalue`, `issecrettable`, `canaccessvalue`, `canaccesstable`, `hasanysecretvalues`, `scrubsecretvalues`, `secretwrap`, `mapvalues`.
  - `canaccesssecrets`: "Returns true if the immediate calling function has appropriate permissions to access or operate on secret values."
  - `dropsecretaccess`: "Removes the ability for the immediate calling function to access secret values."

  — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)
- [REPORTED, empirical Forever] A ShardGrid changelog entry: "attempt to compare a secret number value, execution tainted by ShardGrid" errors in Blizzard nameplates, caused by using Blizzard Settings controls. Tainting Blizzard code makes Blizzard's own secret comparisons fail. — local `reference/shard-grid/CHANGELOG.md:204`
- [REPORTED] "Combat events are in a black box; addons can change the size or shape of the box" (attributed to Blizzard). — [Hearthproof](https://hearthproof.com/systems/addons/)
- [REPORTED] A blocked protected call can fail without a Lua error, and `pcall` does not clear it. — [wowforeverguides](https://wowforeverguides.com/addons/developers)

### Inferences
- Storing a secret GUID in a table **value** (not a key) for later is legal, but it stays secret forever: you can never compare it to a later readable GUID. Re-reading the unit after combat only works if the unit token still points to the same mob, which is not true of a dead mob you tabbed away from.
- For display only (e.g., "Shard from: <name>"), a secret name can go straight to `FontString:SetText`. Persisting it in SavedVariables is pointless or unsafe: there is no evidence secrets serialise, and this is untested.
- The workaround is to capture identity while readable: on `PLAYER_TARGET_CHANGED` and at cast start, read GUID, name, level and type. If readable, cache by GUID; if secret, cache level and classification (not tagged secret) plus timestamps.

### Gaps
- Whether secret values written to SavedVariables are dropped, scrubbed or error is not documented in the sources found.
- Whether `string.format` of a secret produces a secret string that `SetText` accepts on Forever (not just Midnight) was not separately verified. WarlockToolsForever deliberately avoids formatting secret strings.

## C_RestrictedActions API surface and events

### Takeaway
`C_RestrictedActions` provides `IsAddOnRestrictionActive(type)`, `GetAddOnRestrictionState(type)`, `InCombatLockdown()` and `CheckAllowProtectedFunctions(object, silent)`. Its events are `ADDON_RESTRICTION_STATE_CHANGED(type, state)`, `ADDON_ACTION_BLOCKED`, `ADDON_ACTION_FORBIDDEN`, `MACRO_ACTION_BLOCKED` and `MACRO_ACTION_FORBIDDEN`. There are six restriction types, including Chat.

### Cited Findings
- [CONFIRMED] Functions (`RestrictedActionsDocumentation.lua`, forever):
  - `CheckAllowProtectedFunctions(object, silent=false) -> protectedFunctionsAllowed`. `silent`: "If true, don't signal blocked action errors…"
  - `GetAddOnRestrictionState(type) -> AddOnRestrictionState`
  - `InCombatLockdown() -> inCombatLockdown`
  - `IsAddOnRestrictionActive(type) -> active`

  Events: `ADDON_ACTION_BLOCKED(isTainted, function)`, `ADDON_ACTION_FORBIDDEN(isTainted, function)`, `ADDON_RESTRICTION_STATE_CHANGED(type, state)`, `MACRO_ACTION_BLOCKED(function)`, `MACRO_ACTION_FORBIDDEN(function)`. — [forever API docs](https://github.com/Gethe/wow-ui-source/tree/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated)
- [CONFIRMED] `Enum.AddOnRestrictionType`:
  - Combat 0: "The player is actively affecting combat."
  - Encounter 1: "…actively participating in an instance encounter."
  - ChallengeMode 2
  - PvPMatch 3
  - Map 4: "…on a map that applies addon restrictions."
  - Chat 5: "…addon chat communications are restricted."

  `Enum.AddOnRestrictionState`: Inactive 0, Activating 1 ("during the dispatch of ADDON_RESTRICTION_STATE_CHANGED… won't be enforced until event dispatch has completed"), Active 2. — `RestrictedActionsConstantsDocumentation.lua` (forever)
- [REPORTED, empirical Forever] `DeleteCursorItem` from a key press raised `ADDON_ACTION_BLOCKED` in a dungeon with DBM. WarlockToolsForever treats Combat, Encounter, ChallengeMode, PvPMatch and Map restrictions as blocking item deletion. — local `reference/WarlockToolsForever/AGENTS.md` ~1090-1100; `Core.lua:42-68`
- [REPORTED, wiki on Midnight] The CVars `addonCombatRestrictionsForced` and `addonEncounterRestrictionsForced` simulate restriction states (not saved across restarts). This is useful for testing. — [warcraft.wiki.gg Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)

### Inferences
- The `Activating` state lets an addon snapshot readable data (e.g., the target GUID and name) in the `ADDON_RESTRICTION_STATE_CHANGED` handler just before Combat restriction enforcement begins. This is a useful hook for pre-combat snapshots, but its usefulness on Forever is untested.

### Gaps
- Whether the CVar-forced restriction simulation works on Forever is untested.

## Blizzard statements and community discussion on Forever addon restrictions

### Takeaway
Blizzard's only explicit statement is that Midnight's addon disarmament, secrets included, is active on Forever. Community pain points are combo points (secret), WeakAuras not returning, combat-log meters impossible, and auras unreadable in combat.

### Cited Findings
- [CONFIRMED, quoted] "The addon disarmament changes introduced in Midnight, including secrets, will also be active in Forever." (2026-09-15) — [wiki notes](https://warcraft.wiki.gg/wiki/Patch_1.60.1/API_changes/notes)
- [CONFIRMED] Beta 4 (70291) added a `FrameStrata` secret aspect (`GetFrameStrata` can return a secret), fixed addons reading minimap rotation during encounters, blocked trial accounts from sending addon messages, and made `SendChatMessage` "EMOTE" require a hardware event outside instances. — [wiki notes](https://warcraft.wiki.gg/wiki/Patch_1.60.1/API_changes/notes)
- [REPORTED] A forum thread (2026-09-17) says combo points are a secret value on Forever. The poster's edit says Blizzard "will be adding combo points to the Personal Resource Display in a future beta build". The thread has no blue post. — [Blizzard forums](https://us.forums.blizzard.com/en/wow/t/combo-points-are-secret-values/2352853)
- [REPORTED] Auras are readable out of combat, not in combat. Threat on the current target is readable via `UnitDetailedThreatSituation`. Raid-marker index is unreadable in combat. These are beta findings. — [Hearthproof](https://hearthproof.com/systems/addons/)
- [REPORTED, empirical Forever] "every aura API refuses AddOn code in combat". The workaround is Blizzard's `AuraContainer` (`CustomAuraContainerTemplate`). — local `reference/WarlockToolsForever/AGENTS.md` ~400-404
- [REPORTED] On beta build 69913, `GetSpellInfo` and `GetItemInfo` globals were absent; use `C_Spell.*` and `C_Item.*`. — [wowforeverguides](https://wowforeverguides.com/addons/developers)
- [REPORTED] Character names have surnames on Forever. `UnitNameUnmodified("player")` returns (name, surname), as AceDB on Forever uses it. Doc constants `Surname` and `SurnameSeparatorWithLinkSeparator` appear in the generated docs. — local `reference/SoulShardForever/Libs/AceDB-3.0/AceDB-3.0.lua:277`; forever API docs grep

### Inferences
- Overall feasibility for the "which enemy did this shard come from" addon: **likely feasible in the open world**. It relies on capturing target identity at Drain Soul or Shadowburn cast time (empirically readable), the shard's appearance in the bags (not combat-restricted), and death via `UnitIsDead` (not secret-tagged) or `PARTY_KILL` when readable. In dungeons and raids (Map and Encounter restrictions), identity values may be secret, so the addon should degrade to "Unknown (secret)" plus level and classification.

### Gaps
- I found no Blizzard blue post that specifically documents unit-identity secrecy in the open world vs instances on Forever.
- I did not fetch the cited Wowhead article and Blizzard "Combat philosophy and addon disarmament in Midnight" post directly. Their content here is via secondary sources.
