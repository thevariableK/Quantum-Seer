# Collection, Bestiary & Trading-Card Prior Art for "Quantum-Seer" (WoW Forever 1.60.1), as of Nov 2026

Research method: CurseForge project pages (fetched 2026-11-14), warcraft.wiki.gg API pages, Hearthstone wiki (citing official Blizzard figures), Bulbapedia, and gamification and design blogs. Download counts and dates are taken from page snapshots on 2026-11-14.

## 1. Existing WoW addons that collect or catalog killed creatures (bestiaries, kill trackers, rare trackers)

### Takeaway
The space is crowded on Classic and Forever. Several Nov-2026 addons already do creature bestiaries (Yonder, KillDex, Bestiarium). Most importantly, **Adventure Journal already turns kills into painted cards in a binder album, with 3D models and "shinier cards for rarer monsters", on WoW Forever 1.60.1**. The biggest technical pitfall: the Forever client hides creature identity from addons inside dungeons and raids, so instance kills cannot be recorded reliably.

### Cited Findings
- **Adventure Journal: Bestiary, Achievements & Cards** (author Evincia). It supports WoW Forever (latest file 1.60.1, Nov 14 2026) and Classic, and was created about 19 days before the snapshot. Each defeated creature is added as a "painted card". You can view the collection as a list or as a "binder-style album". "Rarer monsters have shinier cards", and cards are displayed as 3D models. It includes a built-in card game (10-card deck, energy, 20-HP hero vs AI, PvP duels in beta) and "Essences" (Brute/Pure/Radiant) that drop, fuse, and boost creatures. It has about 2,000 account-wide achievements and daily goals. 901 downloads, All Rights Reserved. The page gives no drop rates and no export or sharing of the collection. — [CurseForge: Adventure Journal](https://www.curseforge.com/wow/addons/adventure-journal)
  - A search snippet also described Adventure Journal as a numbered bestiary that shows silhouettes for creatures you have only spotted, with data from QuestieDB. The fetched page did not confirm numbering or silhouettes. — [Search result → CurseForge](https://www.curseforge.com/wow/addons/adventure-journal)
- **KillDex** (Nargoyle). Supports Midnight, WoW Forever, MoP Classic, TBC Classic and Classic Era, with 54.2K downloads, updated Nov 14 2026. Features:
  - per-mob kill counts, account-wide and per character, shown in tooltips
  - boss, elite and rare markers
  - milestone messages at 10/25/50/100/250/500/1000 kills
  - a loot tab with drop rates
  - creature-type filters and a searchable database
  - LibDBIcon minimap button
  - **export string → killdex.com** profiles and leaderboards (Battle.net sign-in), `/kd export` and `/kd paste` for moving collections between machines, and import from KillTrack
  
  Key limitation it states: "Since patch 12.0, Blizzard hides which mob died inside instances on Midnight and WoW Forever." It still counts instance kills that grant XP or leave lootable corpses, but at max level, kills with no loot cannot be counted. — [CurseForge: KillDex](https://www.curseforge.com/wow/addons/killdex)
- **Yonder: Riddles, Rare Hunts & an Explorer's Journal** supports Classic Era (incl. Hardcore and SoD), Classic TBC and WoW Forever. Features:
  - a bestiary of about 3,500 creatures in 96 families, with naturalist notes, kill counts, first sightings, loot and observed drop rates
  - a "rotating model" on creature pages
  - sorting by name, kills, recent kill or first discovery
  - undiscovered creatures hidden, with "no total" shown of how many remain
  - per-character storage and `/yonder export` / `import CODE` for settings
  - creature data from CMaNGOS Classic DB, QuestieDB and Wowhead
  - an opt-in "Share what you see" feature that uploads via an external app (Ravenpost) after logout
  
  It states: "The game hides creatures inside Forever dungeons and raids from addons, so the journal can't record them there." 1,069 downloads, v0.16.0 on Nov 14 2026. — [CurseForge: explorers-field-journal URL, which served the Yonder page](https://www.curseforge.com/wow/addons/explorers-field-journal). Note: the "explorers-field-journal" URL returned the Yonder page, so the project may have been renamed. A search snippet had described "Explorer's Field Journal" as a bestiary using CMaNGOS data.
- **Bestiarium** is a searchable bestiary recording every mob you fight. Its Classic or Forever support was not confirmed. — [CurseForge: Bestiarium](https://www.curseforge.com/wow/addons/bestiarium)
- **RareScanner** (about 144.5M downloads). Supports Retail, Forever (file 1.60.1.7, Nov 14 2026), MoP Classic, Classic and TBC.
  - Detection: on Retail it uses vignettes. On Classic and Forever it relies mainly on nameplates (users can raise `nameplateMaxDistance` to 80), plus an optional "Forever macro" that you click repeatedly while walking through spawn areas.
  - Alert UI: a center-screen button showing a miniature NPC image, the name and possible loot. The page does not mention 3D model previews.
  - Sharing: an experimental `/rs telemetry export` sends data to a web portal for respawn estimates.
  
  — [CurseForge: RareScanner](https://www.curseforge.com/wow/addons/rarescanner)
- Other kill counters: KillTracker (Swiftyy), Creature Kill Tracker, KillTrack (the import source for KillDex), and Deathlog (Hardcore). — [Search results: CurseForge KillTracker](https://www.curseforge.com/wow/addons/killtracker-swiftyy), [KillTrack](https://www.curseforge.com/wow/addons/killtrack), [Deathlog](https://www.curseforge.com/wow/addons/deathlog)

### Inferences
- Quantum-Seer is **not** the first "kills become cards with album + 3D models + shine-by-rarity" addon on Forever, because Adventure Journal launched about late Oct 2026. Its differentiators would be:
  - Warlock-exclusive capture via Drain Soul (a deliberate, skill-gated act rather than every kill)
  - explicit foil and 1st Edition mechanics with published odds
  - zone and type sets with completion
  - sharing
- Restricted instance identity is a hard constraint. Design capture around open-world Drain Soul, and expect dungeon or raid captures to degrade. Soul Journal's after-combat recovery (Section 6) is a fallback pattern. Consider whether dungeon or boss cards should exist at all, or be resolved from a curated bossID list via XP or loot events.
- Shipping a static creature DB (QuestieDB or CMaNGOS-derived) is the norm for showing set totals and silhouettes. Yonder deliberately hides totals, while Adventure Journal reportedly shows silhouettes. For set completion motivation, showing totals and silhouettes is preferable (see Section 4).

### Gaps
- No page stated SavedVariables sizes or file formats for any of these addons. To measure them, inspect sources (KillDex and Adventure Journal are All Rights Reserved, so sources were not fetched).
- I could not confirm how Adventure Journal renders models (PlayerModel vs ModelScene) or pools them. Rarity (the mount-tracking addon), NPCScan, PetTracker and Questie internals were not fetched within budget.
- The official patch notes behind "creature hidden in instances since 12.0" were not fetched. The only sources are the KillDex and Yonder author statements.

## 2. Trading-card, foil and rarity presentation: existing addons and the official WoW TCG

### Takeaway
Within WoW addons, only Adventure Journal does card presentation with rarity-based "shine". No addon with foil, 1st Edition or set-numbering mechanics was found. The official WoW TCG used **WoW item-quality colours for rarity** (Common = white, etc.) and Loot cards in Common, Rare and Very Rare tiers, which gives a ready, lore-consistent rarity palette.

### Cited Findings
- WoW TCG: published by Upper Deck from Oct 2005 to Mar 2010, then Cryptozoic from Mar 2010 to Aug 2013. It was succeeded by Hearthstone. — [warcraft.wiki.gg: WoW TCG](https://warcraft.wiki.gg/wiki/WoW_TCG)
- Loot cards were special versions of normal cards with a scratch-off code, redeemable for in-game items. Loot rarity tiers were Common, Rare and Very Rare. — [warcraft.wiki.gg: WoW TCG](https://warcraft.wiki.gg/wiki/WoW_TCG)
- The colours of the card number and set name "come directly from the MMORPG" item-quality colours, and Common cards are white. — [warcraft.wiki.gg: Trading Card Game](https://warcraft.wiki.gg/wiki/Trading_Card_Game) (via search snippet; fan wiki)
- War of the Ancients numbered its cards purely numerically, with no order of precedence, and Hero cards were excluded from set numbering because they were shared across the block. — [warcraft.wiki.gg: War of the Ancients (TCG)](https://warcraft.wiki.gg/wiki/War_of_the_Ancients_(TCG))
- Adventure Journal: "Rarer monsters have shinier cards", painted cards and a binder album. — [CurseForge: Adventure Journal](https://www.curseforge.com/wow/addons/adventure-journal)
- Soul Provenance uses rarity-like "Potency" tiers mapped from WoW classification: Normal (Common), Rare/Elite (Potent), Rare Elite (Greater) and World Boss (Ancient). — [CurseForge: Soul Provenance](https://www.curseforge.com/wow/addons/soul-provenance)

### Inferences
- Mapping Common through Legendary to WoW quality colours (white, green, blue, purple, orange) matches both the official TCG and players' existing mental model. "Uncommon = green" comes from WoW item quality, and the TCG source only confirmed Common = white.
- Set numbering in "001/120" style with a separate block for special or hero-like cards follows WoW TCG precedent.

### Gaps
- I found no reliable details on WoW TCG foil treatment, foil insertion rates or "1st Edition" use. Foil-rate references must come from other TCGs (not researched within budget).

## 3. UI techniques: shimmer or foil, card frames, glows, toasts, model pooling

### Takeaway
LibCustomGlow (MIT, Stanzilla) is the standard library for rarity glows (Pixel, AutoCast and Button glow, keyed so a frame can carry several). Blizzard's AlertFrame queued-subsystem API is how achievement-style toasts are built on Retail, but its availability on Classic and Forever is unconfirmed. LibToast is an alternative with its own API. I found no dedicated sources on foil-shimmer techniques.

### Cited Findings
- LibCustomGlow provides `PixelGlow_Start(frame, color, N lines, frequency, length, thickness, xOff, yOff, border, key)`, `AutoCastGlow_Start` (particle groups of 4 particles, default 4 groups) and `ButtonGlow_Start(frame, color, frequency)`, which is based on Blizzard's glow. Defaults: colour {0.95,0.95,0.32,1} and ButtonGlow frequency 0.125. Each start function has a matching Stop. The `key` parameter allows multiple glows per frame. The readme advises listing LibCustomGlow and Masque in OptionalDependencies. MIT licence, last pushed May 2025. — [CurseForge: LibCustomGlow](https://www.curseforge.com/wow/addons/libcustomglow); [ecosyste.ms: stanzilla/libcustomglow](https://awesome.ecosyste.ms/projects/github.com%2Fstanzilla%2Flibcustomglow)
- LibOrbitGlow is a separate, newer library with six glows built from Blizzard UI art and a different API (`lib.Show(frame, "Pixel", ...)`). — [CurseForge: LibOrbitGlow](https://www.curseforge.com/wow/addons/liborbitglow)
- Achievement alerts are created with `AlertFrame:AddQueuedAlertFrameSubSystem(template, setupFunc, maxAlerts, maxQueued)`. Live frames can be restyled by iterating the subsystem's `alertFramePool`. — [WoWInterface thread](https://www.wowinterface.com/forums/showthread.php?p=314833); [WoWInterface archive t-53443](https://wowinterface.com/forums/archive/index.php/t-53443.html)
- LibToast-1.0 uses its own `RegisterToast` / `SpawnToast` API, independent of AlertFrame. — [WowAce: LibToast API](https://www.wowace.com/projects/libtoast-1-0/pages)
- LootAquiredMover repositions the Loot Acquired AlertFrame on Classic, which suggests AlertFrame exists in Classic clients. — [CurseForge: LootAquiredMover](https://www.curseforge.com/wow/addons/lootaquiredmover)
- Existing Forever Warlock addons use simpler capture feedback. Soul Provenance shows chat messages plus a "top-center banner" toggled with `/spv notices` and `/spv testnotice`. Soul Journal posts varied chat lines and shows a ghost model "turning slowly inside a crystal shard". — [Soul Provenance](https://www.curseforge.com/wow/addons/soul-provenance); [Soul Journal](https://www.curseforge.com/wow/addons/soul-journal)

### Inferences
- A practical stack:
  - LibCustomGlow for rarity glows: AutoCast for Epic and Legendary, Pixel for Rare.
  - A custom foil overlay texture: an additive-blend (`SetBlendMode("ADD")`) gradient whose texcoords scroll via OnUpdate or AnimationGroup Translation inside a clipping frame.
  - A custom toast frame, falling back to a self-managed queue if `AlertFrame:AddQueuedAlertFrameSubSystem` is absent on 1.60.1.
  
  This combines standard WoW UI practice with my engineering judgment. The foil technique specifically is not sourced.
- Album pages: the page-at-a-time model (Soul Journal) avoids rendering many 3D models at once. For a grid album, reuse a fixed pool of model frames per page (e.g. 9) and re-assign display IDs on page turn instead of creating frames per card.

### Gaps
- I found no source documenting foil or holographic shimmer techniques in WoW addons, and no source on PlayerModel or ModelScene pooling costs. I also found no warcraft.wiki.gg page for AlertFrame (the URL returned 404), so whether `AddQueuedAlertFrameSubSystem` exists on 1.60.1 needs in-client verification, e.g. `/dump AlertFrame.AddQueuedAlertFrameSubSystem`.

## 4. Game design guidance: rarity distribution, shiny and foil odds, pity, duplicates, set completion, fatigue

### Takeaway
Published benchmarks:
- Hearthstone (official, Mar 2023): golden Legendary 0.6% per pack with a 361-pack pity, normal Legendary 5% per pack with a 40-pack pity, and duplicate protection.
- Pokémon shiny: 1/8192 early generations, 1/4096 from Gen VI, with Shiny Charm and Masuda boosts.
- Marvel Snap: finish rarity tiers with late-unlocking "Ultimate" finishes.

Design writing agrees that sets should be finite, visible and fillable, that duplicates need a purpose, that rarity must be legible, and that completion needs an endpoint.

### Cited Findings
- Hearthstone official normal-pack rates (Blizzard support article 32545, Mar 14 2023):

  | Card type | Average chance per pack | Pity timer (packs) |
  |---|---|---|
  | Golden Common | 8.1% | 25 |
  | Golden Rare | 7.1% | 29 |
  | Golden Epic | 1.4% | 142 |
  | Golden Legendary | 0.6% | 361 |
  | Signature Legendary | 0.6% | 361 |
  | Epic | 21% | 10 |
  | Legendary | 5.0% | 40 |
  
  At least one Rare or better is guaranteed per 5-card pack, and Commons fill about 3.5 cards per pack. — [Hearthstone wiki: Card pack statistics](https://hearthstone.wiki.gg/wiki/Card_pack_statistics)
- Hearthstone duplicate protection:
  - Since Jun 2017, you get no duplicate Legendary until the set's Legendaries are complete.
  - Since Mar 26 2020, you get no more than 2 copies of any Common, Rare or Epic until you have 2 of every card of that rarity.
  
  — [Hearthstone wiki: Card pack statistics](https://hearthstone.wiki.gg/wiki/Card_pack_statistics)
- Pity timers are tracked per expansion, and the first 10 packs of a new set guarantee a Legendary. — [Hearthstone wiki: Pity Timer](https://hearthstone.wiki.gg/wiki/Pity_Timer); [hearthstone-decks.net](https://hearthstone-decks.net/?p=168918)
- Golden cards cost several times more dust, and crafting costs 4 to 8 times the disenchant value. For a Legendary, crafting costs 1600 dust (golden 3200). — [Hearthstone wiki: Crafting](https://hearthstone.wiki.gg/wiki/Crafting); [cardgamer.com](https://cardgamer.com/games/hearthstone/how-card-rarities-work-in-hearthstone-explained/)
- Pokémon shiny base rate:
  - 1/8192 in Gen II to V, and 1/4096 from Gen VI.
  - Masuda method: about 1/1638 (Gen IV), 1/1365 (Gen V), 1/683 (Gen VI+).
  - Shiny Charm: about 1/2731 (Gen V) and 1/1365 (Gen VI+), and up to 1/512 combined with Masuda.
  
  — [Bulbapedia: Shiny Pokémon](https://bulbapedia.bulbagarden.net/wiki/Shiny_Pok%C3%A9mon); [r/ShinyPokemon odds wiki](https://reddit.birdcat.cafe/r/ShinyPokemon/wiki/odds)
- Marvel Snap Infinity Splits:
  - Finishes are Foil, Prism, Ink and Gold, giving 128 split combinations per card.
  - Finish and flare rarity tiers: Foil = Rare, Ink and Gold Prism = Ultimate.
  - Gold and Ink do not appear until roughly the 5th or 6th split of a card. Sources disagree, with one saying Ink unlocks after the 3rd split.
  - Variants are alternate art treated as separate cards. Rare variants cost 700 Gold, Super Rare 1200 Gold and Ultimate 5000 Collector's Tokens.
  
  — [marvelsnapzone: Infinity Splits](https://marvelsnapzone.com/infinity-splits/); [progameguides: variant rarity](https://progameguides.com/marvel-snap/marvel-snap-variant-rarity-explained/); [mobilematters.gg](https://mobilematters.gg/gaming/marvel-snap-infinity-split-rates-surface-reveal-effects)
- Collection-design heuristics (practitioner opinion, not peer-reviewed):
  - Partial visibility of the goal beats none, and every visible gap must be fillable. — [mechanics playbook: set collection](https://mechanics-playbook-next.pages.dev/mechanics/set-collection/)
  - Collectors hit a "40–60% tipping point" after which they commit to finishing. — [Yu-kai Chou: collection sets](https://yukaichou.com/gamification-study/game-design-technique-collection-sets/)
  - Duplicates need secondary value, such as crafting or merging (e.g. 3 duplicates → a higher tier). — [Yu-kai Chou: merging loop](https://yukaichou.com/advanced-gamification/merging-loop-duplicate-drops-anticipation/); [Practical Game Design (O'Reilly)](https://www.oreilly.com/library/view/practical-game-design/9781787121799/82b3905b-ee78-4479-b8ed-1c2e281694dd.xhtml)
  - "If no one can see that your item is rare, much of its psychological value disappears." A completed collection loses its power, so plan post-completion rewards or chained sets. — [Yu-kai Chou: collection set design](https://yukaichou.com/gamification-analysis/collection-set-design-cd4-engagement-guide/)

### Inferences
- Rarity should come mostly from the creature (classification and level), not from RNG on the capture. That keeps every capture deterministic and respects the effort of Drain Soul. Reserve RNG for the foil roll.
- Foil rate: Hearthstone's golden rate per Common slot is roughly 2 to 8% per pack. Shiny 1/4096 is far too harsh for a few hundred captures per character. A foil rate of about 1 to 5% per capture, plus a bad-luck counter (e.g. guaranteed foil after N foil-less captures, as with Hearthstone pity), seems appropriate. This is my estimate, not sourced.
- "1st Edition" (the first capture of each creature) behaves like duplicate protection in reverse: it makes first captures special without RNG.
- Show set totals and silhouettes, since visible fillable gaps help completion. Exclude unobtainable entries such as instance-hidden mobs (Section 1) from completion counts, so no set becomes impossible.
- Duplicates should feed something, such as a "copies" counter, an upgrade to foil after N copies, or a Healthstone or Soulstone flavour sink.

### Gaps
- I did not research foil rates in physical TCGs (e.g. Pokémon TCG and MTG foil-per-pack rates), and found no peer-reviewed studies on grind fatigue in collection systems.

## 5. Inter-player sharing: addon messages, custom hyperlinks, TRP3 and WeakAuras

### Takeaway
`C_ChatInfo.SendAddonMessage` allows a 16-character prefix and a 255-byte message, with a per-prefix budget of 10 messages refilling at 1 per second. On Forever, SAY and YELL exist but CHANNEL does not. Forever 1.60.1 also blocks addon messages from Trial and Veteran accounts. Retail's `addon:` hyperlink type is local-only and cannot be sent in chat. For sharing, use AceComm (chunking over ChatThrottleLib) for card data, and an export string for out-of-game sharing.

### Cited Findings
- SendAddonMessage limits:
  - Prefix up to 16 characters. Message up to 255 characters, with bytes 1 to 255 allowed and NUL not allowed.
  - Classic-only SAY and YELL are heavily throttled and disallow some characters. WHISPER is limited to same or connected realms.
  - Per-prefix budget: 10 messages, refilling at 1 per second. Exceeding it returns `AddonMessageThrottle` (3). Whispers outside instances are exempt.
  - Sending too much across several prefixes can disconnect the client.
  - Return codes 0 to 12, e.g. 11 `AddOnMessageLockdown` and 12 `TargetOffline` (added in 12.0.0).
  - The wiki recommends ChatThrottleLib or AceComm.
  
  — [warcraft.wiki.gg: C_ChatInfo.SendAddonMessage](https://warcraft.wiki.gg/wiki/API_C_ChatInfo.SendAddonMessage)
- 12.1.7 PTR: "Trial and Veteran accounts can no longer send addon messages. Forever 1.60.1 has the same change." Classic 1.13.4 applied much harsher throttling, and Classic 4.4.0 added per-prefix throttling to match Retail. — [warcraft.wiki.gg: C_ChatInfo.SendAddonMessage](https://warcraft.wiki.gg/wiki/API_C_ChatInfo.SendAddonMessage)
- The `addon` hyperlink type (`addon:name:data`) was added in 10.1.0. These links are "local-only and cannot be sent in chat messages". Clicks are handled via `EventRegistry:RegisterCallback("SetItemRef", ...)` or by hooking `SetItemRef`. `SendChatMessage` may reject malformed links. 10.0.5 removed `shareachieve`, `shareitem` and `sharess`. The page does not say whether `addon:` links exist on Classic or Forever. — [warcraft.wiki.gg: Hyperlinks](https://warcraft.wiki.gg/wiki/Hyperlinks)
- AceComm-3.0 splits large payloads into chunks, sends them at intervals and reassembles them automatically, with a per-chunk progress callback. — [WowAce: AceComm-3.0 API](https://wowace.com/projects/ace3/pages/api/ace-comm-3-0)
- TRP3 protocols:
  - TRP2P, its own protocol, carries "almost every data exchange" and accepts any Lua value.
  - The Mary Sue Protocol (MSP) is a compatibility protocol: string values only, with 2-character field IDs.
  - A separate "Roleplay Broadcast Protocol" over a shared channel is used for map locations.
  
  — [TRP3 wiki: Communication system](https://github-wiki-see.page/m/Total-RP/Total-RP-3/wiki/Communication-system)
- Precedents for out-of-game sharing: KillDex's export string to killdex.com, RareScanner's telemetry export, and Yonder's external Ravenpost uploader. — [KillDex](https://www.curseforge.com/wow/addons/killdex); [RareScanner](https://www.curseforge.com/wow/addons/rarescanner); [Yonder page](https://www.curseforge.com/wow/addons/explorers-field-journal)

### Inferences
- A compact card record fits in one addon message, about 30 bytes: npcID + rarity + foil + 1st Edition flag + capture date + zoneID. "Show a card to party or guild" is therefore cheap. Full album sync needs AceComm and should be on demand only.
- The common pattern for clickable chat links in Classic-era addons is to post plain text such as `[Quantum-Seer: Defias Pillager ★]` and use `ChatFrame_AddMessageEventFilter` on receivers to rewrite it into a local clickable link. This is standard community practice that I did not source here. Senders without the addon see plain text.
- The Trial and Veteran restriction means some players can receive but never send. The UI should degrade gracefully.

### Gaps
- I could not confirm `addon:` hyperlink availability on Forever 1.60.1, so test it in-client. WeakAuras' share-link mechanism was not found in search, and I did not fetch the TRP3 or WeakAuras source.

## 6. Novelty check: existing Warlock "soul collection" addons

### Takeaway
The Warlock soul-collection idea is **already taken on Forever and Classic**, by Soul Journal (Forever-only, Oct 2026), Soul Provenance (Forever and Classic, Oct 2026) and SoulKeeper (Classic). None has trading-card mechanics: rarity tiers as cards, foil, 1st Edition, sets or albums. The card-collection idea exists separately in Adventure Journal, for all kills and any class. **Quantum-Seer's novel combination is Warlock-only Drain Soul capture with TCG presentation.** Each half has direct prior art less than two months old.

### Cited Findings
- **Soul Journal** (PhillypHenning, MIT, GitHub issue tracker):
  - Versions: WoW Forever 1.60.1 only, Warlock only. v1.1.0-forever on Oct 20 2026, with 219 downloads.
  - Capture: records souls when Drain Soul or Shadowburn produces a shard. Each record holds name, level, classification, creature type, race and class for players, zone and subzone, timestamp, player level and spell.
  - "Sparks": partial shards from Drain Soul ticks. If the target escapes, the soul is marked "slipped away" and shown as a faint ghost.
  - UI: right-click a shard or type `/souls`. One soul is shown per page as a ghost model "turning slowly inside a crystal shard". Clicking plays a sound, and repeated clicks can make the ghost emote. Tabs are Bound and Released.
  - Release tracking: assumes a FIFO spend order and records release reasons (Healthstone, Ritual of Summoning, vendor, trade, mail).
  - Tooltips and storage: shard tooltips show the newest soul, and data is stored per character.
  - Pitfalls it documents: hidden names and levels in combat or instances, recovered after combat from the targeted corpse or the XP message, otherwise logged as "Lost Soul" with a generic human ghost. Player souls always use the human ghost model.
  
  — [CurseForge: Soul Journal](https://www.curseforge.com/wow/addons/soul-journal)
- **Soul Provenance** (veraku, MIT):
  - Versions: Forever 1.60.1 (v1.10.0, Oct 23 2026) and Classic 1.15.9 (v1.8.3, Aug 2026), with 80 downloads.
  - Records: a permanent Soul Number per soul, and Potency tiers Common, Potent, Greater and Ancient. Each soul also gets a Temperament (8 personalities), an age-based State, a biography and earned titles.
  - Demon reclamation: rebinds the soul when a demon's shard is refunded.
  - UI: a "Ledger of the Damned" filterable list, Grimoire stats, Trophies, 8 addon achievements, a minimap button and a top-center capture banner.
  - Storage: `SoulProvenanceDB` plus `SoulProvenanceBackupDB`, with `/spv backup`. No sharing.
  
  — [CurseForge: Soul Provenance](https://www.curseforge.com/wow/addons/soul-provenance)
- **SoulKeeper** (Classic) saves the name and last words of the victim behind each soul shard. — [CurseForge: SoulKeeper](https://www.curseforge.com/wow/addons/soulkeeper) (search snippet only)
- Shard-management addons (Shard Grid, SoulMate, ShardLimiter, Necrosis Classic, Soul Ledger and Dark Ledger TBC) focus on counting or capping shards, not collection. — [search results, e.g. Necrosis Classic](https://www.curseforge.com/wow/addons/necrosis-classic), [Soul Ledger - TBC](https://www.curseforge.com/wow/addons/soul-ledger-tbc)

### Inferences
- Quantum-Seer should treat Soul Journal and Soul Provenance as reference implementations for the hard problems:
  - shard-to-soul binding with FIFO spend
  - Drain Soul tick vs kill classification
  - demon shard refunds
  - recovering names hidden in instances
  - backup SavedVariables
  
  Both are MIT-licensed, so their code can legally inform implementation with attribution.
- Users may run Soul Journal or Soul Provenance alongside Quantum-Seer, since both use `/souls`. Avoid that slash command and consider interoperability.
- Card identity should be per creature (npcID) for the album, with per-capture instances for foil, 1st Edition and copies. That differs from Soul Provenance's per-soul identity and from Adventure Journal's per-kill model.

### Gaps
- I did not fetch the GitHub sources for Soul Journal or Soul Provenance to see their capture-detection code or SavedVariables layout. That is a recommended follow-up.
- I did not check Wago or WoWInterface for further Forever soul addons. There may be other recent ones given how fast this niche is growing.
