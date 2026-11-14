# Quantum-Seer — Plan

A Warlock soul shard addon for **WoW Forever (1.60.1, Interface 16001)**. It covers what the
five reviewed shard addons do (see [notes/addons/](../notes/addons/README.md)) and adds a
**soul collection**: every enemy you drain becomes a trading card. Each card shows the
creature's model and has a rarity, foil finishes and set progress.

- Background research: [WoW Forever addon API research](../notes/research/reports/WoW%20Forever%20addon%20API%20research.md),
  with the source notes in `notes/research/research_notes/`.
- Slash command: `/seer`, with `/qs` as an alias. Don't use `/souls`; both Soul Journal and
  Soul Provenance use it.

### Forever facts that shape this plan
- Forever runs on the **modern retail client**, not the Classic one. Midnight's addon
  restrictions apply, including **secret values** (values addons may hold but not read or
  compare).
  - **Code rule:** check that an API exists before using it; don't branch on the game version.
- **Drain Soul can give a shard from damage alone.** Blizzard reduced that chance on Nov 2 but
  didn't remove it. A Drain Soul kill always gives a shard. So a new shard does **not** prove a
  kill.
- **Shadowburn** gives a shard if the target dies within **8 s** (5 s in Classic).
- **Addons can't use the combat log.** `PARTY_KILL` and `UNIT_DIED` exist instead.
- **Enemy identity can be hidden:** an enemy's `UnitGUID`, `UnitName` and creature type may come
  back as secret values.
  - Other authors report these are readable in open-world combat but hidden in dungeons and
    raids. **We still have to test this in game.**
  - `UnitLevel`, `UnitClassification` and `UnitIsDead` are never marked secret in the API docs.
- **GUIDs change between logins**, because Forever scrambles their server and spawn parts. The
  npcID part stays the same.

### Prior art
Both halves of the idea already exist on Forever:
- **Adventure Journal** (All Rights Reserved): kills of any class become painted 3D cards in a
  binder album, "rarer monsters have shinier cards", and it has a card game.
- **Soul Journal** (MIT): records Drain Soul and Shadowburn souls, with "Sparks" from damage
  shards and "slipped away" ghosts.
- **Soul Provenance** (MIT): soul numbers and Potency tiers.

**What makes ours different:** *the Warlock's shard bag is a booster pack.* Capture takes skill
(it has to be a Drain Soul or Shadowburn kill), and we add what nobody combines:
- trading-card mechanics with **published odds**, 1st Edition stamps and numbered zone and type
  sets;
- **Echo cards** for souls that escape;
- the best shard manager of the five reviewed.

Read Soul Journal's and Soul Provenance's capture code before milestone 5. Both are MIT.

---

## 1. Scope

### In v1
| Area | Features | Learned from |
|---|---|---|
| Shard counting | Item 6265; bags 0..`NUM_TOTAL_EQUIPPED_BAG_SLOTS` (includes bag 5, the reagent slot); soul bags found by the bag-family bit | SG, WTF, SSF |
| Cap and deletion | Per-character cap, **off by default**; separate caps with and without a soul bag; "match soul bag size"; one deletion per key press or click; deletion waits while any restriction is active; refused deletes handled; pause/stockpile mode | all |
| Created vs moved-in shards | Shards brought in from the bank, mail or trade are never auto-deleted; the keep count is **saved across `/reload`** | WTF (bugs fixed) |
| Soul bag filling and sorting | Moves loose shards into free soul bag slots from bag events (moves need no key press); waits on locked items | SSF, SG, IncoreShards |
| Display | Count on the soul bag button; movable counter; Soul Grid (the shard grid); low-shard alert; data broker feed with a minimap button | SG, SSF, SoulSort |
| **Soul collection** | Capture → card → album. See §3. | new |

### In v2
- **Stones:** Soulstone and Healthstone bar, trading a Healthstone, expiry reminders, and the
  "use one or make one" macro.
- **Summons:** watching chat for summon requests, a summon list, editable messages (with
  surnames).
- **Card sharing** with other Quantum-Seer users through addon messages. Limits:
  - one message is 255 bytes, enough for a compact card;
  - 10 messages per prefix, refilling at 1 per second;
  - **Trial and Veteran accounts can't send addon messages.**

