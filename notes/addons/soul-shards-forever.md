# Soul Shards Forever (SSF) — competitor review

Source: `reference/SoulShardForever` (v1.13.1, author msromike, MIT, CurseForge 1715240).
Read-only review; nothing was executed. Paths below are relative to that folder.
SSF began as a fork of SoulSort (see `notes/addons/soulsort.md`) and was rewritten from scratch on Ace3 (`Core.lua:6-7`).
The whole codebase is written by AI sessions; the owner does not write Lua (`CLAUDE.md:3-5`). That explains
the very long comments and the strict, process-heavy rules.

## 1. Overview

**What it does:** it keeps Soul Shards at a cap. Each user press (a keybind, `/ssf delete` in a macro, `/click SSFDelete`, or
the window button) deletes **one** shard if you are over the cap, then moves loose shards into the soul bag. It also shows
the count on the bag bar in a color for its state, glows the bag button when you are low, has a pause mode for
stockpiling, and offers one-click macro creation, a minimap/LDB object, 6 locales and a first-run welcome.

| Item | Value |
|---|---|
| `.toc` Interface | `16001` (= 1.60.1) `SoulShardForever.toc:1` |
| SavedVariables | `SoulShardForeverDB`, AceDB, **per-character profile** (`Core.lua:142`) |
| Category / icon | `## Category: Class`, `## IconTexture` read back at runtime via `C_AddOns.GetAddOnMetadata` (`Core.lua:37-39`) |
| Class gate | `UnitClassBase("player") ~= "WARLOCK"` disables the addon **and every module** (`Core.lua:133-139`); all setup is in `OnEnable`, never `OnInitialize` (the 1.0.1 fix for "index field 'db'" errors on other classes) |
| Libs (`embeds.xml`) | LibStub, CallbackHandler, AceAddon, AceEvent, AceHook, AceTimer, AceBucket, AceDB, AceConsole, AceLocale, AceGUI, AceConfig (Cmd/Dialog/Registry), AceTab (**locally patched**, `tools/externals.txt:22-25`), LibSharedMedia, AceGUI-SharedMediaWidgets, LibCustomGlow, LibDataBroker, LibDBIcon |
| LDB / Titan | One LDB `data source` named "SSF", text = the count in its state color, value = N, click toggles the window, tooltip shows the count, cap, low mark and paused state (`Modules/Broker.lua:30-66`); LibDBIcon minimap button with `profile.minimap.hide` |
| Keybindings | `Bindings.xml`: `SSF_DELETE` → `SSFDelete:Click()`, `SSF_PAUSE` → `Options:TogglePause()`. Labels are set as `BINDING_NAME_*` globals for every class (`Modules/KeyBindings.lua:24-44`) |

**Slash commands** (`Modules/Options.lua:453-481`, handler `588-613`): `/ssf` (opens the window and prints the status), `/ssf help` / `?`,
`/ssf delete`, `/ssf pause on|off`, `/ssf setmax N` (turns AutoMax off), `/ssf log on|off` (with `announce` as a hidden legacy
alias), `/ssf options|settings`. Typed with no value, a stateful command reports its current value; with a value it applies
and echoes it (`Modules/Settings.lua:8-12`). Every command except `delete` is echoed in grey as `> /ssf ...`; `delete` is
not, so macros don't flood chat (`Options.lua:594`). AceTab gives tab completion over whole phrases (`Options.lua:547-582`).

**Localization:** the strings are plain tables (`L["English"] = true` in enUS) handed to `NS.RegisterLocale` (`Locales/Register.lua:21-28`),
which feeds AceLocale **and** keeps every table in `NS.locales`. This allows an **in-game Language override**:
`SSF:ApplyLanguage` rewrites the shared AceLocale table in place (`Core.lua:114-126`), then `Options:Rebuild()`. esES maps to
the esMX table. The `.toc` has `## Notes-xxXX` lines. `tools/check.py` enforces locale completeness and placeholder parity.

**Structure:** about 24 one-job modules talk only through AceEvent messages (`SSF_*`) and public methods. `ARCHITECTURE.md` is
**generated** from the file headers and code (message → sender/listeners table, events per module). The key modules:
Bags (read-only queries plus one bucketed change signal), Cap (thresholds and state), Trimmer (delete), Sorter (moves),
DeleteButton, Counter/LowGlow (bag bar), Options/Settings/Window/Widgets (UI), Macros/MacroGlow/Welcome (onboarding).

