# Warlock Tools Forever (WTF) — competitor review

Source: `reference/WarlockToolsForever` (v1.8.0, MIT, author "Dyre" / John Bedworth).
I only read the code. I did not run or change anything. Paths below are relative to that folder.

## 1. Overview

**Purpose:** a set of small Warlock quality-of-life modules for WoW: Forever. Each module has its own on/off switch and settings subpage (README.md:1-7).

**TOC** (`WarlockToolsForever.toc:1-11`):
- `## Interface: 16001, 11509`: one package serves Forever (1.60.1, build 70450) and Classic Era 1.15.9.
- `## SavedVariables: WarlockToolsForeverAccountDB` holds account-wide data: `focusWorks` and learned DoT ticks.
- `## SavedVariablesPerCharacter: WarlockToolsForeverDB` holds everything else.
- `## OptionalDeps: DyresToolkit, DyresShieldsUp, DyresBuffReminders`. It never uses `## Dependencies`. CurseForge marks them as required, so the CurseForge app installs them anyway (AGENTS.md "Moved out in 1.3.0").
- `## Group: DyresToolkit` nests WTF under the Toolkit in the AddOn list.
- Load order: Commands, Core, Layout, Shards, Stones, Announce, Tracker, Dots, Predict, Control, Probe, Options.
- `Bindings.xml` is not listed in the TOC; the client loads it automatically. It holds one binding, `CLICK WarlockToolsForeverSmartCurse:LeftButton`.

**Features:**
- Shard auto-delete above a limit, plus a shard counter.
- Stones bar: Soulstone and Healthstone secure buttons, macro writing, Healthstone trading, reminders.
- Announcements: summon, Soulstone, Healthstone gift whisper, stone-creation emotes.
- DoT tracker with target, focus and up to 8 other mobs; nameplate DoT dots; DoT damage prediction with a "DIES" badge.
- Control bar for Fear and Banish, a Focus box, "! Fear", "! Banish" and "! Seduction" macros.
- Smart curse.
- Shield display, chimes and the Demon Armor reminder used to live here. They moved to sister AddOns in 1.3.0 (Core.lua:107-141).

**Slash commands:** `/wtf` with backup `/warlocktools` (Core.lua:970-972). The dispatch table is at Commands.lua:9-28.
- User commands: `options`, `shards`, `stones`, `unlock`, `lock`, `reset`, `reset_focus`.
- Diagnostics: `debug_shards`, `debug_delete`, `debug_stones`, `debug_click`, `debug_tracker`, `debug_predict`, `debug_control`, `debug_announce`, `debug_emote`, `debug_restrictions`.
- An `always = true` flag lets a diagnostic run even when the AddOn has disabled itself.
- Core.lua:684-695 scans `SlashCmdList` for another AddOn that also claims `/wtf` and points the user to the backup command.

**Dyre's Toolkit** is a separate library AddOn exposed as the global `DyresToolkit`. WTF reaches it in two ways:
- **Layout contract** (`RequireContract(1)` plus a function-presence check, Core.lua:75-99): `RegisterAddOn`, `RegisterSettingsCategory`, `RegisterMovable`, `BeginDrag`, `EndDrag`, `ScaleKeepingCenter`, `Lock`, `Unlock`, `IsLocked`, `OpenSettings`. This gives one lock/unlock mode, a grid and snapping across every Dyre's AddOn, and a shared "Dyre's AddOns" settings parent.
- **Shared engines** (Core.lua:887-933, `SHARED_FUNCTIONS`, about 40 functions):
  - Aura containers, with a Lua fallback on Era (`Auras.lua`).
  - Secret-safe unit helpers: `ReadGUID`, `ReadName`, plate memory, `FindUnit`, `LockCast` (`Units.lua`).
  - HealthBars drawing adapters for ThreatPlates, Plater, Platynator, SUF, EllesmereUI and BetterBlizzFrames.
  - `Spells`, `Macros` (`CreateMacroKeeper`), `Announce` (`FormatAnnouncement`), `ClassData`.
  - `Owners`: `ClaimOwner` with a priority, so sibling AddOns don't both draw or claim the same thing.
  - Full engines: `CreateDotTracker`, `CreatePlateDots`, `CreateDotPredictor`, `CreateControlTracker`.