### Out of scope
- **The DoT, Fear and Banish trackers from Warlock Tools Forever.** They aren't about shards, and
  WTF needs its own toolkit addon to run them.

---

## 2. Deleting shards

1. **Cap off by default.** Never delete anything the player didn't ask for, and never change a
   saved default in a way that lowers an existing cap. SSF 1.6.1 did that.
2. **Bag events only plan.** They work out `excess = count - max(keepCount, cap)` and queue it.
3. **The deletion runs on a key press or click**, one shard per press. Ways in:
   - (a) the `QS_DELETE` keybind;
   - (b) `/seer delete` inside a macro, which calls the delete directly from the slash command;
   - (c) the delete button on the counter;
   - (d) *optional, off by default:* a keyboard listener that lets keys through plus
     post-hooks on `UseAction`/`CastSpellByID`. Turn off any hook whose deletes keep getting
     refused (Shard Grid does this). `SetPropagateKeyboardInput` can't be called during combat,
     so set the listener up outside combat.
   - **Don't document `/click`** until test D1 passes. Forever's `/click` skips buttons that
     have access constraints, and secure buttons are one of those.
   - **Probe `C_Item.DeleteItem(itemGUID)`**, a new restricted function on Forever that Blizzard's
     delete popup uses. If addons can call it on a key press, we can skip picking the shard up.
4. **Each deletion step is checked:**
   - cursor empty → `PickupContainerItem` → cursor holds 6265;
   - **no restriction active**: check `C_RestrictedActions` for Combat, Encounter,
     ChallengeMode, PvPMatch and Map, not just combat (WTF had a key-press delete blocked inside
     a dungeon);
   - `DeleteCursorItem`.
   - **How we know a delete was refused:** the shard is **still on the cursor afterwards**. Then
     put it back.
   - `ADDON_ACTION_BLOCKED` is only a backup signal. On Forever it names the function
     `UNKNOWN()`, and other actions trigger it too. Show the player at most one notice.
5. **Which shard goes:** loose shards before soul bag shards, highest bag and slot first. The
   soul queue (§3.4) decides which *soul* goes.
6. Don't copy IncoreShards' secure click button pointed at `StaticPopup1Button1`.

---

## 3. Soul collection

### 3.1 Capture: linking each new shard to a kill
Shards are identical items, so a shard can't carry data. We link each **new shard** to an
**event**, and **only a confirmed kill makes a full card.** This uses the same signal as WTF's
created-vs-moved-in keep count, so both features share one `ShardWatcher`.

**Inputs:**
```
UNIT_SPELLCAST_CHANNEL_START(player, Drain Soul) ─► channeling = true, snapshot(target)
UNIT_SPELLCAST_CHANNEL_STOP(player, *)          ─► channeling = false   (any STOP; spellID may be secret)
UNIT_SPELLCAST_SUCCEEDED(player, Shadowburn)    ─► 9 s Shadowburn window, snapshot(target)
PLAYER_TARGET_CHANGED / UNIT_HEALTH(target)     ─► refresh snapshot while a window is open
ADDON_RESTRICTION_STATE_CHANGED (Activating)    ─► snapshot just before a restriction starts
PARTY_KILL(attackerGUID, victimGUID)            ─► kill = attacker is player and victim = snapshot GUID
UNIT_DIED(guid)                                 ─► backup death signal
```
**How each shard increase is classified** (on `BAG_UPDATE_DELAYED`):

| Situation | Result |
|---|---|
| **Kill:** inside a window, and either `PARTY_KILL` named the player, or the channel ended with `UnitIsDead("target")` true | Full card |
| **Damage shard:** inside the Drain Soul window but the target is still alive | **Echo** (§3.3); never counts as a capture |
| Outside any window | Foreign shard: no card, and the keep count goes up |

- **Shadowburn costs a shard when cast**, so watch the *net* change across its window.
- **Never save a secret value.** Saved records use the npcID (stable across sessions), name,
  level, classification, creature type and family, displayID, mapID and time. The GUID is only
  a key for matching within the session.
- **Snapshots** are taken at channel start, on target change, and when a restriction is about
  to start. Every read goes through `IsSecret` guards.
