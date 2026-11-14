# Shard Grid (v1.8.2) — competitor review

Source: `reference/shard-grid/` (read only, never run). Paths below are relative to that folder;
`SG:` = `ShardGrid.lua`, `ST:` = `Styles.lua`, `SK:` = `ShardGrid_Skins.lua`.

## 1. Overview

**Purpose.** A Warlock soul shard window shaped like a bag (the grid), plus everything else shards are used for:
low-shard alert, auto-delete of extra shards, a summon request queue, a soulstone tracker, Soulstone and
Healthstone "go make one" alerts, and putting a Healthstone in the trade window for you. It aims to look like
part of the default UI (it uses the bag-window art) and also has Dark and EllesmereUI skins.

**Feature list**
- Grid: soul bag slots (filled/free), plus loose shards from ordinary bags shown in orange. Shards past the
  keep limit, which will be deleted next, are red. Shards are packed together so the slot that empties is
  the one next to free space. Width is set with a drag grip, plus slot size, scale, minimum rows and
  "empty slots first". A tooltip gives the breakdown (SG:1835-1871).
- Toss-in / flash-out animations: a quadratic-bezier "throw" with a trail, a burst and a land sound (SG:1277-1670).
- Low shard alert: a movable pulsing icon with a sound (SG:682-800).
- Soulstone alert ("Soulstone ready" or "Make a Soulstone") and a "No Healthstone" alert. Both are clickable
  secure buttons (SG:802-1194).
- Auto-delete of extra shards, with a "fill the soul bag first" step (SG:231-530).
- Summon requests: watches chat for keywords and queues the players in a window. A row is a secure button
  that does /targetexact + /cast Ritual of Summoning. It sends a whisper and a group message, has an
  Invite/Suggest button, and removes players once they arrive (SG:1923-2600).
- Floating summon button that uses macro conditionals (SG:3112-3269). A "nobody is clicking" /say reminder
  (SG:2955-3109).
- Soulstone tracker: bars with timers and the caster, a report-to-raid button, and an optional announce
  (SG:3272-3903).
- Healthstone trading: puts a stone in the trade on TRADE_SHOW, adds a "Create Healthstone" secure button to
  TradeFrame, and allows each player one stone per N minutes (SG:2598-2822). A title-bar button posts
  "Healthstones! Trade me…" (SG:4831-4874).
- Options: a canvas page in Settings > AddOns, with a standalone window as fallback (SG:3905-4829).
  Minimap button (SG:5057-5137).
- `/shards debug`: a full diagnostic dump that includes a "refusal log" (SG:5259-5285).

**Slash commands** (SG:5287-5424, README:45-68): `/shards` and `/shardgrid` open the options. Subcommands:
`width|cols|w N`, `size N`, `scale N`, `style [auto|blizzard|dark]`, `anim [on|off]`, `alert N|on|off`,
`sound ID`, `lock|unlock`, `show|hide`, `summons [test]`, `stones [report]`, `minimap`,
`cog [list|grab [N]]`, `bar grab|reset|fill N|…`, `reset`, `debug`.

**SavedVariables:** `ShardGridDB`, **per character** (`.toc:7`). Defaults are at SG:38-85. Migration flag
`v11` (SG:5147). A `SUPERSEDED` table removes saved values that equal an old default text, so changes to the
default text reach players (SG:5163-5179). Nice touch. Runtime-learned keys: `noKeyListener` and `hookRefused`
(inputs the client refused), and `stoneUsedAt` (wall-clock `time()`, so the cooldown survives a reload).

**.toc** (`ShardGrid.toc`): `## Interface: 16001`, Version 1.8.2, `## LoadSavedVariablesFirst: 1`, and
`## OptionalDeps: EllesmereUI`. Load order: ShardGrid.lua, Styles.lua, ShardGrid_Skins.lua.
SG:5218-5229 notes that SavedVariables can still arrive *after* ADDON_LOADED on this client. They re-adopt
`ShardGridDB` on PLAYER_LOGIN/PEW if the global table was swapped. Worth copying defensively.

**Libs:** none, no Ace and no LibStub. EllesmereUI is used only through `EllesmereUI.RegisterSkin` (ST:847).
Packaged with BigWigs packager (`.pkgmeta`). A GitHub workflow posts RELEASE-NOTES.md to Discord on each tag.

