# Steward's Initiative — Design Spec

**Date:** 2026-07-03
**Target game version:** Crusader Kings III 1.19.0.6 ("Scribe"), verified against the locally installed game files
**Repo:** new public GitHub repo under `corwinfontaine-creator`, MIT license

## 1. Problem & Goal

The player wants a CK3 mod that automatically upgrades buildings on their behalf, so they don't have to manually click through the building UI for every holding every session. Two independent spending targets:

1. **My Domain** — every holding the player personally holds (their capital and every other county/barony in their domain).
2. **Vassals** — every holding held by any of the player's vassals, direct or indirect, across their entire realm. The player subsidizes this construction out of their own treasury (a "royal patron" model), not the vassal's own gold.

Both are gated by a player-configurable gold reserve floor (never spend below it) and a player-configurable priority order across 6 building categories. The whole system is configured and adjustable at any time via a decision in the decisions panel, which opens a settings menu.

## 2. Non-goals (v1)

- No per-holding/per-county exclusion list. (Noted as a plausible v2 addition if it turns out to be needed in play.)
- No support for AI-controlled realms other than the player's own — this is a player-facing convenience feature, not a rebalance of AI behavior.
- No true native "under construction" progress-bar visual for auto-built buildings (see Section 6 — this is a verified engine limitation, not a scope cut).
- No multiplayer-specific handling beyond whatever singleplayer-safe scripting naturally provides.

## 3. Mod Structure

Standard CK3 mod layout. The mod's internal folder/id is `stewards_initiative` (no apostrophe — Steam Workshop and cross-platform paths shouldn't contain one); the in-launcher display name remains "Steward's Initiative":

```
stewards_initiative/
  .mod                          (descriptor, supported_version = 1.19.*)
  descriptor.mod
  common/
    decisions/                  (the entry-point decision)
    scripted_effects/           (spend/build/settings-menu helper effects)
    scripted_triggers/          (eligibility/affordability triggers)
    scripted_values/            (building-cost lookups, category weighting)
    on_action/                  (monthly pulse hook)
  events/                       (settings-menu pop-up chain)
  localization/english/         (all player-facing text)
```

## 4. Entry Point

One decision, **"Steward's Initiative,"** available to any playable character holding at least one title. Its `effect` does not build anything directly — it fires `trigger_event` to open the settings menu. This is what makes it "a decision you can do and modify as you go": reopening the decision at any time re-enters the same menu with your current settings pre-populated.

## 5. Settings Data Model

All settings are stored as **character variables** on the player character, so settings are per-character and do not carry over automatically across succession (a new ruler starts with the mod fully off until they configure it — avoids surprise behavior from an inherited config):

| Variable | Type | Meaning |
|---|---|---|
| `si_domain_enabled` | bool | Master toggle for domain auto-upgrade |
| `si_vassal_enabled` | bool | Master toggle for vassal subsidy |
| `si_domain_reserve` | number | Domain gold reserve floor |
| `si_vassal_reserve` | number | Vassal-subsidy gold reserve floor |
| `si_priority_1` … `si_priority_6` | category id | Player's ranked order of the 6 building categories |

**Reserve floor semantics:** "never let treasury/gold drop below this number." The mod spends freely on eligible upgrades as long as the balance stays at or above the floor after paying.

**Reserve floor input:** a preset list (0 / 100 / 250 / 500 / 1000 / 2500 / 5000 / 10000 / 25000 / 50000), chosen via the settings-menu event chain. CK3 decisions only support fixed on/off effects, not freeform numeric or drag-and-drop input, without bespoke GUI work; a chain of multiple-choice pop-up events (the same "settings menu made of pop-ups" pattern used by several existing CK3 mods) is the reliable, low-risk way to collect this input. A custom slider widget was considered and rejected as the riskiest, most fragile piece to build and maintain across patches.

**Category ranking input:** a wizard flow — pick your 1st-priority category from all 6, then 2nd from the remaining 5, and so on, each choice narrowing the next pop-up's options via `is_shown` gated on which categories are already assigned.

## 6. Building Categories

Six categories, chosen to mirror how the game itself tags building effects:

1. **Tax / Economy** — flat and % tax buildings (workshops, tradeports, farms, plantations, etc.)
2. **Development Growth** — buildings with development-growth modifiers
3. **Levy** — troop-count buildings (barracks, pastures, warrior lodges, etc.)
4. **Men-at-arms** — MaA damage/toughness/maintenance buildings (stables, smiths, wind furnace, regimental grounds)
5. **Fortification** — fort level / garrison buildings (hill forts, ramparts, curtain walls, watchtowers)
6. **Specials** — holding-type-specific buildings (guild halls, monastic schools, scriptorium, megaliths) and duchy capital buildings

The mod maintains a small hand-authored catalog mapping each building chain to one of these categories, based on the building's actual defined modifiers in the current patch. This is a one-time authoring effort (~40 building chains) with the same maintenance profile as any CK3 content mod that catalogs vanilla buildings — it may need a pass when Paradox ships new buildings in future DLC/patches.

## 7. Monthly Processing Pipeline

An `on_action` fires monthly on the player character, gated so it's a no-op unless `si_domain_enabled` or `si_vassal_enabled` is true.

**Domain pass** (if enabled): for each of the 6 categories in the player's ranked order, find the cheapest eligible next-upgrade across the player's directly-held holdings (via a native-sorted `ordered_x`-style list-builder, not a manual scripted loop), buy it if the resulting balance stays at/above `si_domain_reserve`, and repeat within that category until no affordable candidate remains — then move to the next category.