- Since 1.5.0, Tracker.lua, Dots.lua, Predict.lua and Control.lua are thin glue around those engines. Without Toolkit 2.5 or later, those four features stay off and WTF prints one chat line.
- Shards, Stones and Announce do not depend on the Toolkit.
- A copy of the Toolkit is vendored in `tests/toolkit/*.lua` for CI only. It is not shipped.

**Code structure:**
- One namespace table: `local addonName, WTF = ...`. Modules export `WTF.Foo` functions.
- Core owns: saved-settings repair, login gating, input hooks, the restriction check, the Toolkit wiring and startup order.
- Startup is explicit: Core's `PLAYER_LOGIN` handler calls each module's `Initialize*` in a fixed order (Core.lua:1041-1060). No module relies on its own `PLAYER_LOGIN` handler, so event-handler order never matters. This is a good pattern.
- **Mixed-install guard:** `REQUIRED_NAMESPACE` (Core.lua:171-201) lists every function the other files must define. If one is missing, WTF says "files are from different versions" and stays off. The motivation is that the client hides Lua errors by default.
- **Gating:** first the files, then the class (Core.lua:203-209), then the shard APIs (Shards.lua:85-111). It never gates on build or Interface number (AGENTS.md, "Client and API verification").

## 2. Shard handling (Shards.lua) in depth

**Constants:**
- Item `6265`.
- Soul bag family bit `4`, decoded without the `bit` library: `math.floor(family/4) % 2 == 1` (Shards.lua:144-147).
- Shard sources are matched **by spell name**, resolved at init from one ID, so every rank counts (Shards.lua:23-26, 515-521):
  - Drain Soul `1120`: window of 3 s after the channel ends.
  - Shadowburn `17877`: window of 9 s. Forever's talent gives a shard if the target dies within 8 s.

**Counting** (Shards.lua:130-142):
- Scans bags 0 through `NUM_TOTAL_EQUIPPED_BAG_SLOTS`, falling back to `NUM_BAG_SLOTS`. That range includes the reagent bag (bag 5).
- Uses `C_Container.GetContainerItemInfo().itemID/stackCount`. The bank is never scanned.
- In-client finding (TESTING.md:131-135): on Forever, shards are a Reagent (class 15) and go to the **reagent bag first**. A soul bag (14 slots, family 4) can sit in the reagent slot. The first version scanned only up to `NUM_BAG_SLOTS` and found 0 shards.

**Limit** (Shards.lua:149-172): either the sum of equipped soul-bag slots (bags 1..last, never the backpack), or a default of 6, or a manual 1-60.

**Telling created shards from moved-in shards:** the "keep count" model (Shards.lua:7-13, 460-482).
- `state.count` is refreshed on `BAG_UPDATE_DELAYED` and `PLAYER_ENTERING_WORLD`.
- Creation window:
  - `UNIT_SPELLCAST_CHANNEL_START` of a source sets `channeling = true`.
  - `CHANNEL_STOP` or `SUCCEEDED` sets `windowEnd = now + window` (Shards.lua:484-498).
  - `InCreationWindow() = channeling or now <= windowEnd`.
- When the count rises:
  - Inside a window: `keep = max(keep, min(count, limit))`. Created shards are kept only up to the limit.
  - Outside a window (bank, mail, trade), or while deletion is off or stopped: `keep = max(keep, count)`. Everything is kept.
- When the count drops: `keep = min(keep, count)`.
- At login, `keep = count`. The login scan is a baseline, not a gain.
- Deletion target: `count > max(keep, limit)` (`KeepNow`, Shards.lua:226-228).
- Why this beats per-event bookkeeping: a missed shard (two shards from one kill, a full soul bag) is still deleted on a later input, and lowering the limit deletes nothing. A shard that moves to another slot is not a rise, so it is never treated as a new shard.

