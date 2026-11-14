# Protected actions and hardware-event rules for deleting and moving Soul Shards (WoW Forever 1.60.1 / Interface 16001)

Labels used throughout: **[confirmed]** = read in Blizzard UI source or an authoritative doc page; **[reported]** = stated by an addon author, wiki or forum, not independently verified; **[unknown]** = no reliable evidence found. "Era" = Classic Era 1.15.x; "retail" = Midnight-era live (12.x). Source snapshots of Gethe/wow-ui-source read for this note: `forever` 1.60.1 (70938), commit bebcb6b, 2026-11-13; `classic_era` 1.15.9 (70576), commit 4192ac6, 2026-10-26; `live` 12.1.5 (70783), commit 297d7aa, 2026-11-04. Local addon paths are under `e:\Code\Quantum-Seer\reference\`.

## 1. When and why DeleteCursorItem became hardware-event protected, and the exact rule

### Takeaway
On retail, DeleteCursorItem needed a hardware event from 9.0.2 (2020) and became one-delete-per-hardware-event and "#noscript" (blocked from /run, /script, loadstring) in 9.1.5 (2021). Classic flavors picked this up later; the exact Classic Era patch is not documented. On Forever 1.60.1 the API is flagged `HasRestrictions = true`. In-client tests by three local addons agree: one delete per key press or click, none from events or timers, and none in combat (blocked with ADDON_ACTION_BLOCKED). Forever 1.60.1 also has a GUID-based `C_Item.DeleteItem(itemGUID)` that is restricted too.

### Cited Findings
- [confirmed] warcraft.wiki.gg marks DeleteCursorItem as restricted with `#hwevent` ("requires a hardware event i.e. keyboard/mouse input") and `#noscript` ("cannot be called directly from /run and /script (RunScript) and loadstring, e.g. WowLua and WeakAuras"). Its patch history: 9.0.2 (2020-11-17), "Requires a hardware event. A single hardware event is enough to destroy multiple items"; 9.1.5 (2021-11-02), "Protected when called from a macro script. Limited to deleting a single item per hardware event." The page does not mention combat. — [warcraft.wiki.gg API DeleteCursorItem](https://warcraft.wiki.gg/wiki/API_DeleteCursorItem)
- [reported] An older wiki note, quoted in a May 2025 forum thread, said "As of 2022-09-17 Classic (Vanilla and Wrath) can delete multiple items per hardware event" and had no NOSCRIPT protection. The thread's author disputed it from their own testing. In the same thread, Fizzlemizz (a long-time UI forum helper) said DeleteCursorItem "can't" be used from a macro or slash command, and that `/run DeleteCursorItem()` should give a "FORCE TAINT STRONG" error. No test result was posted, so the Classic behaviour is left unresolved. — [Blizzard forums: DeleteCursorItem() never working (May 2025)](https://us.forums.blizzard.com/en/wow/t/deletecursoritem-never-working/2108765)
- [reported] In June 2021, TBC Classic and retail had "crippled DeleteCursorItem() by requiring a hardware event", while Classic Era's Scrap_Cleaner still deleted from events at that time. The poster complained about needing "a hardware event before each deletion". — [WoWInterface t-58788](https://www.wowinterface.com/forums/archive/index.php/t-58788.html)
- [confirmed] DeleteCursorItemFix (esuvii, "WoW Classic", game versions 1.15.2 / 2.5.4, file dated Apr 17 2024) says: "Due to API changes `DeleteCursorItem()` cannot be called from macros or WeakAuras anymore." It "creates an invisible button that allows the command to be called once per hardware input". Usage is `/run ClearCursor()`, then `C_Container.PickupContainerItem(bag,slot)`, then `/click DelItem`, which runs DeleteCursorItem() and then ClearCursor(). The page includes a soul shard example macro. It does not mention ExecuteFrameScript or StaticPopup. — [CurseForge DeleteCursorItemFix](https://www.curseforge.com/wow/addons/deletecursoritemfix)
- [confirmed] Forever 1.60.1 API docs (and classic_era 1.15.9) list `DeleteCursorItem` with `HasRestrictions = true` and no further detail. — [wow-ui-source forever GameCursorDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/GameCursorDocumentation.lua)
- [confirmed] Forever 1.60.1 has `C_Item.DeleteItem(itemGUID)` with `HasRestrictions = true` and `SecretArguments = "AllowedWhenUntainted"`. The DELETE_ITEM / DELETE_GOOD_ITEM popups call `C_Item.DeleteItem(data.itemGUID)` when they have a GUID and otherwise `DeleteCursorItem()`. This API did not appear in the classic_era or live snapshots I grepped. — [forever ItemDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/ItemDocumentation.lua); [forever GameDialogDefs.lua ~L1563/L1592](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_StaticPopup_Game/GameDialogDefs.lua)
- [reported, in-client on Forever] Shard Grid: "the client allows a single protected item action per hardware event; a second DeleteCursorItem in the same key press is refused ('Interface action failed because of an AddOn')". Deletes fired from bag events are blocked; deletes fired from a click work. — `shard-grid\ShardGrid.lua` L234-L283
- [reported, in-client on Forever] WarlockToolsForever: DeleteCursorItem from event handlers raised `ADDON_ACTION_BLOCKED` (function `UNKNOWN()`) in and out of combat, and the shard went back to its bag. From a key-press hook, the next press deleted it. A delete from a key press also raised ADDON_ACTION_BLOCKED inside a dungeon. — `WarlockToolsForever\TESTING.md` L139-L148; `AGENTS.md` L145-L153; `Shards.lua` L193-L194
- [reported] SoulSort's comment says "Classic vanilla only supports deleting 1 item per hardware event … I guess no version can delete more than 1 shard now." — `SoulSort\SoulSort.lua` L371-L372
- [reported] Forum report (Jan 2026, client unspecified): a hover-and-keybind `/run … PickupContainerItem(b,s) DeleteCursorItem()` macro "no longer works". — [Blizzard forums: Click-to-delete addon? (Dec 2025-Jan 2026)](https://us.forums.blizzard.com/en/wow/t/click-to-delete-addon/2221343)

### Inferences
- The working rule on Forever: call DeleteCursorItem from Lua that runs synchronously inside a real key press or mouse click handler, out of combat, and call it once per input. This is consistent across four sources (wiki rule, Shard Grid, WTF, DeleteCursorItemFix wording). The "#noscript" part means `/run DeleteCursorItem()` typed or put in a macro is blocked. Addon code that runs from a key press, including an addon slash command handler, is the pattern every local addon relies on. That is untested by me.
- The WTF dungeon block may reflect Forever/Midnight "restricted content" states (encounter, M+ style lockdowns) rather than combat alone. That is speculative.

### Gaps
- No Blizzard patch note pins when Classic Era (1.15.x) adopted the one-per-event and noscript rules. Evidence brackets it between a 2022-09 wiki note (multi-delete allowed) and 2024-04 (DeleteCursorItemFix for 1.15.2).
- Combat: no official source says DeleteCursorItem is blocked in combat. That claim rests only on Forever in-client reports (WTF; SSF and Shard Grid design around it).
- Whether `C_Item.DeleteItem(guid)` is callable by addons on a hardware event (it would avoid the cursor pickup) is unknown. Its `HasRestrictions` flag does not say which restriction applies.

## 2. Does /click work on addon-created buttons (Era, retail Midnight, Forever)?

### Takeaway
The source shows a real change. On Era 1.15.9, `/click` calls `button:Click()` on any non-forbidden Button. On Forever 1.60.1 and live 12.1.5 it calls Click() only if `not button:HasAccessConstraints()` and the button lacks the `ScriptedInput` forbidden aspect. Both versions send a release-only click by default. WarlockToolsForever's in-client claim that /click presses no addon buttons on Forever conflicts with SoulShardForever's documented `/click SSFDelete`. The source alone does not settle which is right for a plain (non-secure) button.

### Cited Findings
- [confirmed] Forever /click handler (`SlashCommands.lua` ~L725-L741): parses `name [mouseButton] [down]`, defaulting to LeftButton with `down=false`, then `if button and button:IsObjectType("Button") and not button:HasAccessConstraints() and not button:HasAnyForbiddenAspects(Enum.ForbiddenAspect.ScriptedInput) then button:Click(mouseButton, down)`. The live 12.1.5 handler is identical (L738). — [forever SlashCommands.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_ChatFrameBase/Shared/SlashCommands.lua); [live SlashCommands.lua](https://github.com/Gethe/wow-ui-source/blob/live/Interface/AddOns/Blizzard_ChatFrameBase/Shared/SlashCommands.lua)
- [confirmed] Classic Era 1.15.9 /click uses the older check `button:IsObjectType("Button") and not button:IsForbidden()`. — [classic_era SlashCommands.lua ~L733](https://github.com/Gethe/wow-ui-source/blob/classic_era/Interface/AddOns/Blizzard_ChatFrameBase/Shared/SlashCommands.lua)
- [confirmed] Forever API docs define `HasAccessConstraints`: "Returns whether this object has any access constraints that may limit access from some Lua execution contexts" (its return is itself a secret under `SecretAspect.ObjectSecurity`). `ForbiddenAspect.ScriptedInput` (64): "Restricts APIs that trigger synthetic input interactions on objects from Lua scripts." `Button:Click` has `ChecksForbiddenAspects = ScriptedInput` and `SecretArguments = "NotAllowed"`. Other aspects include `AlwaysPropagateInput` (32): "Forces propagation of mouse and keypress input for this object." — [forever SimpleFrameScriptObjectAPIDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/SimpleFrameScriptObjectAPIDocumentation.lua); [ForbiddenAspectConstantsDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/ForbiddenAspectConstantsDocumentation.lua); [SimpleButtonAPIDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/SimpleButtonAPIDocumentation.lua)
- [confirmed] The SecureActionButton `type=click` action has the same new gate on Forever: `if delegate and not delegate:HasAccessConstraints() and not delegate:HasAnyForbiddenAspects(Enum.ForbiddenAspect.ScriptedInput) then delegate:Click(button)`. — [forever SecureTemplates.lua ~L557-L563](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_FrameXML/SecureTemplates.lua)
- [reported, in-client on Forever] WarlockToolsForever says `/click WarlockToolsForeverHealthstone` did nothing, even after its secure buttons were set to act on release (useOnKeyDown false). Its conclusion: "on Forever `/click` does not press AddOn buttons". It switched to `/use` macros. The tested buttons were SecureActionButtons. The notes do not show whether a plain non-secure button was tested. — `WarlockToolsForever\TESTING.md` L168-L176; `AGENTS.md` L220-L221
- [reported] SoulShardForever documents `/click SSFDelete` on a raw, named, non-secure `CreateFrame("Button")` that runs the delete in its OnClick. The changelog (L328) and README (L93) present it as working. I found no in-client verification note. — `SoulShardForever\Modules\DeleteButton.lua` L3-L11; `README.md` L93
- [reported] Retail/SoD forum reports since 11.0 / Sept 2024: macros can no longer /click buttons that run other macros ("Macros cannot click other macros as of 11.0", user claim, no blue post). One reply says /click works from a keybound macro but not when driven by WeakAuras or another addon. — [Blizzard forums: /click not working after SoD patch (Sept 2024)](https://us.forums.blizzard.com/en/wow/t/click-function-not-working-after-todays-sod-patch/1967897); [Click macros no longer work?](https://us.forums.blizzard.com/en/wow/t/click-macros-no-longer-work/1971596)

### Inferences
- The Forever/live check most likely blocks frames that are protected or secure (SecureActionButtonTemplate frames are protected, so they plausibly "have access constraints"). That would explain WTF's failure with secure buttons. A plain insecure addon Button would seem to pass the check, which favours SSF. This is an inference from API wording, not a test.
- For a plain button, `/click` sends a release-only click (`down=false`). A default `RegisterForClicks("LeftButtonUp")` button will fire OnClick. Whether that OnClick still counts as inside the hardware event of the macro key press is what decides whether DeleteCursorItem succeeds there. It should, because it is the same synchronous call stack, and DeleteCursorItemFix depends on exactly that on Classic.

### Gaps
- I found no direct in-client test of `/click <plain insecure button>` on Forever 1.60.1. A ten-second test settles it: a named plain Button whose OnClick prints, then `/click Name` in a keybound macro.
- I found no Blizzard post explaining HasAccessConstraints, or which frames it is true for.

## 3. ExecuteFrameScript: what it is, whether it keeps the hardware-event context, whether addons may use it

### Takeaway
ExecuteFrameScript(frame, scriptName, ...) runs a frame's script handler directly. Blizzard's own UI uses it on Forever, Era and live (e.g. `ExecuteFrameScript(row, "OnEnter")`). I found no documentation of restrictions or of its interaction with hardware events. IncoreShards uses it to run a hooked `DeleteCursorItem()` synchronously from its slash command. The delete then happens inside the key-press stack of the slash command or macro, so it plausibly inherits the hardware event. The SecureActionButton `type=click`/`clickbutton=StaticPopup1Button1` setup looks incidental.

### Cited Findings
- [confirmed] Blizzard UI calls `ExecuteFrameScript(self:GetParent(), "OnEnter")` and similar on all three branches. — [forever Blizzard_AuctionHouseTableBuilder.lua L80-L86, L227](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_AuctionHouseUI/Shared/Blizzard_AuctionHouseTableBuilder.lua)
- [confirmed] The warcraft.wiki.gg page `API_ExecuteFrameScript` returned 404, and ExecuteFrameScript has no entry in the generated API documentation I grepped. Its restrictions are undocumented. — [warcraft.wiki.gg (404)](https://warcraft.wiki.gg/wiki/API_ExecuteFrameScript)
- [confirmed, local code] IncoreShards creates `CreateFrame("Button","DelItem",UIParent,"SecureActionButtonTemplate")`, sets `type=click`, `clickbutton=_G.StaticPopup1Button1`, and adds `btn:HookScript("OnClick", function() DeleteCursorItem() end)`. Its slash handler does `C_Container.PickupContainerItem(b,s)` and then `ExecuteFrameScript(btn,"OnClick")`, then returns, so one shard is deleted per command. Its TOC credits esuvii, the author of DeleteCursorItemFix. — `IncoreShards\IncoreShards.lua` L350-L353, L366-L408; `IncoreShards.toc` L5
- [confirmed] On Forever, a `type=click` secure action calls `delegate:Click()` only when the delegate has no access constraints. StaticPopup1Button1 is a Blizzard frame, so whether it passes is unknown. Under Forever's /click and ScriptedInput rules, a run started from insecure code may not do anything secure. — [forever SecureTemplates.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_FrameXML/SecureTemplates.lua)

### Inferences
- Hardware-event state belongs to the current input dispatch (the key press that ran the macro or slash command), not to the function called. ExecuteFrameScript runs the handler synchronously inside that stack, so the post-hooked DeleteCursorItem plausibly runs "during" the hardware event. In practice IncoreShards would then be equivalent to calling DeleteCursorItem directly from its slash handler. The pattern is probably an attempt to route around the `#noscript` "called from a macro" taint check. Whether that matters on Forever is unknown.
- Because the button is a SecureActionButton, the secure part may silently fail or be skipped when the script is run from tainted code. The part doing the work is the HookScript post-hook.

### Gaps
- I found no source (wiki, forum or Blizzard) stating whether ExecuteFrameScript is protected, taints, or keeps hardware-event context. This needs a client test, for example a slash command that calls DeleteCursorItem directly versus via ExecuteFrameScript.

## 4. Do SetPropagateKeyboardInput listeners and hooksecurefunc post-hooks count as hardware events? Did Midnight restrict propagation?

### Takeaway
Two Forever addons report that a non-consuming keyboard listener (`EnableKeyboard(true)` + `SetPropagateKeyboardInput(true)` + OnKeyDown) is a valid hardware-event context for DeleteCursorItem. RestedXP uses the same pattern for junk deletion. Since 10.1.5, SetPropagateKeyboardInput cannot be called by insecure code in combat, and Forever/live flag it `HasRestrictions`. The addons disagree on world mouse clicks: WTF uses `WorldFrame:HookScript("OnMouseDown")`, while Shard Grid reports world mouse/wheel deletes refused. Shard Grid reports action hooks (UseAction etc.) as partly working, and records refused hooks individually.

### Cited Findings
- [confirmed] SetPropagateKeyboardInput is restricted: "May no longer be called by insecure code while in combat" since patch 10.1.5 (2023-07-11). The wiki lists no 11.x or 12.0 change. — [warcraft.wiki.gg Frame:SetPropagateKeyboardInput](https://warcraft.wiki.gg/wiki/API_Frame_SetPropagateKeyboardInput)
- [confirmed] Live 12.1.5 docs: `SetPropagateKeyboardInput` has `HasRestrictions = true` and `SecretArguments = "AllowedWhenUntainted"`. The new `ForbiddenAspect.AlwaysPropagateInput` lets Blizzard force propagation on chosen frames. — [live SimpleFrameAPIDocumentation.lua ~L1426](https://github.com/Gethe/wow-ui-source/blob/live/Interface/AddOns/Blizzard_APIDocumentationGenerated/SimpleFrameAPIDocumentation.lua); [forever ForbiddenAspectConstantsDocumentation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/ForbiddenAspectConstantsDocumentation.lua)
- [confirmed] Forever docs: gamepad functions say "Insecure code can only call this once in response to gamepad input hardware events". This is the same once-per-hardware-event idiom. — [forever InputDocumentation.lua L276-L329](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/InputDocumentation.lua)
- [reported, in-client on Forever] WarlockToolsForever: a Drain Soul shard above the limit "was deleted on the next key press" via a propagating keyboard frame. It enables the frame only after `GetPropagateKeyboardInput()` confirms it, so keys are never swallowed. It also hooks `WorldFrame:HookScript("OnMouseDown")`, following RestedXP (`RXPGuides/InventoryManager.lua`). — `WarlockToolsForever\TESTING.md` L139-L148; `Core.lua` L697-L740
- [reported, in-client on Forever] Shard Grid: "Mouse buttons and the wheel in the game world are deliberately not hooked: tested in game, the client refused a delete from them, pressed or released, and showed 'Interface action failed' each time." It runs the delete pass after UseAction, casts, equipped-item use, jump and interact hooks. It records any refused hook by name (e.g. "UseAction in combat") and switches that hook off. It also says the key listener cannot see controller buttons because `EnableGamePadButton` is protected. — `shard-grid\ShardGrid.lua` L487-L500, L440-L471
- [reported] SoulShardForever runs deletes only from a slash command or button click, and says nothing about hooks. — `SoulShardForever\Modules\Trimmer.lua` L8-L10

### Inferences
- Key presses via a propagating listener: two independent Forever reports agree that these work out of combat. Treat as reliable.
- World-click hooks: the reports conflict. One possible explanation, not verified, is that Shard Grid's tests involved mouselook or right-click camera drags, or used OnMouseUp/wheel, while WTF hooked OnMouseDown. Another is that WorldFrame clicks carry a different input context on Forever. Treat world clicks as unreliable.
- hooksecurefunc post-hooks on UseAction / CastSpellByName run inside the Blizzard secure call. They count only if that call itself came from a hardware event, such as an action-bar key or click. Shard Grid's per-hook refusal log suggests some count and some do not, especially in combat. No source confirms this.

### Gaps
- No official statement on whether Midnight or Forever further restricted keyboard propagation beyond the 10.1.5 combat rule. The `AlwaysPropagateInput` aspect exists, but I did not find where Blizzard applies it.
- No authoritative statement on whether WorldFrame OnMouseDown counts as a hardware event for DeleteCursorItem on Forever. A client test is required.

## 5. Is PickupContainerItem / bag-to-bag movement protected or hardware-gated? Limits per frame, item locks?

### Takeaway
In none of the three branches is C_Container.PickupContainerItem documented as hardware-gated or `HasRestrictions`. On Forever and live it only carries `SecretArguments = "AllowedWhenUntainted"`. Three local addons move shards from events or passes without input and report it working. Moves are asynchronous: the source slot stays locked until the server confirms. Addons therefore plan moves from a snapshot, skip locked slots, and re-run on bag-update or ITEM_LOCK_CHANGED events. I found no documented per-frame move cap. WTF's comment that the client allows moves only during a key press or click is not backed by any documented restriction.

### Cited Findings
- [confirmed] Forever 1.60.1 and live 12.1.5: `PickupContainerItem` is `Type = "Function"` with `SecretArguments = "AllowedWhenUntainted"`, without `HasRestrictions`. Classic Era 1.15.9 has no restriction annotations at all. Compare `DeleteCursorItem`, which has `HasRestrictions = true`. — [forever ContainerDocumentation.lua ~L420](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_APIDocumentationGenerated/ContainerDocumentation.lua); [classic_era ContainerDocumentation.lua ~L353](https://github.com/Gethe/wow-ui-source/blob/classic_era/Interface/AddOns/Blizzard_APIDocumentationGenerated/ContainerDocumentation.lua)
- [reported] SoulShardForever Sorter: one pass "plans every move from one snapshot and fires them all; it never reads the bags back between moves (a move in flight shows its shard locked at the source and maybe not yet at the target)". Locked shards are skipped, and leftovers are retried on the next bag-changed event up to MAX_RERUNS. "Nothing runs in combat". — `SoulShardForever\Modules\Sorter.lua` L1-L21, L41-L60
- [reported] Shard Grid `ns.FillSoulBag` moves one unlocked stack per call (pickup source, then pickup empty soul-bag slot). It treats a still-full cursor as a refusal and puts the item back. — `shard-grid\ShardGrid.lua` L314-L345
- [reported] WarlockToolsForever's header comment says "The client lets AddOns delete or move items only during a key press or click". It also gates moves on `InCombatLockdown()` or `UnitAffectingCombat` (`WTF.MoveRestricted`). Its notes give no in-client evidence that moves outside input were blocked. — `WarlockToolsForever\Core.lua` L41-L56, L697-L701
- [reported] IncoreShards registers `ITEM_LOCK_CHANGED`. WTF's test harness models `BAG_UPDATE_DELAYED` and `ITEM_LOCK_CHANGED` for move completion. — `IncoreShards\IncoreShards.lua` L686; `WarlockToolsForever\TESTING.md` L78

### Inferences
- Bag-to-bag moves are not hardware-gated on Forever, according to the API docs and the working event-driven sorters in SSF and Shard Grid. WTF's input-only rule is a conservative design choice, not a client rule.
- Out-of-combat moves look safe. In combat, the docs show no restriction, but all three addons avoid it. The cause is unknown and may be caution.
- The practical limit is item locks, not a per-frame cap. Firing several pickup pairs in one frame works only when each targets distinct unlocked slots. Reading bag state between moves in the same frame gives stale or locked results.

### Gaps
- No source documents a server-side throttle on item moves per frame or second. No source documents whether moves in combat are blocked on Forever.

## 6. Evidence for "the server reverts deletions made in combat"

### Takeaway
None found. The only source is SoulShardForever's own code comment. The better-documented observation (WTF) is that in-combat DeleteCursorItem is blocked client-side (ADDON_ACTION_BLOCKED) and the shard returns to its bag immediately. That could easily be mistaken for a "server revert".

### Cited Findings
- [reported, uncorroborated] SSF Trimmer: "In combat it does nothing: the server reverts in-combat deletes when combat ends." — `SoulShardForever\Modules\Trimmer.lua` L10
- [reported, in-client on Forever] WTF: a shard created in combat; DeleteCursorItem "raised ADDON_ACTION_BLOCKED (UNKNOWN()), the shard went back to its bag". — `WarlockToolsForever\TESTING.md` L139-L141
- [confirmed absence] Searches of the wiki, Blizzard forums and WoWInterface found no report of deleted items reappearing after combat. — [warcraft.wiki.gg API DeleteCursorItem](https://warcraft.wiki.gg/wiki/API_DeleteCursorItem); [Blizzard forums search results: DeleteCursorItem changed](https://us.forums.blizzard.com/en/wow/t/deletecursoritem-changed/1168993)

### Inferences
- The likely explanation is that the delete is blocked at call time in combat, so the item never leaves. Either way, the safe policy is the same: never delete in combat.

### Gaps
- A real server-side revert cannot be ruled out without a client test. One test: delete in combat from a key press, check that no ADDON_ACTION_BLOCKED fires, then recount after combat.

## 7. ADDON_ACTION_BLOCKED / ADDON_ACTION_FORBIDDEN behaviour and popup suppression

### Takeaway
On Forever 1.60.1, `ADDON_ACTION_FORBIDDEN` is routed to `StaticPopup_Show("ADDON_ACTION_FORBIDDEN", addonName, ...)`, a Disable / Ignore dialog. `ADDON_ACTION_BLOCKED` goes to a separate handler (in Classic routing, `GameEvent.HandleAddonActionBlocked`), which produces the "Interface action failed because of an AddOn" message. Addons cannot unregister Blizzard's handling. The local addons listen to both events to detect refusals, and Shard Grid hides the popups with `StaticPopup_Hide`.

### Cited Findings
- [confirmed] Forever routing: Shared `EventRouting.lua` maps `ADDON_ACTION_FORBIDDEN` to `GameEvent.HandleAddonActionForbidden`, which calls `StaticPopup_Show("ADDON_ACTION_FORBIDDEN", arg1, nil, arg1)` (`EventImplementation.lua` L91). `StaticPopupDialogs["ADDON_ACTION_FORBIDDEN"]` has buttons DISABLE / IGNORE_DIALOG (`GameDialogDefs.lua` L718-L721). Classic routing maps `ADDON_ACTION_BLOCKED` to `GameEvent.HandleAddonActionBlocked`; Mainline maps it to `GameEvent.HandleActionBlocked`. — [forever Blizzard_Game/Shared/EventImplementation.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_Game/Shared/EventImplementation.lua); [forever Blizzard_Game/Classic/EventRouting.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_Game/Classic/EventRouting.lua); [forever GameDialogDefs.lua](https://github.com/Gethe/wow-ui-source/blob/forever/Interface/AddOns/Blizzard_StaticPopup_Game/GameDialogDefs.lua)
- [reported, local code] Shard Grid registers both events (via pcall), logs refusals during its delete pass, and calls `pcall(StaticPopup_Hide, "ADDON_ACTION_FORBIDDEN")` and `pcall(StaticPopup_Hide, "ADDON_ACTION_BLOCKED")`. WTF counts blocked-action events only while a deletion is in progress. It notes that blocked events name the addon with function `UNKNOWN()` on Forever (not `DeleteCursorItem()`). — `shard-grid\ShardGrid.lua` L109-L141; `WarlockToolsForever\Shards.lua` L504-L531; `TESTING.md` L127-L135
- [reported] On Forever, registering `COMBAT_LOG_EVENT_UNFILTERED` itself raises ADDON_ACTION_BLOCKED (`UNKNOWN()`), so these events are noisy beyond item actions. Calling `SetRaidTarget` raises ADDON_ACTION_FORBIDDEN. — `WarlockToolsForever\TESTING.md` L127-L133; `tests\run.lua` L279-L283

### Inferences
- A refused DeleteCursorItem can be detected reliably by checking `GetCursorInfo()` right after the call: the item is still on the cursor if it was refused. Shard Grid and WTF both do this. The ADDON_ACTION_BLOCKED event is a secondary signal. Use ClearCursor() to put the item back.
- Hiding the FORBIDDEN popup with StaticPopup_Hide works cosmetically, but the dialog's appearance is Blizzard's intended taint warning. Avoiding the forbidden call is the real fix.

### Gaps
- I did not read the body of `GameEvent.HandleAddonActionBlocked` on Forever. Whether it shows a popup, a UI error message, or only taint-log output was not verified in source. Shard Grid's quoted text "Interface action failed because of an AddOn" suggests a UI error-frame message.