## 2. Shard handling in depth

### Counting and bag detection (`Modules/Bags.lua`)
- Shard = itemID `6265` (`Core.lua:34`). It scans bags `0 .. Enum.BagIndex.ReagentBag` (0-5) (`Bags.lua:29`). **On Forever
  a soul pouch equips in the reagent bag slot (bag 5).** Before 1.1.0 SSF never scanned it (CHANGELOG:301-303).
- Soul bag = `bit.band(GetContainerNumFreeSlots family, C_Item.GetItemFamily(6265)) ~= 0` (`Bags.lua:69-81`). It is
  family-based, not ID-based, so it works for any soul bag in any slot. A regular bag means family 0; a special bag that
  is not a soul bag (herb, reagent) is excluded from sorting (`Bags.lua:105-111`). This fixed the 1.3.1 bug "Only Reagents can be
  placed in Reagent Bags".
- **"Secret values"**: every container return is wrapped in `num()`, so a value that is not a plain number counts as
  "not a shard" (`Bags.lua:31-36`), and an `IsLocked` value that is not a boolean counts as locked (`Bags.lua:54-59`). This hints that the Forever engine
  is a modern client build where some API returns can be opaque "secret" values. **We should guard the same way.**
- Lock checks use `ItemLocation:CreateFromBagAndSlot` + `C_Item.IsLocked` (`Bags.lua:53-59`).
- `Count()` walks every slot of bags 0-5. It is not cached: every listener (Counter, Broker, Window, Status, Cap) rescans on
  each `SSF_BAGS_CHANGED`. The cost is trivial (about 6 bags × 20-36 slots).
- One change signal: `BAG_UPDATE` and `BAG_CONTAINER_UPDATE` are bucketed into one `SSF_BAGS_CHANGED` every 0.5 s (`Bags.lua:174-176`).

### Sort order (one order for everything) — `Bags.lua:8-12, 105-131`
The soul bags come first (ascending id), then the regular bags 4 → 0. Inside a bag the walk goes from the **bottom slot (highest number) first**.
With the legacy "Sort top down" mode, regular bags walk top first; soul bags always walk bottom first.

### Cap and "Match soul bag size" (`Modules/Cap.lua`)
- `Get()`: if `autoMax` is on and a soul bag is equipped, the cap is the **sum of the slots of all equipped soul bags** (`Cap.lua:72-77`). Otherwise
  it is `maxShards`, clamped to 1-100 and floored. With no soul bag, AutoMax silently falls back to the manual value.
- While AutoMax is on, `Recompute` **writes** the bag total into `maxShards`. Unticking it then keeps the number the player last saw
  instead of jumping back to an old one (`Cap.lua:6-9, 107`). `lowMark` is clamped down to the cap when the cap drops
  (`Cap.lua:108`), and it is not restored when the cap rises.
- `State(count)` → `paused` > `quiet` > `over` > `low` > `ok` (`Cap.lua:87-93`). **"Quiet"** means Drain Soul (spell 1120) is not
  known yet: no colors and no deletes, because a pre-level-10 warlock can't get shards anyway (`Cap.lua:16-26, 55, 83-85`).
  Spell knowledge can arrive late at login, so the module watches `SPELLS_CHANGED` and `LEARNED_SPELL_IN_SKILL_LINE` while quiet.
- `Trimmer:Delete` calls `Get()` fresh on every press, so a missed event can't give a stale cap (`Cap.lua:30-32`).

### Which shard it deletes — `Bags:LastShard` (`Bags.lua:154-166`)
It deletes the **last unlocked shard in sort order**: the shard in the bag nearest the backpack, at its top slot (or at its bottom slot under the top-down mode).
Result: **loose shards are deleted before soul bag shards**, and the soul bag stays full. The delete runs **before** the sort pass,
because the sort's moves lock the very shards `LastShard` would pick (`Trimmer.lua:6-7`).