**Deletion under hardware-event protection:**
- `DeleteCursorItem` raises `ADDON_ACTION_BLOCKED` when called from event handlers, both in and out of combat (TESTING.md:136-145). The fix was copied from RestedXP's junk deleter (Core.lua:697-739):
  - A plain frame with `SetPropagateKeyboardInput(true)`. `EnableKeyboard` is turned on only if `GetPropagateKeyboardInput()` reads back true, so keys are never swallowed. The frame's `OnKeyDown` runs the input handlers.
  - `WorldFrame:HookScript("OnMouseDown", ...)`.
  - Shards registers `OnInput -> ProcessDeletions` (Shards.lua:350-352, 529).
  - A generic `WTF.RunOnNextInput(fn)` queue is reused for emotes and trade placement.
- So deletion is lazy: at most **one shard per key press or world click**. There is no popup or prompt.
- `/wtf debug_delete` relies on the fact that a slash command runs inside the Enter key press, which is a hardware event (Shards.lua:540-576).

**Steps in `DeleteOne`** (Shards.lua:240-275):
1. Check `DeletionRestricted()`. If restricted, return "held".
2. `SplitContainerItem(bag, slot, 1)` for a stack, otherwise `PickupContainerItem`.
3. Verify `GetCursorInfo()` returns `"item", 6265`. An empty cursor returns "missed". Anything else: `ClearCursor()` and return "unexpected".
4. Check the restriction again.
5. Set `settling = {expected = count-1, untilTime = now+2}` *before* the delete, because the client may report bags before `DeleteCursorItem` returns.
6. Set `state.deleting = true`, call `DeleteCursorItem()`, then set `deleting = false`.
7. `ADDON_ACTION_BLOCKED`/`FORBIDDEN` naming this AddOn while `deleting` is true sets `refused = true` (Shards.lua:504-508). This assumes the event fires synchronously inside the call.
8. If refused, or the cursor is not empty: `ClearCursor()` (the shard goes back) and return "blocked".

**Handling the result** (`ProcessDeletions`, Shards.lua:313-348):
- Guards: `processing`, `blocked`, the enabled flag, `Settling()`, the restriction, a non-empty cursor, and `count <= keep`.
- "held", or "blocked" while restricted: wait silently.
- "missed": count a miss for that slot. After 3 misses the slot is skipped.
- "unexpected", or "blocked" outside a restriction: `StopDeleting`, which warns once and stays off until `/reload`. This is the right fail-safe for an irreversible action.

**Which shard is deleted** (Shards.lua:280-296): an unlocked shard *outside* soul bags first, because that frees normal bag space. Within that, the last slot in bag order. Soul-bag shards are used only as a fallback.

**Combat and restrictions** (`WTF.MoveRestricted`, Core.lua:46-69). Deletion is held when any of these is true:
- `InCombatLockdown()`.
- `UnitAffectingCombat("player")`, where a secret answer counts as true.
- `C_RestrictedActions.IsAddOnRestrictionActive` is true for any of `Combat`, `Encounter`, `ChallengeMode`, `PvPMatch`, `Map`.

Even a key press in a dungeon raised `ADDON_ACTION_BLOCKED` (Core.lua:40-45). In practice, **shards are deleted only after combat**, on the first input.

**Secret values:**
- A secret `spellID` opens no window. It sets `secretCasts` for diagnostics (Shards.lua:180-190). This fails safe: the shard is kept.
- In-client: the player's own `UNIT_SPELLCAST_*` spell IDs stay readable in open-world combat (TESTING.md:194-197).

**Diagnostics** (`/wtf debug_shards`, Shards.lua:596-696): bag families and sizes, ring logs of the last 10 gains (each with the preceding spell event) and the last 10 deletions, the last attempt's result, the stop reason, and frame placement details for the counter. These are very good for remote bug reports.

