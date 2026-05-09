# Token Action HUD — Imperium Maledictum

Developer reference for `packages/token-action-hud-impmal`. Covers architecture, ImpMal data structures, the TAH Core integration pattern, and known quirks discovered during development.

---

## Architecture

The module follows the **system adapter pattern** used by all TAH system modules. TAH Core provides the HUD frame, settings, and rendering pipeline; the adapter supplies system-specific action data and click handling.

```
TAH Core (token-action-hud-core v2.1.x)
  │
  ├─ fires tokenActionHudCoreApiReady(coreModule)
  │       └─ adapter creates classes via factory functions
  │          registers SystemManager via tokenActionHudSystemReady
  │
  └─ calls SystemManager methods:
       getActionHandler()   → ActionHandlerImpmal
       getRollHandler()     → RollHandlerImpmal
       registerDefaults()   → layout + groups
       registerSettings()   → (empty for now)
```

### Critical: deferred class inheritance

TAH Core base classes (`coreModule.api.ActionHandler`, etc.) only exist after `tokenActionHudCoreApiReady` fires. All three adapter classes are therefore created via **factory functions** called inside the hook, not as top-level class declarations:

```javascript
// modules/token-action-hud-imperium-maledictum.mjs
Hooks.on('tokenActionHudCoreApiReady', async (coreModule) => {
    const ActionHandlerImpmal = createActionHandler(coreModule)
    const RollHandlerImpmal   = createRollHandler(coreModule)
    const SystemManagerImpmal = createSystemManager(coreModule, ActionHandlerImpmal, RollHandlerImpmal)
    // ...
})
```

Defining classes at module-load time causes a `TypeError` on `globalThis.coreModule` (undefined), which silently prevents `tokenActionHudSystemReady` from ever firing — meaning TAH Core never loads its Handlebars templates, and the HUD renders blank.

---

## Groups and groupIds

TAH Core calls `buildSystemActions(groupIds)` where `groupIds` is a flat array of **group `id` values** (not `nestId` values) for all currently active groups, including both category-level and leaf-level entries.

Example groupIds received at runtime:
```
["characteristics","skills","talents","combat","powers","inventory","utility",
 "characteristic","specialisation","talent","trait","boonLiability",
 "weapon","ammo","combatAction","warpCharge","power",
 "protection","forceField","equipment","augmetic","utility","restRecover"]
```

**Consequence**: check `groupIds.includes(item.type)` or `groupIds.includes(groupId)` using the leaf group `id`, not the `nestId`. Pass `tah.groups[type]` (which has the correct `id`) to `addActions()`.

### DEFAULTS layout

`registerDefaults()` must return `{ layout, groups }`:
- `layout` — array of category objects, each with a `groups` array of leaf group objects
- `groups` — flat array of all leaf group objects (used for TAH settings UI)

Group name strings must be **pre-localised** before returning — TAH Core does not call `game.i18n.localize()` on them. `SystemManager.registerDefaults()` maps all names through `game.i18n.localize()`.

Tab order in the layout array controls display order in the HUD. Current layout:

| Category | Leaf groups (in order) |
|----------|------------------------|
| Characteristics | characteristic |
| Skills | specialisation |
| Talents & Traits | talent, trait, boonLiability |
| Combat | weapon, ammo, combatAction |
| Psychic Powers | warpCharge, power |
| Inventory | protection, forceField, equipment, augmetic |
| Utility | utility, restRecover |

The Psychic Powers tab is hidden automatically by TAH Core when no actions are added to it; the adapter gates both `#buildWarpCharge` and `#buildPowers` on `actor.items.some(i => i.type === 'power')`.

---

## TAH Core action template

TAH Core renders each action via `action.hbs`. Key fields on the action object:

