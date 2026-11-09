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

- **`lib`** (optional, top level of any content file) — shared helper code for the whole mod. See [8.1](#81-shared-helpers-lib).

- Duplicate `manifest.json` ids across subscribed items: the second one is rejected outright with a "duplicate mod id" error, and none of its content loads.
- Load order: mods load after the base game's content pools exist, and before any save is loaded (saves can reference mod content by id string).

### Namespacing

Every content item's `id` must:

- Match `^[a-z0-9]+_[a-z0-9_]+$` — lowercase, with at least one underscore separating the namespace from the rest.
- Start with your mod's manifest `id` plus an underscore (mod id `bogpack` → every item id starts with `bogpack_`).
- **Relics, potions and heroes:** not collide with any existing base-game or already-loaded mod relic/potion id (relics and potions share one id space — heroes are checked against this id space too), and, for heroes, not collide with any existing hero id.
- **Enemies:** enemy ids are not checked for collisions against anything. Only the display name is checked (see below), but you should still keep enemy ids unique and namespaced, since the namespace prefix is enforced.

**Enemies have one more constraint:** the display name (`n`) must be globally unique against every enemy in the base game and every other loaded mod, checked case-insensitively. The engine keys bestiary tracking, save resolution, and boss-encounter history by display name rather than id, so a duplicate name would silently corrupt another enemy's tracking data.

**Relics, potions and heroes** all require a `name` (max 48 chars) and a `desc` (max 300 chars).

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
| `ac` | 0–20 | Hard ceiling at load time — the player's roll is itself capped at 20 (`Math.min(20, nat + bonus)`), so a higher AC would be unreachable by any roll and is rejected rather than published as a lie. (Hook code can push an enemy's AC higher at runtime via `setEnemyStat` — see section 7.6.) |
| `dmg` | 0–1,000 | |
| `shield` | 0–100,000, optional (default 0) | Stripped by Arcane damage. |
| `armor` | 0–100,000, optional (default 0) | Stripped by Shadow damage. |
| `atk` | one of the twelve attack types (below) | |
| `effect` | one of the 13 effect keys (below), optional | |

**Attack types (12):** `bite, crush, dark, fire, ice, lightning, pierce, poison, psychic, slash, steel, water`. Each carries the same hidden rider it has on base-game enemies (e.g. `slash` steals gold per attack, `steel` is −2 to your d20 rolls, `water` is −1 hand size). The same list is valid for relic `resistTypes` and `vsTypes`.

### Effects apply at every tier

The engine does not gate effects on boss status — it reads `effect` unconditionally, and every consumer of it checks the effect key regardless of tier. The bestiary tooltip and combat ability badge display effects identically at every tier. Base-game enemies only put effects on bosses by design convention, not engine restriction — a modded grunt can carry any effect.

**Bosses are the exception.** Every run rolls a random curse for its boss, so a boss's `effect` field is ignored. Leave it out on boss-tier enemies.

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

Every relic needs an `id`, a `name` (max 48 chars), a `rarity`, and a `desc` (max 300 chars). A relic also needs at least one hook or one recognized stat field — an empty relic is rejected. Unknown fields are rejected loudly, never dropped silently. This includes most hero-only fields in sections 5.2–5.5, which fail validation as unknown fields on relics. The exceptions are listed under **Hero scoring fields on relics** in 3.2.

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
| `oddChips` | −999 to 999 | Per odd-rank card scored (3, 5, 7, 9, A) |
| `monkMult` | −99 to 99 | Flat Mult when 3+ distinct suits are played in one hand. On a hero with the `monkMultX` flag this flat bonus is replaced — see `monkMultX` in 5.3. |
| `highCardMult` | −99 to 99 | Flat Mult on High Card hands only |
| `pairMult` | −99 to 99 | Flat Mult on every hand **except** High Card (despite the name, not just Pairs) |
| `chipsPerDiscard` | −100 to 100 | Mana per discard you still have left this battle (Hoarded Potential uses 10) |
| `zeroDiscMult` | −99 to 99 | Flat Mult while you have 0 discards left (Conviction uses 3) |
| `rarityBoost` | 0 to 3 | Added weight toward rarer relics whenever a relic is offered (shop, event, treasure, camp). Stacks with the hero fields `craftsmanRelicBoost` and `craftsmanRarityBoost`. |
| `curatorSlots` | 0 to 5 | Extra curator-style relic slots in the shop |

**Armor & combat rolls**

| Field | Range | Notes |
|---|---|---|
| `armor` | 0 to 200 | Flat damage reduction per hit taken |
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
| `shopDiscount` | 0 to 0.75 | % off shop prices (rerolls included) |
| `potionPotency` | −0.5 to 2 | Multiplier on potion effect strength |
| `healAfter` | 0 to 200 | Flat heal after battle win |
| `goldAfter` | 0 to 500 | Flat gold after battle win |
| `healAfterPct` | 0 to 0.5 | % max-HP heal after battle win |

**Hero scoring fields on relics**

These hero fields are also valid on relics. They stack with the hero's own value, respect Relic Jam, and count as stat fields for Echo Chamber auto-mirroring.

| Field | Range | Notes |
|---|---|---|
| `ratkingRelicMult` | 0 to 2 | Multiplicative: Mult × (1 + value × relics held) |
| `allOddMult` | −10 to 10 | Flat Mult if every played card is odd-ranked |
| `multPerAttack` | 0 to 5 | Flat Mult per attack already made this battle |
| `depthMana` | 0 to 20 | Flat Mana per floor reached |
| `calmStreakMult` | 0 to 5 | Flat Mult per non-combat node since your last battle |
| `rangerMissingCardMult` | 0 to 5 | Mult per card the hand is short of 5 |
| `twinsoulMonoMana` | 0 to 100 | Flat Mana on mono-sigil hands |

Reserved keys that are never validated as stat fields (used for other purposes): `id, name, rarity, desc, descT, svg, hooks, echoCompatKeys, echoIncompatible, combo, vsTypes, vsTier, resistTypes`.

`descT` is optional on relics but currently ignored: it passes validation and isn't stored. It's only used on potions.

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

Note that `relicCount` does not count a copy that is currently jammed (e.g. by the `relicjam` effect).

### 3.5 Persistent state

Three scopes are available, all JSON-serializable. A non-serializable value fails with a logged mod error rather than corrupting the save.