**Edge cases and bugs I see:**
- **A stuck `channeling` flag.** `OnSpellEvent` returns early when `SourceWindow(spellID)` is nil. That includes a *secret* ID on `CHANNEL_STOP` (Shards.lua:484-496). If START was readable and STOP was secret (for example an encounter restriction beginning mid-channel), `channeling` stays true until `/reload`. From then on, every rise counts as "created", so **bank, mail or trade shards above the limit would be deleted**. Our fix: always clear `channeling` on any of the player's `CHANNEL_STOP` events, whatever the ID. Also add a timeout, and reset on `PLAYER_ENTERING_WORLD`.
- The creation window is time-based. A bank withdrawal or mail pickup within 3 s (9 s after Shadowburn) of the source cast counts as created. The risk is low but real.
- The keep count lives only in memory. A `/reload` after an in-combat kill (shard not yet deleted) makes that shard part of the baseline, and it is never deleted.
- **Cost per input.** Every key press, including movement keys, runs `CountShards()`, a full scan of about 5 bags × N slots. Each `GetContainerItemInfo` call allocates a table, and `GetShardLimit` also scans bag families. The `count <= keep` early return comes only *after* the scan (Shards.lua:321). `state.count` from `BAG_UPDATE_DELAYED` is already cached and could gate this cheaply.
- `DebugDeleteShard` ignores `isLocked`. On a locked slot it reports a misleading "cursor did not hold one Soul Shard" (Shards.lua:559-575).
- `DeleteOne` dereferences `info.stackCount` without a nil check. It is safe today only because `DeletionSlot` ran in the same frame.
- If UIParent is hidden (Alt+Z), the keyboard frame (parented to UIParent) stops receiving input. World clicks still work.
- `OnInput` runs handlers without isolation (Core.lua:713-724). An error in one handler stops the rest of the queue for that input.

**Shard counter** (Shards.lua:354-458): a plain, non-secure frame, so it can move and resize in combat. It shows the item icon, dimmed, with a THICKOUTLINE count: green at or above the limit, red at or below the low setting (default 2), white otherwise. The default position is a fraction of UIParent (55%, 42%) so it works at any resolution and UI scale. The scale is applied before the position is restored, because the saved position is in the frame's own scale.

## 3. Other modules (high level)

**Stones** (Stones.lua):
- Rank tables use Classic spell and item IDs (Stones.lua:13-26).
- The bar has two `SecureActionButtonTemplate` buttons with attributes set by mouse button (Stones.lua:309-358):
  - Soulstone, left: `macrotext1 = "/use [@target,help,nodead][@player] item:ID"`.
  - Right: cast the best create rank.
  - Healthstone middle: always use your own stone.
  - With a friendly *player* targeted, Healthstone left switches to "trade" mode, or "create" when you carry no stone they can use. The create rank is limited by the target's level (`C_Item.GetItemInfo` minLevel). It skips ranks you already carry, because Healthstones are unique per rank.
- Attributes and visibility change only out of combat. In combat, `stones.pending` is set and the update runs on `PLAYER_REGEN_ENABLED`.
- `useOnKeyDown = false`.
- **Trading** (Stones.lua:427-552, 798-813):
  1. `PostClick` on release calls `InitiateTrade("target")`.
  2. On `TRADE_SHOW` within 10 s, it whispers the gift text.
  3. Then `PickupContainerItem` → `ClickTradeButton(freeSlot)`.
  4. If that is refused or restricted, it retries once on the next input. After that it warns: "drag it in yourself".
  5. It never presses Trade for you.
- **Macros** (Stones.lua:160-306): it keeps "! Healthstone" and "! Soulstone" in the per-character macro tab.
  - Lines: `/use [btn:2] Create Healthstone(Rank N)` + `/stopmacro [btn:2]`, then `/use item:` for every rank.
  - Rewritten out of combat, only when the body changes.
  - Leaves 2 character macro slots free.
  - Renames old macro names in place, so action bar slots survive the rename.
  - The macro limit comes from `Constants.MacroConsts.MAX_CHARACTER_MACROS`. The global exists only after Blizzard's macro UI loads.
- **Soulstone status:** pairs `UNIT_SPELLCAST_SENT` (target name, castGUID) with `SUCCEEDED`, saves the target and an end time of 30 min (`time()`), and uses `C_Timer.After` to announce the end. Reminders fire on entering an instance, `GROUP_JOINED` and `PLAYER_REGEN_ENABLED`, at most every 10 min.