- **Fallbacks when identity is hidden** (expected in instances):
  1. if the name is readable but the GUID isn't → a **name-keyed** record, upgraded to an
     npcID the next time the creature is seen in the open world;
  2. after combat, recover the name from the corpse or the XP message (Soul Journal's
     method);
  3. otherwise a **Lost Soul** card built from level, classification and zone, which are
     always readable.
- **Instance captures are best effort.** They never count toward set totals.
- Look up each spell's name once with `C_Spell.GetSpellName`. Also call
  `GetSpellCastSecrecy(1120)` once at load to learn whether Drain Soul's spell ID is always
  hidden.

### 3.2 Card data
A card is a **capture record**, rolled once and saved, so a `/reload` never re-rolls it.

```lua
-- account-wide: QuantumSeerDB
species = {
  [npcID or "n:"..name] = {
    name, type, family, classification, displayID, modelOK,
    captures = 12, echoes = 4, foils = 1,
    best = 4,                             -- best rarity tier owned
    counts = { [0]=7, [1]=3, [2]=1, [4]=1 },
    finishes = { shadow = 1 },
    first = { t, mapID, char, level },    -- the "1st Edition" card
    seen = true, instanceOnly = false,    -- for set totals and silhouettes
    fav = false,
  },
}
pity = { foil = 37 }                      -- captures since the last foil
log  = { ... }                            -- ring buffer, last 500 captures
-- per character: QuantumSeerCharDB
settings, keepCount, soulQueue = { {key, rarity, finish, t}, ... }
```
- **Seen but not captured:** creatures the player has targeted or moused over in the open
  world are recorded with `seen = true`, to drive the silhouettes and set totals (§3.7).
- Saved data stays small: mapIDs instead of zone names, and duplicates collapsed into counts.

### 3.3 Rarity, foil and Echo
**Rarity** is worked out once from facts about the creature, **with no randomness**:

| Signal | + |
|---|---|
| classification `elite` | 2 |
| `rare` | 3 |
| `rareelite` | 4 |
| `worldboss` / level −1 / boss in a dungeon | 5 |
| enemy level ≥ player level + 3 | +1 |
| killed in a raid instance | +1 |

**Tiers** use the item-quality colors (`ITEM_QUALITY_COLORS`), as the official WoW trading card
game did:

| Score | Tier |
|---|---|
| 0–1 | Common |
| 2 | Uncommon |
| 3 | Rare |
| 4 | Epic |
| 5+ | Legendary |

**Finishes** are where the randomness lives. The odds are shown in the album's help page.

| Finish | Chance | Look |
|---|---|---|
| Plain | — | — |
| **Shadow Foil** | 2.5%, doubled for Rare and up | Purple shimmer |
| **Fel Foil** | Demons only, 1% | Green flame shimmer |
| **Ember Foil** | Shadowburn kills only, 3% | Ember shimmer |
| **Void Prism** | 0.25%; pity: guaranteed by 400 captures | Rainbow, the top chase card |

- **Foil pity:** a capture is guaranteed to be at least Shadow Foil after 60 non-foil
  captures in a row. Each pity counter resets when its finish drops.
  - For comparison, Hearthstone gives a golden Legendary 0.6% per pack with a 361-pack pity.
- **1st Edition:** the stamp goes on the first capture of each species, with its date and zone.
- **Echo** (a damage shard from a target that got away) is a translucent ghost variant of the
  species card. Echoes are counted separately, never upgrade rarity, and **give duplicates and
  near-misses a use**:
  - 5 Echoes of a species not yet captured reveal its silhouette and zone;
  - Echoes of a species already captured add to its "Resonance" progress bar, purely cosmetic.
- All numbers live in one table in `Souls/Rarity.lua` so they're easy to tune.

### 3.4 The soul queue (where collecting meets shard management)
Each character keeps an ordered list of the souls currently held in their shards. When the shard
count drops, because of Summon, Soulstone, Healthstone, Shadowburn or our own auto-delete,
**souls leave the queue in this order:**
1. foreign shards and Echoes;
2. lowest rarity;
3. duplicates of species already owned;
4. oldest.