**Size/structure:** ShardGrid.lua has 5424 lines in one chunk, Styles.lua 868 (a skin engine shared across
the author's addons), Skins 195. Everything hangs off `ns` plus file-level locals. The author notes the main
chunk is "near the client's 200-local limit" (SG:316-317), which is why later functions live on `ns`. The
code says "offline tests" drive some handlers (SG:2595, 2761), but no tests ship.

## 2. Shard handling in depth

### Counting (SG:146-229)
- `SHARD_ID = 6265` (SG:13). The bag API is chosen once: `C_Container.*` with the legacy globals as fallback
  (SG:149-155). `GetSlotInfo` handles both the `C_Container.GetContainerItemInfo` table shape
  (`stackCount`, `isLocked`) and the legacy multi-return (SG:158-167).
- Scans bags `0..MAX_BAG`, where `MAX_BAG = max(NUM_TOTAL_EQUIPPED_BAG_SLOTS, NUM_BAG_SLOTS+1,
  Enum.BagIndex.ReagentBag or 5)` = 5 (SG:16-19). **Key client quirk (CHANGELOG 1.4.0):** on WoW Forever a
  soul bag can sit in the *reagent bag slot (bag 5)*. Reading only bags 0-4 missed it.
- **Soul bag detection** (SG:169-185): first `GetContainerNumFreeSlots(bag)` returns the bag family, checked
  with `bit.band(family, 4)`. Fallback: `ContainerIDToInventoryID` → `GetInventoryItemID` →
  `GetItemInfoInstant` with classID 1 / subclassID 1. Bag 0 is never treated as a soul bag.
- Every occupied soul-bag slot counts as a shard **without checking the item ID** (SG:201-205). That holds
  only because soul bags accept nothing else. Our addon should check the ID anyway.
- Output: `stats = {soulSlots, inSoul, outside, free, hasSoulBag}` plus a list of cells. With no soul bag,
  loose shards are plain "shard" and not "overflow" (SG:220-224).
- **Not scanned:** bank, keyring, mail, the trade window. Bank shards are invisible (the README says nothing
  about them). The total counts bags only.

### Choosing which shards to delete
- `Excess(stats)` (SG:270-276) = `inSoul + outside - (soulSlots + keepExtra)`, capped at `outside`.
  **Returns 0 while the soul bag has free slots and shards sit outside**, so they get moved in first.
  `onlyWithBag` pauses deletion when no soul bag is equipped.
- Safety rule: while auto-delete is off, `keepExtra` is parked at KEEP_MAX = 40. Turning auto-delete on
  never deletes on the spot; the player has to lower the slider (SG:248-265, 5155). Good UX to copy.
- Order: **highest bag and highest slot first**, never from a soul bag, skipping locked slots
  (SG:285-292). The grid repaints the same shards red and moves them next to the free space
  (SG:1706-1736), so the display matches the order deletion will use.
- `count <= excess - issued` (SG:292) means a stack bigger than the remaining excess is skipped. That is
  harmless because shards don't stack, but it would leave a stuck queue if they ever did.

### How deletion works under hardware-event protection (the core trick)
- The comment at SG:234-237 says DeleteCursorItem only works during a real key press or click (tested in
  game). The comment at SG:280-282 says **only one protected item action per hardware event**: a second
  DeleteCursorItem in the same key press is refused.
  So the design is a **queue**: bag events only compute `pendingExcess` (`RunAutoDelete`, SG:359-377).
  `ns.HardwareDeletePass` then runs from an input handler and deletes **one** stack per press (SG:381-439).
- Input sources that trigger the delete pass:
  1. **Pass-through keyboard listener** (SG:445-485): a 1x1 frame with `EnableKeyboard(true)` and
     `SetPropagateKeyboardInput(true)`. Its `OnKeyDown` runs the pass and does not swallow the key. It is
     built at file load, inside a "stage" so a refusal can be traced to it. If the client refuses it, it is
     torn down and `db.noKeyListener` is saved so it is never tried again (SG:132-135, 465-485).
     Typing into an EditBox doesn't reach it (the edit box has focus).
  2. **Post-hooks** with `hooksecurefunc` on `UseAction`, `CastSpellByName`, `CastSpellByID`,
     `UseInventoryItem`, `JumpOrAscendStart` and `InteractUnit` (SG:507-530). These run inside the player's own
     button press, so controller users and mouse-clickers count. This works because the client accepts
     a DeleteCursorItem from the tail of a hardware-initiated secure call. Mouse-look / world clicks were
     tested and get refused (SG:487-493, CHANGELOG 1.6.1).
  3. Clicks: OnMouseUp on the grid and on the alert (SG:5051-5054), and the "Delete extras now" button
     (SG:4447-4456).
- **Refusal detection:** a global watcher on `ADDON_ACTION_BLOCKED/FORBIDDEN` filtered to its own addon
  (SG:113-142). It sets `passBlocked` (via `ns.OnAddonActionBlocked`, SG:2824-2831), **calls
  `StaticPopup_Hide`** on the client's popup, and clears the cursor. A hook input that gets refused is
  disabled by key: `"UseAction"` and `"UseAction in combat"` are judged separately, and the result is saved
  in `db.hookRefused` so the "Interface action failed" message shows at most once ever (SG:496-505).
  Three refused passes in a row with nothing achieved set `deleteBlocked` until the player toggles the
  option (SG:417-425).
- **Verify after acting:** PickupSlot, then `GetCursorInfo()` must be `"item", 6265`, then
  DeleteCursorItem. If the cursor still holds something afterwards, the delete was refused, so ClearCursor
  returns the item to the bag (SG:293-306). Same pattern in FillSoulBag (SG:338-343).
- **Debounce:** after a delete or move, `awaitingUpdate = true` blocks further passes until
  BAG_UPDATE/BAG_UPDATE_DELAYED, with a 1.5 s C_Timer as backstop for a lost event (SG:395-401, 428-437,
  5230). Every pass rescans first so a stale queue can't over-delete (SG:383-384).
- **Fill-before-delete** (`ns.FillSoulBag`, SG:318-350): pick up the last loose shard and click an empty
  soul-bag slot. It is also gated to hardware passes, one move per press.
  *Observation:* the Healthstone trade code does PickupContainerItem + ClickTradeButton from a `C_Timer`
  (SG:2693-2696, 2864-2865) and that apparently works, so PickupContainerItem itself is not
  hardware-protected here. The fill could probably run straight from BAG_UPDATE_DELAYED without waiting for
  a key press. **Verify in game before relying on it.**
- **Edge cases handled:** something already on the cursor means do nothing (SG:293, 319, 2683). Locked
  slots are skipped. `SpellIsTargeting()` skips the pass, but **only in the hook path** (SG:516).
- **Bug: the SpellIsTargeting guard is missing from the key-listener path and the grid/alert click
  path.** SG:457 and SG:5052-5054 call `HardwareDeletePass()` without that check. If a key is pressed while
  an item-targeting spell is pending (e.g. a weapon enchant, or "use X on item"), PickupContainerItem on the
  shard could become the spell's target. CHANGELOG 1.6.1 claims "never", but the code only covers hooks.
- **Combat:** deletion is *not* blocked in combat. The hooks track an "in combat" variant per input.
  Secure UI (summon rows, cast buttons) is deferred (see §3).
- Bank/mail/trade: not considered at all for counting or deletion. Moving shards into the bank won't
  register. Fine for deletion (it only touches bags 0-5), but the totals are bag-only.

## 3. Other features

**Summon requests** (SG:1923-2600)
- Events: `CHAT_MSG_PARTY[_LEADER]`, `RAID[_LEADER|_WARNING]`, `INSTANCE_CHAT[_LEADER]`, `WHISPER`
  (SG:2835-2840). Every arg goes through an `issecretvalue` guard before use (SG:2874). Your own lines
  are ignored (SG:2875). Sender GUID is arg 12, then `GetPlayerInfoByGUID` for the class (SG:2880, 2514).
- Keyword match: the text is normalised (lowercase, punctuation → space). Keywords of ≤2 chars ("1") must
  be the whole message; longer ones match as whole words (SG:1989-2002). Default list at SG:1961. Includes
  "1" and "123", which can false-positive in raids where people answer "1".
- Rows are `SecureActionButtonTemplate`. Left button: `type1=macro`,
  `macrotext1="/targetexact <name>\n/cast Ritual of Summoning"`. Right button: `type2=spell` +
  `unit2=<raidN>` (SG:2436-2440). **`unit2` is a raid/party unit token captured out of combat.** If the
  roster reshuffles during combat, the token can point at someone else.
- `PostClick` only acts on the down or up edge that matches the CVar `ActionButtonUseKeyDown`, which avoids
  double handling (SG:2356-2361). Also used on the summon button (SG:3201-3203). Copy this.
- Messages go out from `UNIT_SPELLCAST_SENT` for **any** Ritual of Summoning cast (action bar, macro,
  click-cast), not only the addon's buttons. There is a 5 s dedupe per name (SG:2222-2226, 2926-2953). When
  the target arg is secret it falls back to `target` if that player is grouped.
- **WoW Forever names have surnames.** UnitName returns (first, surname), and whispers need
  `first .. Constants.CharacterNameSeparatorConsts.CHARACTERNAME_SURNAME_SEPARATOR .. surname`
  (SG:2891-2904). Group matching compares both the full name and the first name (SG:2013-2021). This is the
  1.8.1 fix and a must-know for us.
- Template placeholders `{name} {zone} {area} {subzone} {minimap} {coords} {shards}`, filled with gsub and a
  table (SG:1949-1959). Coordinates come from `C_Map.GetBestMapForUnit` / `GetPlayerMapPosition`
  (SG:2048-2070).
- Arrival check: a 2 s ticker. `UnitInRange` can be secret, so the fallback is `UnitIsVisible`, then a 150 s
  offer timeout, then "left the group" (SG:2546-2596).
- **/say needs a hardware event outside instances** (CHANGELOG 1.7.0, 1.8.2). The /say line is queued and
  posted on the next key press (the same key listener) or a click on a pop-up button. Inside an instance it
  goes at once (SG:3031-3046). A "nobody is clicking" nudge after N seconds of channelling uses
  `UnitChannelInfo`/`UnitCastingInfo` spell IDs. Ritual starts as a cast and only then becomes a channel,
  so a 1.5 s gap is tolerated (SG:3048-3107).
- Floating summon button: a secure macro
  `/cast [@mouseover,exists,noharm,nodead][@target,exists,noharm,nodead][@raidN,exists,nodead] Ritual`
  (SG:3237-3249). This avoids `UnitCanAssist`, which fails for out-of-zone party members (CHANGELOG 1.4.3).
- Combat: secure rows can't change, so every update sets `summonDirty` and replays on PLAYER_REGEN_ENABLED
  (SG:2423, 2849-2854). An opaque, mouse-enabled "In combat" banner over the rows swallows clicks
  (SG:2151-2182). The window can't be hidden in combat, and the X button prints why (SG:2091-2096).

**Soulstone tracker** (SG:3272-3903)
- Reads every group unit's auras through `C_UnitAuras.GetAuraDataByIndex`, or `UnitAura` as fallback. It
  matches the English name "Soulstone Resurrection" or spell IDs 20707…27239 (SG:3275-3318).
- **Aura reads are withheld from addons in combat on this client** (CHANGELOG 1.4.2). A non-string name or
  a secret duration counts as "unknown", not "gone". The last good reading (`stoneMemory`) keeps counting
  down to its fixed expiry (SG:3320-3371). Good pattern.
- Events: UNIT_AURA (**unfiltered**), GROUP_ROSTER_UPDATE, PEW (plus rescans at +1/3/6 s, because existing
  buffs fire no event after a reload), and a 1 s ticker with a full rescan every 5 s (SG:3869-3902).
- Combat log is **forbidden** to addons on this client (CHANGELOG 1.3.0 "Removed the addon's use of the
  combat log"). The caster comes from aura `sourceUnit`.
- Report: packs entries into lines of 230 chars or less (SG:3827-3854). Announce only your own casts, keyed
  by `name:expires` to avoid repeats (SG:3754-3773).
- Bar art: tries the client's cooldown-bar atlas pieces, and `/shards bar grab` copies textures off whatever
  bar you hover (SG:3462-3662). Clever, but a lot of code.

**Soulstone/Healthstone alerts** (SG:802-1194)
- `ns.ReadyStone`: the first stone item in the bags (`GetItemCount`) and `C_Container.GetItemCooldown`.
  A duration of 1.5 s or less is ignored as the GCD (SG:996-1011). The cooldown end is the max of the item
  cooldown, `db.stoneUsedAt + 1800` (wall clock, noted on `UNIT_SPELLCAST_SUCCEEDED` of Soulstone
  Resurrection), and your own stone's expiry on a group member (SG:1046-1062). It wakes itself at the
  ready time with a generation-counted `C_Timer.After` (SG:1064-1075).
- Clickable alerts: a separate secure button, **not parented or anchored to the icon**, so the icon can
  still show and hide in combat while the button only changes out of combat (SG:887-939).
  **Macro trick for combat state changes:** `"/use item:<made stone id>\n/cast Create Soulstone"`. The
  same button uses the stone if one exists and makes one otherwise, so it never needs reconfiguring
  mid-fight (SG:1114-1128, CHANGELOG 1.8.2).
- Events: BAG_UPDATE_COOLDOWN, PEW, SPELLS_CHANGED, PLAYER_REGEN_ENABLED, UNIT_SPELLCAST_SUCCEEDED(player)
  (SG:1181-1193). They also update on every Refresh.

**Healthstone trading** (SG:2598-2822)
- TRADE_SHOW → after 0.3 s `OfferHealthstone`: group-only check (`UnitInParty("NPC")`), repeat limit
  check, cursor-empty check, then `PickupContainerItem` + `ClickTradeButton(firstEmptySlot)` (SG:2661-2700).
  Once per trade. A stone created during the trade is offered on a later bag update (SG:2705-2719).
- "Who got one": `tradeStaged` is set on TRADE_PLAYER_ITEM_CHANGED / ACCEPT_UPDATE when a slot holds a
  stone. It is confirmed by `UI_INFO_MESSAGE == ERR_TRADE_COMPLETE`, or by the stone missing from the bags
  1 s after TRADE_CLOSED (SG:2725-2762). The partner key is `UnitGUID("NPC")`.
- Healthstone detection: an ID list, **plus a fallback that checks every bag item's link for the English
  word "Healthstone"** (SG:2605-2619). That is both a locale bug and a performance cost (see §4).
- The "Create Healthstone" secure button is parented to **TradeFrame** and created lazily out of combat
  (SG:2781-2811). It listens for "AnyUp","AnyDown" because of the cast-on-key-down CVar (CHANGELOG 1.4.4).

**Animations** (SG:1277-1670): pooled textures on a full-screen HIGH-strata layer. A single OnUpdate driver
is attached only while something is animating and removed when idle (SG:1501, 1504-1508). Changes are
detected by diffing the previous cell kinds against the new ones (SG:1623-1670). Well done.

**Skins** (ST, SK): Styles.lua is a mini skin engine with Dark and EllesmereUI back-ends sharing one API
(`S.Shell/Button/Font/SquareIcon/...`). It restyles by fading the original art, so leaving a style needs a
reload, offered through the addon's own prompt rather than a StaticPopup (ST:664-760). Secure frames are
skinned through `Styles.OutOfCombat` (ST:52-66). It hooksecurefuncs the global `PanelTemplates_*` functions
for tab painting (ST:439-445).

**Options** (SG:4755-4829): `Settings.RegisterCanvasLayoutCategory` only. **No proxy settings. Registering
addon values with the Settings system tainted Blizzard nameplates** ("attempt to compare a secret number
value", CHANGELOG 1.0.1). It opens the panel only from a click and checks `IsVisible()` rather than trusting
the return value of `OpenToCategory`.

## 4. Code quality

**Architecture/patterns**
- One huge file with forward-declared locals and `ns.*` for anything past the 200-local limit. Readable
  thanks to excellent "why" comments, but tightly coupled: `Refresh()` drives grid, alerts, summon window,
  summon button and trade (SG:1684-1699).
- Very defensive: 71 `pcall`s, every API feature-detected (`C_Container or global`, `C_Spell or
  GetSpellInfo`, atlases tested with `C_Texture.GetAtlasInfo`). An `issecretvalue` guard is on nearly
  every value read from the game.
- A `report` table plus load "stages" (`ns.SetStage`) means any refusal says which subsystem caused it
  (SG:109-144). Combined with `/shards debug`, that makes for great support ergonomics.
- Dirty flags plus PLAYER_REGEN_ENABLED replay for secure frames. A secure button kept separate from a
  non-secure visual so the visual can change in combat.

**Bugs/risks spotted**
1. SpellIsTargeting guard missing from the key and click delete paths (SG:457, 5052-5054). See §2.
2. Soulstone window likely **can't stay closed while grouped**. `UpdateStones` re-shows it whenever it's
   enabled and (grouped or stones>0) (SG:3784-3822), and it runs on every UNIT_AURA and every 5 s. The X
   only calls `Hide()` (SG:3383).
3. `unit2` in the summon rows is a stale unit token after a roster change in combat (SG:2440).
4. The blockWatcher calls `ClearCursor()` on *any* ShardGrid refusal (SG:137). That could drop an item the
   player is holding if an unrelated action of theirs was blamed on the addon.
5. English string matches: "Soulstone Resurrection" (SG:3276), "Healthstone" link search (SG:2613, 2641).
   ERR_TRADE_COMPLETE is used correctly.
6. Scan counts any item in a soul bag as a shard (SG:201-205).
7. `HEALTHSTONE_SPELLS` lists WotLK IDs (47878, 47871) and the alert list doesn't (SG:1140 vs 2766). This
   is harmless, but the two lists are duplicated and out of sync. CREATE_HEALTHSTONE is also iterated with
   `IsPlayerSpell or IsSpellKnown` inside the loop.
8. Auto-delete on with `keepExtra` already low: the first bag event after login queues deletes, and the
   first key press deletes. That is by design, but there is no confirmation.

**Performance**
- **Refresh runs twice per bag change**: on BAG_UPDATE (throttled to 1 per 50 ms, leading edge) *and* on
  BAG_UPDATE_DELAYED (SG:5230-5239). Each Refresh does a full Scan (IsSoulBag per bag, per scan),
  `UpdateStoneAlert` (6x GetItemCount, 6x IsPlayerSpell, cooldown reads), and `UpdateHealthAlert` →
  `FindHealthstone`, which calls **GetContainerItemLink on every non-stone item in every bag**. When a
  trade is open it runs again. Then `ns.SyncConfig()`. BAG_UPDATE fires constantly while looting or
  vendoring. Better: coalesce to one pass per frame or on BAG_UPDATE_DELAYED, and cache the soul-bag
  status per BAG_CONTAINER_UPDATE.
  The author moved to the immediate BAG_UPDATE on purpose so the toss-in animation doesn't lag (CHANGELOG
  1.3.0).
- **UNIT_AURA is registered unfiltered** (SG:3870). Every aura change on any unit (target, nameplates,
  all 40 raiders) triggers a full 41-unit × 40-aura scan plus UpdateStoneAlert. In a raid this is the
  biggest cost. Use `RegisterUnitEvent` or filter `unit` to group tokens, check `updateInfo` if available,
  and throttle.
- BAG_UPDATE_COOLDOWN (SG:1182) fires on every GCD, and each time redoes the stone/health checks.
- Three always-on tickers: 2 s summon arrivals, 0.5 s say-nudge, 1 s stones (SG:2596, 3108, 3890). They are
  cheap when idle but never stop.

**Taint**
- hooksecurefunc post-hooks on UseAction/CastSpell* are the safe kind (they don't replace the function).
  But they run addon code inside protected call chains on every button press. The addon also calls
  protected-ish item functions there. Accepted by this client, per their testing.
- It deliberately avoids Settings proxy values (nameplate taint), StaticPopup for its own prompts
  (ST:668-669), and HideUIPanel in combat (SG:621-627). It does call `StaticPopup_Hide` on the client's
  ADDON_ACTION popups (SG:138-141), which touches the shared popup system from addon code.
- Global hooks on `PanelTemplates_*` in Styles (ST:443-444) run for every tabbed Blizzard frame.

## 5. Borrow / avoid for Quantum-Seer

**Borrow**
- The queue-then-act-on-hardware-event model with a **pass-through key listener**
  (`EnableKeyboard` + `SetPropagateKeyboardInput(true)`) plus **hooksecurefunc on UseAction / CastSpell* /
  JumpOrAscendStart / InteractUnit** for controller and click players. One protected action per event.
- Verify each step through GetCursorInfo, with ClearCursor to roll back. Rescan right before acting. An
  `awaitingUpdate` latch with a timer backstop.
- A per-input refusal memory saved to SavedVariables, so the "Interface action failed" error is seen at
  most once. ADDON_ACTION_BLOCKED/FORBIDDEN filtered by addon name as the refusal signal.
- "Keep extra" parked at max while auto-delete is off: enabling it can never wipe shards.
- Fill the soul bag before deleting. Test whether the move works outside hardware events (§2).
- Delete from the highest bag and slot first, and color the doomed shards in the UI.
- Scan bag 5 (reagent slot) for soul bags. Use the bag-family bit with an item class/subclass fallback.
- Secret-value hygiene: wrap every game-sourced value with `issecretvalue`. Treat "can't read" as
  *unknown* and keep the last good value.
- Surname-aware names (`CHARACTERNAME_SURNAME_SEPARATOR`) for whispers and group matching.
- PostClick edge filter on the `ActionButtonUseKeyDown` CVar. RegisterForClicks("AnyUp","AnyDown").
- A secure button split from its visual. A use-or-create macro for state that changes in combat.
- Canvas-only Settings page (no proxy settings). `/debug` dump with a stage-tagged refusal log.
- SUPERSEDED-default migration for user-editable text.

**Avoid / do better**
- One 5k-line file near the local limit. Split into modules (bags, deleter, summons, stones, trade, UI).
- Full rescans on every BAG_UPDATE and BAG_UPDATE_DELAYED, plus unfiltered UNIT_AURA. Coalesce and
  filter by unit.
- English name matching. Use spell and item IDs only, and global strings where text is needed.
- Missing SpellIsTargeting / CursorHasItem / `GetCurrentKeyBoardFocus` checks on some input paths. Put
  every precondition in one function that all trigger paths call.
- Stale unit tokens in secure attributes. Prefer `/targetexact name` or rebuild on GROUP_ROSTER_UPDATE
  and mark the rows dirty.
- Window auto-show logic that overrides the user closing it.
- Feature sprawl (bar-art grabbing, cog art grabbing) adds a lot of code for little value.
- Bank shards are ignored. Consider showing a bank count (cached on BANKFRAME_OPENED) if players ask.

## 6. What the CHANGELOG reveals about WoW Forever (1.60.1) API pain points

- **DeleteCursorItem requires a hardware event, and allows only one per event** (1.3.0). Mouse-look and
  world clicks don't count (1.6.1). The key listener can't see controller buttons
  (`EnableGamePadButton` is protected), so action hooks were needed (1.6.1). Players asked for deletes
  to work for controller and click users.
- **SendChatMessage("SAY") outside instances needs a hardware event.** A message posted from an event
  handler gets "blocked: UNKNOWN()" (1.7.0, 1.8.2).
- **Secret values exist on this classic-era client** (`issecretvalue`). Aura data is withheld in combat
  (1.4.2). `UnitInRange` may be secret (1.4.1). UNIT_SPELLCAST_SENT args can be secret (1.7.0). The
  **combat log is off-limits** (1.3.0).
- **Settings proxy settings taint Blizzard nameplates** ("compare a secret number value", 1.0.1).
  HideUIPanel and Settings panel closing are refused in combat (1.6.0).
- **The soul bag can be in the reagent bag slot** (1.4.0). Bag gaps: players want shards packed, not
  mirrored slot by slot (1.4.0).
- **Names have surnames.** UnitName gives the first name only, and chat gives the full name. This broke
  whispers, group matching and unit lookup (1.8.1).
- `UnitCanAssist` fails for party members out of zone. Use group membership instead (1.4.3).
- The cast-on-key-down CVar breaks buttons that only listen for Up (1.4.4).
- Ritual of Summoning is a cast that turns into a channel (1.8.2). Item cooldown reads were unreliable
  enough that they also track their own `stoneUsedAt` (1.8.0).
- Secure buttons can't be shown or hidden in combat. An alert that appears in combat can't be clicked
  until combat ends (1.8.0). A stone made in combat needed the use-or-create macro (1.8.2).
- SavedVariables can arrive after ADDON_LOADED (`LoadSavedVariablesFirst` in .toc, SG:5218).
- Player requests visible in the log: low-shard alert, keep-N shards, controller support, summon queue
  with whispers and /say, a soulstone "who has it" report, one Healthstone per player, matching
  EllesmereUI's look, a Dark theme, and stable window size (min rows).