**Vassal pass** (if enabled, runs after the domain pass completes): a cheap gate first — only proceed if remaining gold/treasury is above `si_vassal_reserve`; if so, run the same category-by-category, cheapest-first sweep over holdings belonging to `every_vassal` (all vassals, direct and indirect), spending the player's own gold down to `si_vassal_reserve`.

**Tie-breaking within a category:** cheapest eligible upgrade first, regardless of which holding it's in.

**Both passes skip** any building slot where `has_ongoing_construction = yes` — i.e. anything already manually queued by the player or (for vassals) already in progress — so the mod never fights the player for the same slot.

## 8. Construction Mechanic — Delayed Grant

CK3 has no scriptable way to start a real, native, progress-bar-visible construction queue. This was verified directly against the installed 1.19.0.6 game files, not assumed:

- Every vanilla usage of the `add_building` effect (tutorials, story-cycle rewards, great projects, decision rewards) grants instantly — there is no queued/timed variant anywhere in the shipped scripts.
- The in-game Build button (`window_county_view.gui`) is wired to `onclick = "[GUIBuildingItem.OnClick]"`, a compiled native GUI binding, structurally separate from the scripting language decisions/events/on_actions run in.
- `common/effect_localization/` — which must contain a tooltip entry for every effect the game has, scripted or hardcoded, because tooltips are generated from it — has an entry for `add_building` and no second entry for anything construction-queue-related.
- `has_ongoing_construction` and `has_construction_with_flag` are real, but read-only triggers, used everywhere in vanilla defensively (checked before an instant `add_building` grant, to avoid clobbering something already queued) — never paired with a "start construction" effect.

Given that, the mod approximates real construction pacing:

1. When the mod decides to build something, it deducts the cost immediately (handling both `gold` and `treasury` currency systems per Section 9).
2. Completion is scheduled via a timed mechanism matching that building's normal `construction_time`.
3. At completion time, the mod **re-verifies** `has_ongoing_construction = no` and that the target slot/tier is still what was expected, before calling `add_building`. If the player (or, for vassal holdings, the vassal) manually queued something into that slot in the meantime, the mod skips silently rather than overwriting — mirroring the exact defensive pattern vanilla uses in its own free-building story events.

The one accepted trade-off: there's no visible progress bar/scaffolding during the wait, since that visual is part of the native queue this system can't reach. The pacing (gold-now, building-later) matches manual construction; the visual feedback doesn't.

## 9. Currency Compatibility

CK3 characters use one of two currency systems (`gold` or the newer `treasury`, gated by `has_treasury = yes`). Every gold check and deduction in this mod uses the standard vanilla pattern:

```
trigger_if = { limit = { has_treasury = yes } treasury >= X }
trigger_else = { gold >= X }
```

so the mod behaves correctly regardless of which system a given character uses.

## 10. Performance Considerations

- **Domain scanning is inherently cheap** — bounded by the player's domain limit (typically well under 30 holdings) regardless of empire size.
- **Vassal scanning is the real cost driver** in a large, late-game blob empire (potentially hundreds of vassals). Mitigations:
  1. A cheap gate (`gold > si_vassal_reserve`, a single comparison) runs before any vassal iteration — skips the expensive scan entirely in the common case where the realm is already fully built up or cash-strapped.
  2. Candidate selection uses native-sorted list-builders (the same mechanism the game's own AI uses to evaluate hundreds of characters/targets daily) rather than manual nested scripted loops.
  3. Monthly cadence is the starting point; if real-world profiling on a large save shows a problem, vassal-checks specifically (not domain-checks) can be dropped to a longer interval without design changes elsewhere.
- Actual performance will be validated empirically during implementation on a large test save, not fully solved on paper up front.

## 11. Edge Cases & Safety

- **Succession/character death:** settings reset to defaults (everything off) on the new character — no inherited config causes surprise spending.
- **Below reserve floor already:** nothing is built; the mod simply does nothing that month.
- **Title/vassal changes mid-month:** each monthly run is a fresh scan with no persistent per-holding state, so newly acquired/lost holdings and vassals are picked up or dropped automatically on the next run.
- **Manual construction conflicts:** guarded twice — once when the mod decides to spend (skips slots already under construction) and again at delayed-grant completion time (Section 8).

## 12. Testing Approach

This is a scripting-only mod (no compiled code). Verification means:

1. Loading the mod in a live CK3 save and checking the game's own error console/log output for script syntax errors on load.
2. Manually driving a test game: enabling each toggle independently, fast-forwarding time, and confirming gold is spent correctly, buildings appear after the expected delay, reserve floors are respected, and category priority order is followed.
3. Using CK3's `-debug_mode` console commands (`script_docs`, `dump_data_types`) during implementation to validate exact effect/trigger syntax against the installed 1.19.0.6 files directly — the same method used to ground this design.

## 13. Open Items Carried Into Implementation Planning

- Exact list of ~40 building chains and their category assignments (Section 6) needs to be authored against the current game files.
- Exact scripted-value formulas for reading a given building's next-tier cost (the game defines these via shared cost-tier script values, e.g. `normal_cost_tier_3`, referenced per-building — confirmed to exist and be systematic, but the full mapping needs to be built out).
- Exact list-builder/trigger names for "cheapest eligible candidate in category X" need final verification per building type during implementation (the mechanism is confirmed to exist; the full syntax for every category needs to be written and tested).