Favorites and Legendary souls are **protected**. If every soul over the cap is protected,
auto-delete pauses and tells the player why. Shards are identical, so this order is purely
cosmetic and always consistent. The permanent collection is never touched by deletion.

On login, the queue is matched to the real shard count:
- extra entries are dropped (lowest first);
- missing ones become foreign entries.

### 3.5 Showing the creature
Addons can't save screenshots, so the picture is drawn live from a stored **displayID**:
- **Getting the displayID:** no Forever API gives a unit's displayID directly.
  - At capture, a hidden `PlayerModel` calls `SetUnit("target")`, then `GetDisplayInfo()`
    (test M1). The result is saved on the species record.
  - If that fails, the displayID is filled in later from any open-world sighting
    (target or mouseover).
- **Main view:** `PlayerModel:SetDisplayInfo(displayID)`. It reads display data the client
  ships with (14k entries), so it shouldn't depend on the client having seen the creature. Add
  `SetPortraitZoom` and a camera preset per creature type, with a slow turn on hover.
- **Grid and thumbnails:** `SetPortraitTextureFromCreatureDisplayID(texture, displayID)` draws a
  cheap 2D portrait with no model frame, so the Soul Grid and album thumbnails can show real
  faces.
- **Fallback chain:**
  1. `SetDisplayInfo`;
  2. `SetCreature(npcID)`, which depends on the client's creature cache and has shown blank
     models for unseen NPCs on retail;
  3. the creature-type art plate. Type icons have to ship with the addon, because
     `GetCreatureTypeInfo` has no icon, though family info does.
  - A model counts as failed when `OnModelLoaded` doesn't fire within about 1 s.
- **Offline data:** the client's own creature table has only 189 entries (pets and critters), so
  there's no npcID→displayID table we can generate. "Creature Display DB" offers
  `GetDisplayIdByNpcId` but is All Rights Reserved. At most, we use it if the player already has
  it installed (§8).
- **Performance:** a pool of at most 9 model frames, used only in the large card view and on the
  visible album page. Everything else uses portrait textures.

### 3.6 Card layout (about 220×308, scales with the window)
```
┌───────────────────────────┐  ← border color = rarity; finish overlay = additive shimmer
│ Gnoll Brute        Lv 19 ◆│  ◆ = elite/rare/boss marker
│ ┌───────────────────────┐ │
│ │                       │ │  PlayerModel(SetDisplayInfo) or portrait / type art plate
│ │     3D model          │ │  background = creature-type art
│ │                       │ │
│ └───────────────────────┘ │
│ Humanoid · Redridge Mts.  │  type icon · zone set symbol
│ Captured 12× · Echoes 4   │
│ first 3/11 by Vexa Nightbloom
│ ★ 1st EDITION   #047/112  │  collector number / set size (seen species)
└───────────────────────────┘
```
- **Foil effect:** one additive texture that scrolls with `SetTexCoord`, animated in `OnUpdate`
  **only while the card is visible**.
- **Glows:** LibCustomGlow is MIT, but we draw our own simple glow to keep dependencies down.
- **Echo variant:** a desaturated, translucent model and a "faded" border.
- **Art:** Blizzard textures and shipped type icons at first; our own `.tga`/`.blp` art goes in
  `media/` later.

### 3.7 Album screen
- **Tabs:** All · By Zone · By Type · Recent · Favorites.
- **Album pages** hold a 3×3 grid of cards. Click a card for the large view with stats.
- **Filters:** rarity, finish, type, zone, 1st Edition, and Echo-only.
- **Sets with visible gaps.** Collection guidance says visible gaps you can fill are what keep
  people collecting.
  - Species that are **seen but not captured** show as **silhouettes**, using their portrait
    with the color removed.
  - Set totals count **open-world species seen**, so "Redridge 14/22" grows as you explore.
  - Species only met in instances are left out of totals.
- **Milestones:** the first Legendary, the first Void Prism, 100 souls, and completing a zone.
- **Capture toast:** a small card slides in from the shard counter, with a sound for Rare and
  above.
  - Use Blizzard's `AlertFrame` queue if it exists (to be tested), otherwise our own frame.
  - It can be turned off and never appears during combat.

---

## 4. Code layout

These decisions are fixed:
- No required dependency on another author's addon. Bundled libraries: LibStub,
  LibDataBroker, LibDBIcon.