### Deletion mechanics and the hardware event (`Modules/Trimmer.lua:34-79`)
```lua
ClearCursor(); C_Container.PickupContainerItem(bag, slot)
local kind, cursorID = GetCursorInfo()
if kind ~= "item" or cursorID ~= SHARD then ClearCursor() return end   -- safety: never delete a non-shard
DeleteCursorItem()
local stillHeld = GetCursorInfo(); ClearCursor(); if stillHeld then return end
```
- **Why it needs a hardware event:** `DeleteCursorItem` is a restricted API. Addon code may call it, but only inside the
  execution of a real user input (a key press or mouse click). On this engine it is also limited to **one delete per hardware event**
  (`Trimmer.lua:8-9`; README "The game lets an addon destroy one item per keypress"). So you can't trim on `BAG_UPDATE`,
  a timer, or a loop: each press removes exactly one shard. Every entry point exists to carry a hardware event into
  `Trimmer:Delete()`:
  - a **keybind**: the `Bindings.xml` body runs in the keypress → `SSFDelete:Click()` → OnClick → `Delete()`;
  - a **macro line** `/ssf delete`: the macro runs from a key or click, and the slash handler runs synchronously inside that
    event, so `/ssf delete` before `/cast Life Tap` costs nothing extra. One press means one cast plus one delete;
  - `/click SSFDelete`: a plain, **non-secure**, sizeless, named `Button` (`Modules/DeleteButton.lua:19-23`). It exists only
    because `/click` needs a global frame name, so keybind addons and WeakAuras can reach it. **It needs no SecureActionButton**,
    because the restriction is on hardware events, not on taint;
  - the window's Delete Shard button, where the mouse click is the event (`Options.lua:236-258`).
- The `stillHeld` check catches a delete the engine refused, so no false `SSF_SHARD_DELETED` is sent.
- **Combat:** it does nothing. The code comment says "*the server reverts in-combat deletes when combat ends*" (`Trimmer.lua:10`). This is a
  Forever-specific claim worth verifying ourselves. The macro help warns that the Bane of Agony macro puts you in combat if you
  have a target (`Modules/DeleteHelp.lua:32-33`).
- `ClearCursor()` runs first, so **whatever the player was holding on the cursor is dropped** when the macro fires (a small UX risk).

### Moving shards into the soul bag (`Modules/Sorter.lua`)
- **SSF Mode (default):** each shard in a later bag than a free soul bag slot moves into that slot. It only moves into *free* soul bag slots and
  never touches other items. With no soul bag or a full soul bag, nothing moves (`Sorter.lua:3-8`).
