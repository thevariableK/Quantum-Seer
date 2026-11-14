# SoulSort v1.6.2 – Easy Soul Shard Management

- Source: https://www.curseforge.com/wow/addons/soulsort-easy-soul-shard-management (project 336356)
- Code: no public repo. Reviewed from the CurseForge zip, extracted to `reference/SoulSort/`.
- Author Anilusion · **GPLv3: read for ideas only, don't copy code** · ~153.6k downloads,
  created ~2019.
- Ships 3 TOCs:
  - `SoulSort.toc`: Interface 16001, v1.6.2, the Forever build.
  - `SoulSort_Vanilla.toc`: 11502.
  - `SoulSort_Wrath.toc`: 30403.
  - The Vanilla and Wrath TOCs still say v1.5 and load the same Lua.

## Overview
- **Files:** `SoulSort.lua` (522 lines) and `SoulSort_Options.lua` (212 lines). The only bundled
  library is LibDataBroker-1.1. The TOC lists `libs\...` but the folder is `Libs\`, which only
  works because Windows paths aren't case-sensitive.
- **Saved settings:** `SoulSortOptions`, per character.
- **Defaults:** `MaxShards = 0`, which means infinite, so it never deletes until the player sets a
  cap. AutoMax is off. The total counter is off; the per-bag counter is on.
- **`/ss` / `/soulsort` subcommands:** `sort`, `max N`, `automax`, `counter`, `reverse`,
  `showinfo`, `combatwarning`, `options`. `autosort` is dead: `SS_IsClassicVersion()` always
  returns false (lines 194-198), and the comment says auto-sorting isn't possible since the API
  update.
- **Data broker feed:** shows "Shards: N (Full)" text.
- **Bag bar counters:** a count on each non-soul bag button plus a total.

## Deletion (lines 355-385)
- `SS_SortShards()` runs from `/ss sort` in a macro, or from the "Sort Now" button. Both are
  hardware events. It deletes first, then sorts.
- Deletion walks bags 4→0 and slots from last to first. It keeps the first `MaxShards` shards it
  finds and deletes the next one: `PickupContainerItem` → `GetCursorInfo()` confirms item 6265 →
  `DeleteCursorItem()` directly from the slash handler. It then **returns after 1 delete.** The
  comment says no version can delete more than one shard per press any more.
- So the simplest pattern works: a slash command in a macro calls `DeleteCursorItem()`
  directly, with no button tricks. SSF's `/ssf delete` uses the same idea. IncoreShards adds an
  `ExecuteFrameScript` button hop, which looks unnecessary.

## Sorting (lines 248-312)
- It plans swaps from one snapshot. Each shard in bags 0→4 goes to the first slot, scanning
  bags 4→0, that doesn't hold a shard and isn't already a destination. Then it runs every
  `PickupContainerItem(src); PickupContainerItem(dst)` pair in one go. The "reverse" option
  scans destination slots from last to first.
- **It swaps shards with whatever item is in the destination slot.** It doesn't target soul bags
  at all. It just packs shards into the "last" bag (bag 4, the far-left bag slot), which is
  why the docs say "Soul Bag must be in the far-left bag slot". A swap that would move a
  non-shard item into a soul bag fails silently.
- No item-lock handling. Many pickups in one frame, on a single snapshot, can mis-swap if
  anything is locked.

## Bag coverage and soul bag detection: Forever gaps
- ⚠️ **Only bags 0-4 are scanned** for counting, sorting and deleting (lines 249, 360, 424, 439).
  **Bag 5, the reagent slot where Forever soul bags often sit, is ignored entirely.** Shards
  there are invisible to the addon.
- ⚠️ `SS_IsSoulBag` (line 414) matches the bag *name* against the English substrings `"Soul"`,
  `"Felcloth"` and `"D'Sak's"`. That breaks on non-English clients and on any soul bag named
  otherwise. Use bag family bits instead.
- AutoMax sets the cap to the total soul bag slots in bags 1-4; with no soul bag, it uses the size
  of the last equipped bag.

## Bugs
- **Options copy-paste bug:** the "Show deleted shards" and "Disable combat warning" checkboxes
  both read `optSortReverse:GetChecked()` (Options lines 172, 176), so they save the *reverse*
  setting. The combat-warning value is also inverted: "Disable" sets `ShowCombatWarning = true`.
- **Broken data broker click:** clicking the data broker calls `InterfaceOptionsFrame:Show()` /
  `InterfaceOptionsFrame_OpenToCategory`, which don't exist on the modern-settings Forever client,
  so it errors (lines 35-38).
- **Leaked globals:**
  - variables `argument`, `cursorItemID`, `hasSoulBag`, `bagName`, `numSlots`, `font`;
  - functions `SS_*`, `RefreshOptions`, `SaveOptions`, `SoulSortSettingsCategory`;
  - checkbox frames `optAutoMax` …, which collide easily with other addons.
- **Unthrottled rescans:** the full count runs on every `BAG_UPDATE`, which fires once per bag
  per change, with no `BAG_UPDATE_DELAYED` coalescing. `PLAYER_REGEN_ENABLED` is registered but
  unused.
- **Counter placement:** the total counter is anchored on `CharacterBag3Slot`, an assumption
  about the bag bar layout.

## Takeaways
- It's the market leader by download count, but its code is legacy. The Forever port is a
  minimal API update: `C_Container`, `C_AddOns` and the `Settings` canvas page.
- It doesn't handle bag 5, the soul bag family, localization or item locks. These are clear
  ways for us to do better while matching its feature set: sort, cap, counters, AutoMax and the
  data broker feed.
- It confirms that a **slash command in a macro calling `DeleteCursorItem()` directly** works,
  one item per press.