**Announce** (Announce.lua):
- Summon and Soulstone group and whisper texts with `%T` and `%Z`.
- Whispers must go to "First-Last". Forever characters have surnames, and `UnitName`'s second return is the surname, not the realm (TESTING.md:155-161).
- Emotes and SAY from cast events are blocked, so emotes wait in `RunOnNextInput` and are dropped after 30 s (Announce.lua:103-117).
- Default texts include a credit line for WTF (Core.lua:260-276). This is self-promotion in group chat. Ours should not do it.

**DoT tracker, nameplate dots, Control bar (Fear/Banish)** are engines in the Toolkit; WTF only passes settings and frame names.
- **The hard Forever fact:** AddOn code cannot read auras in combat. `GetAuraDataByIndex`, `GetUnitAuras` and `GetUnitAuraInstanceIDs` raise "Auras cannot be accessed when secret while tainted" and taint the caller (TESTING.md:190-193).
- The solution is Blizzard's `CreateFrame("AuraContainer", nil, row, "CustomAuraContainerTemplate")` after loading `Blizzard_AuraContainer`.
  - Each slot is `AddAuraSlot` with filter `HARMFUL|PLAYER` and `candidateFilters.includeSpellIDs`.
  - Curses use `includeDispelTypes = {Curse=true}` because the curse debuff IDs matched nothing.
  - The engine draws timers into regions that WTF creates in `initializeFrame`. WTF never reads timers.
  - Touching engine-owned bars later raised "forbidden object ... tainted by an AddOn" (AGENTS.md "Panel opacity").
- Mob identity:
  - Lock the GUID at `UNIT_SPELLCAST_SENT`, using the target name and the first unit with that name, and use it at `SUCCEEDED`.
  - Plate GUIDs can be secret, so a token→GUID memory is learned via `UnitIsUnit` against target and focus.
  - In instances, mob IDs are hidden and mobs are tracked by nameplate (CHANGELOG 1.6.0).
- Secure pinned rows use `RegisterStateDriver(row,"visibility","[@target,harm,nodead] show; hide")`. List rows cannot be clickable, because the restricted environment has no `UnitGUID`.
- Health: enemy health is secret in combat. It goes straight into `StatusBar:SetMinMaxValues/SetValue`, which accept secrets, plus `UnitHealthPercent(unit, true, CurveConstants.ScaleTo100)`, as separate font strings.
- The combat log: **registering `COMBAT_LOG_EVENT_UNFILTERED` raises `ADDON_ACTION_BLOCKED`**, even out of combat (TESTING.md:124-130). Prediction on Forever uses the player's casts plus `UNIT_COMBAT` instead.
- `SetRaidTarget` is forbidden even out of combat, so markers come only from `/tm` lines in macros.
- `/click` macros do not press AddOn buttons on Forever (TESTING.md:162-171).
- Focus does not work on Era. A "focusWorks" flag is learned and stored account-wide.

## 4. Probe.lua

This is **not** the capability gate. The real gating is `CheckShardCapabilities` (Shards.lua:85-111), `CheckStoneCapabilities` (Stones.lua:42-57) and `CheckTrackerCapabilities` (Tracker.lua:29-42), each a list of required functions checked with `type(...) == "function"`. A failure prints one line naming the missing API, and only that module, or the whole AddOn for shard APIs, stays off.

Probe.lua is `/wtf debug_restrictions`, a read-only field-research tool:
- Restriction states from `C_RestrictedActions.GetAddOnRestrictionState` over `Enum.AddOnRestrictionType`.
- `C_CombatLog.IsCombatLogRestricted()`.
- Whether the last cast's spell and target are readable.
- On target, focus and up to 3 nameplates, every aura access path: `GetAuraDataByIndex`, `GetUnitAuraInstanceIDs`, `GetUnitAuras`, `GetAuraDataBySpellName`, `GetUnitAuraBySpellID`, and `GetAuraDuration` with its `HasSecretValues`.
- Each check is wrapped in its own `pcall` (`Step`, Probe.lua:22-27), so one refused API doesn't hide the others, and failures print to chat because the client hides Lua errors.
- Secret values are never touched. Everything goes through `WTF.Describe`, which prints "secret".
- Nameplate units are found by `UnitExists("nameplateN")`, because plate frames carry no token.