- **Legacy "Sort top down" / "Sort bottom up"** (SoulSort's behavior): a shard moves to any earlier slot in the walk, and the target may hold another
  item, which swaps into the shard's old slot (`Sorter.lua:10-15`). `IsSwappableAt` excludes locked items and secret values.
- The algorithm (`Sorter.lua:40-89`) works in 3 steps. (1) It takes a **single snapshot** of the targets and the movable shards, each with a rank (the bag position, or in legacy mode the
  walk index). (2) It pairs the last shard with the first target, working inward while the target ranks earlier. (3) It fires `Pickup(src); Pickup(dst); ClearCursor()`
  for every pair **without re-reading the bags**, because moves in flight show stale data.
- **Leftovers:** if any shard was locked or anything moved, `reruns` increments and later `SSF_BAGS_CHANGED` events re-run `Pass()` up to
  `MAX_RERUNS = 5`. **Important evidence:** these reruns run from a bag event, *not* a hardware event. So on this engine
  `PickupContainerItem` moves **do not need a hardware event**. Only the delete does.
- No pass runs in combat, and nothing is queued for later (`Sorter.lua:41-44`).

### Pause
`profile.paused` stops both moves and deletes. Each paused press out of combat can play a sound (LSM-registered Blizzard file
567559 / kit 1200, played with a volume override through `C_Sound.PlaySound(kit,nil,false,false,nil,volume)`, `Modules/Sounds.lua:90-100`) and/or print a
chat line. While paused, the counter is white and the glow is off ("stockpiling on purpose", `Cap.lua:16-18`). Toggles: the checkbox, `/ssf pause`, or
the `SSF_PAUSE` keybind. All three go through one code path (`Options.lua:89-91`).

### Edge cases handled
- A shard locked in flight is skipped by both delete and sort. The count is taken before the sort, because a moving shard can appear twice (`Trimmer.lua:75`).
- At a fresh login the bags are not readable at `PLAYER_ENTERING_WORLD` (the count is 0), so the status line waits for `BAG_UPDATE_DELAYED`, with a 10 s
  timer fallback (`Modules/Status.lua:108-124`).
- Bag 5 can hold a reagent bag or a soul pouch. A bag that is neither a soul bag nor a regular bag is never a target.
- The macro slot limits come from `Constants.MacroConsts` because the `MAX_*_MACROS` globals are nil on Forever (`Modules/Macros.lua:40-45`).

## 3. UI

- **Window** (`Modules/Window.lua`, `Widgets/SSFFrame.lua`): an AceConfigDialog table fed into a custom AceGUI `Frame` extension.
  It auto-fits width, then height (`Window.lua:211-217`). Its position is saved together with the **screen size**, and it reopens
  centered if the resolution or UI scale changed (`Window.lua:262-277`). It closes on Escape through `UISpecialFrames` (`SSFFrame.lua:441`). It **closes on
  `PLAYER_REGEN_DISABLED` and refuses to open in combat** (`Window.lua:73, 307-310`). This is necessary because the welcome side
  note contains a secure button, and hiding it later would be blocked. The window re-feeds the whole AceConfig tree on every bag bucket while it is shown.
- **Help mode:** an "i" button toggles option tooltips on and off for the session. The tooltips also work on *greyed* controls, which normally fire no
  OnEnter, so the wrapper frame takes the mouse instead (`Modules/Help.lua:15-23, 61-72`). This is a nice touch.
- **Delete Shard button** sits beside the count. It is greyed out with the reason written on its face (cap is N / SSF paused / no Drain Soul yet) via
  `Trimmer:WhyNot` (`Trimmer.lua:57-64`, `Options.lua:146-161`). The changelog says a button that silently did nothing "looked broken".
- **Counter on the bag bar** (`Modules/Counter.lua`): a raw FontString on `CharacterBag3Slot`, or on the last equipped soul bag's button
  (`CharacterReagentBag0Slot` for bag 5). The font size is auto-fit by stepping down from 48 to 6 until the text fits 70% of the face, outline included (`Counter.lua:105-116`).
  A 60%-alpha black texture under the number keeps it readable on light bag icons. The game's own count on the soul bag button
  is hidden with `SetAlpha(0)`, not `Hide()`, because the game re-shows it (`Counter.lua:12-16, 90-100`). It is restored when the counter moves.
  The text is colored by state (red low / green ok / yellow over / white paused or quiet, `Modules/Colors.lua:4-9`).
- **Low glow** (`Modules/LowGlow.lua`): `LibCustomGlow.PixelGlow_Start` on the same button: red, 8 lines, a 3 s period, 3 px thick, keyed "SSF".
  The proc glow was rejected because it spills onto the neighboring bag buttons and reads as "press this" (`LowGlow.lua:6-8`).
- **Macro creation** (`Modules/Macros.lua`): it makes `SSF_LT` and `SSF_BoA` with body `#showtooltip\n/ssf delete\n/cast <localized
  name>` (spell names come from `C_Spell.GetSpellName`, since there is **no GetSpellInfo on this client**, `Macros.lua:47-55`). It finds macros by name, so a renamed macro counts
  as missing. It puts them in character slots if both fit, else in account slots, else shows a UIErrorsFrame message; it never splits the pair.
  Because **a newly created macro shows a stale icon until a reload** (a game bug), Create returns `"/reload"`, which the welcome note's
  **SecureActionButton runs as macrotext in PreClick** (`Widgets/SSFSideNote.lua:316-327`). This works around `ReloadUI()` being blocked
  from addon code (CHANGELOG 1.11.2). After the reload, `LOADING_SCREEN_DISABLED` opens the MacroFrame on the macros. **MacroGlow**
  hooks `MacroSelector.RunSetup` and button-glows the macros that are not yet on an action bar. It finds them by scanning the action bars' text from
  `C_ActionBar.GetActionText` (`Modules/MacroGlow.lua:29-72`).
- **Welcome** (`Modules/Welcome.lua`): it opens after `LOADING_SCREEN_DISABLED` with help mode on, and it is versioned (`WELCOME_VERSION = 11000`). It is marked seen
  only when the player closes it outside combat. It is shown again when Drain Soul is learned, after `TRAINER_CLOSED`, so it never pops up over the trainer.
- **KeyBindings:Open** opens Blizzard's modern Settings panel at SSF's section, expanded by setting `initializer.data.expanded`
  (`KeyBindings.lua:32-42`). This confirms that Forever has the retail `Settings`/`SettingsPanel` API.

## 4. Tooling

- **Headless tests** (`tests/`): plain **Lua 5.1** (asserted, `tests/wow_stub.lua:26`). `wow_stub.lua` provides:
  - inert frames that accept any method;
  - a pure-Lua `bit` implementation and an `xpcall` that passes its arguments through, which Ace3's safecall needs;
  - `C_Container`/`C_Item` backed by a `WoW.bags` table with `WoW.SoulBag(slots, items, locked)` / `PlainBag` builders;
  - `PickupContainerItem` calls recorded in `WoW.picks`;
  - `WoW.combat`, `WoW.unknownSpells` and `WoW.FireEvent`.

  `run.lua` loads the **real Ace3 libs** plus Core and the logic modules (no
  GUI), seeds an old SavedVariables file to test the profile-key migration, then runs every `test_*.lua`. Tests assert planned moves as
  strings like `"0:2>1:3 0:9>1:2"` (`tests/test_sorter.lua:68-93`). **This is directly reusable for us.**
- **`tools/check.py`** (782 lines) is the pre-commit and CI gate:
  - luacheck, the tests, locale completeness and placeholders;
  - every `L[...]` used must exist in enUS;
  - colors may only be typed in `Colors.lua`;
  - a file-header contract lint, and header *truth* (Listens/Sends/Saves claims checked against the code);
  - no raw hooks; the shard ID may only appear in Core; every listened message must also be sent;
  - **every Blizzard API/global used is checked against a clone of the wow-ui-source `forever` branch** (no deprecated shims, every `C_` call must be in the API docs);
  - it regenerates `ARCHITECTURE.md`;
  - it reports lib drift.
- `tools/blizzard_reaches.txt` lists every Blizzard UI internal touched (MacroFrameMixin, CharacterBag*Slot, SettingsPanel…), verified
  on build 1.60.1 (70862). It is a **checklist for patch day**.
- `tools/sync_libs.py`: Libs/ are **vendored and committed**. `.pkgmeta` has no externals (`.pkgmeta:4-5`). The script diffs against the
  upstreams in `externals.txt` and copies them in only with `--apply`.
- `.luacheckrc`: std lua51 with a curated `read_globals` whitelist ("add a name only when the game has it").
- CI: `.github/workflows/check.yml` runs check.py on push and PR, then `git diff --exit-code ARCHITECTURE.md`. `release.yml` runs
  BigWigsMods/packager@v2 on a `v*` tag (flag `-u` keeps LF line endings) and posts a GitHub Release for Strongbox. CurseForge builds from the tag via a webhook.
  `.pkgmeta` uses `manual-changelog: CHANGELOG.md` and ignores dev files.

## 5. Code quality

**Strengths:** very clear separation of concerns; pure query functions (Bags) are easy to test; a single source of truth for
thresholds (Cap), wording (Status) and colors (Colors); defensive guards against secret values and locked slots; and a cursor sanity check before the delete.
Combat is handled consistently: Trimmer, Sorter, Macros.Create and Window all bail out.

**Weaknesses and risks:**
- **Over-engineering / verbosity:** the comments often outweigh the code, and the 635-line Options module plus 4 custom AceGUI widgets
  (~1.4k lines) exist mostly for pixel layout. AceGUI internals are read in several places (`Window.lua:34-42`, Tooltip).
  The authors acknowledge this ("a newer AceGUI in any other addon runs SSF's widgets on code nobody tested", `CLAUDE.md:43-45`).
- **Bug (minor):** `Sorter:OnBagsChanged` re-runs passes from bag events **without checking `paused`** (`Sorter.lua:91-93`). Pausing mid-chain
  still allows up to 5 more passes, and under the legacy modes those passes swap items around while you loot.
- **Bug (minor):** the rerun counter is never reset by a new press ("a press mid-chain does not reset the count", `Sorter.lua:32`). Sequences are
  capped oddly, but this is harmless.
- **Edge case:** right after `DeleteCursorItem`, the same press runs `Sorter:Pass()`. The deleted slot may still read as a shard (locked → skipped, or
  unlocked → a ghost pickup that does nothing). This is benign but untested.
- **AceDB default footgun:** 1.6.1 changed the default `maxShards` from 20 to 8, and **every user still on the default silently got cap 8,
  so their next presses deleted shards** (CHANGELOG:201-206). AceDB doesn't store values equal to the default.
- `ClearCursor()` at the start of every press drops whatever the player is holding.
- **Taint surface (low):**
  - writes `initializer.data.expanded` into Blizzard's Settings data (`KeyBindings.lua:36-40`);
  - calls `MacroFrame:ChangeTab/SelectMacro` from addon code;
  - adds FontStrings, textures and glows to Blizzard bag buttons and calls `SetAlpha(0)` on the game's Count text;
  - re-anchors the shared AceConfigDialog tooltip.

  None of these are protected frames. The secure macro button is handled correctly (attributes are only set out of combat, in PreClick, and the window closes on REGEN_DISABLED).
- **Performance:** there is no caching, so every bucket triggers about 5 full scans, plus a full AceConfigDialog rebuild if the window is open. This is fine at this scale. MacroGlow
  scans 180 action slots only once when the window opens; afterwards it reads only the changed slot.
- **Locale:** `Macros` uses a localized spell name in `/cast`. That is correct for the client, but if the player switches SSF's Language setting, the macro bodies still use
  the client's spell names, which is also correct.

## 6. Borrow / avoid / what the CHANGELOG teaches

**Borrow:**
1. The delete design: **one shard per hardware event**, entered through a named `/click` button, a `Bindings.xml` binding, and `/xxx delete` in a macro. Add the
   cursor-itemID verification and the `stillHeld` check.
2. Family-bit soul bag detection over bags 0-5 (**the reagent slot!**). Return guards for secret values. Lock checks via `C_Item.IsLocked`.
3. Delete loose shards first, then fill only free soul bag slots. Plan the moves from a single snapshot. Use bounded event-driven reruns, but check pause.
4. "Quiet until Drain Soul is known", so the addon doesn't nag low-level warlocks.
5. Match soul bag size that **writes** the value back into the visible setting.
6. Delete button greyed out with the reason shown on its face; tooltips on greyed controls.
7. Counter on the bag button: auto-fit font, dark backing, the game Count hidden via alpha. LibCustomGlow pixel glow instead of the proc glow.
8. One-click macros with a localized `#showtooltip /x delete /cast <spell>`, created with a slot-aware character/account fallback. Use a **secure button with macrotext
   "/reload"** whenever a reload is needed.
9. Headless test harness (Lua 5.1 + stub + real Ace libs). Check the API names against wow-ui-source. A `blizzard_reaches.txt` patch checklist.
   Vendored libs. A BigWigs packager release on tag.
10. `BAG_UPDATE_DELAYED` + timer for the first count at login. Bucket bag events (0.5 s).

**Avoid:**
- 4 custom AceGUI widgets and reliance on AceGUI internals for cosmetic layout. Vendoring and patching AceTab to get tab completion is not worth it.
- Changing AceDB defaults that drive destructive actions. Store an explicit value or migrate instead.
- Using `ClearCursor()` without first checking whether the cursor already holds something the player is dragging.
- Re-feeding the whole options dialog on every bag change.

**CHANGELOG → Forever API pain points:**
- The soul pouch lives in **bag 5 (reagent slot)**. Reagent bags there must be excluded from sorting (1.1.0, 1.3.1).
- **Forever character naming "Name Surname" (regional unique names)** changed AceDB's character key between AceDB 36 and 39. Without
  a migration (`Core.lua:82-106`), profiles reset (1.9.0).
- `ReloadUI()` from addon code can be blocked, so use a secure macro button instead (1.11.2). New macros show a stale icon until a reload (1.8.0).
- Calls on Blizzard's deprecated list were replaced (1.12.0). There is no `GetSpellInfo`, and `MAX_*_MACROS` globals are nil (use `Constants.MacroConsts`).
- Deletes are one per hardware event and never in combat. In-combat deletes are reportedly reverted by the server.
- The bag bar never collapses on Forever, so `CharacterBag3Slot` always exists (1.10.2).
- `C_Sound.PlaySound` accepts a per-play volume override on Forever.

**What users asked for (as seen in the changelog):**
- pause/stockpile for raids;
- a counter on the soul bag;
- a mode that moves nothing else in your bags, as opposed to SoulSort's swapping;
- a broker text with just the count;
- help for new players and one-click macros;
- reasons why a button is greyed;
- localization (6 languages);
- tab completion;
- quiet behavior before level 10.