| Field | Type | Rendering | Notes |
|-------|------|-----------|-------|
| `id` | string | — | Must be unique within the group |
| `name` | string | escaped text | Button label |
| `useRawHtmlName` | boolean | — | Set `true` to render `name` as raw HTML (triple-brace) |
| `img` | string | background-image URL | Set `''` to suppress the icon |
| `encodedValue` | string | — | Passed to `handleActionClick` |
| `cssClass` | string | applied to `<button>` | Use for visual states |
| `active` | boolean | — | TAH Core's own active state; adds a highlight class |
| `tooltip` | string | shown on hover | |
| `info1` | object | see below | First info badge (top-right area) |
| `info2` | object | see below | Second info badge |
| `info3` | object | see below | Third info badge |

**Info badge fields** — each `infoN` object accepts either `.text` or `.icon`, not both:

| Field | Rendering |
|-------|-----------|
| `{ text: "40" }` | Escaped string — safe for numbers, plain text |
| `{ icon: '<i class="fa-solid fa-hand"></i>' }` | Raw HTML via triple-brace `{{{...}}}` — use for Font Awesome icons |

Only 3 info slots exist in the template. Plan badge usage accordingly.

---

## ImpMal data structures

### Characteristics

```javascript
actor.system.characteristics  // object keyed by characteristic id
// Keys: ws, bs, str, tgh, ag, int, per, wil, fel
// Each entry: { total, starting, modifier, advances, bonus, ... }
// bonus = Math.floor(total / 10)

game.impmal.config.characteristics  // { ws: "IMPMAL.WeaponSkill", ... }
// Values are i18n keys — pass through game.i18n.localize()
```

### Skills

```javascript
actor.system.skills  // object keyed by skill id
// Each entry: { characteristic, advances, modifier, total, specialisations[] }
// specialisations[] — array of specialisation Item documents

game.impmal.config.skills  // { awareness: "Awareness", ... }
// Values are already localised strings (not i18n keys)
```

### Specialisations (items)

```javascript
// item.type === 'specialisation'
item.system.total     // getter: parent skill total + (5 * advances)
```

Display convention used in the HUD: `"Awareness – Sight"` (en-dash between skill and specialisation name).

### Weapons (items)

```javascript
// item.type === 'weapon'
item.system.skillTotal              // computed total of the skill used for this weapon
item.system.damage.value            // base damage value
item.system.traits.has('twohanded') // boolean — is this weapon two-handed?
```

### Hands (character actors only)

```javascript
actor.system.hands.isHolding(itemId)
// Returns: { left: item|null, right: item|null }

item.system.equip('right')  // equip to right hand (also 'left'; two-handed weapons fill both)
item.system.equip('left')
item.system.unequip()       // remove from all hands
```

`isHolding` is only available on `character` type actors. For NPCs, use `item.system.isEquipped` directly:

```javascript
if (item.system.isEquipped) await item.system.unequip()
else await item.system.equip()
```

### Combat Actions

```javascript
actor.system.combat.action          // key of currently active combat action (string), or ''
game.impmal.config.actions          // { aim: { label: "IMPMAL.Aim", ... }, charge: {...}, ... }
// Keys: aim, charge, defend, fullauto, guarded, run, standard, swift
```

### Warp Charge

```javascript
actor.system.warp.charge      // current warp charge (integer)
actor.system.warp.threshold   // max before overcharge (Willpower bonus)
actor.system.warp.state       // warp state object passed to psychic mastery test
```

The `charge > threshold` condition determines Psychic Mastery vs Purge:
- `charge <= threshold`: clicking triggers a Purge roll
- `charge > threshold`: clicking triggers a Psychic Mastery test

This is encoded into the `encodedValue` at action-build time so the RollHandler does not need to re-read warp state.

### Psychic Powers (items)

