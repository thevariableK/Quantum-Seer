# Competitor addon review — Warlock soul shard managers (WoW Forever)

Reviewed 2026-11-14. Per-addon notes are in this folder. Source clones live in `reference/`
(gitignored).

| Addon | Author | License | Downloads | Source reviewed? | Notes |
|---|---|---|---|---|---|
| Shard Grid 1.8.2 | Vatik | MIT | ~5.8k | Yes (github.com/jonlipin/shard-grid) | [shard-grid.md](shard-grid.md) |
| Warlock Tools Forever 1.8.0 | Dyre_Straytes | MIT | ~6.2k | Yes (github.com/jbedworth/WarlockToolsForever) | [warlock-tools-forever.md](warlock-tools-forever.md) |
| Soul Shards Forever 1.13.1 | msromike | MIT | ~29.7k | Yes (github.com/msromike/SoulShardForever) | [soul-shards-forever.md](soul-shards-forever.md) |
| SoulSort 1.6.2 | Anilusion | **GPLv3** | ~153.6k | Yes (CurseForge zip; no public repo) | [soulsort.md](soulsort.md) |
| IncoreShards 1.0.4 | Incore97 | MIT | ~635 | Yes (CurseForge zip; no public repo) | [incoreshards.md](incoreshards.md) |

## The core technical problem everyone solves the same way
- `DeleteCursorItem` is **protected by a hardware event**. If an event handler or timer calls
  it, the result is `ADDON_ACTION_BLOCKED`. All five addons delete **one shard per key press or
  click**. The approaches differ only in how they catch that input:
  - **Shard Grid** catches the most kinds of input: a keyboard listener that lets keys through
    (`SetPropagateKeyboardInput(true)`), post-hooks on `UseAction`/`CastSpellBy*` and others,
    and UI clicks.
  - **WTF** uses a keyboard listener that lets keys through, plus a `WorldFrame` OnMouseDown
    hook. Shard Grid's notes say clicks in the game world get refused, which conflicts with
    WTF. Test this.
  - **SSF** uses explicit entry points: a keybind, `/ssf delete` in a macro, a button, and
    `/click SSFDelete`.
  - **SoulSort** uses the simplest method: `/ss sort` in a macro calls `DeleteCursorItem()`
    directly from the slash handler.
  - **IncoreShards** puts `/incore` in a macro, picks up the shard, then calls
    `ExecuteFrameScript(btn,"OnClick")` on an embedded DeleteCursorItemFix-style button. Its
    button is a secure click proxy to `StaticPopup1Button1`, which is risky, so don't copy it.
- **Conflict to verify in game:** SSF documents `/click SSFDelete`, but the WTF review says
  `/click` doesn't press addon buttons on Forever.
- **Combat:** deletion is refused in combat and in restricted content
  (`C_RestrictedActions`). Queue deletions and run them on the first input after combat.
- **Moving items appears NOT to need a hardware event.** SSF sorts from bag events, and Shard
  Grid moves trade items from a timer. So filling the soul bag can happen automatically, and
  only the delete waits for a key press.

## WoW Forever API facts (from all three code reviews)
- Shard is item 6265. Use `C_Container`. **Soul bags can sit in the reagent slot (bag 5)**, so
  scan bags 0–`NUM_TOTAL_EQUIPPED_BAG_SLOTS`, not `NUM_BAG_SLOTS`. Detect soul bags by
  bag-family bit (4).
- The client has **secret values** (`issecretvalue`), aura reads are blocked in combat, and
  merely registering `COMBAT_LOG_EVENT_UNFILTERED` is blocked. Guard every API return value.
- Characters have **surnames**: whisper targets need the
  `CHARACTERNAME_SURNAME_SEPARATOR`. The AceDB profile key also changed (SSF needed a
  migration).
- `GetSpellInfo` is gone, so use `C_Spell.GetSpellName`. The `MAX_*_MACROS` globals are nil, so
  use `Constants.MacroConsts`. The client has the retail `Settings` API. Registering addon
  values with it tainted the nameplates (Shard Grid); register a canvas page only.
- /say and emotes sent from events need a hardware event.

## Design lessons for Quantum-Seer
1. **Safe defaults.** Never delete by surprise. Shard Grid keeps the cap at its maximum while
   auto-delete is off. SSF 1.6.1 silently lowered existing users' cap from 20 to 8 by changing
   an AceDB default.
2. **Tell created shards from moved-in ones.** WTF's "keep count" leaves shards from the bank,
   mail or trade alone. Fix its bugs: persist the keep count across `/reload` and clear the
   channel flag even when the spell ID is secret.
3. **Check every delete step:** cursor empty → pick up → confirm the cursor holds 6265 →
   re-check restrictions → delete → confirm the cursor is empty. On a refusal, put the shard
   back. Prefer loose shards over soul bag shards, and the highest bag and slot first.
4. **Performance:** don't run a full rescan on every key press, `BAG_UPDATE` or `UNIT_AURA`.
   Cache the count, coalesce updates on `BAG_UPDATE_DELAYED`, and register unit events narrowly.
5. **Stay self-contained.** Avoid WTF's dependency on Dyre's Toolkit and SSF's Ace3 bulk plus
   patched widgets. Shard Grid shows that a dependency-free addon is viable, but don't build it
   as one 5k-line file.
6. **Tooling worth copying:** headless Lua 5.1 tests with WoW mocks that model secret values
   and hardware events (WTF has 352 tests). SSF's `tools/check.py` validates the API names the
   code uses against wow-ui-source's forever branch. Build with the BigWigs packager, triggered
   by a tag.
7. **Features players asked for:** a pause/stockpile mode for raid nights, a count on the soul
   bag button, a low-shard alert, soulstone and healthstone helpers, summon requests (with
   surnames), one-click macros, reasons on greyed-out buttons, a data broker display and
   localization.
8. **GPL caution:** SoulSort is GPLv3, so read it for ideas only. The other three are MIT.

## Open questions to test in game
- Does `/click <AddonButton>` work on Forever?
- Does a click in the game world (via a WorldFrame hook) count as a valid hardware event for deletion?
- Is a second `DeleteCursorItem` in the same key press always refused?
- Do the server's combat reverts happen (SSF's claim)?
- Can `PickupContainerItem` for soul bag moves run from events in all cases?

- Does `ExecuteFrameScript` keep the hardware-event context? IncoreShards depends on it.

## Forever gaps in the older addons (we can beat these)
- **Bag 5 (reagent slot):** SoulSort ignores it entirely. IncoreShards never counts or deletes
  shards that sit in a non-soul reagent bag.
- **Soul bag detection:** SoulSort matches English bag names (`"Soul"`, `"Felcloth"`,
  `"D'Sak's"`). Use the bag family bit instead.
- **Forever soul bag:** IncoreShards hardcodes the Felblood soul bag, item **279271**, a
  Forever-only item.
- **SoulSort options page:** two checkboxes save the wrong setting, and the data broker click
  calls the removed `InterfaceOptionsFrame`.
- **Globals:** both addons leak many generic globals.