- No Ace3: our settings page is a canvas, avoiding the Settings registration that caused
  nameplate errors in Shard Grid.
- No globals other than the saved variables and binding names.

```
QuantumSeer/
  QuantumSeer.toc            Interface 16001; SavedVariables QuantumSeerDB; SavedVariablesPerCharacter QuantumSeerCharDB
  Bindings.xml               QS_DELETE, QS_PAUSE, QS_ALBUM
  Core/      Init.lua        namespace, event bus, throttled refresh, DB defaults + versioned migrations
             Compat.lua      API shims, IsSecret/Read* guards, restriction state, surname helper
             Probe.lua       /seer probe — read-only API/secret-value report (§6)
  Shards/    Bags.lua        bag scan + cache, soul-bag detection, counts by location
             Watcher.lua     windows, kill/damage/foreign classification, keepCount (persisted)
             Deleter.lua     input sources, safe delete sequence, cursor-based refusal check
             Mover.lua       fill soul bags, sort modes, lock-aware multi-pass
  Souls/     Capture.lua     snapshots, PARTY_KILL/UNIT_DIED, fallbacks → capture records
             Display.lua     displayID capture (hidden model), sighting backfill
             Rarity.lua      scoring, finishes, pity (all tunables in one table)
             Collection.lua  account DB, seen species, sets/milestones
             Queue.lua       per-char soul queue + login reconcile
  UI/        Counter.lua · SoulGrid.lua · Card.lua · Album.lua · Toast.lua · Options.lua · Broker.lua
  Locales/   enUS.lua (+ de, fr, es, pt, ru later)
  media/     type icons, card frames, foil textures
tests/       Lua 5.1 headless runner + WoW mocks (secret values, hardware-event gate, bags, cursor, restrictions)
tools/       check_api.py — every Blizzard API name used, checked against wow-ui-source forever branch
.pkgmeta     BigWigs packager; GitHub Actions: luac -p, tests, check_api, package on tag
```

Performance rules:
- Recount only on `BAG_UPDATE_DELAYED`.
- Cache the shard count.
- Never do a full scan per key press, per `ITEM_LOCK_CHANGED` or per `UNIT_AURA`.
- Register unit events for specific units only (`RegisterUnitEvent`).

---

## 5. Milestones

| # | Milestone | Done when |
|---|---|---|
| **0** | **Probe (in game)** | §6 priorities 1–3 answered and written to `notes/forever-api.md`; priorities 4–5 before milestone 3 |
| 1 | Skeleton | TOC, namespace, saved data and migrations, test runner, CI, `check_api`; addon loads with no errors |
| 2 | Counting and display | Count is correct in bags 0–5 with and without soul bags; counter; data broker feed |
| 3 | Cap and deletion | Tested ways to delete work; refusals detected by the cursor check; waits on any restriction; pause; foreign shards protected |
| 4 | Soul bag fill and sort | Event-driven moves that wait on item locks |
| 5 | Capture | *Read Soul Journal and Soul Provenance first.* Kill/damage/foreign split; `PARTY_KILL`; fallbacks; displayID capture |
| 6 | Cards | Card widget (SetDisplayInfo → SetCreature → art plate), rarity, finishes and pity, Echo variant, toast |
| 7 | Album and Soul Grid | Paging, filters, silhouettes and set totals, milestones, soul queue with protected souls |
| 8 | Polish and release | Options page, locales, help with published odds, macro creator, CurseForge listing |
| v2 | Stones, Summons, Sharing | — |

---

## 6. Phase 0 — in-game tests, by priority
The status column is what the research found ([report](../notes/research/reports/WoW%20Forever%20addon%20API%20research.md)).
`/seer probe` runs the read-only parts. It wraps each call in `pcall` and never reads a secret
value.

**Priority 1: can we capture at all?**

| # | Test | Status |
|---|---|---|
| C1 | During Drain Soul in combat, check `issecretvalue` on the target's `UnitGUID`, `UnitName`, `UnitCreatureType`, `UnitLevel` and `UnitClassification`. Do it in the open world, in a dungeon, and on a unit you only see as a nameplate | Reported: readable in the open world, hidden in instances |
| C2 | Are the `PARTY_KILL` / `UNIT_DIED` GUIDs readable? Is `UnitIsDead("target")` readable at the kill? | `UnitIsDead` isn't marked secret in the docs; the rest is unknown |