**Worth copying wholesale.** A `/qs probe` command that players can run in combat and paste back is the cheapest way to learn what Forever 1.60.x really allows.

## 5. Tooling

**tests/run.lua** (7.4k lines, 352 tests, Lua 5.1, no framework):
- Each `fixture()` builds a fresh sandbox `env` with `setmetatable({}, {__index=_G})`. It loads each runtime file with `loadfile` + `setfenv(chunk, env)` and passes `addonName, ns` as varargs (run.lua:1175-1201). Then it fires `ADDON_LOADED` and `PLAYER_LOGIN`.
- The mocks model the important client rules:
  - **Secret values** are tables whose metatable errors on `==`, `<`, `..`, `tostring` and indexing (run.lua:62-73). `issecretvalue` is backed by a registry. A mocked `string.format` returns a secret when given one.
  - Enemy health turns secret in combat.
  - **Hardware events:** `state.press()`, `state.click()` and `state.slash()` set `state.hardware` only while they run (run.lua:1217-1246). `DeleteCursorItem` emits `ADDON_ACTION_BLOCKED` and leaves the cursor full unless `hardware` is true and nothing is restricted (run.lua:614-628).
  - Bags with a family field and lock flags. A cursor whose `ClearCursor` puts the item back. A `BAG_UPDATE_DELAYED` emitted after a delete, with an option for a slow report.
- About 50 shard tests read like a spec. Examples: "shards moved in from the bank are never deleted", "two shards created in one bag update, in one stack, are both deleted", "a blocked-action event outside a deletion never stops deletion", "a secret spell ID opens no window", "the key hook passes every key on to the game".
- Output is TAP-like. The process exit code reflects failures.
- The Toolkit is loaded from `../DyresToolkit` when present, otherwise from the vendored copy (run.lua:26-41).
- TESTING.md:16-18 is honest about the limits: mocks cannot prove protected-action rules, taint or layout. In-client observations override mocks, and each one becomes a regression test.

**Python** (standard library only):
- `scripts/build_release.py` is a deterministic ZIP builder: fixed timestamps and file modes, an allowlist of package files.
- It validates that the TOC `Interface` and the load order match `RUNTIME_FILES`, checks the version and tag, the license text and any TGA headers.
- It extracts the CHANGELOG section for the release notes and writes a BigWigs-format `release.json` (flavor `forever`, interface 16001) for WowUp.
- `verify_release.py`, `publish_curseforge.py` and `curseforge_versions.py` cover publishing. `tests/test_*.py` cover them.

**CI** (`.github/workflows/release.yml`):
- On push or PR: `luac5.1 -p` on every file, `lua5.1 tests/run.lua`, the Python unit tests, then build and unzip-check the package.
- On a tag: a GitHub release (draft, verified, then published), then a CurseForge upload. Both are behind the `release` environment with a required reviewer, and both need repository variables set to opt in.
- Actions are pinned by SHA. Re-running a CurseForge upload is refused. `gate-check.yml` verifies the approval gate.

**Reusable for us:** the `setfenv` sandbox loader, the secret-value metatable, the hardware-flag mock for protected calls, the deterministic ZIP builder with a TOC and file-list consistency check, and the CHANGELOG-to-release-notes step.

## 6. Code quality

**Good:**
- Defensive feature detection everywhere: `type(x) == "function"`.
- Repair of saved variables on load (`SanitizeDB`, Core.lua:537-680). Every field is clamped or whitelisted, and old defaults are migrated with version flags such as `colorsVersion` and `markersVersion`.
- Secret-value discipline: `IsSecret` and `Describe`.
- Combat-deferred secure changes.
- Event-driven updates, with no idle OnUpdate.
- Excellent player-facing diagnostics.
- Clear comments that explain *why*, often citing in-client evidence.

