# Dunjin Modding Reference

Mods add enemies, relics, potions, and heroes to Dunjin via Steam Workshop. This document covers file structure, every content type, the full field/validation list for each, and the mod API available to hook code.

## Contents

1. [Setup & File Structure](#1-setup--file-structure)
2. [Enemies](#2-enemies)
3. [Relics](#3-relics)
4. [Potions](#4-potions)
5. [Heroes](#5-heroes)
6. [Hooks](#6-hooks)
7. [Mod API Reference](#7-mod-api-reference)
8. [Sandbox Restrictions](#8-sandbox-restrictions)

---

## 1. Setup & File Structure

Mods are Steam Workshop items — not manually-uploaded folders on disk that the game reads directly. You can both consume and publish them without leaving the game.

### Installing a mod

1. **Enable mods**: Settings → Advanced → Enable Mods. Stored as `META.modsEnabled`; takes effect after a restart.
2. **Subscribe** to a mod, either on the Steam Workshop website or from the **Browse** tab of the in-game MODS panel (searchable, paginated, with a subscribe/unsubscribe toggle per item — no need to leave the game).
3. **Restart the game.** On boot, if mods are enabled, the game calls a Steam bridge (`window.dunjinDesktop.steam.getWorkshopModsSync`) to read all subscribed Workshop items.
4. The **MODS** button on the title screen opens a panel with three tabs: **Installed** (enable/disable individual mods, view load errors), **Browse** (search and subscribe to Workshop items in-game), and **Upload** (publish your own mod — see below).
5. If mods are enabled but no Steam bridge is present (e.g. a browser build), mod loading is silently skipped — no error, just no mods.

### Publishing a mod (Upload tab)

Mod authors don't need any external tooling — the Upload tab drives Steam Workshop publishing directly:

1. **Pick your mod folder** — a local folder containing `manifest.json` and your content JSON files. The game reads the manifest immediately and pre-fills the title field from `manifest.name` if you haven't typed one yet.
2. **Pick a preview image** — Steam requires one; publishing is blocked without it.
3. Fill in **title** (required), **description**, and a **change note** (defaults to "Updated" if left blank).
4. Choose **visibility**: Private, Unlisted, Friends only, or Public.
5. Click publish. Progress is reported live (preparing → uploading files → uploading preview → finishing). Publishing again with the same folder updates the existing Workshop item (its id is remembered) rather than creating a new one.

If your Steam account hasn't accepted the Workshop legal agreement yet, the upload either completes with a warning or is blocked outright until you accept it.

### Mod folder contents

A mod (synced from Workshop) is a folder containing:

- **`manifest.json`** — identity only:
  | Field | Notes |
  |---|---|
  | `id` | Required. Lowercase letters/digits only: `^[a-z0-9]+$`. Becomes the mod's namespace. |
  | `name` | Display name. |
  | `version` | Free-form string. |
  | `author` | Free-form string. |

- **One or more content JSON files**, each with any of:
  ```json
  { "enemies": [...], "relics": [...], "potions": [...], "heroes": [...] }
  ```
  A mod doesn't need all four. `heroes` may also be spelled `classes` — both keys are read. As a shorthand, an enemies file may also just be a bare JSON array instead of `{"enemies": [...]}`.

- Duplicate `manifest.json` ids across subscribed items: the second one is rejected outright with a "duplicate mod id" error, and none of its content loads.
- Load order: mods load after the base game's content pools exist, and before any save is loaded (saves can reference mod content by id string).

### Namespacing

Every content item's `id` must:

- Match `^[a-z0-9]+_[a-z0-9_]+$` — lowercase, with at least one underscore separating the namespace from the rest.
- Start with your mod's manifest `id` plus an underscore (mod id `bogpack` → every item id starts with `bogpack_`).
- Not collide with any existing base-game or already-loaded mod relic/potion id (relics and potions share one id space), or, for heroes, any existing hero id.

**Enemies have one more constraint:** the display name (`n`) must be globally unique against every enemy in the base game and every other loaded mod, checked case-insensitively. The engine keys bestiary tracking, save resolution, and boss-encounter history by display name rather than id, so a duplicate name would silently corrupt another enemy's tracking data.

### Error handling

Bad items are rejected loudly, never silently dropped. Each rejection logs a specific reason (e.g. `bogpack_ward: rarity must be one of common|uncommon|rare|epic|legendary|mythic`) to the mod error log, viewable from the MODS panel after a restart. A malformed JSON file is reported per-file and skipped; other files in the same mod still load.

For a full picture during development, open the browser devtools console and run `dunjinMods()`. It prints a complete load report: Steam bridge connection status, every loaded mod with its version/author/item count, how many heroes/relics/potions/enemies were merged into the live pools, and every logged error in one place — faster than digging through the MODS panel one mod at a time.

### ID immutability

Once a save references an item id, that id is permanently baked into that save. Renaming or removing an id after players have used it will corrupt those saves — there's no migration path. Treat published ids as immutable; if an item needs a substantial redesign, ship a new id and leave the old one in place (even as a no-op).

---

## 2. Enemies

Enemies are pure data — no hooks, no code.

```json
{"enemies":[{"id":"bogpack_reaver","n":"Swamp Reaver","tier":"grunt","hp":70,"ac":9,"dmg":8,"shield":10,"armor":20,"atk":"poison","effect":"unsteady"}]}
```

| Field | Range / values | Notes |
|---|---|---|
| `id` | — | See namespacing above. |
| `n` | 1–40 chars | Display name. Globally unique, case-insensitive. |
| `tier` | `grunt` \| `vanguard` \| `elite` \| `boss` | Sets internal spawn-pool placement and encounter-tracking flags. |
| `hp` | 1–100,000 | |
| `ac` | 0–20 | Hard engine ceiling — the player's roll is itself capped at 20 (`Math.min(20, nat + bonus)`), so a higher AC would be unreachable by any roll and is rejected rather than published as a lie. |
| `dmg` | 0–1,000 | |
| `shield` | 0–100,000, optional (default 0) | Stripped by Arcane damage. |
| `armor` | 0–100,000, optional (default 0) | Stripped by Shadow damage. |
| `atk` | one of the ten attack types (below) | |
| `effect` | one of the 13 effect keys (below), optional | |

**Attack types:** `bite, crush, dark, fire, ice, lightning, pierce, poison, psychic, slash`.

### Effects apply at every tier

The engine does not gate effects on boss status — it reads `effect` unconditionally, and every consumer of it checks the effect key regardless of tier. The bestiary tooltip and combat ability badge display effects identically at every tier. Base-game enemies only put effects on bosses by design convention, not engine restriction — a modded grunt can carry any effect.

### Effect list (13)

| Key | Name | Effect |
|---|---|---|
| `numbed` | Trembling Hands | −2 hand size for the battle |
| `vulnerable` | Cracked Ward | You take +25% damage for the battle |
| `relicjam` | Faulty Toolbelt | Disables a random relic every turn |
| `unsteady` | Shaky Ground | −3 to all d20 rolls for the battle |
| `potionseal` | Sealed Satchel | Potions disabled for the battle |
| `sigilsour` | Hindrance | A random sigil is fully nullified each hand (scores no Mana or Mult); it changes each hand |
| `patternread` | Open Book | Your most-played hand type this run is fully nullified |
| `weakened` | Leaden Arms | You deal −25% damage for the battle |
| `sigilread` | Tipped Hand | Your most-played sigil so far this battle is fully nullified |
| `noescape` | Cornered Rat | 0 discards for the battle |
| `demand` | Solemn Order | A demanded sigil (changes every hand) must be played or you deal 50% less damage |
| `courtspurge` | Court's Purge | Face cards (J/Q/K) are fully nullified for the battle |
| `waning` | Waning Hour | −1 attack for the battle |

---

## 3. Relics

Relics come in two flavors: **stat-only** (pure JSON) and **hook-driven** (JavaScript). Prefer stat-only whenever the effect fits — it's less error-prone and gets Echo Chamber mirroring and Relic Jam suppression for free, since the engine's relic-summing logic already walks every relic (mod or base-game) and its Echo Chamber mirror the same way.

A relic needs at least one hook or one recognized field — an empty relic is rejected. Unknown fields are rejected loudly, never dropped silently.

`rarity` alone drives shop price, sell value, and the item's color/border everywhere it's displayed — the same lookup table base-game relics use. There's no separate price or sell-value field to set; picking the right rarity string is all that's needed.

### 3.1 Stat-only example

```json
{"relics":[{"id":"bogpack_ward","name":"Marsh Ward","rarity":"rare","desc":"Take 25% less Poison damage.","resistTypes":["poison"],"resistPct":0.25}]}
```

### 3.2 Stat fields

**Hand economy**

| Field | Range | Notes |
|---|---|---|
| `chips` | −999 to 999 | Flat Mana added to every hand |
| `mult` | −99 to 99 | Flat Mult added to every hand |
| `multX` | 0 to 10 | Mult multiplier, applied after flat mult |
| `multPerCard` | −20 to 20 | Mult per card in the played hand |
| `combo` | any `HAND_TYPES` key, or `any` | Bonus when that hand type is played. Requires at least one of `chips`/`mult`/`multX` also set. |
| `handSize`, `discards`, `hands` | −10 to 10 each | Run-wide resource modifiers. Floors: hand size 1, attacks (`hands`) 1, discards 0. Negative `hands`/`discards` apply at battle start only — picking up a relic with a negative value mid-combat has no effect on that battle. Negative `handSize` takes effect as you play cards down. |

**Per-sigil scoring** — all four schools are symmetric; each has both a Chips and a Mult field:

| School | Chips | Mult |
|---|---|---|
| Arcane | `arcChips` [−999, 999] | `arcMult` [−99, 99] |
| Divine | `divChips` [−999, 999] | `divMult` [−99, 99] |
| Shadow | `shadowChips` [−999, 999] | `shadowMult` [−99, 99] |
| Martial | `marChips` [−999, 999] | `marMult` [−99, 99] |

```json
{"relics":[{"id":"bogpack_warcry","name":"War Cry Standard","rarity":"uncommon","desc":"+1 Mult per Martial card scored.","marMult":1}]}
```

**Other scoring fields**

| Field | Range | Notes |
|---|---|---|
| `faceChips` | −999 to 999 | Per face card (J/Q/K) scored |
| `faceMult` | −99 to 99 | Per face card scored |
| `oddChips` | −999 to 999 | Per odd-rank card scored |
| `monkMult` | −99 to 99 | Mult when 3+ distinct suits are played in one hand |
| `rarityBoost` | 0 to 3 | Added weight toward rarer relics whenever a relic is offered (shop, event, treasure, camp). Stacks with the hero field `craftsmanRelicBoost`. |
| `curatorSlots` | 0 to 5 | Extra curator-style relic slots in the shop |

**Armor & combat rolls**

| Field | Range | Notes |
|---|---|---|
| `armor` | 0 to 200 | Flat armor granted |
| `roll` | −5 to 5 | Flat bonus to every d20 roll |
| `rollFloor` | 0 to 20 | Minimum on rolled results |
| `critWiden` | 0 to 5 | Lowers the natural roll needed to crit |
| `noFumble` | flag | Prevents natural 1s from being a full miss |
| `rerollOne` | flag | Rerolls a natural 1 once |

**Resistances & type interactions**

| Field | Range | Notes |
|---|---|---|
| `resistTypes` + `resistPct` | array of attack types + [0, 0.9] | % less damage taken from listed types. Must be declared together. |
| `vsTypes` + `vsTypesPct` | array of attack types + [0, 2] | Bonus damage dealt vs those attack types. Must be declared together. |
| `vsTier` + `vsTierPct` | one of `grunt`\|`vanguard`\|`elite`\|`boss` + [0, 2] | Bonus damage dealt vs that enemy tier. Must be declared together. |
| `hpDmgPct`, `armorDmgPct`, `shieldDmgPct` | −0.9 to 2 each | % modifier to damage dealt against HP / armor / shield layers specifically |

**Economy & misc**

| Field | Range | Notes |
|---|---|---|
| `luckBonus` | −0.5 to 0.5 | Added chance on event/treasure/camp rolls |
| `shopDiscount` | 0 to 0.75 | % off shop prices |
| `potionPotency` | −0.5 to 2 | Multiplier on potion effect strength |
| `healAfter` | 0 to 200 | Flat heal after battle win |
| `goldAfter` | 0 to 500 | Flat gold after battle win |
| `healAfterPct` | 0 to 0.5 | % max-HP heal after battle win |

Reserved keys that are never validated as stat fields (used for other purposes): `id, name, rarity, desc, descT, svg, hooks, echoCompatKeys, echoIncompatible, combo, vsTypes, vsTier, resistTypes`.

`descT` is optional on relics, same as on potions — a secondary description line base-game items use for a dynamic/numeric variant of the flavor text (e.g. showing the live rolled value of an effect). It's accepted and stored but not required.

### 3.3 Hooks on relics

See [section 6](#6-hooks) for the mechanism and full payload reference. Relics may declare any hook from the relic/hero hook list.

### 3.4 Echo Chamber mirroring

Mirroring is automatic for pure-stat relics and opt-in for hook-driven ones, since a hook can't be introspected the way a stat field can:

- **No hooks, has recognized stat fields, no explicit declaration** → auto-mirrored for free.
- **Has hooks, no explicit declaration** → defaults to `echoIncompatible` (silently un-mirrorable). This is deliberate: a silent no-op "mirror" would be worse for players than an obvious omission.
- **`echoIncompatible: true`** → explicitly blocks mirroring. Use for state-machine or once-per-run relics where duplication wouldn't make sense.
- **`echoCompatKeys: ["some_key", ...]`** → declares the relic mirror-eligible; each key becomes a truthy flag directly on the relic object (what the mirroring lookup tests for).

**Reading your own mirror count from inside a hook** — a hook only fires once per real event; Echo Chamber doesn't replay hook execution, only stat-field lookups. To scale a hook-driven relic's effect by its own mirror count, combine `api.id` (this relic's own id) with `api.run.relicCount(api.id)`, which already accounts for Echo Chamber mirrors pointed at it:

```json
{"hooks":{"onDamageTaken":"const n = api.run.relicCount(api.id); api.state.set((api.state.get(0)) + n);"}}
```

### 3.5 Persistent state

`api.state.get(default)` / `api.state.set(value)` gives per-item-id, JSON-serializable scratch space — auto-namespaced by the item's own id, so two mods (or two relics) never collide. A non-serializable value passed to `set` fails silently (logged as a mod error) rather than corrupting the save.

---

## 4. Potions

Potions are entirely hook-driven — there's no stat-only form. A potion's `use` hook is required; without it the potion is rejected.

```json
{"potions":[{"id":"bogpack_venomdraft","name":"Venom Draft","rarity":"uncommon","desc":"Deal 20 poison damage to the enemy.","combatOnly":true,"hooks":{"use":"api.dealDamage(20, {type:'martial'});"}}]}
```

| Field | Notes |
|---|---|
| `id`, `name`, `rarity`, `desc` | Same rules as relics. |
| `descT` | Optional. Max 300 chars — a secondary/technical description line. |
| `svg` | Optional icon, sanitized (see below). |
| `combatOnly` | Optional boolean. Restricts the potion to combat use. |
| `hooks.use` | **Required.** Fires when the potion is consumed. |
| `hooks.*` | Optional additional hooks — a narrower list than relics/heroes (see [section 6](#6-hooks)). |

No other fields are accepted — a potion only takes `id, name, rarity, desc, descT, svg, hooks, combatOnly`.

Inside `use`, `ctx.potencyMult` reflects the game's potion-potency scaling (including hero `potionPotency`/`potionPotencyMaxRarity` bonuses) — read it to scale your effect rather than hardcoding a flat number if you want the potion to interact normally with potency-boosting relics and heroes. Returning `false` from `use` signals that the potion failed to do anything (e.g. no valid target) and blocks it from being consumed; any other return value lets consumption proceed normally.

---

## 5. Heroes

Heroes combine base run stats with a `bonus` object (internally `run.clsBonus`). Fields in `bonus` draw from three pools:

1. **Relic-shared fields** — anything from [section 3.2](#32-stat-fields) works identically here.
2. **Hero-only numeric/flag/string fields** — read directly off `run.clsBonus` by dedicated engine checks, and never flow through the general relic-summing logic. (Setting one of these on a *relic* validates fine but silently does nothing — relics never populate `run.clsBonus`.)
3. **Hero-only array field** (`relicSpawnGate`) — same non-relic-summed rule, validated as a list of rarity strings.

Heroes may also declare `hooks`, using the same mechanism as relics.

```json
{"heroes":[{"id":"bogpack_ranger","name":"Bog Ranger","rarity":"rare","desc":"Marsh hero.","blurb":"Patient tracker.","perks":"100 HP · +1 hand size · +1 Mult per Arcane card scored","hp":100,"gold":25,"color":"#7cbf57","bonus":{"handSize":1,"arcMult":1},"relics":["oath"],"potions":["heal"]}]}
```

### 5.1 Base fields

| Field | Range / notes |
|---|---|
| `hp` | 1 to 500 |
| `gold` | 0 to 500, optional (default 0) |
| `blurb` | Required, max 200 chars |
| `perks` | Required, max 300 chars. Manually-written bullet summary shown on hero-select — nothing auto-generates it from `bonus`, so keep it in sync by hand. |
| `color` | Hex value like `#9fd0ff` (defaults to `#cdb47e`) |
| `relics`, `potions` | Arrays of starting item ids, max 3 entries each. Must be base-game ids or ids belonging to this same mod — a hero can't depend on another mod's content. |
| `cost` | Optional, 0–9999. If set, the hero is coin-purchasable instead of free-from-start. `unlockDesc` defaults to "Purchase for N coins" (max 120 chars) if not supplied. |
| `difficulty` | Optional, 1–5. Star rating on hero-select. Unrelated to the run's overall difficulty setting. |
| `mimic` | Optional. Offers this hero's perks as Mimic major/minor draw candidates. See [5.6](#56-mimic-compatibility-major--minor-perks). |

Mod heroes cannot use a JS-predicate `unlockReq` the way base-game heroes can (hooks are sandboxed strings; a predicate over save data would need a second sandbox). A mod hero is therefore either free-from-start or cost-gated — no "beat this boss to unlock" path is available to mods. Omitting both `cost` and `unlockReq` defaults to unlocked.

### 5.2 Hero-only numeric fields

Every hero-only mechanic in the base game has a field here that any custom hero can set — none are tied to a specific base hero id.

| Field | Range | Notes |
|---|---|---|
| `startShield` | 0–100 | Flat starting shield; also raises max-shield tracking |
| `knightArmor` | 0–50 | Flat starting shield; functionally similar to `startShield` but doesn't touch max-shield tracking |
| `faceMultX` | 0–3 | Mult multiplier per face card scored |
| `allOddMult` | −10 to 10 | Bonus Mult if every card in the hand is odd-ranked |
| `allEvenMultX` | 0–5 | Mult multiplier if every card in the hand is even-ranked |
| `arcSuitMul` / `arcSuitMonoMul` | 0–5 each | Mult multiplier when the hand's dominant perk suit is Arcane / when the hand is Arcane-only |
| `divSuitMul` / `divSuitMonoMul` | 0–5 each | Same pattern, Divine |
| `shadowSuitMul` / `shadowSuitMonoMul` | 0–5 each | Same pattern, Shadow |
| `marSuitMul` / `marSuitMonoMul` | 0–5 each | Same pattern, Martial |
| `bruteRageX` | 0–1 | Mult multiplier per hit taken this battle, stacking as the fight goes on |
| `maxPotions`, `maxRelicsBonus` | −3 to 6 each | Capacity adjustments |
| `critLow` | 2–20 | Crit threshold — lower widens the crit window (default trigger is a natural 20) |
| `glancePct` | 0–1 | Damage % dealt on a glancing blow (base default 0.5) |
| `bardLuck` | −0.5 to 0.5 | Added chance on event/treasure/camp rolls; stacks additively with relic field `luckBonus` |
| `goldInterestPct` | 0–1 | After a battle win, adds this % of current gold as bonus interest on top of the normal reward |
| `ratkingRelicMult` | 0–2 | Mult per relic held |
| `duelistCritX` | 1–5 | Crit damage multiplier, replacing the default 2× |
| `duelistCritWiden` | 0–5 | Lowers the natural roll needed to crit — a hero-only bonus-field equivalent of the relic field `critWiden` |
| `necroGlanceMult` | 0–20 | Mult granted on the hand following a glancing blow |
| `tankEliteMulX` | 0–5 | Mult multiplier when attacking a Vanguard or Elite enemy; also multiplies your own damage on a one-shot kill against one |
| `tankGruntDmgMulX` | 0–5 | Multiplier on incoming damage from Grunt-tier enemies specifically (bosses/vanguards/elites unaffected) |
| `craftsmanRelicBoost` | 0–1 | Added chance weight toward relic drops from events/treasure/camp, and feeds the rarity-weighting roll on any relic pick. Stacks additively with relic field `rarityBoost`. |
| `craftsmanOneShotChance` | 0–1 | Chance of an extra bonus relic when your first hand of a battle kills a full-HP enemy outright |
| `fatespinnerCritRollGain` | 0–5 | How much the natural-roll floor permanently increases per crit landed, up to the existing +5 cap |
| `bardDiscardMana` | 0–50 | Flat bonus Mana added to your *next* hand for every card discarded this turn |
| `broodmotherPostBattlePotions` | 0–10 | Grants N potions after every battle win (converted to gold if the potion belt is full) |
| `broodmotherPotionMana` | 0–10 | Every potion consumed permanently adds this much flat Mana to every hand for the rest of the run |
| `wardenArmorMultDivisor` | 1–100 | Pairs with the `wardenArmorMult` flag (default divisor 25 if that flag is set without this) |
| `graveBurialDivisor` | 1–52 | **Validated but not currently read by any scoring code.** `graveBurial`'s bonus is hardcoded at +0.25 Mult per missing card; this field has no observable effect. Treat as reserved. |
| `monkMultXFactor` | 1–5 | **Validated but not currently read anywhere.** The 3-distinct-suit case of `monkMultX` does not multiply by this field — see `monkMultX` below. Treat as reserved. |
| `monkMultXFactor4` | 1–5 | Pairs with `monkMultX` — the multiplier applied at 4+ distinct suits (default 2 if omitted). This is the field `monkMultX` actually reads. |
| `rangerMultXFactor` | 1–5 | Pairs with `rangerMultX` — the multiplier applied on High Card/Pair hands (default 2 if omitted) |
| `clericHealMult` | 0–5 | Every time the player absorbs `clericHealMultPer` HP of actual healing (overheal at full HP doesn't count; the remainder banks between heals), grants this much permanent Mult |
| `clericHealMultPer` | 5–200 | HP-healed threshold that triggers `clericHealMult` (default 50 if `clericHealMult` is set without it) |

### 5.3 Hero-only flags (true/false only)

| Flag | Effect |
|---|---|
| `wardenHeal` | All healing also restores shield instead of just HP |
| `wardenArmorMult` | +1 Mult per 25 shield currently held. The 25 is adjustable via `wardenArmorMultDivisor` |
| `monkMultX` | Changes how `monkMult` behaves by distinct suits played, **asymmetrically at 3 vs. 4 suits**: at exactly 3 distinct suits, grants a flat **+25 Mana** bonus — independent of the `monkMult` stat value and not adjustable by `monkMultXFactor` despite it being a validated field; at 4+ distinct suits, multiplies Mult by `monkMultXFactor4` instead (default 2). Without this flag, `monkMult`'s bonus is instead added flatly at 3+ suits as an ordinary stat field. |
| `graveBurial` | The weakest unenchanted card is removed from the deck after every battle; **+0.25 Mult per card missing from a full 52-card deck, fixed.** `graveBurialDivisor` is validated but does not adjust this rate (see above); card removal itself is not adjustable by any field. |
| `twinSchoolDeck` | The 52-card deck is restricted to two randomly-chosen schools at run start |
| `randomDeck` | The 52-card deck is entirely randomized (any school, any rank, duplicates allowed) |
| `rangerMultX` | Doubles Mult on High Card/Pair hands. Multiplier adjustable via `rangerMultXFactor`. |
| `noDiscards` | Converts what would be discards into extra attacks instead |
| `duelistGlanceFumble` | A natural 1 becomes a glancing blow instead of a fumble |
| `necroRebornCrit` | A natural 1 is instead treated as a natural 20 (a crit). Stacks oddly with `duelistGlanceFumble` if both are set on one hero — the nat-1 rewrite checks run in a fixed order, so test any hero combining roll-altering flags. |
| `ratkingCommonOnly` | Every relic offered, sold, or found while this hero is active is restricted to common rarity. Predates and is narrower than `relicSpawnGate` (section 5.4), which can express the same restriction plus any other rarity subset — prefer `relicSpawnGate` for new heroes. |
| `fatespinnerRollDmg` | Multiplies all damage dealt by (natural roll ÷ 10) — a natural 20 deals double, a natural 4 deals well under normal. Pair carefully with anything that also floors or rerolls the natural die. |
| `startRandomCommonRelic` | Grants one random common-rarity relic at run start, bypassing `relicSpawnGate` entirely — this is a one-time grant, not a spawn-pool restriction |
| `templarPlasma` | A substantial, high-risk scoring rewrite: replaces `chips × mult` scoring with `((chips + mult) / 2)²`; as a tradeoff, triples enemy HP scaling and triples flee-damage scaling |

### 5.4 Hero-only array field

`relicSpawnGate` (array of rarity strings, e.g. `["epic","legendary"]`) — restricts which relic rarities this hero can find, from every source that draws from the relic pool (shop, event rewards, treasure, camp). The engine's single relic-spawn choke point checks this field, so declaring it covers every source automatically.

- Must be a non-empty array; each entry one of the six rarities. Unrecognized entries are rejected; duplicates are silently deduped.
- Does not affect relics granted outside the normal spawn pool — `startRandomCommonRelic` and Ratking's tribute-event relic grant both bypass the spawn gate and hand out a specifically common relic regardless of this field.

### 5.5 Hero-only string field

`potionPotencyMaxRarity` (one of the six rarities) — caps which potion rarities benefit from this hero's `potionPotency` bonus; any potion above the given rarity gets none of it.

### 5.6 Mimic compatibility (major / minor perks)

The Mimic is a base-game hero whose entire kit is randomized: at run start it rolls one **major** perk and one **minor** perk, each borrowed wholesale from a different hero, and plays the run with that combined bonus. This is driven by a lookup table of hand-authored major/minor perks — one per base-game hero, whichever ones that hero's designer chose to offer.

Mod heroes are not added to this table automatically. By default, a mod hero can be picked, played, and fought entirely normally — it simply never comes up as something the Mimic can borrow from. Declaring the optional `mimic` field opts a mod hero into that draw pool.

```json
{"heroes":[{
  "id":"bogpack_ward","name":"Marsh Warden","rarity":"rare","desc":"Marsh hero.",
  "blurb":"Keeper of the bog.","perks":"110 HP · +1 Mult per Arcane card scored",
  "hp":110,"bonus":{"arcMult":1},
  "mimic":{
    "major":{"desc":"Bog Ward: +25% resist to Poison damage, permanently","bonus":{"resistTypes":["poison"],"resistPct":0.25}},
    "minor":{"desc":"+1 relic slot","bonus":{"maxRelicsBonus":1}}
  }
}]}
```

| Field | Notes |
|---|---|
| `mimic` | Optional object. Omit entirely if this hero should never be offered to the Mimic — there's no minimum, and most mod heroes can simply skip it. |
| `mimic.major` | Optional. A single perk object: `{desc, bonus, relics?}` (see below). |
| `mimic.minor` | Optional. Either a single perk object, or an **array** of perk objects for a hero with more than one minor variant (the base game does this for Broodmother and Rat King — each entry becomes its own equal-odds draw, distinct from that hero's other minor(s)). Max 4 entries if an array. |

At least one of `major`/`minor` is required if `mimic` is present at all — an empty `mimic: {}` is rejected the same way an empty relic is. A hero can offer just a major, just a minor, or both, exactly like base-game heroes (Highroller, for instance, only offers a major).

**Perk object fields** (used identically for `major`, and for each entry under `minor`):

| Field | Notes |
|---|---|
| `desc` | Required, max 150 chars. Shown to the player as the rolled Mimic's perk description — keep it as terse as the base game's own entries (e.g. *"+1 Mult per Martial card scored"*). |
| `bonus` | Required, at least one field. Validated exactly like the hero's own top-level `bonus` field (section 5) — any relic-shared field from [3.2](#32-stat-fields), or any hero-only numeric/flag/string field from [5.2](#52-hero-only-numeric-fields)–[5.5](#55-hero-only-string-field). |
| `relics` | Optional. Array of starting item ids (max 3), granted only when this specific perk is the one rolled. Same rule as the hero's own `relics`/`potions`: must be base-game ids or ids belonging to this same mod. |

**How the draw works, and what modders should know:**

- A Mimic run rolls exactly one major (from every hero — base-game or modded — that has declared one) and one minor (same pool, excluding whichever hero supplied the major), each with equal odds among eligible entries. Your hero's major and minor are independent draws and can end up combined with perks from two different other heroes, or with each other, or not picked at all in a given run.
- The Mimic's own bonus is the sum of the rolled major's `bonus` and the rolled minor's `bonus` — not your hero's regular top-level `bonus`, which is never used by the Mimic. Design major/minor as self-contained bonuses, not as references to the rest of the hero's kit.
- `relics`/`potions` under `mimic.major` or `mimic.minor` are a separate grant from the hero's own top-level `relics`/`potions` — the Mimic never receives your hero's own starting items, only whatever the specific rolled perk(s) specify.
- Because `bonus` reuses the full hero-only field set, it's technically possible to put a whole-kit flag like `randomDeck`, `twinSchoolDeck`, `templarPlasma`, or `startRandomCommonRelic` on a *minor* perk. Nothing stops this validation-wise, but several of these were designed as one hero's entire identity (or, like `startRandomCommonRelic`, interact with a different part of the engine than the normal relic grant) and may combine strangely when layered under an unrelated major. Treat those flags as major-perk material, and stick to additive numeric/flag fields for minors, unless you've specifically tested the combination.
- As with any hero content, `mimic` perk ids aren't a thing on their own — the whole `mimic` block lives inside your hero's entry and is validated when that hero loads. There's no separate namespacing step; a bad `mimic` block simply fails your hero the same way a bad `bonus` or `hooks` block would (see [Error handling](#error-handling)), with the specific field named in the mod error log.
- Design intent, not an engine restriction, decides how "on-theme" the perks are — the Mimic doesn't care whether a major/minor makes narrative sense with the donor hero's kit, same as it doesn't for base-game heroes.

If a save's rolled major/minor donor is later removed (mod disabled/uninstalled), that slot's bonus just stops applying on reload — same missing-mod toast/log as a dropped relic or potion, not a crash.

---

## 6. Hooks

Relics, potions, and heroes can all declare a `hooks` object. Every hook body receives `(api, ctx)` — `api` is the mod API (section 7); `ctx` is a payload specific to that hook.

### 6.1 Where the hook fires and what it can change

Two shapes exist:

- **Mutating hooks** fire mid-calculation. Whatever the hook does to the relevant `api` value (via `api.addMult`/`api.addMana`/`api.reduceDamage`/etc., or directly returning it — see each hook's row) feeds back into the actual result.
- **Observational hooks** fire after the fact, for reacting only — nothing you do in them changes the outcome that already happened.

| Hook | Fires | `ctx` payload | Mutating? |
|---|---|---|---|
| `onScoreHand` | While a hand's chips/mult are being calculated | `{handType, cardCount}` | Yes — `api.addMana`/`api.addMult`/`api.multiplyMult` feed into the final chips/mult |
| `onHandResolved` | After a hand is fully scored and played | `{chips, mult, handType}` | No |
| `onDamageTaken` | While incoming damage is being applied to the player | `{amount, raw, ignoreArmor, cause, type}` | Yes — `api.reduceDamage`/`api.increaseDamage` change the final amount |
| `onHeal` | While a heal is being applied to the player | `{amount}` | Yes — `api.reduceHeal` lowers the final amount |
| `onBattleStart` | At the start of combat | `{enemyName, boss, elite}` | No |
| `onBattleEnd` | At the end of combat, win or lose | `{outcome: "win"|"lose", enemyName, boss}` (`boss` only present on win) | No |
| `onDiscard` | When cards are discarded | `{count, discardsLeft}` | No |
| `onShopEvent` | On a shop transaction | `{event, kind, cost}` (fields vary by event) | No |
| `onPickup` | When a relic is picked up | `{relicId}` | No |
| `onRoll` | After a d20 roll resolves | `{roll, total, bonus, mode, crit, fumble, hit, glance, ac, critThreshold}` | No |
| `onNodeEnter` | When entering a map node | `{type, depth, x, combatStreak}` | No |
| `onPotionUse` | When a potion is consumed | `{potionId, rarity}` | No |
| `onRelicRemoved` | When a relic is discarded or sold, before removal | `{relicId, isMod, goldGained}` | No |
| `onDamageDealt` | After the player deals damage, once shield/armor/HP layer-peeling is resolved | `{hp, shield, armor, total, mode, crit}` (`total` = shield+armor+HP combined). Fires unconditionally, including on 0-damage hits, and reports damage *before* any later cursed-enemy multiplier is applied. | No |
| `onCardAdded` | When a card is added to the deck, from any source (mod, base relic, shop, etc.) | `{uid, r, s, ench, toDraw}` | No |
| `onCardRemoved` | When a card is removed from the deck, from any source | `{uid, r, s, ench}` | No |

`onCardAdded`/`onCardRemoved` are fire-and-observe broadcasts — they tell you a card was added/removed, not *why*.

### 6.2 Which content types get which hooks

- **Relics and heroes** share the full list above.
- **Potions** get a narrower set — the same list minus `onShopEvent`, `onPotionUse`, and `onRelicRemoved` — plus a potion-only `use` hook (required). `use` fires when the potion is consumed, with `ctx = {potencyMult}` (see section 4). Returning `false` from `use` blocks consumption; anything else lets it proceed.

### 6.3 Mod effects (`api.armEffect`)

A potion's `use` hook can call `api.armEffect(...)` to register a temporary or permanent buff/debuff under the potion's own id. Once armed, the engine looks that id up **in the potion pool** each time it fires relic-style hooks — so this mechanism is specifically for potions to grant a multi-turn or multi-battle effect rather than a single instant one; calling it from a relic or hero hook arms an effect that will never be found and so never fires. See `armEffect`/`clearEffect` in section 7 for the exact interface.

### 6.4 Requirements & length limits

- A relic needs at least one hook or one recognized stat field.
- A hero may have zero hooks and zero bonus fields and still be valid — there's no minimum for heroes.
- A potion always needs at least the `use` hook.
- Hook bodies are capped at 4,000 characters (see section 8 for full sandbox rules).

---

## 7. Mod API Reference

Every hook body receives `(api, ctx)`. `api` exposes read accessors under `api.run` / `api.combat` / `api.account`, plus top-level action methods.

### 7.1 `api.id`

The id of the relic/potion/hero currently executing this hook. Use this instead of hardcoding your own id string — especially combined with `api.run.relicCount(api.id)` for self-mirror-counting (section 3.4).

### 7.2 `api.run` (read-only accessors)

| Method | Returns |
|---|---|
| `gold()` | Current gold |
| `hp()` / `maxHp()` | Current / max HP |
| `depth()` | Current run depth |
| `deckSize()` | Card count in the full deck |
| `deck()` | Array of `{r, s, ench}` card copies |
| `hand()` | Array of `{r, s, ench}` card copies, current hand |
| `relics()` | Array of `{id, rarity, jammed}` |
| `relicCount(id)` | Live copies of a relic by id, **including Echo Chamber mirrors** (direct or chained). Works for stat-only and hook-driven relics, as long as the target isn't `echoIncompatible`. |
| `hasRelic(id)` | Shorthand for `relicCount(id) > 0` |
| `discardsLeft()` / `handsLeft()` | Remaining discards / hands this battle |
| `armor()` | Total armor (relic sum + Ironform bonus) |
| `shield()` | Current shield |
| `maxRelics()` / `maxPotions()` | Current capacity |
| `potions()` | Array of `{id, name, rarity}` |
| `handSizeMax()` | Max hand size |
| `upgrades()` | Copy of `run.upgrades` |
| `loop()` | Current endless-mode loop number |
| `endless()` | Whether in endless mode |
| `difficulty()` | Run difficulty string |
| `combatStreak()` | Consecutive-combat streak counter |
| `luckyDie()` | Whether Lucky Die is active |
| `fated()` | `{armed, hand, factor}` |
| `echo()` | `{charges, copies}` |
| `rebound()` | Current rebound-heal percentage |
| `relicSellValues()` | Array of `{id, sell}` |
| `nodeType()` | Type of the current map node |
| `seeded()` | Whether this is a seeded run |
| `handUpgrade(key)` | `{chips, mult}` bonus banked for a given hand type |
| `drawPile()` / `discardPile()` | Array of `{r, s, ench}` card copies |
| `goldSpent()` | Total gold spent this run |
| `deckAdds()` | Count of cards added to the deck this run |
| `handPlayCounts()` | Copy of per-hand-type play counts |
| `suitPlayCounts()` | Copy of per-suit play counts |
| `deckCounts()` | `{bySuit, byRank, enchanted, total}` |

### 7.3 `api.combat` (read-only, combat only)

| Method | Returns |
|---|---|
| `enemy()` | `{n, hp, maxHp, armorHp, shield, ac, atkType, effect, boss, elite, vanguard, cursed}` or `null` outside combat |
| `handType()` | The hand type key currently being scored (`onScoreHand` only) |
| `handsMax()` / `discardsMax()` | Battle caps |
| `enemyAttackType()` | Current enemy's attack type |
| `sigilCounts()` | Copy of sigils played so far this battle |
| `demandSigil()` | Current Demand-effect sigil, if any |
| `handTypeCounts()` | Copy of hand types played so far this battle |
| `discardsUsed()` | Discards used so far this battle |
| `tookHit()` | Whether the player has taken a hit this battle |
| `cardsPlayed()` | Cards in the hand currently being scored |
| `roll()` / `rollTotal()` / `rollBonus()` / `rollMode()` / `critThreshold()` | Last roll's raw natural, total, bonus, mode string, and crit threshold |
| `isCrit()` / `isFumble()` / `isHit()` / `isGlance()` | Booleans derived from the last roll's mode |

### 7.4 `api.state` (persistent scratch space)

See section 3.5.

### 7.5 `api.account` (read-only)

| Method | Returns |
|---|---|
| `discoveryPct()` | Fraction of base-game relics/potions the player has seen |
| `unlockAllUsed()` | Whether the player has ever used an unlock-all cheat/option |

### 7.6 Top-level action methods

All action methods are no-ops (returning `0`/`false`/`null` as appropriate) outside their required context (a live run, combat, etc.), and all clamp their inputs to sane bounds.

| Method | Effect |
|---|---|
| `dealDamage(n, opts)` | Deals `n` damage to the current enemy. `opts.type` may be `arcane`\|`martial`\|`divine`\|`shadow` to route it through that school's damage type; `opts.armorOnly` restricts it to the armor layer. |
| `heal(n)` | Heals the player |
| `gainArmor(n)` | Grants armor |
| `addGold(n)` / `spendGold(n)` | Adds/spends gold (spend fails if insufficient) |
| `addMult(n)` / `multiplyMult(x)` | Adds to / multiplies the in-progress Mult (`onScoreHand` only — mutates the scope, not `run` directly) |
| `addMana(n)` | Adds to the in-progress Mana/chips (`onScoreHand` only) |
| `drawCards(n)` | Draws up to `n` cards, capped by hand-size max |
| `toast(msg)` | Shows a toast notification (HTML-escaped, 200-char cap) |
| `sfx(name)` | Plays a sound effect — must be one of `relic, potion, coin, shieldbreak, hit, crit, win, dealHand, click, error` |
| `upgradeHandType(handKey, opts)` | Permanently upgrades a hand type's base chips/mult (`opts.chips` 0–200, `opts.mult` 0–20) |
| `addMaxHp(n)` | Adjusts max HP (1–9999), and current HP proportionally |
| `addHands(n)` / `addDiscards(n)` | Adjusts remaining hands/discards this battle |
| `addTempMult(n)` | Adds temporary Mult for the run (clamped ±99) |
| `grantRelic()` / `grantPotion()` | Grants a random base-game relic/potion the player doesn't already have (relic) or any base-game potion (potion), respecting capacity. Returns the granted id or `null`. |
| `addRelicSellValue(n)` | Adds to every held relic's sell-value bonus |
| `setLuckyDie(on)` | Toggles Lucky Die |
| `setFated(handKey, factor)` | Arms/disarms a Fated-hand bonus (`factor` 1–10, default 2). Pass `handKey: null` to disarm. |
| `addEchoCharges(n, copies)` | Adjusts Echo charges (0–99) and/or sets Echo copy count (1–10) |
| `setRebound(pct)` | Sets rebound-heal percentage (0–1) |
| `armEffect(opts)` | Registers a temporary/permanent mod effect under the calling item's id — `opts.hands` or `opts.battles` sets a duration in hands/battles; omitting both makes it permanent. See section 6.3. |
| `clearEffect()` | Removes the calling item's armed effect |
| `setNullSigil(sigil)` / `nullSigil()` | Sets/reads the currently-nullified sigil (like the `sigilsour` enemy effect). Pass `null` to clear. |
| `setDemandSigil(sigil)` | Sets the currently-demanded sigil (like the `demand` enemy effect). Pass `null` to clear. |
| `enchantCards(ench, which, n)` | Enchants up to `n` cards. `which` is `"played"` (the cards just played), a school name, or `"random"` (default: any unenchanted card in hand) |
| `addRelicSlots(n)` / `addPotionSlots(n)` | Adjusts capacity (1–20 cap) |
| `clearEnemyEffect()` | Clears the current enemy's effect |
| `setEnemyStat(key, val)` | Sets `hp`\|`shield`\|`armorHp`\|`ac` on the current enemy, clamped to sane bounds |
| `reduceDamage(n)` / `increaseDamage(n)` | Adjusts in-flight damage (`onDamageTaken` only; `increaseDamage` capped at +50) |
| `reduceHeal(n)` | Adjusts in-flight healing (`onHeal` only) |
| `hand.discardAll()` | Discards the whole hand and redraws to cap |
| `hand.discard(indices)` | Discards specific hand indices (array or single number) and redraws to cap |
| `hand.enchant(index, ench)` | Enchants a specific hand-index card |
| `deck.add(card)` | Adds a card (`{r: 2-14, s: school, ench?}`) to the deck (200-card cap). Returns the new card's uid. |
| `deck.remove(indices)` | Removes deck cards by index (array or single number), never below a 5-card minimum |

---

## 8. Sandbox Restrictions

Hook bodies are compiled with `new Function(...)`, not `eval`, and run with the following restrictions:

- **Length cap**: 4,000 characters per hook body.
- **No loops**: `while`, `for`, and `do` are banned in the source text — use `map`/`filter`/`reduce`/`forEach` instead.
- **No `constructor`, `eval`, `__proto__`/`__defineGetter__`/`__lookupGetter__`, `import`/`export`, or `debugger`** — all banned by pattern match on the hook source.
- **Shadowed globals**: `window, document, globalThis, self, top, parent, frames, opener, run, combat, META, RELICS, POTIONS, ENEMIES, localStorage, sessionStorage, indexedDB, fetch, XMLHttpRequest, WebSocket, Worker, Function, require, process, module, exports, navigator, location, history, alert, open, postMessage, Reflect, Proxy, saveMeta, saveGame, loadGame, applyDamage, damagePlayer, toast, sfx` are all passed in as function parameters bound to `undefined`, so referencing any of them by name inside a hook resolves to `undefined` rather than the real object — the only way to touch game state is through `api`.
- A hook that throws is caught, logged as a mod error (naming the offending item and hook), and treated as a no-op for that call — it doesn't crash the run.

### SVG icons (`svg` field on enemies, relics, potions, heroes)

Custom icons are optional and go through a strict sanitizer:

- Max 24,000 bytes, max 400 nodes.
- Must parse as valid SVG with a root `<svg>` element.
- Only these tags are allowed: `svg, g, path, circle, rect, ellipse, line, polygon, polyline, defs, linearGradient, radialGradient, stop, use, title`.
- Only a fixed allowlist of presentation/geometry attributes is allowed (`fill`, `stroke-*`, `d`, `x/y/cx/cy/r`, gradient stops, etc.) — no `style`, no `on*` event attributes, no scripts.
- `<script>`, `<foreignObject>`, `<iframe>`, `<image>`, `<style>`, `<animate>`, `<set>`, and `<handler>` elements are rejected outright, as is any element bearing an `on*` attribute.
- Internal `id`/`href`/`url(#...)` references are allowed only for local same-document gradient/`<use>` references, and are automatically namespaced to avoid colliding with the base game's own ids.
- If an `svg` is supplied but fails sanitization, the item still loads — with a placeholder icon and a logged mod error — rather than being rejected outright.