| API | Scope | Lifetime |
|---|---|---|
| `api.state.get(default)` / `set(value)` | This item id | This run, saved with it |
| `api.shared.get(key, default)` / `set(key, value)` | Every item in this mod | This run, saved with it |
| `api.persist.get(key, default)` / `set(key, value)` | Every item in this mod | Across runs, on this profile. 8,192 bytes per mod. Pass `undefined` to delete a key. |

`persist` differs between players, so anything it influences breaks seeded-run reproducibility. Use it for unlocks and tallies, not for rolls.

---

## 4. Potions

Potions are entirely hook-driven — there's no stat-only form. A potion's `use` hook is required; without it the potion is rejected.

```json
{"potions":[{"id":"bogpack_venomdraft","name":"Venom Draft","rarity":"uncommon","desc":"Deal 20 poison damage to the enemy.","combatOnly":true,"hooks":{"use":"api.dealDamage(20, {type:'martial'});"}}]}
```

| Field | Notes |
|---|---|
| `id`, `name`, `rarity`, `desc` | Same rules as relics (`name` max 48 chars, `desc` max 300 chars). |
| `descT` | Optional. Max 300 chars — a secondary/technical description line. |
| `svg` | Optional icon, sanitized (see below). |
| `combatOnly` | Optional boolean. Restricts the potion to combat use. |
| `hooks.use` | **Required.** Fires when the potion is consumed. |
| `hooks.*` | Optional additional hooks — a narrower list than relics/heroes (see [section 6](#6-hooks)). |

No other fields are accepted — a potion only takes `id, name, rarity, desc, descT, svg, hooks, combatOnly`.

Inside `use`, `ctx.potencyMult` reflects the game's potion-potency scaling (including hero `potionPotency`/`potionPotencyMaxRarity` bonuses) — read it to scale your effect rather than hardcoding a flat number if you want the potion to interact normally with potency-boosting relics and heroes. Returning `false` from `use` signals that the potion failed to do anything (e.g. no valid target) and blocks it from being consumed; any other return value lets consumption proceed normally.

Players start with **3 potion slots** (see `maxPotions` in 5.2).

---

## 5. Heroes

Heroes combine base run stats with a `bonus` object (internally `run.clsBonus`). Fields in `bonus` draw from three pools:

1. **Relic-shared fields** — anything from [section 3.2](#32-stat-fields) works identically here.
2. **Hero-only numeric/flag/string fields** — read directly off `run.clsBonus` by dedicated engine checks. These are only accepted on heroes (and Mimic perks); on a relic they are rejected as unknown fields, apart from the short list in 3.2.
3. **Hero-only array field** (`relicSpawnGate`) — same rule, validated as a list of rarity strings.

Heroes may also declare `hooks`, using the same mechanism as relics.

**Every base-game hero perk is reproducible.** No hero mechanic is gated on a base hero's id — every one is driven purely by `bonus` fields, so a mod hero that sets the same fields gets the same behavior (and the same in-run Mana/Mult breakdown rows, labelled with your hero's name). Section 5.7 lists the exact `bonus` for every base hero's major and minor perk.

```json
{"heroes":[{"id":"bogpack_ranger","name":"Bog Ranger","rarity":"rare","desc":"Marsh hero.","blurb":"Patient tracker.","perks":"100 HP · +1 hand size · +1 Mult per Arcane card scored","hp":100,"gold":25,"color":"#7cbf57","bonus":{"handSize":1,"arcMult":1},"relics":["oath"],"potions":["heal"]}]}
```

### 5.1 Base fields

| Field | Range / notes |
|---|---|
| `id` | See namespacing. |
| `name` | Required, max 48 chars. |
| `desc` | Required, max 300 chars. |
| `rarity` | Optional on heroes (not used for anything). If given, must be one of the six rarity strings. |
| `hp` | 1 to 500 |
| `gold` | 0 to 500, optional (default 0) |
| `blurb` | Required, max 200 chars |
| `perks` | Required, max 300 chars. Manually-written bullet summary shown on hero-select — nothing auto-generates it from `bonus`, so keep it in sync by hand. |
| `color` | Hex value like `#9fd0ff` (defaults to `#cdb47e`) |
| `bonus` | Optional object — see the field pools above. |
| `relics`, `potions` | Arrays of starting item ids, max 3 entries each. Must be base-game ids or ids belonging to this same mod — a hero can't depend on another mod's content. |
| `startDeck` | Optional. Replaces the 52-card deck. Array of `{r, s, ench?, count?}` holding 5–200 cards in total (`r` 2–14, `s` a sigil, `count` 1–52). Can't be combined with `randomDeck` or `twinSchoolDeck`. |
| `startDeckAdd` | Optional. Same card format, 1–100 cards, added on top of the starting deck. Applied after `startDeck` and the deck flags. |
| `startHandUpgrades` | Optional. Object keyed by hand type: `{"pair":{"chips":10,"mult":2}}` (`chips` 0–200, `mult` 0–20). Starts the run with those upgrades banked. |
| `cost` | Optional, 0–9999. If set, the hero is coin-purchasable instead of free-from-start. `unlockDesc` defaults to "Purchase for N coins" (max 120 chars) if not supplied. |
| `difficulty` | Optional, 1–5 (default 1). Star rating on hero-select. Unrelated to the run's overall difficulty setting. |
| `mimic` | Optional. Offers this hero's perks as Mimic major/minor draw candidates. See [5.6](#56-mimic-compatibility-major--minor-perks). |

Mod heroes cannot use a JS-predicate `unlockReq` the way base-game heroes can (hooks are sandboxed strings; a predicate over save data would need a second sandbox). A mod hero is therefore either free-from-start or cost-gated — no "beat this boss to unlock" path is available to mods. Omitting both `cost` and `unlockReq` defaults to unlocked.

**Perk text conventions.** The base game names major perks only (e.g. *"Flourish: Each Crit grants +1 permanent Mult"*); minor perks are never given a name (e.g. *"+1 relic slot"*). Follow the same pattern in `perks` and in `mimic` descriptions so mod heroes read like the base roster. In-game hero notifications use the same rule: hero icon, then the major's name or the hero's name for a minor, then " - " and the effect.

### 5.2 Hero-only numeric fields

A value of `0` is normally treated as "not set" and dropped. The two **override** fields (`monkChips`, `graveDeckMult`) are the exception: an explicit `0` is kept, which is how you switch off the default value their paired flag would otherwise supply.

**Defense & capacity**

| Field | Range | Notes |
|---|---|---|
| `startShield` | 0–100 | Starting armor; also raises peak-armor tracking (what `tankHeal` restores up to) |
| `knightArmor` | 0–50 | Starting armor that doesn't touch peak-armor tracking |
| `maxPotions`, `maxRelicsBonus` | −3 to 6 each | Slot adjustments from the base **3 potion slots** and **6 relic slots** |

> **Removed:** `tankGruntDmgMulX` no longer exists. Using it gets the item rejected as an unknown field. Use `wardenGruntDmgMulX` (see Scoring below) for extra damage taken from Grunt-tier enemies.

**Scoring**

| Field | Range | Notes |
|---|---|---|
| `faceMultX` | 0–3 | Mult × (1 + value × face cards scored), **and +1 gold per face card scored**. Nullified faces (Court's Purge, soured/read sigils) don't count. |
| `allOddMult` | −10 to 10 | Flat Mult if every played card is odd-ranked |
| `allEvenMultX` | 0–5 | Mult multiplier if every played card is even-ranked |
| `arcSuitMul` / `arcSuitMonoMul` | 0–5 each | Mult multiplier when the hand's dominant perk suit is Arcane / when the hand is Arcane-only |
| `divSuitMul` / `divSuitMonoMul` | 0–5 each | Same pattern, Divine |
| `shadowSuitMul` / `shadowSuitMonoMul` | 0–5 each | Same pattern, Shadow |
| `marSuitMul` / `marSuitMonoMul` | 0–5 each | Same pattern, Martial |
| `twinsoulMonoMana` | 0–100 | Flat Mana on mono-sigil hands (every card the same sigil). No longer used by a base hero, still supported. |
| `twinsoulHtChips` | 0–20 | Each time a hand type is played, it permanently gains this much Mana for the rest of the run. Shares the per-hand-type store Scholar's Abacus writes to, so the two stack. |
| `rangerMissingCardMult` | 0–5 | Mult per card the played hand is short of 5 (a 1-card hand gets ×4 of this) |
| `multPerAttack` | 0–5 | Flat Mult per attack already made this battle (the first attack gets none, the third gets ×2 of this) |
| `depthMana` | 0–20 | Flat Mana per floor reached this run |
| `calmStreakMult` | 0–5 | Flat Mult per non-combat node (shop, event, camp, treasure) entered since your last battle. Entering a battle locks the count in for that fight and resets it — the mirror image of Pilgrim's Scar |
| `ratkingRelicMult` | 0–2 | Mult per relic held |
| `monkChips` | 0–100 | **Override.** Flat Mana when a hand contains 3+ distinct suits. Works standalone, or overrides the implicit **+15** that `monkMultX` supplies (set `0` to remove it). |
| `wardenGruntDmgMulX` | 0–5 | Multiplier on incoming damage from Grunt-tier enemies only |
| `wardenEliteMulX` | 0–5 | Mult multiplier vs Vanguards/Elites; also multiplies your damage on a one-shot kill against one |
| `wardenVanguardMana` | 0–100 | Every Vanguard defeated permanently adds this much flat Mana to every hand for the rest of the run |
| `wardenEliteMult` | 0–10 | Every Elite defeated permanently adds this much flat Mult to every hand for the rest of the run |
| `graveDeckMult` | 0–2 | **Override.** Mult per card your deck is below 52. Works standalone (pair with any deck-thinning), or overrides the implicit **+0.5** that `graveBurial` supplies (set `0` to remove it). |
| `bruteRageX` | 0–1 | Mult × (1 + value × hits taken this battle) |
| `bardDiscardMana` | 0–50 | Flat Mana added to your next hand per card discarded this turn |
| `broodmotherPotionMana` | 0–10 | Each potion consumed permanently adds this much flat Mana to every hand |
| `monkMultXFactor4` | 1–5 | Pairs with `monkMultX` — the multiplier at 4 distinct suits (default 2) |
| `rangerMultXFactor` | 1–5 | Pairs with `rangerMultX` — the multiplier on High Card/Pair (default 2) |
| `tankArmorMultDivisor` | 1–100 | Pairs with `tankArmorMult` (default 25) |

> **Removed:** `tankEliteMulX`, `tankVanguardMana` and `tankEliteMult` no longer exist. Using any of them gets the item rejected as an unknown field. Use `wardenEliteMulX`, `wardenVanguardMana` and `wardenEliteMult` (listed above) instead.

Gains from `wardenVanguardMana` and `wardenEliteMult` are saved with the run, shown as rows in the in-run Mana/Mult breakdown, and announced with a notification each time they grow.

**Rolls & crits**

| Field | Range | Notes |
|---|---|---|
| `critLow` | 2–20 | Crit threshold — lower widens the crit window (default natural 20) |
| `glancePct` | 0–1 | Damage % dealt on a glancing blow (default 0.5) |
| `duelistCritX` | 1–5 | Crit damage multiplier, replacing the default 2× |
| `duelistCritWiden` | 0–5 | Lowers the natural roll needed to crit (hero-only equivalent of relic `critWiden`) |
| `duelistRiposteMult` | 0–5 | Permanent Mult gained every time you land a crit |
| `necroGlanceMult` | 0–20 | Mult stored by a glancing blow, spent on your next full hit or crit |
| `fatespinnerCritRollGain` | 0–5 | Permanent d20 bonus gained per crit, up to +5 total; at +5 you become fumble-immune |

**Healing**

| Field | Range | Notes |
|---|---|---|
| `clericHealMult` | 0–5 | Permanent Mult gained per `clericHealMultPer` HP of actual healing (overheal doesn't count; the remainder banks between heals) |
| `clericHealMultPer` | 5–200 | Threshold for `clericHealMult` (default 50) |

**Economy & loot**

| Field | Range | Notes |
|---|---|---|
| `bardLuck` | −0.5 to 0.5 | Added chance on event/treasure/camp rolls; stacks with relic `luckBonus` |
| `goldInterestPct` | 0–1 | After a battle win, adds this % of current gold as interest |
| `freeRerolls` | 0–5 | Free shop rerolls per shop visit |
| `craftsmanRelicBoost` | 0–1 | Added chance toward relic drops from events/treasure/camp, and feeds rarity weighting on any relic pick. Stacks with relic `rarityBoost`. |
| `craftsmanRarityBoost` | 0–1 | Shifts relic rarity weighting toward rarer relics on any relic pick (the Craftsman's "boosted relic rarity"). Stacks with `craftsmanRelicBoost` and relic `rarityBoost`. |
| `craftsmanOneShotChance` | 0–1 | Chance of a bonus relic when your first hand kills a full-HP enemy |
| `broodmotherPostBattlePotions` | 0–10 | Potions granted after every battle win (gold if the belt is full) |

**Deprecated (validated for backward compatibility, no effect)**

| Field | Range | Notes |
|---|---|---|
| `graveBurialDivisor` | 1–52 | Never read. Use `graveDeckMult` to change Gravekeeper's per-card rate. |
| `monkMultXFactor` | 1–5 | Never read. Use `monkChips` (3+ suits) and `monkMultXFactor4` (4 suits). |

### 5.3 Hero-only flags (true/false only)

| Flag | Effect |
|---|---|
| `knightValor` | +10 Mana on every hand while at or above 50% HP; +2 Mult instead while below 50% |
| `tankHeal` | Healing can also restore armor, up to your peak armor this run |
| `tankArmorMult` | +1 Mult per 25 armor held (divisor via `tankArmorMultDivisor`) |
| `monkMultX` | Monk's suit engine. At 3+ distinct suits: flat Mana equal to `monkChips` (implicitly **+15** if `monkChips` isn't set). At 4 distinct suits: Mult × `monkMultXFactor4` (default 2) — **this multiplier only fires if you also have a non-zero `monkMult` from somewhere** (Monk sets `monkMult: 1` for exactly this reason). With the flag set, `monkMult`'s flat bonus is no longer added. |
| `graveBurial` | After every battle, the weakest unenchanted card is removed from the deck. Also supplies an implicit **+0.5 Mult per card below 52** unless `graveDeckMult` is set. |
| `twinSchoolDeck` | The deck is built from two randomly-chosen schools at run start |
| `randomDeck` | The deck is fully randomized (any school, any rank, duplicates allowed) |
| `rangerMultX` | Mult × `rangerMultXFactor` (default 2) on High Card/Pair hands |
| `noDiscards` | 0 discards; any bonus discards from other sources become extra attacks instead |
| `duelistGlanceFumble` | A natural 1 becomes a glancing blow instead of a fumble. No longer used by a base hero, still supported. |
| `necroRebornCrit` | A natural 1 is instead treated as a crit. Nat-1 rewrites run in a fixed order, so test any hero combining this with `duelistGlanceFumble`. |
| `ratkingCommonOnly` | Every relic offered, sold, or found is restricted to common rarity. Narrower predecessor of `relicSpawnGate` (5.4) — prefer that for new heroes. |
| `fatespinnerRollDmg` | Clean hits and crits deal damage × (d20 total ÷ 10), where the total includes roll bonuses and caps at 20 |
| `templarPlasma` | Replaces `chips × mult` with `((chips + mult) / 2)²`; triples enemy HP/defense scaling and flee-damage scaling |
| `templarReactor` | Overkill damage on any killing blow is banked as Mana (overkill ÷ ~67, or ÷ 200 with `templarPlasma`), capped at **+50**. The whole bank is released on your first scoring hand against an Elite or Boss. |
| `bruteIntimidate` | Each time an enemy attacks you, its ATK drops by 1 for the rest of the battle (minimum 0) |
| `reweaveFate` | Adds a **Reweave Fate** choice to every camp: instead of resting, your most-played hand this run becomes your fated hand. Disabled until you have played a hand type that isn't already your fate. Uses no randomness |
| `attackDebuffImmune` | Ignore every enemy attack-type rider (Water's −1 hand size, Steel's −2 roll, Slash's gold theft and so on), exactly like holding Crudestone |
| `startRandomRelic` | One random **common or uncommon** relic at run start (rarity-weighted). Respects `relicSpawnGate`/`ratkingCommonOnly` and skips relics you already hold. |
| `startRandomCommonRelic` | One random **common** relic at run start. Bypasses `relicSpawnGate` (skips archived, non-spawnable and already-held relics). If both relic flags are set, this one wins. |

### 5.4 Hero-only array field

`relicSpawnGate` (array of rarity strings, e.g. `["epic","legendary"]`) — restricts which relic rarities this hero can find, from every source that draws from the relic pool (shop, event rewards, treasure, camp). The engine's single relic-spawn choke point checks this field, so declaring it covers every source automatically.

- Must be a non-empty array; each entry one of the six rarities. Unrecognized entries are rejected; duplicates are silently deduped.
- The one exception is `startRandomCommonRelic`, which ignores the gate and always grants a common relic at run start (e.g. a hero gated to `["epic","legendary"]` with this flag still starts with one common). `startRandomRelic`, event relics and every other grant respect the gate.

### 5.5 Hero-only string field

`potionPotencyMaxRarity` (one of the six rarities) — caps which potion rarities benefit from this hero's `potionPotency` bonus; any potion above the given rarity gets none of it.

### 5.6 Mimic compatibility (major / minor perks)

The Mimic is a base-game hero whose entire kit is randomized: at run start it rolls one **major** perk and one **minor** perk, each borrowed from a different hero, and plays the run with that combined bonus. Every base-game hero currently offers exactly one major and one minor (section 5.7).

Mod heroes are not added to this pool automatically. By default, a mod hero can be picked and played normally but never comes up as something the Mimic can borrow. Declaring the optional `mimic` field opts it in.

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
| `mimic` | Optional object. Omit entirely if this hero should never be offered to the Mimic. |
| `mimic.major` | Optional. A single perk object: `{desc, bonus, relics?}`. |
| `mimic.minor` | Optional. Either a single perk object, or an **array** of up to 4 perk objects for a hero with several minor variants — each entry becomes its own equal-odds draw. (No base-game hero currently uses the array form.) |

At least one of `major`/`minor` is required if `mimic` is present — an empty `mimic: {}` is rejected. A hero can offer just a major, just a minor, or both.

**Perk object fields** (used identically for `major` and each `minor` entry):

| Field | Notes |
|---|---|
| `desc` | Required, max 150 chars. Shown as the rolled perk's description — keep it terse like the base game's (e.g. *"+1 Mult per Martial card scored"*). Name a major (`"Name: effect"`); leave a minor unnamed. |
| `bonus` | Required, at least one field. Validated exactly like the hero's own `bonus` — any field from [3.2](#32-stat-fields) or [5.2](#52-hero-only-numeric-fields)–[5.5](#55-hero-only-string-field). |
| `relics` | Optional. Array of starting relic ids (max 3), granted only when this specific perk is rolled. Base-game ids or ids from this same mod. There is no `potions` equivalent. |

**How the draw works, and what modders should know:**

- A Mimic run rolls one major (equal odds among every hero that declares one) and one minor (same pool, excluding the hero that supplied the major; each array entry counts as a separate candidate). Rolls are seeded, so a seeded run always gets the same pair.
- The Mimic's bonus is the sum of the rolled major's `bonus` and the rolled minor's `bonus` (numbers add; flags/strings from the minor overwrite). Your hero's top-level `bonus`, `relics`, `potions`, `startDeck`, `startDeckAdd` and `startHandUpgrades` are never used.
- **Implicit defaults are off in a Mimic run.** `monkMultX` no longer supplies its +15 Mana and `graveBurial` no longer supplies its +0.5 Mult per missing card — each must come from an explicit `monkChips`/`graveDeckMult` in the rolled perks. This is how the base game splits Monk and Gravekeeper into separate major/minor perks, so put the numeric part of a flag-plus-number mechanic in its own field if you want it to travel with the perk.
- Whole-kit flags (`randomDeck`, `twinSchoolDeck`, `templarPlasma`, the two start-relic flags) are technically allowed on a minor, but were designed as one hero's identity and may combine strangely. Test such combinations.
- A bad `mimic` block fails the whole hero, with the specific field named in the mod error log.
- If a save's rolled donor is later removed (mod disabled/uninstalled), that slot's bonus stops applying on reload — same missing-mod toast/log as a dropped relic or potion, not a crash.

### 5.7 Base-game hero perks (reference)

Every base-game hero's full kit is its major + minor combined, plus any hero-only extras listed under the table (Monk and Gravekeeper also rely on the implicit defaults above when played directly). Copy any row into a mod hero's `bonus` or `mimic` block to reproduce it exactly.

| Hero | Major `bonus` | Minor `bonus` |
|---|---|---|
| Knight | `{"knightValor":true}` | `{"armor":5}` |
| Cleric | `{"clericHealMult":1,"clericHealMultPer":50}` | `{"healAfterPct":0.08}` |
| Duelist | `{"duelistRiposteMult":1}` | `{"duelistCritX":2.5,"duelistCritWiden":1}` |
| Scavenger | `{"goldInterestPct":0.08}` | `{"shopDiscount":0.25}` |
| Craftsman | `{"craftsmanOneShotChance":0.5}` | `{"craftsmanRelicBoost":0.2,"craftsmanRarityBoost":0.532}` |
| Highroller | `{"faceMultX":0.25}` | `{"freeRerolls":1}` |
| Bard | `{"bardDiscardMana":3}` | `{"discards":2,"bardLuck":0.25}` |
| Fatespinner | `{"fatespinnerRollDmg":true}` | `{"fatespinnerCritRollGain":1}` |
| Twinsoul | `{"twinSchoolDeck":true}` | `{"twinsoulHtChips":1}` |
| Lottery Saint | `{"randomDeck":true}` | `{"startRandomRelic":true}` |
| Necromancer | `{"necroRebornCrit":true}` | `{"necroGlanceMult":3}` |
| Cosmonaut | `{"templarPlasma":true}` | `{"templarReactor":true}` |
| Tank | `{"tankHeal":true,"startShield":50}` | `{"tankArmorMult":true}` |
| Warden | `{"wardenEliteMulX":2,"wardenGruntDmgMulX":2}` | `{"wardenVanguardMana":3,"wardenEliteMult":0.5}` |
| Brute | `{"bruteRageX":0.5}` | `{"bruteIntimidate":true}` |
| Bombardier | `{"hands":3,"noDiscards":true}` | `{"glancePct":0.75}` |
| Mage | `{"handSize":2}` | `{"arcMult":0.5}` |
| Oddball | `{"oddChips":5,"allOddMult":3}` | `{"maxRelicsBonus":1}` |
| Monk | `{"monkMult":1,"monkMultX":true}` | `{"monkChips":15}` |
| Ranger | `{"rangerMultX":true}` | `{"rangerMissingCardMult":0.5}` |
| Broodmother | `{"broodmotherPotionMana":1,"broodmotherPostBattlePotions":2}` | `{"maxPotions":3,"maxRelicsBonus":-3}` |
| Rat King | `{"ratkingRelicMult":0.2}` | `{"maxRelicsBonus":3,"ratkingCommonOnly":true}` |
| Gravekeeper | `{"graveBurial":true}` | `{"graveDeckMult":0.5}` |

Hero-only extras that belong to the heroes themselves, not their Mimic perks:

- **Starting relics:** Cleric (Cleric's Blessing), Fatespinner (Steady Hand), Mage (Mage's Focus) and Tank (Bulwark Oath). Add them via `relics` if you want them.
- **Fatespinner's camp option:** his own `bonus` also has `{"reweaveFate":true}`, which isn't part of his Mimic perks.

---

## 6. Hooks

Relics, potions, and heroes can all declare a `hooks` object. Every hook body receives `(api, ctx)` — `api` is the mod API (section 7); `ctx` is a payload specific to that hook.

### 6.1 Where the hook fires and what it can change

Two shapes exist:

- **Mutating hooks** fire mid-calculation. Whatever the hook does to the relevant `api` value (via `api.addMult`/`api.addMana`/`api.reduceDamage`/etc., or directly returning it — see each hook's row) feeds back into the actual result.
- **Observational hooks** fire after the fact, for reacting only — nothing you do in them changes the outcome that already happened.

Hooks marked **pure** run with effects off, because the game calls them repeatedly for display. Only the scope methods named in their row work there.

| Hook | Fires | `ctx` payload | Mutating? |
|---|---|---|---|
| `onScoreHand` | While a hand's chips/mult are being calculated | `{handType, cardCount}` | Yes — `api.addMana`/`api.addMult`/`api.multiplyMult` feed into the final chips/mult |
| `onHandResolved` | After a hand is fully scored and played | `{chips, mult, handType}` | No |
| `onDamageTaken` | While incoming damage is applied to the player: enemy attacks, poison, Fire scorch and event damage. Fires after armor and shield. | `{amount, raw, ignoreArmor, cause, type}` | Yes — `api.reduceDamage`/`api.increaseDamage`/`api.multiplyDamage` change the final amount |
| `onHeal` | While a heal is being applied to the player | `{amount}` | Yes — `api.reduceHeal`/`api.increaseHeal` change the final amount |
| `onDeath` | HP reached 0, after Soul Anchor. Not fired when fleeing a boss, which is always fatal. | `{cause, enemyName}` | Yes — `api.survive(hp)` cancels the death (default 1 HP). The first hook to survive stops the rest. |
| `onRunStart` | Once, when a new run is created, after the deck, relics and potions are set. Not on resume. | `{seed, seeded, heroId, difficulty}` | No |
| `onBattleStart` | Before combat is built, so `api.combat.*` and combat actions don't work yet. Use `onTurnStart` with `ctx.first`. | `{enemyName, boss, elite}` | No |
| `onTurnStart` | Each player turn, once the hand is dealt | `{turn, first, handsLeft}` | No |
| `onTurnEnd` | After the enemy's attack, before redrawing. Not fired if the enemy died or you did. | `{turn, handsLeft}` | No |
| `onCardPlayed` | Once per card in a played hand, before `onHandResolved` | `{uid, r, s, ench, index, count, handType}` | No |
| `onBeforeRoll` | After the d20 is rolled, before hit or crit is decided | `{roll, bonus, total, ac, critThreshold}` | Yes — `api.setRoll`/`api.addRoll`/`api.setCritThreshold` |
| `onBeforeDamageDealt` | Before damage reaches the enemy's shield, armor and HP. Skipped for `api.dealDamage` called from inside this hook. Kill previews don't include it. | `{amount, mode, type}` | Yes — `api.reduceDamage`/`api.increaseDamage`/`api.multiplyDamage` |
| `onShopPrice` | **Pure.** Once per shop item whenever prices are calculated, after discounts | `{kind, id, rarity, base, price}` | Yes — `api.setPrice`/`api.multiplyPrice` |
| `onRelicOffer` | **Pure.** Once per relic whenever a spawn pool is built | `{id, rarity, isMod}` | Yes — `api.banOffer()` removes it from that pool. `grantRelic(id)` ignores this. |
| `onCampRest` | Rest or Deep Sleep at a camp, before healing | `{amount, kind: "rest"|"sleep"}` | Yes — `api.reduceHeal`/`api.increaseHeal` |
| `onEventResolved` | After an event or camp choice takes effect | `{title, choice, label, camp}` | No |
| `onEffectExpired` | An armed effect's duration ran out. Not fired by `clearEffect()`. | `{kind: "hands"|"battles"}` | No |
| `onBattleEnd` | At the end of combat, win or lose | `{outcome: "win"|"lose", enemyName, boss}` (`boss` only present on win) | No |
| `onDiscard` | When cards are discarded | `{count, discardsLeft}` | No |
| `onShopEvent` | On a shop transaction | `{event, kind, cost}` (fields vary by event) | No |
| `onPickup` | When a relic is picked up | `{relicId}` | No |
| `onRoll` | After a d20 roll resolves | `{roll, total, bonus, mode, crit, fumble, hit, glance, ac, critThreshold}` | No |
| `onNodeEnter` | When entering a map node | `{type, depth, x, combatStreak}` | No |
| `onPotionUse` | When a potion is consumed | `{potionId, rarity}` | No |
| `onRelicRemoved` | After a relic is discarded, sold or destroyed. Fires on your other relics and also on the removed relic itself, so it can clean up. | `{relicId, isMod, goldGained}` | No |
| `onDamageDealt` | After the player deals damage, once shield/armor/HP layer-peeling is resolved | `{hp, shield, armor, total, mode, crit}` (`total` = shield+armor+HP combined). Fires unconditionally, including on 0-damage hits, and reports damage *before* any later cursed-enemy multiplier is applied. | No |
| `onCardAdded` | When a card is added to the deck, from any source (mod, base relic, shop, etc.) | `{uid, r, s, ench, toDraw}` | No |
| `onCardRemoved` | When a card is removed from the deck, from any source | `{uid, r, s, ench}` | No |

`onCardAdded`/`onCardRemoved` are fire-and-observe broadcasts — they tell you a card was added/removed, not *why*.

### 6.2 Which content types get which hooks

- **Relics and heroes** share the full list above.
- **Potions** get a narrower set — the same list minus `onRunStart`, `onShopEvent`, `onPotionUse`, `onPickup`, and `onRelicRemoved` — plus a potion-only `use` hook (required). A potion's other hooks only fire while its effect is armed. `use` fires when the potion is consumed, with `ctx = {potencyMult}` (see section 4). Returning `false` from `use` blocks consumption; anything else lets it proceed.

### 6.3 Mod effects (`api.armEffect`)

`api.armEffect(...)` means two different things depending on who calls it.

**From a potion**, it keeps the potion's hooks firing after it's consumed. The engine looks the id up in the potion pool each time it fires hooks, until the effect expires.

**From a relic or hero**, it's a timer. Held relics and heroes already fire every hook, so arming never fires them a second time. Gate your own hooks on `api.effect()`, and react to the end in `onEffectExpired`:

```json
{"hooks":{
  "onBattleStart":"api.armEffect({hands:3});",
  "onScoreHand":"if (api.effect()) api.addMult(5);",
  "onEffectExpired":"api.toast('The surge fades');"
}}
```

- Each item id has one effect. Arming again replaces it.
- An effect armed in `onHandResolved` doesn't lose a hand to the hand that armed it. The same goes for `battles` in `onBattleEnd`.
- A relic's effect, counter and rows are removed when its last copy leaves.

### 6.4 Requirements & length limits

- A relic needs at least one hook or one recognized stat field.
- A hero may have zero hooks and zero bonus fields and still be valid — there's no minimum for heroes.
- A potion always needs at least the `use` hook.
- Hook bodies are capped at 4,000 characters (see section 8 for full sandbox rules). Move shared or longer code to `lib`.

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
| `relicCount(id)` | Live copies of a relic by id, **including Echo Chamber mirrors** (direct or chained), **excluding jammed copies**. Works for stat-only and hook-driven relics, as long as the target isn't `echoIncompatible`. |
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
| `handNumber()` | 1-based number of the hand being scored, or of the last hand played. Matches between the preview and the real score, so it's safe in `api.rng` keys. |
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

See section 3.5, which also covers `api.shared` and `api.persist`.

`api.effect()` returns this item's armed effect as `{hands, battles, permanent}`, or `null`. See section 6.3.

`api.lib` is this mod's helper object. See section 8.1.

### 7.5 `api.account` (read-only)

| Method | Returns |
|---|---|
| `discoveryPct()` | Fraction of base-game relics/potions the player has seen |
| `unlockAllUsed()` | Whether the player has ever used an unlock-all cheat/option |

### 7.6 Top-level action methods

All action methods are no-ops (returning `0`/`false`/`null` as appropriate) outside their required context (a live run, combat, etc.), and all clamp their inputs to sane bounds.

Enchantment arguments (`enchantCards`, `hand.enchant`, `deck.add`'s `ench`, `deck.enchant`) must be one of the game's twelve enchantment keys: `chips, mult, fortune, wax, block, vamp, chainmail, flow, gild, goldblock, tincture, item`. Anything else is rejected and logged as a mod error. Sigil/school arguments are `arcane`, `martial`, `divine` or `shadow`.

| Method | Effect |
|---|---|
| `dealDamage(n, opts)` | Deals `n` damage to the current enemy. `opts.type` may be `arcane`\|`martial`\|`divine`\|`shadow` to route it through that school's damage type; `opts.armorOnly` restricts it to the armor layer. |
| `heal(n)` | Heals the player |
| `gainArmor(n)` | Grants armor |
| `addGold(n)` / `spendGold(n)` | Adds/spends gold (spend fails if insufficient) |
| `addMult(n)` / `multiplyMult(x)` | Adds to / multiplies the in-progress Mult (`onScoreHand` only — mutates the scope, not `run` directly) |
| `addMana(n)` | Adds to the in-progress Mana/chips (`onScoreHand` only) |
| `drawCards(n)` | Draws up to `n` cards (max 20), capped by hand-size max |
| `toast(msg)` | Shows a toast notification (HTML-escaped, 200-char cap) |
| `sfx(name)` | Plays a sound effect — must be one of `relic, potion, coin, shieldbreak, hit, crit, win, dealHand, click, error` |
| `upgradeHandType(handKey, opts)` | Permanently upgrades a hand type's base chips/mult (`opts.chips` 0–200, `opts.mult` 0–20) |
| `addMaxHp(n)` | Adjusts max HP (1–9999), and current HP proportionally |
| `addHands(n)` / `addDiscards(n)` | Adjusts remaining hands/discards this battle. `addDiscards` with a positive value does nothing against an enemy with the `noescape` effect (unless the player has Wardstone active). |
| `addTempMult(n)` | Adds temporary Mult for the run (clamped ±99) |
| `grantRelic(id?)` / `grantPotion(id?)` | With an `id`, grants that exact relic/potion — base-game, your own mod's, or another loaded mod's (returns `null` and logs an error if the id isn't loaded). Without one, grants a random base-game item using the game's own rules: normal rarity weighting (plus Craftsman/rarity boosts for relics), and for relics the hero's spawn restrictions (`relicSpawnGate`, `ratkingCommonOnly`), no archived or non-spawnable relics, and nothing already held. Both respect capacity and return the granted id or `null`. |
| `addRelicSellValue(n)` | Adds to every held relic's sell-value bonus |
| `setLuckyDie(on)` | Toggles Lucky Die |
| `setFated(handKey, factor)` | Arms/disarms a Fated-hand bonus (`factor` 1–10, default 2). Pass `handKey: null` to disarm. |
| `addEchoCharges(n, copies)` | Adjusts Echo charges (0–99) and/or sets Echo copy count (1–10) |
| `setRebound(pct)` | Sets rebound-heal percentage (0–1) |
| `armEffect(opts)` | Registers a temporary/permanent mod effect under the calling item's id — `opts.hands` or `opts.battles` sets a duration in hands/battles; omitting both makes it permanent. Works differently for potions and for relics/heroes: see section 6.3. |
| `clearEffect()` | Removes the calling item's armed effect |
| `setNullSigil(sigil)` / `nullSigil()` | Sets/reads the currently-nullified sigil (like the `sigilsour` enemy effect). Pass `null` to clear. |
| `setDemandSigil(sigil)` | Sets the currently-demanded sigil (like the `demand` enemy effect). Pass `null` to clear. |
| `enchantCards(ench, which, n)` | Enchants up to `n` cards. `which` is `"played"` (the cards just played), a school name, or `"random"` (default: any unenchanted card in hand) |
| `addRelicSlots(n)` / `addPotionSlots(n)` | Adjusts capacity (1–20 cap). Persists through later slot recalculations. |
| `rng(key)` | See below. |
| `clearEnemyEffect()` | Clears the current enemy's effect |
| `setEnemyStat(key, val)` | Sets `hp`\|`shield`\|`armorHp`\|`ac`\|`dmg` on the current enemy, clamped to sane bounds (`ac` clamps to 0–30 here, unlike the 0–20 limit in `setEnemyFlag` and in enemy JSON). `dmg` is the enemy's ATK and updates its badge immediately. |
| `setEnemyFlag(key, val)` | Combat only. `key` is `cursed` or `poisonTick` (set to `!!val`), or `ac` (0–20). Returns `false` for any other key. |
| `reduceDamage(n)` / `increaseDamage(n)` / `multiplyDamage(x)` | Adjusts in-flight damage (`onDamageTaken`, `onBeforeDamageDealt`). `increaseDamage` is capped at +50 per call, `multiplyDamage` at ×10. |
| `reduceHeal(n)` / `increaseHeal(n)` | Adjusts in-flight healing (`onHeal`, `onCampRest`) |
| `survive(hp)` | `onDeath` only. Cancels the death and sets HP (default 1, capped at max HP). |
| `setRoll(n)` / `addRoll(n)` / `setCritThreshold(n)` | `onBeforeRoll` only. `setRoll` sets the natural roll (1–20), and setting 1 becomes 2 if the player is fumble-immune. `addRoll` adds to the bonus (±20 per call). `setCritThreshold` takes 2–20. |
| `setPrice(n)` / `multiplyPrice(x)` | `onShopPrice` only. Rounded, minimum 0. |
| `banOffer()` | `onRelicOffer` only. |
| `removeRelic(target, opts)` | `target` is a relic id, a relic index, or empty for this relic. Destroys it for no gold, or pays its sell value with `{sell:true}`. Echo Chamber targets, slot caps and `onRelicRemoved` are handled for you. Not allowed during `onScoreHand`. Returns `true` on success. |
| `removePotion(target)` | `target` is a potion id or index. No gold. Can't remove the potion being used. Returns `true` on success. |
| `setCounter(value)` | Shows a badge of up to 4 characters on this relic's chip. Pass `null` to clear. Relics only, since heroes and potions have no chip. |
| `setRow(kind, value, opts)` | Adds a row for this item to the held section of the Mana/Mult breakdown. `kind` is `mult` or `mana`. `opts.how` is the description (100 chars); `opts.x` shows it as a multiplier and leaves it out of the total. Pass `null` or `0` to clear. **Display only:** it doesn't change scoring, so call it alongside `addMult`/`addMana`. |
| `hand.discardAll()` | Discards the whole hand and redraws to cap |
| `hand.discard(indices)` | Discards specific hand indices (array or single number) and redraws to cap |
| `hand.enchant(index, ench)` | Enchants a specific hand-index card |
| `deck.add(card)` | Adds a card (`{r: 2-14, s: school, ench?}`) to the deck (200-card cap). Returns the new card's uid. |
| `deck.remove(indices)` | Removes deck cards by index (array or single number), never below a 5-card minimum |
| `deck.setCard(index, {r?, s?})` | Changes a deck card's rank (2–14) and/or sigil in place; a copy currently in hand updates too |
| `deck.enchant(index, ench)` | Enchants a deck card by index; a copy currently in hand updates too |

### 7.7 Randomness (`api.rng`)

Use `api.rng` for every random decision. It's a pure function of the run seed, your item id, your `key`, and the current loop, depth and map position. It doesn't advance the game's RNG, so previews, other mods and other rolls aren't affected, and the result survives save and reload.

| Method | Returns |
|---|---|
| `api.rng(key)` | Float from 0 up to 1 |
| `api.rng.int(n, key)` | Integer from 0 to `n − 1` |
| `api.rng.chance(p, key)` | `true` with probability `p` |
| `api.rng.pick(array, key)` | One element |

**The same key at the same node always returns the same value.** That's what keeps a preview and the real score in agreement. For a fresh roll, change the key:

```js
if (api.rng.chance(0.25, 'proc' + api.combat.handNumber())) api.addMult(10);
```

Use different keys for different decisions in one hook, or they'll be correlated.

---

## 8. Sandbox Restrictions

Hook bodies are compiled with `new Function(...)`, not `eval`, and run with the following restrictions:

- **Length cap**: 4,000 characters per hook body (32,000 for `lib`).
- **No loops**: `while`, `for`, and `do` are banned in the source text — use `map`/`filter`/`reduce`/`forEach` instead.
- **No `constructor`, `eval`, `__proto__`/`__defineGetter__`/`__lookupGetter__`, `import`/`export`, or `debugger`** — all banned by pattern match on the hook source. `constructor` and `__proto__` are also rejected when split across string concatenation. `\u` escapes aren't allowed anywhere in hook source.
- **Randomness and time**: `Math.random` inside a hook returns a seeded value, so older mods stay reproducible, but it won't match between a preview and the real score. Use `api.rng`. `Date`, `performance` and `crypto` aren't available.
- **Allowed globals**: hooks can only see `Math` (seeded `random`), `Array`, `Number`, `String`, `Boolean`, `JSON`, `Map`, `Set`, `WeakMap`, `WeakSet`, `Symbol`, `BigInt`, `RegExp`, `parseInt`, `parseFloat`, `isNaN`, `isFinite`, `console`, and `Object` limited to `keys`, `values`, `entries`, `fromEntries`, `assign` and `freeze`. Every other name resolves to `undefined`, including all browser APIs, timers and game functions. The only way to touch game state is through `api`.
- **Locked built-ins**: once a mod with code loads, built-in prototypes are frozen for that session and `.constructor` on functions is `undefined`. Adding to `Array.prototype` throws.
- A hook that throws is caught, logged as a mod error (naming the offending item and hook), and treated as a no-op for that call — it doesn't crash the run.

### 8.1 Shared helpers (`lib`)

Hooks compile separately, so they can't see each other's variables. Put shared code in a top-level `lib` string in a content file. It runs once at load and must return an object. Every hook in the mod can then reach it as `api.lib`.

```json
{
  "lib": "const scale = (api, n) => n * api.run.relicCount(api.id); return { scale };",
  "relics": [{"id":"bogpack_tide","name":"Tide","rarity":"rare","desc":"+3 Mult per copy.","hooks":{"onScoreHand":"api.addMult(api.lib.scale(api, 3));"}}]
}
```

- Capped at 32,000 characters per file, with the same banned patterns as hooks, including the loop ban.
- `api` and `ctx` don't exist in `lib`, so pass them to your helpers.
- Several files can each have a `lib`. Their helpers merge, and a duplicate name is a load error.

### Display text

Text fields (`name`, `desc`, `descT`, `blurb`, `perks`, `unlockDesc` and Mimic `desc`) are HTML-escaped, so they show literally and can't carry markup. Enemy `n` can't contain `<`, `>`, `&` or `"`.

### SVG icons (`svg` field on enemies, relics, potions, heroes)

Custom icons are optional and go through a strict sanitizer:

- Max 24,000 bytes, max 400 nodes.
- Must parse as valid SVG with a root `<svg>` element.
- Only these tags are allowed: `svg, g, path, circle, rect, ellipse, line, polygon, polyline, defs, linearGradient, radialGradient, stop, use, title`.
- Only a fixed allowlist of presentation/geometry attributes is allowed (`fill`, `stroke-*`, `d`, `x/y/cx/cy/r`, gradient stops, etc.) — no `style`, no `on*` event attributes, no scripts.
- `<script>`, `<foreignObject>`, `<iframe>`, `<image>`, `<style>`, `<animate>`, `<set>`, and `<handler>` elements are rejected outright, as is any element bearing an `on*` attribute.
- Internal `id`/`href`/`url(#...)` references are allowed only for local same-document gradient/`<use>` references, and are automatically namespaced to avoid colliding with the base game's own ids.
- If an `svg` is supplied but fails sanitization, the item still loads — with a placeholder icon and a logged mod error — rather than being rejected outright.