```javascript
// item.type === 'power'
item.system.rating               // Warp Rating (integer)
item.system.overt                // boolean — overt powers use orange text in HUD
item.system.skill                // skill key (string) or Item document for the test skill
item.system.difficulty           // difficulty key (e.g. 'average', 'hard')
item.system.damage.value         // base damage (0 = no damage)
item.system.damage.SL            // boolean — damage scales with SL

game.impmal.config.difficulties  // { average: { modifier: 0 }, hard: { modifier: -10 }, ... }
```

Adjusted skill total for display:
```javascript
const skill = sys.skill
const skillTotal = skill instanceof Item
    ? skill.system.total
    : this.actor.system.skills[skill]?.total ?? 0
const diffMod  = game.impmal.config.difficulties?.[sys.difficulty]?.modifier ?? 0
const adjusted = skillTotal + diffMod
```

### Wounds

```javascript
actor.system.characteristics.tgh.bonus   // Toughness bonus (floor(TGH/10))
actor.system.combat.wounds.value          // current wound count (increases with damage taken)
// Heal = subtract from wounds.value, floor at 0
```

---

## Roll API (ImpMal actor methods)

| Action | Method | Notes |
|--------|--------|-------|
| Characteristic | `actor.setupCharacteristicTest(key)` | key e.g. `"ws"` |
| Base skill | `actor.setupSkillTest({ key })` | key e.g. `"awareness"` |
| Specialisation | `actor.setupSkillTest({ itemId })` | itemId of the specialisation item |
| Weapon | `actor.setupWeaponTest(itemId)` | itemId of the weapon item |
| Psychic power | `actor.setupPowerTest(itemId)` | itemId of the power item |
| Psychic mastery | `actor.setupSkillTest({ key: 'psychic' }, { warp: actor.system.warp.state })` | second arg pre-fills warp state |
| Purge | `actor.purge()` | opens confirmation dialog in non-combat |
| Select combat action | `actor.useAction(key)` | key from `game.impmal.config.actions` |
| Clear combat action | `actor.clearAction()` | removes current action selection |
| Roll initiative | `actor.rollInitiative({ createCombatants: true })` | |

---

## Encoded values

Actions pass data to the RollHandler via a pipe-delimited `encodedValue` string. The delimiter is `this.delimiter` (set by TAH Core, typically `"|"`).

| Format | Example | Handler |
|--------|---------|---------|
| `characteristic\|key` | `characteristic\|ws` | `setupCharacteristicTest(key)` |
| `skill\|key` | `skill\|awareness` | `setupSkillTest({ key })` |
| `specialisation\|itemId` | `specialisation\|abc123` | `setupSkillTest({ itemId })` |
| `weapon\|itemId` | `weapon\|abc123` | `setupWeaponTest(itemId)` |
| `ammo\|itemId` | `ammo\|abc123` | opens sheet on Combat tab |
| `combatAction\|key` | `combatAction\|aim` | `useAction`/`clearAction` |
| `warpCharge\|mastery` | — | `setupSkillTest` psychic mastery |
| `warpCharge\|purge` | — | `actor.purge()` |
| `power\|itemId` | `power\|abc123` | `setupPowerTest(itemId)` |
| `utility\|initiative` | — | `actor.rollInitiative(...)` |
| `utility\|endTurn` | — | `game.combat.nextTurn()` |
| `utility\|rest6h` | — | heal TGH bonus wounds + chat msg |
| `utility\|restDay` | — | heal 2× TGH bonus wounds + chat msg |

The `warpCharge` encoded value (`mastery` vs `purge`) is set at action-build time based on `charge > threshold`. Right-clicking either state always triggers a purge.

---

## Click handling in RollHandler

`handleActionClick(event, encodedValue)` uses an early-exit pattern for action types that need special handling before the `isRenderItem()` check:

