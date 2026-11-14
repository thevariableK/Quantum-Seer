# IncoreShards v1.0.4

- Source: https://www.curseforge.com/wow/addons/incoreshards (project 1704876)
- Code: no public repo. Reviewed from the CurseForge zip, extracted to `reference/IncoreShards/`.
- Author Incore97 · MIT · Forever only (`## Interface: 16001`) · ~635 downloads · released 2026-10-20.

## Overview
- **Files:** one file, `IncoreShards.lua` (763 lines), with no libraries.
- **Saved settings:** `IncoreShardsSettings`, account-wide. It is not per character, so every
  warlock on the account shares one cap.
- **Load guard:** the file returns early when the player isn't a Warlock (lines 1-5).
- **Slash commands:** `/incore` deletes one extra shard and is meant to go in a macro.
  `/incoreshards` opens the settings page.
- **Two caps:** one applies when no soul bag is equipped (`limit`), the other when one is
  (`limit2`). Each has a "No Limit" checkbox. **Both default to No Limit**, so it never deletes
  until the player opts in. That is a good safe default.
- **Settings UI:** a hand-built canvas page registered with `Settings.RegisterCanvasLayoutCategory`
  (lines 345-347). It shows bag icons, arrows and a live counter preview.
- **Counter display:** a movable shard counter (Alt + drag) with an opacity slider. The count
  pulses red with orbiting "sparks" when shards are over the cap.
- **"FILL" button:** a popup soul bag button with sparks appears when loose shards could move into
  free soul bag slots. Clicking it moves them.

## Deletion — the interesting part (lines 349-411)
```lua
local btn = CreateFrame("Button", "DelItem", UIParent, "SecureActionButtonTemplate")
btn:SetAttribute("type", "click")
btn:SetAttribute("clickbutton", _G["StaticPopup1Button1"])
btn:HookScript("OnClick", function() DeleteCursorItem() end)
...
SlashCmdList["INCORESHARDS"] = function()
    ClearCursor()
    ... count shards in the normal (non-soul, non-bag-5) bags ...
    C_Container.PickupContainerItem(b, s)
    ExecuteFrameScript(btn, "OnClick")   -- fires the hooked DeleteCursorItem
    return -- one shard per command
end
```
- `/incore` runs inside a macro the player triggers, for example
  `#showtooltip / /cast Drain Soul / /incore`. The hardware event comes from that key press.
  The addon picks up the last shard (highest bag and slot first), then calls
  **`ExecuteFrameScript(btn, "OnClick")`**, which runs the button's OnClick script, including the
  hooked `DeleteCursorItem()`, in the same call stack.
- This is DeleteCursorItemFix (by esuvii) embedded, with `ExecuteFrameScript` instead of `/click`.
  **Worth testing ourselves:** does `ExecuteFrameScript` keep the hardware-event context on
  Forever? If it does, it avoids the open question of whether `/click` presses addon buttons
  (see README).
- ⚠️ **Risky:** the button is a `SecureActionButtonTemplate` with `type=click` and
  `clickbutton=StaticPopup1Button1`. Running OnClick also runs the secure handler, which may
  "click" the first button of whatever popup is showing. That could accept a "Delete item?",
  "Release spirit" or "Accept quest" dialog at the moment of deletion. Untested, but
  there's no reason for this button to be a secure click proxy, so don't copy it. The button's
  global name `DelItem` also collides with the standalone DeleteCursorItemFix addon if both
  are installed.
- It doesn't confirm the cursor holds a shard before deleting. It relies on finding item 6265
  in the slot it picked up, and there's no follow-up check either.

## Counting and bag logic
- `GetBagInfo` (line 356): a soul bag is one whose bag type, from
  `C_Container.GetContainerNumFreeSlots`, equals 4. This compares the whole family value
  instead of masking the bit, but works for soul bags.
- Scans bags 0-5 (`maxNumBags = 5`).
- **Only shards in normal bags count against the cap.** Soul bag shards are never deleted.
  Bag 5 (the reagent slot) is excluded from "normal bags" (lines 375, 625).
  - ⚠️ **Bug:** other reviews found that Forever puts shards into the reagent bag first. With
    a non-soul reagent bag equipped, shards there are neither counted against the cap nor
    deleted.
- The counter scans every slot with `GetContainerItemLink` + `string.find` on each
  `BAG_UPDATE_DELAYED`, `ITEM_LOCK_CHANGED`, `PLAYER_REGEN_ENABLED` and `GET_ITEM_INFO_RECEIVED`
  (lines 684-693).
  - `ITEM_LOCK_CHANGED` fires for every pickup and drop, so this does a full rescan per lock
    change.
  - `GET_ITEM_INFO_RECEIVED` can fire in bursts.
  - It's cheap at this scale but not ideal.
- The "FILL" button (lines 509-546) runs several `PickupContainerItem(src)` +
  `PickupContainerItem(dst)` pairs in one click, planned from one snapshot. Moving several shards
  per click works, which suggests moving items has no one-per-press limit.

## Bugs and code smells
- **Single-backslash texture paths:** `"Interface\Buttons\WHITE8X8"`,
  `"Interface\Buttons\ButtonHilight-Square"` and `"Interface\Icons\Inv_misc_bag_corefelclothbag"`
  (lines 476, 489, 496, 602, 612). Lua 5.1 drops unknown escapes, so these become broken paths.
  The highlight texture and the fallback bag icon silently fail; the sparks still work because
  `SetColorTexture` overrides the path.
- **Leaked globals:**
  - `UpdateHudState` is a global function.
  - Frames get generic global names (`Border_0`, `EditBox_0`, `CheckBox_0` …), which collide
    easily with other addons.
- **Hardcoded Forever items:** the Felblood soul bag item ID **279271** and Traveler's Backpack
  4500 are used for icons. 279271 is a useful fact: a Forever-specific soul bag.
- **Account-wide settings,** even though the cap is really a per-character preference.
- **The `OnUpdate` loop runs every frame** for the life of the session, even when no sparks show
  (lines 559-581).

## Takeaways
- Copy: both caps default to "no limit", separate caps for with and without a soul bag, the
  "move shards into soul bag" button and prompt, and the visual config preview.
- Investigate: does `ExecuteFrameScript` keep the hardware event?
- Avoid:
  - the secure click proxy to `StaticPopup1Button1`;
  - globals;
  - ignoring the reagent bag;
  - a full rescan on `ITEM_LOCK_CHANGED`;
  - single-backslash paths.