**Weak or risky:**
- **Heavy coupling to a private shared library.** Most features need Dyre's Toolkit at a specific minimum version (2.5–2.9). The AGENTS.md spec is 1,180 lines, and many behaviours are "the Toolkit owner's" call. Duplicate copies of engines (MTF, WTF, Toolkit) caused drift.
- Feature creep. The CHANGELOG shows features moving in and out (shield, chimes, sounds). That is a lot of churn for a 0.x → 1.8 project in about 4 weeks.
- `REQUIRED_NAMESPACE` is a manual list that needs updating every release.
- Global frame names are numerous: `WarlockToolsForever*`, which is required for secure frames and bindings. The globals are prefixed, which is fine.
- **Taint:** WTF never touches Blizzard frames. The exceptions are `WorldFrame:HookScript` (a hook, which is safe) and drawing on third-party health bars. The keyboard frame calls `SetPropagateKeyboardInput` once at login, outside combat. The Healthstone trade calls `InitiateTrade` from a secure button's PostClick, which is not protected.
- **Performance:** a bag scan on every key press, described in §2. `GetShardLimit` runs a soul-bag scan on each counter refresh. Tracker work happens in the Toolkit engines.
- The bugs I found are listed in §2: the stuck `channeling` flag (the most serious), the time-window false positives, and the in-memory keep count.

## 7. Borrow, avoid, and API pain points

**Borrow:**
1. Lazy deletion from input hooks: a propagating keyboard frame enabled only after `GetPropagateKeyboardInput()` reads back true, plus `WorldFrame:HookScript("OnMouseDown")`. One item per input. This is the only approach proven to work on Forever.
2. The keep-count model instead of per-shard tagging. It is robust to missed events, multi-shard kills and limit changes.
3. Pick up, verify the cursor, delete, verify the cursor is empty, put the item back on failure. Treat `ADDON_ACTION_BLOCKED` during the call as a refusal. Stop on the first unexpected result.
4. `MoveRestricted()` covering combat, `UnitAffectingCombat`, and the `C_RestrictedActions` types.
5. Matching shard sources by spell name, so every rank counts. Scanning through `NUM_TOTAL_EQUIPPED_BAG_SLOTS`, so the reagent bag is included.
6. Explicit startup from Core, a mixed-install guard, saved-variable repair, and one line saying why the AddOn is off.
7. Players' `/x debug_*` commands with ring logs, and a read-only restrictions probe with per-check `pcall`.
8. The Lua 5.1 test harness with a secret-value mock and a hardware flag, plus the deterministic packager.

**Avoid:**
- An unconditional early return on an unknown or secret spell ID before clearing channel state. Clear state on *any* player channel stop.
- Rescanning bags on every key press. Gate on the cached count first.
- A hard dependency web, or credit spam in default chat texts.
- `/click` macros, `SetRaidTarget`, CLEU registration, and reading auras in combat on Forever. All are dead ends.
- Relying on the `MAX_CHARACTER_MACROS` global.

**Forever API pain points revealed by the CHANGELOG, TESTING.md and comments:**
- `DeleteCursorItem` needs a hardware event, and is refused even then in combat and encounters (0.3.4-beta; TESTING.md:136).
- Midnight-style secret values and AddOn restrictions on a Classic-like client: `issecretvalue`, `C_RestrictedActions`, `C_CombatLog.IsCombatLogRestricted() == true` always. Registering CLEU itself is blocked.
- Auras are unreadable in combat. You need Blizzard's AuraContainer engine (1.3.2, 1.6.0). Curse debuff IDs don't match Classic IDs. Mob GUIDs and plate GUIDs are secret in instances.
- Emote and SAY from events are blocked, so they need an input. Whispers need "First-Last" because of surnames.
- `/click` does not reach AddOn buttons. `SetRaidTarget` is forbidden for AddOns. Focus is missing on Era.
- Shards are reagents: they fill the reagent bag first, and the soul bag may sit in the reagent slot. Classic spell and item IDs are mostly valid, but treat them as unverified and match by name.
- Classic Era and Forever ship in one package (`## Interface: 16001, 11509`). Behaviour is chosen by API detection, never by build number.