```javascript
// Weapon: double-click, right-click, and left-click all differ
if (actionType === tah.actions.weapon) {
    if (event.detail >= 2) return this.#openSheetOnCombatTab()   // double-click
    if (this.isRenderItem()) return this.#handleWeaponEquip(id)  // right-click
    await this.#handleWeapon(id)                                  // left-click
    return
}

// Combat action: right-click opens journal, left-click toggles selection
if (actionType === tah.actions.combatAction) {
    if (this.isRenderItem()) return this.#openActionsJournal()
    await this.#handleCombatAction(id)
    return
}

// Warp charge: right-click always purges; left-click depends on encoded value
if (actionType === tah.actions.warpCharge) {
    if (this.isRenderItem() || id === 'purge') await this.actor.purge()
    else await this.actor.setupSkillTest({ key: 'psychic' }, { warp: this.actor.system.warp.state })
    return
}
```

`isRenderItem()` returns `true` on right-click (TAH Core's term for the secondary/context click).

### Weapon equip cycle

For character actors (hands model available):
```
Unequipped  →  Right hand  →  Left hand  →  Unequipped
Two-handed: Unequipped  →  Both hands  →  Unequipped
```

For NPC actors: toggles `isEquipped` directly (no hand tracking).

### Sheet navigation

```javascript
const sheet = this.actor.sheet
await sheet.render({ force: true })
sheet.changeTab?.('combat', 'primary')  // ApplicationV2 tab API
```

Used for both ammo left-click and weapon double-click.

### Actions journal

```javascript
// constants.journals.actions = 'JournalEntry.hdElQAwiBr5AyoRf.JournalEntryPage.xf46pBDy93sT0ZDl'
const page = await fromUuid(constants.journals.actions)
if (page) page.parent.sheet.render(true, { pageId: page.id })
```

---

## Styling

CSS lives in `styles/imperium-maledictum.css`. The `cssClass` field is applied to the `<button class="tah-action-button ...">` element. Button text is in a child `<div class="tah-button-text">`, so colour rules must target that child.

```css
/* Overt psychic power — faded orange label */
button.tah-impmal-overt .tah-button-text {
    color: rgba(210, 115, 45, 0.8);
}

/* Active combat action — orange outline + tinted background + bright label */
button.tah-impmal-active {
    outline: 1px solid rgba(210, 115, 45, 0.9);
    background: rgba(210, 115, 45, 0.15);
}
button.tah-impmal-active .tah-button-text {
    color: rgba(210, 115, 45, 1);
}
```

`active: true` on an action object triggers TAH Core's own active state (adds a separate highlight class). The adapter also sets `cssClass: 'tah-impmal-active'` for its own orange styling.

Setting `img: ''` on an action suppresses the icon that TAH Core would otherwise show.

---

## Build and deployment

Rollup watch mode (`npm run build`) reads `module.json` for the module ID, bundles `modules/token-action-hud-imperium-maledictum.mjs`, and copies `module.json`, `languages/`, and `styles/` to the path returned by `foundry-path.js`.

Copy `foundry-path.example.js` → `foundry-path.js` and set the path to your local Foundry `Data/modules/token-action-hud-imperium-maledictum/` directory. This file is gitignored.

---

## TAH Core version notes

- **Installed version**: `token-action-hud-core` v2.1.1 by Larkinabout (`https://github.com/Larkinabout/fvtt-token-action-hud-core`)
- The Drental repo (`https://github.com/Drental/fvtt-tokenactionhud`) is the old original codebase and should not be confused with the maintained fork
- Set `requiredCoreModuleVersion: '2.1'` in `constants.mjs` to match
- In Foundry v14, the core uses `foundry.applications.handlebars.loadTemplates()` to register Handlebars partials — this only runs if `tokenActionHudSystemReady` fires successfully from the adapter

---

## Known TODOs

- Ammo display: in ImpMal, ammo items are linked to specific weapons rather than being standalone inventory. Consider grouping available ammo under its parent weapon in the Combat tab rather than as a separate flat list.
- Right-click on skills/characteristics could open the actor sheet focused on that stat (currently unhandled — falls through to default).
- No vehicle or patron actor support — `buildSystemActions` only handles `character` and `npc` types.