**Priority 2: card art**

| # | Test | Status |
|---|---|---|
| M1 | Hidden `PlayerModel:SetUnit("target")`, then `GetDisplayInfo()`: does it return the creature's displayID? | Unknown, **most valuable** |
| M2 | After clearing the WDB cache, do `SetDisplayInfo(id)` and `SetPortraitTextureFromCreatureDisplayID` render? How do they compare with `SetCreature(npcID)`? Does `OnModelLoaded` fire, and when? | `SetCreature` has shown blank models on retail; the rest is unknown |

**Priority 3: classifying captures**

| # | Test | Status |
|---|---|---|
| K1 | Over 20+ Drain Soul casts: START/STOP spellIDs, `GetSpellCastSecrecy(1120)`, and how many shards arrive while the target is alive vs dead | STOP spellID can be hidden; how often damage shards happen after the Nov 2 change is unknown |
| K2 | How long from the kill to `PARTY_KILL` to `BAG_UPDATE_DELAYED` (this sets the window sizes) | Unknown |

**Priority 4: ways to delete**

| # | Test | Status |
|---|---|---|
| D1 | `/click` on a plain (non-secure) addon button | Probably works (Blizzard's code only skips constrained buttons) |
| D2 | A second `DeleteCursorItem` in the same key press | Reported refused |
| D3 | `C_Item.DeleteItem(itemGUID)` from a key press | Unknown |
| D4 | Deleting in combat from a key press, then recounting after combat | Reported blocked when called, not reverted later |
| D5 | `WorldFrame` OnMouseDown vs OnMouseUp | Reported, conflicting |

**Priority 5: the rest**

| # | Test | Status |
|---|---|---|
| R1 | Move a shard into the soul bag from a bag event, in and out of combat | Reported to work out of combat |
| R2 | Does a delete run through `ExecuteFrameScript` keep the key press? | Unknown, low priority |
| R3 | Does a canvas Settings page cause nameplate errors? | Unknown |
| R4 | Does `AlertFrame:AddQueuedAlertFrameSubSystem` exist? Do the CVars that force restrictions on work, so we can test without combat? | Unknown |

---

## 7. Risks
| Risk | Effect | Mitigation |
|---|---|---|
| Identity hidden more widely than reported, or Blizzard tightens it mid-season | No names or models | Cards built from data that can't be hidden (level, classification, displayID); fill gaps from later sightings; Lost Soul; instance captures best effort |
| **Damage shards create false cards** | "Captured 12×" stops meaning kills | Kill confirmation (`PARTY_KILL` / dead target); damage shards become Echoes |
| No way to get a displayID | Thumbnails fall back to type art | `SetCreature` fallback; good type art plates; optional Creature Display DB |
| **Crowded niche** (Adventure Journal, Soul Journal, Soul Provenance) | Hard to stand out | Warlock-only capture, Echoes, published odds, sets, and the best shard manager |
| Blizzard restricts the delete paths further | Auto-delete stops working | Keybind and macro remain; refusal is detected; one notice |
| Saved data grows | Slow logins | Collapsed records, capped capture log |
| Surname names | Broken whispers and profile keys | One "First Surname" helper |
| Accidental deletion | Users lose shards and trust us | Cap off by default, foreign shards protected, defaults never lowered, delete steps checked |

---

## 8. Decisions
**Settled:**
- One collection shared across the account, with "captured by" kept per card.
- DoT tracking stays out.
- Rarity has no randomness; finishes are random, with pity.
- Echoes ship in v1.

**To confirm:**
- **Set totals:** count only open-world species you've *seen* (planned), or also ship a creature
  list per zone? **Recommendation: seen only;** shipping a list needs data we can't generate.
- **Creature Display DB:** use it if the player already has it installed, to fill missing
  displayIDs, even though it bends the "no required dependency" rule? **Recommendation: yes,
  optional only.**
- **Name:** `Quantum-Seer` / `/seer`, or give the card side its own name, e.g. "Soul Binder"?
