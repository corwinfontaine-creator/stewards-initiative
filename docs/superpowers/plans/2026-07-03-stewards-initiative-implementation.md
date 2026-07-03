# Steward's Initiative Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a CK3 mod (targeting 1.19.x "Scribe") that lets the player configure, via an in-game decision and settings-menu event chain, automatic gold-reserve-respecting building upgrades across their own domain and (as a liege-funded subsidy) their vassals' holdings realm-wide, prioritized by a player-ranked list of 6 building categories.

**Architecture:** All settings live as character variables. A monthly `on_action` runs a domain pass then a vassal pass, each walking the player's ranked category list and using CK3's native `ordered_x` list-builder (sorted by a per-province "cheapest eligible candidate in this category" scripted_value) to find and fund the single cheapest affordable upgrade, repeating until the relevant reserve floor is hit. Construction is a "delayed grant": gold is deducted immediately, the building is actually applied via `add_building` after a scheduled delay matching the real building's `construction_time`, re-validated at grant time to avoid clobbering manual construction.

**Tech Stack:** CK3 script only (decisions, events, on_actions, scripted_effects/triggers/values, localization `.yml`). No compiled code.

## Global Constraints

- Target game version: CK3 1.19.x ("Scribe"). `.mod`/`descriptor.mod` `supported_version = "1.19.*"`.
- Mod internal id/folder: `stewards_initiative` (no apostrophe). Display name: "Steward's Initiative".
- All script identifiers use the `si_` prefix to avoid collisions with vanilla or other mods.
- Every gold check/deduction must handle both currency systems: `trigger_if = { limit = { has_treasury = yes } treasury >= X } trigger_else = { gold >= X }` (and the equivalent for deduction).
- Settings are character variables, reset to defaults (mod fully off) on a new character — no persistence across succession beyond what character-variable scoping gives for free.
- No per-holding/county exclusion list in this plan (out of scope per spec Section 2).
- Duchy capital buildings are explicitly out of scope for this plan (see Task 7 note) — "Specials" covers guild_halls, monastic_schools, scriptorium, megalith only.
- Local dev testing loop: this machine already has CK3 1.19.0.6 installed at `C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III`, and the mod's registration pointer lives at `C:/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/mod/stewards_initiative.mod`, pointing `path` at this repo (`C:/Users/corwi/CK3_Auto_Upgrade_Mod`). Every task's verification step should check `C:/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log` for new errors after the user reloads the mod (the game's own script-error log — grep for the mod's own file names to isolate our errors from pre-existing ones), plus a short manual in-game check the user performs (this repo has no automated CK3 test framework; this is the established testing method — see spec Section 12).

---

## File Structure

```
stewards-initiative/  (repo root = mod content folder)
  descriptor.mod
  common/
    decisions/
      si_settings_decision.txt
    on_action/
      si_on_actions.txt
    scripted_effects/
      si_settings_effects.txt        (defaults init, menu navigation helpers)
      si_building_catalog_effects.txt (per-building select/build effects, Task 6-7)
      si_construction_effects.txt     (delayed-grant mechanic, Task 9)
      si_pass_effects.txt             (domain pass / vassal pass, Task 10-11)
    scripted_triggers/
      si_building_catalog_triggers.txt (per-building eligibility triggers, Task 6-7)
      si_gold_triggers.txt            (currency-agnostic affordability helpers)
    scripted_values/
      si_building_catalog_values.txt  (per-building next-tier-cost values, Task 6-7)
  events/
    si_menu_events.txt                (settings menu hub + domain/vassal/priority submenus, Task 2-5)
  localization/
    english/
      si_l_english.yml
```

---

### Task 1: Mod scaffold & local dev registration

**Files:**
- Create: `descriptor.mod`
- Create: `common/decisions/si_settings_decision.txt`
- Create: `localization/english/si_l_english.yml`
- Create: `C:/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/mod/stewards_initiative.mod` (outside the repo — the game's mod registry pointer)

**Interfaces:**
- Produces: the decision id `si_settings_decision`, invocable from later tasks' events via `effect = { trigger_event = si_menu.1 }` once Task 2 defines `si_menu.1`. For Task 1, its effect is a no-op placeholder (`custom_tooltip = si_settings_decision_placeholder_tt`) that Task 2 will replace.

- [ ] **Step 1: Create the repo-root descriptor**

`C:\Users\corwi\CK3_Auto_Upgrade_Mod\descriptor.mod`:

```
version="1.19.0.6"
tags={
	"Gameplay"
	"Utilities"
}
name="Steward's Initiative"
supported_version="1.19.*"
```

- [ ] **Step 2: Register the mod with the local game install**

`C:\Users\corwi\Documents\Paradox Interactive\Crusader Kings III\mod\stewards_initiative.mod`:

```
version="1.19.0.6"
tags={
	"Gameplay"
	"Utilities"
}
name="Steward's Initiative"
supported_version="1.19.*"
path="C:/Users/corwi/CK3_Auto_Upgrade_Mod"
```

- [ ] **Step 3: Create the minimal proof-of-life decision**

`common/decisions/si_settings_decision.txt`:

```
si_settings_decision = {
	picture = "gfx/interface/illustrations/decisions/decision_generic_council.dds"

	is_shown = {
		is_playable_character = yes
		any_held_title = {
			always = yes
		}
	}

	is_valid_showing_failures_only = {
		always = yes
	}

	ai_check_interval = 0 # AI never considers this — player-only feature

	cost = {
		gold = 0
	}

	effect = {
		custom_tooltip = si_settings_decision_placeholder_tt
	}
}
```

- [ ] **Step 4: Add localization for the decision**

`localization/english/si_l_english.yml` (must be UTF-8 with BOM — use the editor/tool's "UTF-8 BOM" save option, not plain UTF-8):

```yaml
l_english:
 si_settings_decision_title:0 "Steward's Initiative"
 si_settings_decision_desc:0 "Configure automatic building upgrades for your domain and vassals."
 si_settings_decision_confirm:0 "Open Settings"
 si_settings_decision_placeholder_tt:0 "The settings menu will open here once Task 2 is implemented."
```

- [ ] **Step 5: Verify the mod loads without script errors**

Ask the user to:
1. Open the Paradox Launcher, find "Steward's Initiative" under mods (it will appear as a local/unmanaged mod), enable it in the active playset, and launch CK3.
2. Load or continue any save as a playable character with at least one title.
3. Open the decisions panel and confirm "Steward's Initiative" appears with the correct title/description and no red error indicator.

Then, grep the error log for anything mentioning our files:

```bash
grep -i "si_settings_decision\|stewards_initiative\|si_l_english" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output (no matches), or only pre-existing unrelated errors if the log wasn't cleared first (recommend clearing/renaming the old error.log before this first load so a clean grep is meaningful).

- [ ] **Step 6: Commit**

```bash
git add descriptor.mod common/decisions/si_settings_decision.txt localization/english/si_l_english.yml
git commit -m "Add mod scaffold and proof-of-life settings decision"
```

(Note: `Documents/Paradox Interactive/Crusader Kings III/mod/stewards_initiative.mod` is outside the repo and is not committed — it's local machine registration, not mod content.)

---

### Task 2: Character variable defaults & settings menu hub

**Files:**
- Modify: `common/decisions/si_settings_decision.txt` (point `effect` at the real menu)
- Create: `common/on_action/si_on_actions.txt` (game-start default init, monthly pulse hook stub)
- Create: `common/scripted_effects/si_settings_effects.txt`
- Create: `events/si_menu_events.txt`
- Modify: `localization/english/si_l_english.yml`

**Interfaces:**
- Consumes: none new.
- Produces: character variables `si_domain_enabled` (bool), `si_vassal_enabled` (bool), `si_domain_reserve` (number), `si_vassal_reserve` (number), `si_priority_1..6` (number, category id 1-6: 1=Economy, 2=Development, 3=Levy, 4=MaA, 5=Fortification, 6=Specials). Event id `si_menu.1` (hub). Later tasks (3, 4, 5) add `si_menu.2`+ and are linked from this hub.

- [ ] **Step 1: Initialize defaults on game start**

`common/on_action/si_on_actions.txt`:

```
on_game_start = {
	effect = {
		every_playable_character = {
			si_initialize_defaults_effect = yes
		}
	}
}
```

- [ ] **Step 2: Write the defaults-init scripted effect**

`common/scripted_effects/si_settings_effects.txt`:

```
si_initialize_defaults_effect = {
	if = {
		limit = { NOT = { has_variable = si_domain_enabled } }
		set_variable = { name = si_domain_enabled value = 0 }
	}
	if = {
		limit = { NOT = { has_variable = si_vassal_enabled } }
		set_variable = { name = si_vassal_enabled value = 0 }
	}
	if = {
		limit = { NOT = { has_variable = si_domain_reserve } }
		set_variable = { name = si_domain_reserve value = 500 }
	}
	if = {
		limit = { NOT = { has_variable = si_vassal_reserve } }
		set_variable = { name = si_vassal_reserve value = 500 }
	}
	if = {
		limit = { NOT = { has_variable = si_priority_1 } }
		set_variable = { name = si_priority_1 value = 1 } # Economy
		set_variable = { name = si_priority_2 value = 2 } # Development
		set_variable = { name = si_priority_3 value = 3 } # Levy
		set_variable = { name = si_priority_4 value = 4 } # Men-at-arms
		set_variable = { name = si_priority_5 value = 5 } # Fortification
		set_variable = { name = si_priority_6 value = 6 } # Specials
	}
}

# Called any time a new character needs variables initialized (e.g. succession) — reuses the same effect
si_ensure_defaults_effect = {
	si_initialize_defaults_effect = yes
}
```

Note: booleans are stored as `0`/`1` numbers because CK3 character variables are numeric/date/string, not native bool — every trigger check against these uses `var:si_domain_enabled = 1` (not a bare `has_variable` check, since the variable always exists after init).

- [ ] **Step 3: Point the decision at the real menu**

Edit `common/decisions/si_settings_decision.txt`, replace the `effect` block:

```
	effect = {
		si_ensure_defaults_effect = yes
		trigger_event = si_menu.1
	}
```

- [ ] **Step 4: Write the settings menu hub event**

`events/si_menu_events.txt`:

```
namespace = si_menu

si_menu.1 = {
	type = character_event
	title = si_menu.1.t
	desc = {
		first_valid = {
			triggered_desc = {
				trigger = { var:si_domain_enabled = 1 }
				desc = si_menu.1.desc_domain_on
			}
			desc = si_menu.1.desc_domain_off
		}
	}
	theme = council
	left_icon = root

	option = { # Configure Domain
		name = si_menu.1.a
		trigger_event = si_menu.2
	}
	option = { # Configure Vassals
		name = si_menu.1.b
		trigger_event = si_menu.4
	}
	option = { # Set Priorities
		name = si_menu.1.c
		trigger_event = si_menu.6
	}
	option = { # Done
		name = si_menu.1.d
	}
}
```

- [ ] **Step 5: Add localization for the hub**

Append to `localization/english/si_l_english.yml`:

```yaml
 si_menu.1.t:0 "Steward's Initiative — Settings"
 si_menu.1.desc_domain_on:0 "Domain auto-upgrades are currently #P ON#! Reserve floor: $si_domain_reserve$ gold."
 si_menu.1.desc_domain_off:0 "Domain auto-upgrades are currently #N OFF#!"
 si_menu.1.a:0 "Configure Domain Upgrades"
 si_menu.1.b:0 "Configure Vassal Subsidies"
 si_menu.1.c:0 "Set Category Priorities"
 si_menu.1.d:0 "Done"
```

- [ ] **Step 6: Verify**

Grep the error log after the user reloads:

```bash
grep -i "si_menu\|si_on_actions\|si_settings_effects" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Ask the user to take the decision in-game and confirm the hub event pops up with working "Configure Domain Upgrades" / "Configure Vassal Subsidies" / "Set Category Priorities" / "Done" buttons (the first three will error/no-op until Tasks 3-5 exist — that's expected at this point; just confirm the hub itself displays and "Done" closes it).

- [ ] **Step 7: Commit**

```bash
git add common/decisions/si_settings_decision.txt common/on_action/si_on_actions.txt common/scripted_effects/si_settings_effects.txt events/si_menu_events.txt localization/english/si_l_english.yml
git commit -m "Add character variable defaults and settings menu hub"
```

---

### Task 3: Domain toggle & reserve floor submenu

**Files:**
- Modify: `events/si_menu_events.txt` (add `si_menu.2`, `si_menu.3`)
- Modify: `localization/english/si_l_english.yml`

**Interfaces:**
- Consumes: `si_domain_enabled`, `si_domain_reserve` (from Task 2).
- Produces: event ids `si_menu.2` (domain toggle) and `si_menu.3` (domain reserve picker). Both return to `si_menu.1` when done.

- [ ] **Step 1: Write the domain toggle event**

Append to `events/si_menu_events.txt`:

```
si_menu.2 = {
	type = character_event
	title = si_menu.2.t
	desc = si_menu.2.desc
	theme = council
	left_icon = root

	option = {
		name = si_menu.2.enable
		trigger = { var:si_domain_enabled = 0 }
		set_variable = { name = si_domain_enabled value = 1 }
		trigger_event = si_menu.3
	}
	option = {
		name = si_menu.2.disable
		trigger = { var:si_domain_enabled = 1 }
		set_variable = { name = si_domain_enabled value = 0 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.2.change_reserve
		trigger = { var:si_domain_enabled = 1 }
		trigger_event = si_menu.3
	}
	option = {
		name = si_menu.2.back
		trigger_event = si_menu.1
	}
}

si_menu.3 = {
	type = character_event
	title = si_menu.3.t
	desc = si_menu.3.desc
	theme = council
	left_icon = root

	option = {
		name = si_menu.3.opt_0
		set_variable = { name = si_domain_reserve value = 0 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_100
		set_variable = { name = si_domain_reserve value = 100 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_250
		set_variable = { name = si_domain_reserve value = 250 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_500
		set_variable = { name = si_domain_reserve value = 500 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_1000
		set_variable = { name = si_domain_reserve value = 1000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_2500
		set_variable = { name = si_domain_reserve value = 2500 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_5000
		set_variable = { name = si_domain_reserve value = 5000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_10000
		set_variable = { name = si_domain_reserve value = 10000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_25000
		set_variable = { name = si_domain_reserve value = 25000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_50000
		set_variable = { name = si_domain_reserve value = 50000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.back
		trigger_event = si_menu.1
	}
}
```

- [ ] **Step 2: Wire the hub's "Configure Domain Upgrades" option to it**

Edit `events/si_menu_events.txt`, in `si_menu.1`, replace the first option:

```
	option = { # Configure Domain
		name = si_menu.1.a
		trigger_event = si_menu.2
	}
```

(No change needed — this already points at `si_menu.2`, which now exists.)

- [ ] **Step 3: Add localization**

Append to `localization/english/si_l_english.yml`:

```yaml
 si_menu.2.t:0 "Domain Upgrades"
 si_menu.2.desc:0 "Automatically upgrade buildings in holdings you personally hold, as long as your treasury stays above your chosen reserve."
 si_menu.2.enable:0 "Enable"
 si_menu.2.disable:0 "Disable"
 si_menu.2.change_reserve:0 "Change Reserve Amount"
 si_menu.2.back:0 "Back"
 si_menu.3.t:0 "Domain Reserve Floor"
 si_menu.3.desc:0 "Never spend below this amount of gold/treasury."
 si_menu.3.opt_0:0 "0"
 si_menu.3.opt_100:0 "100"
 si_menu.3.opt_250:0 "250"
 si_menu.3.opt_500:0 "500"
 si_menu.3.opt_1000:0 "1,000"
 si_menu.3.opt_2500:0 "2,500"
 si_menu.3.opt_5000:0 "5,000"
 si_menu.3.opt_10000:0 "10,000"
 si_menu.3.opt_25000:0 "25,000"
 si_menu.3.opt_50000:0 "50,000"
 si_menu.3.back:0 "Back"
```

- [ ] **Step 4: Verify**

```bash
grep -i "si_menu" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Ask the user to reload, take the decision, click "Configure Domain Upgrades," enable it, pick a reserve amount, and confirm they land back on the hub with the description now showing "ON" and the chosen amount.

- [ ] **Step 5: Commit**

```bash
git add events/si_menu_events.txt localization/english/si_l_english.yml
git commit -m "Add domain toggle and reserve floor submenu"
```

---

### Task 4: Vassal toggle & reserve floor submenu

**Files:**
- Modify: `events/si_menu_events.txt` (add `si_menu.4`, `si_menu.5`)
- Modify: `localization/english/si_l_english.yml`

**Interfaces:**
- Consumes: `si_vassal_enabled`, `si_vassal_reserve`.
- Produces: event ids `si_menu.4` (vassal toggle), `si_menu.5` (vassal reserve picker). Mirrors Task 3's structure exactly, on the vassal variables.

- [ ] **Step 1: Write the vassal toggle and reserve events**

Append to `events/si_menu_events.txt`:

```
si_menu.4 = {
	type = character_event
	title = si_menu.4.t
	desc = si_menu.4.desc
	theme = council
	left_icon = root

	option = {
		name = si_menu.4.enable
		trigger = { var:si_vassal_enabled = 0 }
		set_variable = { name = si_vassal_enabled value = 1 }
		trigger_event = si_menu.5
	}
	option = {
		name = si_menu.4.disable
		trigger = { var:si_vassal_enabled = 1 }
		set_variable = { name = si_vassal_enabled value = 0 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.4.change_reserve
		trigger = { var:si_vassal_enabled = 1 }
		trigger_event = si_menu.5
	}
	option = {
		name = si_menu.4.back
		trigger_event = si_menu.1
	}
}

si_menu.5 = {
	type = character_event
	title = si_menu.5.t
	desc = si_menu.5.desc
	theme = council
	left_icon = root

	option = {
		name = si_menu.3.opt_0
		set_variable = { name = si_vassal_reserve value = 0 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_100
		set_variable = { name = si_vassal_reserve value = 100 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_250
		set_variable = { name = si_vassal_reserve value = 250 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_500
		set_variable = { name = si_vassal_reserve value = 500 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_1000
		set_variable = { name = si_vassal_reserve value = 1000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_2500
		set_variable = { name = si_vassal_reserve value = 2500 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_5000
		set_variable = { name = si_vassal_reserve value = 5000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_10000
		set_variable = { name = si_vassal_reserve value = 10000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_25000
		set_variable = { name = si_vassal_reserve value = 25000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.opt_50000
		set_variable = { name = si_vassal_reserve value = 50000 }
		trigger_event = si_menu.1
	}
	option = {
		name = si_menu.3.back
		trigger_event = si_menu.1
	}
}
```

(Reuses the `si_menu.3.opt_*`/`si_menu.3.back` localization keys from Task 3 since the option labels are identical amounts — only the event's own `t`/`desc` are new keys.)

- [ ] **Step 2: Add localization**

Append to `localization/english/si_l_english.yml`:

```yaml
 si_menu.4.t:0 "Vassal Subsidies"
 si_menu.4.desc:0 "Automatically fund building upgrades in your vassals' holdings, realm-wide, out of your own treasury — as long as your treasury stays above your chosen reserve."
 si_menu.4.enable:0 "Enable"
 si_menu.4.disable:0 "Disable"
 si_menu.4.change_reserve:0 "Change Reserve Amount"
 si_menu.4.back:0 "Back"
 si_menu.5.t:0 "Vassal Subsidy Reserve Floor"
 si_menu.5.desc:0 "Never spend below this amount of gold/treasury on vassal subsidies. Domain upgrades (if enabled) are always paid for first."
```

- [ ] **Step 3: Verify**

```bash
grep -i "si_menu" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Ask the user to reload and confirm "Configure Vassal Subsidies" behaves the same way domain configuration did.

- [ ] **Step 4: Commit**

```bash
git add events/si_menu_events.txt localization/english/si_l_english.yml
git commit -m "Add vassal toggle and reserve floor submenu"
```

---

### Task 5: Category priority ranking wizard

**Files:**
- Modify: `events/si_menu_events.txt` (add `si_menu.6` through `si_menu.11`)
- Modify: `localization/english/si_l_english.yml`

**Interfaces:**
- Consumes: `si_priority_1..6`.
- Produces: a 6-step wizard (`si_menu.6` picks 1st priority, `si_menu.7` picks 2nd from the remaining 5, ... `si_menu.11` picks the 6th automatically since only one remains). Each step writes one `si_priority_N` variable and uses a scratch variable `si_wizard_taken_<category>` to hide already-picked categories from later steps.

- [ ] **Step 1: Write a helper effect to reset wizard scratch state**

Append to `common/scripted_effects/si_settings_effects.txt`:

```
si_reset_priority_wizard_effect = {
	set_variable = { name = si_wizard_taken_1 value = 0 }
	set_variable = { name = si_wizard_taken_2 value = 0 }
	set_variable = { name = si_wizard_taken_3 value = 0 }
	set_variable = { name = si_wizard_taken_4 value = 0 }
	set_variable = { name = si_wizard_taken_5 value = 0 }
	set_variable = { name = si_wizard_taken_6 value = 0 }
	set_variable = { name = si_wizard_step value = 1 }
}
```

- [ ] **Step 2: Point the hub's "Set Category Priorities" at a reset + the wizard**

Edit `events/si_menu_events.txt`, in `si_menu.1`, replace the third option:

```
	option = { # Set Priorities
		name = si_menu.1.c
		si_reset_priority_wizard_effect = yes
		trigger_event = si_menu.6
	}
```

- [ ] **Step 3: Write the 6-step wizard**

Append to `events/si_menu_events.txt`. Each step offers only categories not yet taken (checked via `var:si_wizard_taken_N = 0`), writes `si_priority_<step>` and the matching `si_wizard_taken_<category>`, then advances `si_wizard_step` and re-fires the SAME event `si_menu.6` — a single self-looping event rather than 6 separate ones, since the "which options are visible" logic is identical each time and only the target slot (`si_wizard_step`) changes:

```
si_menu.6 = {
	type = character_event
	title = si_menu.6.t
	desc = {
		first_valid = {
			triggered_desc = { trigger = { var:si_wizard_step = 1 } desc = si_menu.6.desc_1 }
			triggered_desc = { trigger = { var:si_wizard_step = 2 } desc = si_menu.6.desc_2 }
			triggered_desc = { trigger = { var:si_wizard_step = 3 } desc = si_menu.6.desc_3 }
			triggered_desc = { trigger = { var:si_wizard_step = 4 } desc = si_menu.6.desc_4 }
			triggered_desc = { trigger = { var:si_wizard_step = 5 } desc = si_menu.6.desc_5 }
			desc = si_menu.6.desc_6
		}
	}
	theme = council
	left_icon = root

	option = {
		name = si_menu.6.economy
		trigger = { var:si_wizard_taken_1 = 0 }
		si_wizard_pick_effect = { CATEGORY = 1 }
	}
	option = {
		name = si_menu.6.development
		trigger = { var:si_wizard_taken_2 = 0 }
		si_wizard_pick_effect = { CATEGORY = 2 }
	}
	option = {
		name = si_menu.6.levy
		trigger = { var:si_wizard_taken_3 = 0 }
		si_wizard_pick_effect = { CATEGORY = 3 }
	}
	option = {
		name = si_menu.6.maa
		trigger = { var:si_wizard_taken_4 = 0 }
		si_wizard_pick_effect = { CATEGORY = 4 }
	}
	option = {
		name = si_menu.6.fortification
		trigger = { var:si_wizard_taken_5 = 0 }
		si_wizard_pick_effect = { CATEGORY = 5 }
	}
	option = {
		name = si_menu.6.specials
		trigger = { var:si_wizard_taken_6 = 0 }
		si_wizard_pick_effect = { CATEGORY = 6 }
	}
}
```

- [ ] **Step 4: Write the pick-and-advance scripted effect**

Append to `common/scripted_effects/si_settings_effects.txt`:

```
si_wizard_pick_effect = { # Records $CATEGORY$ into the current wizard step's si_priority_N, then advances or returns to hub
	set_variable = { name = si_wizard_taken_$CATEGORY$ value = 1 }

	if = {
		limit = { var:si_wizard_step = 1 }
		set_variable = { name = si_priority_1 value = $CATEGORY$ }
	}
	else_if = {
		limit = { var:si_wizard_step = 2 }
		set_variable = { name = si_priority_2 value = $CATEGORY$ }
	}
	else_if = {
		limit = { var:si_wizard_step = 3 }
		set_variable = { name = si_priority_3 value = $CATEGORY$ }
	}
	else_if = {
		limit = { var:si_wizard_step = 4 }
		set_variable = { name = si_priority_4 value = $CATEGORY$ }
	}
	else_if = {
		limit = { var:si_wizard_step = 5 }
		set_variable = { name = si_priority_5 value = $CATEGORY$ }
	}
	else_if = {
		limit = { var:si_wizard_step = 6 }
		set_variable = { name = si_priority_6 value = $CATEGORY$ }
	}

	if = {
		limit = { var:si_wizard_step < 6 }
		change_variable = { name = si_wizard_step add = 1 }
		trigger_event = si_menu.6
	}
	else = {
		trigger_event = si_menu.1
	}
}
```

- [ ] **Step 5: Add localization**

Append to `localization/english/si_l_english.yml`:

```yaml
 si_menu.6.t:0 "Set Category Priorities"
 si_menu.6.desc_1:0 "Pick your #1 priority category — upgrades in this category are always funded first."
 si_menu.6.desc_2:0 "Pick your #2 priority category."
 si_menu.6.desc_3:0 "Pick your #3 priority category."
 si_menu.6.desc_4:0 "Pick your #4 priority category."
 si_menu.6.desc_5:0 "Pick your #5 priority category."
 si_menu.6.desc_6:0 "Only one category remains — confirm to finish."
 si_menu.6.economy:0 "Tax / Economy"
 si_menu.6.development:0 "Development Growth"
 si_menu.6.levy:0 "Levy (Troop Count)"
 si_menu.6.maa:0 "Men-at-arms"
 si_menu.6.fortification:0 "Fortification"
 si_menu.6.specials:0 "Specials (Guild Halls, Monastic Schools, Scriptorium, Megaliths)"
```

- [ ] **Step 6: Verify**

```bash
grep -i "si_menu\|si_wizard" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Ask the user to reload, run "Set Category Priorities," pick all 6 in some order, and confirm each step correctly removes the already-picked options and the wizard returns to the hub after the 6th.

- [ ] **Step 7: Commit**

```bash
git add events/si_menu_events.txt common/scripted_effects/si_settings_effects.txt localization/english/si_l_english.yml
git commit -m "Add category priority ranking wizard"
```

---

### Task 6: Building catalog — Economy, Development, Levy

**Files:**
- Create: `common/scripted_triggers/si_building_catalog_triggers.txt`
- Create: `common/scripted_values/si_building_catalog_values.txt`
- Create: `common/scripted_effects/si_building_catalog_effects.txt`

**Interfaces:**
- Consumes: none new (uses vanilla triggers `add_random_building_trigger`, `generic_economic_building_innovation_trigger`, `generic_recruitment_building_innovations_trigger`, `has_building_or_higher`).
- Produces: for each building chain X in this task's scope, `si_X_eligible_trigger` (province scope, true if X's next tier can be built now), `si_X_next_cost_value` (province scope, gold cost of X's next tier, only meaningful when `si_X_eligible_trigger` is true), `si_build_X_effect` (province scope, calls `add_building` on X's next tier — actual application deferred to Task 9's delayed-grant mechanic, so this effect is invoked by that mechanic, not directly). Also produces `si_economy_best_cost_value`, `si_development_best_cost_value`, `si_levy_best_cost_value` (province scope, cheapest eligible cost across all buildings in that category in this province, or `999999` if none) and `si_build_cheapest_economy_effect` / `si_build_cheapest_development_effect` / `si_build_cheapest_levy_effect` (province scope, builds whichever specific building was cheapest) — these three category-level constructs are what Task 10/11's pass logic actually calls.

Grounding data (verified directly against installed game files, see design spec Section 6 and this plan's research): all 21 buildings below use `cost_gold = <class>_building_tier_N_cost` (confirmed class per building) and their next-tier eligibility is `add_random_building_trigger` (fresh, empty slot) OR (`has_building_or_higher = X_01` AND not yet at `X_08` AND the listed innovation-family trigger, or the listed bespoke condition).

| Building | Category | Cost class | Innovation gate |
|---|---|---|---|
| outposts | Economy | cheap | recruitment |
| logging_camps | Economy | cheap | economic |
| peat_quarries | Economy | cheap | economic |
| plantations | Economy | cheap | economic |
| quarries | Economy | cheap | economic |
| hunting_grounds | Economy | cheap | economic |
| cereal_fields | Economy | normal | economic |
| hill_farms | Economy | cheap | economic |
| hospices | Economy | normal | economic |
| orchards | Economy | normal | economic |
| farm_estates | Economy | normal | economic |
| paddy_fields | Economy | normal | economic |
| common_tradeport | Economy | normal | bespoke (see Step 3) |
| caravanserai | Economy | expensive | bespoke (see Step 3) |
| watermills | Economy | expensive | bespoke (see Step 3) |
| windmills | Economy | expensive | bespoke (see Step 3) |
| qanats | Development | normal | bespoke (see Step 3) |
| murex_farm | Development | normal | bespoke (see Step 3) |
| waterworks | Development | normal | economic |
| pastures | Levy | normal | economic |
| barracks | Levy | normal | recruitment |

- [ ] **Step 1: Write the two shared innovation-family templates**

`common/scripted_triggers/si_building_catalog_triggers.txt`:

```
# Eligibility for a "generic economic" building (crop_rotation/manorialism/guilds/cranes progression)
si_generic_economic_eligible_trigger = { # BUILDING, requires province scope, scope:build_owner already set
	NOT = { has_building_or_higher = $BUILDING$_08 }
	OR = {
		add_random_building_trigger = { BUILDING = $BUILDING$ }
		AND = {
			has_building_or_higher = $BUILDING$_01
			generic_economic_building_innovation_trigger = { BUILDING = $BUILDING$ }
		}
	}
}

# Eligibility for a "generic recruitment" building (barracks/burhs/castle_baileys/royal_armory-style progression)
si_generic_recruitment_eligible_trigger = { # BUILDING, requires province scope, scope:build_owner already set
	NOT = { has_building_or_higher = $BUILDING$_08 }
	OR = {
		add_random_building_trigger = { BUILDING = $BUILDING$ }
		AND = {
			has_building_or_higher = $BUILDING$_01
			generic_recruitment_building_innovations_trigger = { BUILDING = $BUILDING$ }
		}
	}
}
```

- [ ] **Step 2: Write the shared next-tier-cost value template**

`common/scripted_values/si_building_catalog_values.txt`:

```
# Returns the gold cost of BUILDING's next tier, given its cost CLASS (cheap/normal/expensive). Province scope.
si_next_tier_cost_value = { # BUILDING, CLASS
	if = {
		limit = { NOT = { has_building_or_higher = $BUILDING$_01 } }
		value = $CLASS$_building_tier_1_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_01 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_02 } }
		value = $CLASS$_building_tier_2_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_02 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_03 } }
		value = $CLASS$_building_tier_3_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_03 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_04 } }
		value = $CLASS$_building_tier_4_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_04 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_05 } }
		value = $CLASS$_building_tier_5_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_05 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_06 } }
		value = $CLASS$_building_tier_6_cost
	}
	else_if = {
		limit = { has_building_or_higher = $BUILDING$_06 }
		limit = { NOT = { has_building_or_higher = $BUILDING$_07 } }
		value = $CLASS$_building_tier_7_cost
	}
	else = {
		value = $CLASS$_building_tier_8_cost
	}
}
```

Note: `$CLASS$_building_tier_1_cost` through `_8_cost` are real vanilla scripted_values defined in `common/script_values/00_building_values.txt` for the `cheap`, `normal`, and `expensive` classes (confirmed by reading that file directly) — this mod does not redefine them, only references them by the standard Jomini text-substitution mechanism.

- [ ] **Step 3: Write the bespoke eligibility triggers**

Append to `common/scripted_triggers/si_building_catalog_triggers.txt` (each mirrors the exact innovation logic vanilla itself uses for that specific building, per `common/scripted_effects/00_building_effects.txt`):

```
si_common_tradeport_eligible_trigger = {
	NOT = { has_building_or_higher = common_tradeport_08 }
	OR = {
		add_random_building_trigger = { BUILDING = common_tradeport }
		AND = {
			has_building_or_higher = common_tradeport_01
			OR = {
				AND = {
					NOT = { has_building_or_higher = common_tradeport_02 }
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_crop_rotation
							has_cultural_parameter = next_level_trade_ports
						}
					}
				}
				AND = {
					has_building_or_higher = common_tradeport_02
					NOT = { has_building_or_higher = common_tradeport_04 }
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_manorialism
							AND = { has_innovation = innovation_crop_rotation has_cultural_parameter = next_level_trade_ports }
						}
					}
				}
				AND = {
					has_building_or_higher = common_tradeport_04
					NOT = { has_building_or_higher = common_tradeport_06 }
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_windmills
							AND = { has_innovation = innovation_manorialism has_cultural_parameter = next_level_trade_ports }
						}
					}
				}
				AND = {
					has_building_or_higher = common_tradeport_06
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_cranes
							AND = { has_innovation = innovation_windmills has_cultural_parameter = next_level_trade_ports }
						}
					}
				}
			}
		}
	}
}

si_qanats_eligible_trigger = {
	NOT = { has_building_or_higher = qanats_08 }
	OR = {
		add_random_building_trigger = { BUILDING = qanats }
		AND = {
			has_building_or_higher = qanats_01
			OR = {
				AND = {
					NOT = { has_building_or_higher = qanats_02 }
					scope:build_owner.culture = { has_cultural_parameter = unlocks_qanat_building }
				}
				AND = { has_building_or_higher = qanats_02 NOT = { has_building_or_higher = qanats_04 } }
				AND = { has_building_or_higher = qanats_04 NOT = { has_building_or_higher = qanats_06 } }
				has_building_or_higher = qanats_06
			}
		}
	}
}

si_murex_farm_eligible_trigger = {
	NOT = { has_building_or_higher = murex_farm_08 }
	county = { NOT = { has_county_modifier = backwater_county_modifier } }
	OR = {
		add_random_building_trigger = { BUILDING = murex_farm }
		has_building_or_higher = murex_farm_01
	}
}

si_wind_furnace_eligible_trigger = {
	NOT = { has_building_or_higher = wind_furnace_08 }
	OR = {
		add_random_building_trigger = { BUILDING = wind_furnace }
		AND = {
			has_building_or_higher = wind_furnace_01
			OR = {
				AND = { NOT = { has_building_or_higher = wind_furnace_02 } scope:build_owner.culture = { has_innovation = innovation_barracks } }
				AND = { has_building_or_higher = wind_furnace_02 NOT = { has_building_or_higher = wind_furnace_04 } scope:build_owner.culture = { has_innovation = innovation_burhs } }
				AND = { has_building_or_higher = wind_furnace_04 NOT = { has_building_or_higher = wind_furnace_05 } scope:build_owner.culture = { has_innovation = innovation_castle_baileys } }
				AND = { has_building_or_higher = wind_furnace_05 NOT = { has_building_or_higher = wind_furnace_06 } scope:build_owner.culture = { has_innovation = innovation_royal_armory } }
				has_building_or_higher = wind_furnace_06
			}
		}
	}
}

si_caravanserai_eligible_trigger = {
	NOT = { has_building_or_higher = caravanserai_08 }
	is_county_capital = yes
	building_caravanserai_requirement_terrain = yes
	scope:build_owner.culture = { has_innovation = innovation_guilds }
	OR = {
		AND = { NOT = { has_building_or_higher = caravanserai_01 } free_building_slots > 0 }
		AND = {
			has_building_or_higher = caravanserai_01
			OR = {
				AND = { NOT = { has_building_or_higher = caravanserai_02 } }
				AND = { has_building_or_higher = caravanserai_04 NOT = { has_building_or_higher = caravanserai_05 } scope:build_owner.culture = { has_innovation = innovation_cranes } }
				has_building_or_higher = caravanserai_02
			}
		}
	}
}

si_watermills_eligible_trigger = {
	NOT = { has_building_or_higher = watermills_08 }
	is_county_capital = yes
	building_watermills_requirement_terrain = yes
	scope:build_owner.culture = { has_innovation = innovation_windmills }
	OR = {
		AND = { NOT = { has_building_or_higher = watermills_01 } free_building_slots > 0 }
		AND = {
			has_building_or_higher = watermills_01
			OR = {
				AND = { NOT = { has_building_or_higher = watermills_02 } }
				AND = { has_building_or_higher = watermills_04 NOT = { has_building_or_higher = watermills_05 } scope:build_owner.culture = { has_innovation = innovation_cranes } }
				has_building_or_higher = watermills_02
			}
		}
	}
}

si_windmills_eligible_trigger = {
	NOT = { has_building_or_higher = windmills_08 }
	is_county_capital = yes
	building_windmills_requirement_terrain = yes
	scope:build_owner.culture = { has_innovation = innovation_windmills }
	OR = {
		AND = { NOT = { has_building_or_higher = windmills_01 } free_building_slots > 0 }
		AND = {
			has_building_or_higher = windmills_01
			OR = {
				AND = { NOT = { has_building_or_higher = windmills_02 } }
				AND = { has_building_or_higher = windmills_04 NOT = { has_building_or_higher = windmills_05 } scope:build_owner.culture = { has_innovation = innovation_cranes } }
				has_building_or_higher = windmills_02
			}
		}
	}
}
```

- [ ] **Step 4: Write per-building select-and-cost wiring for this task's 21 buildings**

Append to `common/scripted_effects/si_building_catalog_effects.txt` (new file). This defines, for each building, a `si_try_X_effect` that — if `si_X_eligible_trigger` holds and its cost is less than the running `var:si_candidate_cost` — records it as the new best candidate (`si_candidate_cost`, `si_candidate_building` as a numeric index per the table order below):

```
# Building index table for this file's $CATEGORY$ effects (order fixed, used by si_build_cheapest_*_effect switches):
# Economy:      0 outposts, 1 logging_camps, 2 peat_quarries, 3 plantations, 4 quarries, 5 hunting_grounds,
#               6 cereal_fields, 7 hill_farms, 8 hospices, 9 orchards, 10 farm_estates, 11 paddy_fields,
#               12 common_tradeport, 13 caravanserai, 14 watermills, 15 windmills
# Development:  0 qanats, 1 murex_farm, 2 waterworks
# Levy:         0 pastures, 1 barracks

si_try_economy_candidate_effect = { # $INDEX$, $ELIGIBLE_TRIGGER$, $COST_VALUE$ — invoked once per Economy building
	if = {
		limit = {
			$ELIGIBLE_TRIGGER$ = yes
			$COST_VALUE$ < var:si_candidate_cost
		}
		set_variable = { name = si_candidate_cost value = $COST_VALUE$ }
		set_variable = { name = si_candidate_building value = $INDEX$ }
	}
}

si_economy_best_cost_value = { # Province scope. Returns cheapest eligible Economy upgrade cost, or 999999 if none.
	value = 999999
	# This value is only used for ordering provinces (Task 8); the actual candidate index is
	# recomputed by si_build_cheapest_economy_effect at spend time to avoid double bookkeeping.
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = outposts } } min = si_next_tier_cost_value = { BUILDING = outposts CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = logging_camps } } min = si_next_tier_cost_value = { BUILDING = logging_camps CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = peat_quarries } } min = si_next_tier_cost_value = { BUILDING = peat_quarries CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = plantations } } min = si_next_tier_cost_value = { BUILDING = plantations CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = quarries } } min = si_next_tier_cost_value = { BUILDING = quarries CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = hunting_grounds } } min = si_next_tier_cost_value = { BUILDING = hunting_grounds CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = cereal_fields } } min = si_next_tier_cost_value = { BUILDING = cereal_fields CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = hill_farms } } min = si_next_tier_cost_value = { BUILDING = hill_farms CLASS = cheap } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = hospices } } min = si_next_tier_cost_value = { BUILDING = hospices CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = orchards } } min = si_next_tier_cost_value = { BUILDING = orchards CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = farm_estates } } min = si_next_tier_cost_value = { BUILDING = farm_estates CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = paddy_fields } } min = si_next_tier_cost_value = { BUILDING = paddy_fields CLASS = normal } }
	if = { limit = { si_common_tradeport_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = common_tradeport CLASS = normal } }
	if = { limit = { si_caravanserai_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = caravanserai CLASS = expensive } }
	if = { limit = { si_watermills_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = watermills CLASS = expensive } }
	if = { limit = { si_windmills_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = windmills CLASS = expensive } }
}
```

**IMPORTANT — verify `min` support before relying on this pattern:** this step assumes Jomini `value = { ... }` blocks support a `min = <value>` operation (clamp running total down to the given value if it's lower). This is very likely correct (it mirrors `add`/`multiply`/`subtract` operations that unquestionably exist in script_values), but was not directly confirmed by grepping the installed game files during planning. Before writing the rest of this step's code, run:

```bash
grep -rn "min = " "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/script_values" | head -5
```

If real usage examples appear, proceed exactly as written above. **If no usage is found**, replace the `si_economy_best_cost_value` pattern with the imperative-effect version instead (this is the fallback already proven safe elsewhere in this plan): rewrite it as `si_economy_best_cost_effect` (an *effect*, not a *value*) using `set_variable`/`change_variable` and `if = { limit = { X < var:si_candidate_cost } set_variable = { name = si_candidate_cost value = X } }` for each building in sequence — functionally identical, just spelled as an effect instead of a value block, and called instead of referenced by `order_by` (Task 8 would then need to call this effect on each candidate province directly rather than using native `ordered_x` sorting — note this as a required adjustment to Task 8 if this fallback is used).

Repeat the same `si_<category>_best_cost_value` pattern (or its effect-based fallback, per whichever the verification above selected) for Development (qanats/normal, murex_farm/normal, waterworks/normal — waterworks uses `si_generic_economic_eligible_trigger`) and Levy (pastures/normal using `si_generic_economic_eligible_trigger`, barracks/normal using `si_generic_recruitment_eligible_trigger`):

```
si_development_best_cost_value = {
	value = 999999
	if = { limit = { si_qanats_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = qanats CLASS = normal } }
	if = { limit = { si_murex_farm_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = murex_farm CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = waterworks } } min = si_next_tier_cost_value = { BUILDING = waterworks CLASS = normal } }
}

si_levy_best_cost_value = {
	value = 999999
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = pastures } } min = si_next_tier_cost_value = { BUILDING = pastures CLASS = normal } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = barracks } } min = si_next_tier_cost_value = { BUILDING = barracks CLASS = normal } }
}
```

- [ ] **Step 5: Write the build-cheapest effects (actually applies `add_building` to whichever candidate is cheapest)**

Append to `common/scripted_effects/si_building_catalog_effects.txt`:

```
si_build_cheapest_economy_effect = { # Province scope. Re-derives the cheapest eligible candidate and calls add_next_building_tier_effect on it.
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = outposts } }        COST_VALUE = { si_next_tier_cost_value = { BUILDING = outposts CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 1  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = logging_camps } }    COST_VALUE = { si_next_tier_cost_value = { BUILDING = logging_camps CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 2  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = peat_quarries } }    COST_VALUE = { si_next_tier_cost_value = { BUILDING = peat_quarries CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 3  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = plantations } }      COST_VALUE = { si_next_tier_cost_value = { BUILDING = plantations CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 4  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = quarries } }         COST_VALUE = { si_next_tier_cost_value = { BUILDING = quarries CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 5  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = hunting_grounds } }  COST_VALUE = { si_next_tier_cost_value = { BUILDING = hunting_grounds CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 6  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = cereal_fields } }    COST_VALUE = { si_next_tier_cost_value = { BUILDING = cereal_fields CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 7  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = hill_farms } }       COST_VALUE = { si_next_tier_cost_value = { BUILDING = hill_farms CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 8  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = hospices } }         COST_VALUE = { si_next_tier_cost_value = { BUILDING = hospices CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 9  ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = orchards } }         COST_VALUE = { si_next_tier_cost_value = { BUILDING = orchards CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 10 ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = farm_estates } }     COST_VALUE = { si_next_tier_cost_value = { BUILDING = farm_estates CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 11 ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = paddy_fields } }     COST_VALUE = { si_next_tier_cost_value = { BUILDING = paddy_fields CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 12 ELIGIBLE_TRIGGER = { si_common_tradeport_eligible_trigger = yes }                             COST_VALUE = { si_next_tier_cost_value = { BUILDING = common_tradeport CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 13 ELIGIBLE_TRIGGER = { si_caravanserai_eligible_trigger = yes }                                 COST_VALUE = { si_next_tier_cost_value = { BUILDING = caravanserai CLASS = expensive } } }
	si_try_economy_candidate_effect = { INDEX = 14 ELIGIBLE_TRIGGER = { si_watermills_eligible_trigger = yes }                                   COST_VALUE = { si_next_tier_cost_value = { BUILDING = watermills CLASS = expensive } } }
	si_try_economy_candidate_effect = { INDEX = 15 ELIGIBLE_TRIGGER = { si_windmills_eligible_trigger = yes }                                    COST_VALUE = { si_next_tier_cost_value = { BUILDING = windmills CLASS = expensive } } }

	# NOTE: this switch only records which building was chosen (si_candidate_building, set above by
	# si_try_economy_candidate_effect) — it does NOT call add_next_building_tier_effect. Per the delayed-grant
	# design (spec Section 8 / Task 9), the actual add_next_building_tier_effect call happens later, in
	# si_apply_pending_economy_effect (Task 9), once the construction delay has elapsed. This effect's only
	# remaining job is stamping how long that delay should be, via si_pending_grant_days (Task 9 Step 2).
}

si_build_cheapest_development_effect = {
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0 ELIGIBLE_TRIGGER = { si_qanats_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = qanats CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 1 ELIGIBLE_TRIGGER = { si_murex_farm_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = murex_farm CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 2 ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = waterworks } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = waterworks CLASS = normal } } }
	# Selection only — see note above. Application happens in si_apply_pending_development_effect (Task 9).
}

si_build_cheapest_levy_effect = {
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0 ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = pastures } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = pastures CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 1 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = barracks } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = barracks CLASS = normal } } }
	# Selection only — see note above. Application happens in si_apply_pending_levy_effect (Task 9).
}
```

Note: `add_next_building_tier_effect` is the real vanilla scripted_effect from `common/scripted_effects/00_building_effects.txt` (confirmed by direct read during planning) — it walks the chain (`if NOT has_building_or_higher X_01 → add X_01, else_if has X_01 → add X_02, ...`) so we don't need to duplicate tier-selection logic here, only eligibility/cost.

- [ ] **Step 6: Verify script loads cleanly**

This task has no in-game UI yet (it's plumbing consumed by Task 8-11), so verification is purely: reload the mod and confirm no new parse errors.

```bash
grep -i "si_economy\|si_development\|si_levy\|si_generic_economic\|si_generic_recruitment\|si_next_tier_cost\|si_common_tradeport\|si_qanats\|si_murex_farm\|si_wind_furnace\|si_caravanserai\|si_watermills\|si_windmills" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output.

- [ ] **Step 7: Commit**

```bash
git add common/scripted_triggers/si_building_catalog_triggers.txt common/scripted_values/si_building_catalog_values.txt common/scripted_effects/si_building_catalog_effects.txt
git commit -m "Add Economy, Development, and Levy building catalog"
```

---

### Task 7: Building catalog — Men-at-arms, Fortification, Specials

**Files:**
- Modify: `common/scripted_triggers/si_building_catalog_triggers.txt`
- Modify: `common/scripted_effects/si_building_catalog_effects.txt`

**Interfaces:**
- Consumes: `si_generic_recruitment_eligible_trigger`, `si_next_tier_cost_value`, `si_try_economy_candidate_effect` (all from Task 6 — reused as-is, no MaA/Fortification/Specials-specific variant needed since the "try candidate" plumbing is category-agnostic).
- Produces: `si_build_cheapest_maa_effect`, `si_build_cheapest_fortification_effect`, `si_build_cheapest_specials_effect` (mirroring Task 6's `si_build_cheapest_*_effect` shape exactly — these three plus Task 6's three are what Task 10/11 call, one per category per priority-ranked pass).

Grounding data:

| Building | Category | Cost class | Innovation gate | Extra requirement |
|---|---|---|---|---|
| military_camps | MaA | cheap | recruitment | — |
| horse_pastures | MaA | cheap | recruitment | — |
| hillside_grazing | MaA | cheap | recruitment | — |
| warrior_lodges | MaA | cheap | recruitment | — |
| camel_farms | MaA | normal | recruitment | — |
| elephant_pens | MaA | normal | economic | — |
| stables | MaA | normal | recruitment | — |
| smiths | MaA | normal | recruitment | — |
| regimental_grounds | MaA | expensive | recruitment | — |
| wind_furnace | MaA | normal | bespoke (Task 6 Step 3) | — |
| workshops | MaA | expensive | bespoke (see Step 1) | — |
| hill_forts | Fortification | cheap | fortification | — |
| ramparts | Fortification | cheap | fortification | — |
| curtain_walls | Fortification | cheap | fortification | — |
| watchtowers | Fortification | cheap | fortification | — |
| monastic_schools | Specials | normal | bespoke (see Step 1) | `has_holding_type = church_holding` |
| guild_halls | Specials | normal | bespoke (see Step 1) | `has_holding_type = city_holding` |
| scriptorium | Specials | normal | economic | `has_holding_type = church_holding`, `scope:build_owner = { has_dlc_feature = legends }` |
| megalith | Specials | normal | bespoke (see Step 1) | `has_holding_type = church_holding` |

Note: this task does not cover the 17 duchy-capital buildings (`dragon_kiln`, `examination_hall`, `tower_of_silence`, `charnel_grounds`, `burial_site`, `royal_garden`, `military_academy`, `march`, `siege_works`, `royal_armory`, `jousting_lists`, `blacksmiths`, `archery_ranges`, `tax_assessor`, `leisure_palace`, `royal_forest`, `great_megalith`). These are culturally-gated, mutually-exclusive one-per-duchy picks — automating "which one to build" is a meaningfully different, higher-stakes decision than routine tier-upgrading (the player likely wants to choose their duchy's unique building themselves), so "Specials" in this mod covers guild_halls/monastic_schools/scriptorium/megalith only. This narrows spec Section 6's "Specials... and duchy capital buildings" — flagging this deviation explicitly rather than silently dropping it.

- [ ] **Step 1: Write the fortification generic template and the two remaining bespoke triggers**

Append to `common/scripted_triggers/si_building_catalog_triggers.txt`:

```
si_generic_fortification_eligible_trigger = { # BUILDING, requires province scope, scope:build_owner already set
	NOT = { has_building_or_higher = $BUILDING$_08 }
	OR = {
		add_random_building_trigger = { BUILDING = $BUILDING$ }
		AND = {
			has_building_or_higher = $BUILDING$_01
			generic_fortification_building_innovations_trigger = { BUILDING = $BUILDING$ }
		}
	}
}

si_workshops_eligible_trigger = {
	NOT = { has_building_or_higher = workshops_08 }
	OR = {
		add_random_building_trigger = { BUILDING = workshops }
		AND = {
			has_building_or_higher = workshops_01
			scope:build_owner.culture = { has_innovation = innovation_advanced_bowmaking }
			OR = {
				NOT = { has_building_or_higher = workshops_02 }
				AND = { has_building_or_higher = workshops_04 NOT = { has_building_or_higher = workshops_05 } scope:build_owner.culture = { has_innovation = innovation_royal_armory } }
				has_building_or_higher = workshops_02
			}
		}
	}
}

si_monastic_schools_eligible_trigger = {
	has_holding_type = church_holding
	NOT = { has_building_or_higher = monastic_schools_08 }
	OR = {
		add_random_building_trigger = { BUILDING = monastic_schools }
		AND = {
			has_building_or_higher = monastic_schools_01
			scope:build_owner.culture = { has_innovation = innovation_city_planning }
			OR = {
				NOT = { has_building_or_higher = monastic_schools_02 }
				AND = { has_building_or_higher = monastic_schools_02 NOT = { has_building_or_higher = monastic_schools_04 } scope:build_owner.culture = { has_innovation = innovation_manorialism } }
				AND = { has_building_or_higher = monastic_schools_04 NOT = { has_building_or_higher = monastic_schools_06 } scope:build_owner.culture = { has_innovation = innovation_windmills } }
				AND = { has_building_or_higher = monastic_schools_06 scope:build_owner.culture = { has_innovation = innovation_cranes } }
			}
		}
	}
}

si_guild_halls_eligible_trigger = {
	has_holding_type = city_holding
	NOT = { has_building_or_higher = guild_halls_08 }
	OR = {
		add_random_building_trigger = { BUILDING = guild_halls }
		AND = {
			has_building_or_higher = guild_halls_01
			OR = {
				AND = {
					NOT = { has_building_or_higher = guild_halls_02 }
					scope:build_owner.culture = { OR = { has_innovation = innovation_crop_rotation has_cultural_parameter = next_level_guild_halls } }
				}
				AND = {
					has_building_or_higher = guild_halls_02
					NOT = { has_building_or_higher = guild_halls_04 }
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_manorialism
							AND = { has_cultural_parameter = next_level_guild_halls has_innovation = innovation_crop_rotation }
						}
					}
				}
				AND = {
					has_building_or_higher = guild_halls_04
					scope:build_owner.culture = {
						OR = {
							has_innovation = innovation_cranes
							AND = { has_cultural_parameter = next_level_guild_halls has_innovation = innovation_guilds }
						}
					}
				}
			}
		}
	}
}

si_megalith_eligible_trigger = {
	has_holding_type = church_holding
	NOT = { has_building_or_higher = megalith_08 }
	scope:build_owner.faith = { has_doctrine_parameter = can_build_megaliths }
	OR = {
		add_random_building_trigger = { BUILDING = megalith }
		AND = {
			has_building_or_higher = megalith_01
			OR = {
				NOT = { has_building_or_higher = megalith_02 }
				AND = { has_building_or_higher = megalith_02 scope:build_owner.culture = { has_innovation = innovation_city_planning } }
			}
		}
	}
}
```

- [ ] **Step 2: Write the MaA, Fortification, Specials build-cheapest effects**

Append to `common/scripted_effects/si_building_catalog_effects.txt`:

```
si_build_cheapest_maa_effect = {
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = military_camps } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = military_camps CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 1 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = horse_pastures } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = horse_pastures CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 2 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = hillside_grazing } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = hillside_grazing CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 3 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = warrior_lodges } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = warrior_lodges CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 4 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = camel_farms } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = camel_farms CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 5 ELIGIBLE_TRIGGER = { si_generic_economic_eligible_trigger = { BUILDING = elephant_pens } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = elephant_pens CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 6 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = stables } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = stables CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 7 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = smiths } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = smiths CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 8 ELIGIBLE_TRIGGER = { si_generic_recruitment_eligible_trigger = { BUILDING = regimental_grounds } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = regimental_grounds CLASS = expensive } } }
	si_try_economy_candidate_effect = { INDEX = 9 ELIGIBLE_TRIGGER = { si_wind_furnace_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = wind_furnace CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 10 ELIGIBLE_TRIGGER = { si_workshops_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = workshops CLASS = expensive } } }
	# Selection only, per Task 6 Step 5's note — application happens in si_apply_pending_maa_effect (Task 9).
}

si_build_cheapest_fortification_effect = {
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0 ELIGIBLE_TRIGGER = { si_generic_fortification_eligible_trigger = { BUILDING = hill_forts } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = hill_forts CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 1 ELIGIBLE_TRIGGER = { si_generic_fortification_eligible_trigger = { BUILDING = ramparts } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = ramparts CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 2 ELIGIBLE_TRIGGER = { si_generic_fortification_eligible_trigger = { BUILDING = curtain_walls } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = curtain_walls CLASS = cheap } } }
	si_try_economy_candidate_effect = { INDEX = 3 ELIGIBLE_TRIGGER = { si_generic_fortification_eligible_trigger = { BUILDING = watchtowers } } COST_VALUE = { si_next_tier_cost_value = { BUILDING = watchtowers CLASS = cheap } } }
	# Selection only — application happens in si_apply_pending_fortification_effect (Task 9).
}

si_build_cheapest_specials_effect = {
	set_variable = { name = si_candidate_cost value = 999999 }
	set_variable = { name = si_candidate_building value = -1 }
	si_try_economy_candidate_effect = { INDEX = 0 ELIGIBLE_TRIGGER = { si_monastic_schools_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = monastic_schools CLASS = normal } } }
	si_try_economy_candidate_effect = { INDEX = 1 ELIGIBLE_TRIGGER = { si_guild_halls_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = guild_halls CLASS = normal } } }
	if = {
		limit = {
			has_holding_type = church_holding
			scope:build_owner = { has_dlc_feature = legends }
			si_generic_economic_eligible_trigger = { BUILDING = scriptorium }
			si_next_tier_cost_value = { BUILDING = scriptorium CLASS = normal } < var:si_candidate_cost
		}
		set_variable = { name = si_candidate_cost value = si_next_tier_cost_value = { BUILDING = scriptorium CLASS = normal } }
		set_variable = { name = si_candidate_building value = 2 }
	}
	si_try_economy_candidate_effect = { INDEX = 3 ELIGIBLE_TRIGGER = { si_megalith_eligible_trigger = yes } COST_VALUE = { si_next_tier_cost_value = { BUILDING = megalith CLASS = normal } } }
	# Selection only — application happens in si_apply_pending_specials_effect (Task 9).
}
```

(`scriptorium`'s extra DLC-gate condition can't be expressed by the plain `si_try_economy_candidate_effect` template since it needs an extra AND clause — it's inlined directly instead, same net effect.)

- [ ] **Step 3: Write the matching `si_<category>_best_cost_value` blocks used by Task 8's ordering**

Append to `common/scripted_values/si_building_catalog_values.txt` (or its imperative-effect fallback, matching whatever Task 6 Step 4 determined `min` support to be):

```
si_maa_best_cost_value = {
	value = 999999
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = military_camps } } min = si_next_tier_cost_value = { BUILDING = military_camps CLASS = cheap } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = horse_pastures } } min = si_next_tier_cost_value = { BUILDING = horse_pastures CLASS = cheap } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = hillside_grazing } } min = si_next_tier_cost_value = { BUILDING = hillside_grazing CLASS = cheap } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = warrior_lodges } } min = si_next_tier_cost_value = { BUILDING = warrior_lodges CLASS = cheap } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = camel_farms } } min = si_next_tier_cost_value = { BUILDING = camel_farms CLASS = normal } }
	if = { limit = { si_generic_economic_eligible_trigger = { BUILDING = elephant_pens } } min = si_next_tier_cost_value = { BUILDING = elephant_pens CLASS = normal } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = stables } } min = si_next_tier_cost_value = { BUILDING = stables CLASS = normal } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = smiths } } min = si_next_tier_cost_value = { BUILDING = smiths CLASS = normal } }
	if = { limit = { si_generic_recruitment_eligible_trigger = { BUILDING = regimental_grounds } } min = si_next_tier_cost_value = { BUILDING = regimental_grounds CLASS = expensive } }
	if = { limit = { si_wind_furnace_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = wind_furnace CLASS = normal } }
	if = { limit = { si_workshops_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = workshops CLASS = expensive } }
}

si_fortification_best_cost_value = {
	value = 999999
	if = { limit = { si_generic_fortification_eligible_trigger = { BUILDING = hill_forts } } min = si_next_tier_cost_value = { BUILDING = hill_forts CLASS = cheap } }
	if = { limit = { si_generic_fortification_eligible_trigger = { BUILDING = ramparts } } min = si_next_tier_cost_value = { BUILDING = ramparts CLASS = cheap } }
	if = { limit = { si_generic_fortification_eligible_trigger = { BUILDING = curtain_walls } } min = si_next_tier_cost_value = { BUILDING = curtain_walls CLASS = cheap } }
	if = { limit = { si_generic_fortification_eligible_trigger = { BUILDING = watchtowers } } min = si_next_tier_cost_value = { BUILDING = watchtowers CLASS = cheap } }
}

si_specials_best_cost_value = {
	value = 999999
	if = { limit = { si_monastic_schools_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = monastic_schools CLASS = normal } }
	if = { limit = { si_guild_halls_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = guild_halls CLASS = normal } }
	if = {
		limit = { has_holding_type = church_holding scope:build_owner = { has_dlc_feature = legends } si_generic_economic_eligible_trigger = { BUILDING = scriptorium } }
		min = si_next_tier_cost_value = { BUILDING = scriptorium CLASS = normal }
	}
	if = { limit = { si_megalith_eligible_trigger = yes } min = si_next_tier_cost_value = { BUILDING = megalith CLASS = normal } }
}
```

- [ ] **Step 4: Verify**

```bash
grep -i "si_maa\|si_fortification\|si_specials\|si_workshops\|si_monastic_schools\|si_guild_halls\|si_megalith\|si_generic_fortification" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add common/scripted_triggers/si_building_catalog_triggers.txt common/scripted_effects/si_building_catalog_effects.txt common/scripted_values/si_building_catalog_values.txt
git commit -m "Add Men-at-arms, Fortification, and Specials building catalog"
```

---

### Task 8: "Find cheapest province for category" selection helper

**Files:**
- Create: `common/scripted_effects/si_selection_effects.txt`

**Interfaces:**
- Consumes: `si_<category>_best_cost_value` (all 6, from Tasks 6-7), `si_build_cheapest_<category>_effect` (all 6, from Tasks 6-7).
- Produces: `si_spend_on_category_effect` (character scope, parameterized by `$CATEGORY_NUM$` 1-6 and `$SCOPE_LIST$`, a list-builder block — either domain holdings or vassal holdings, supplied by the caller). Repeatedly finds and funds the cheapest eligible upgrade in that category across the given scope until either no eligible candidate remains or funding it would drop the relevant reserve below its floor. This is the shared core both Task 10 (domain pass) and Task 11 (vassal pass) call once per category, per priority-ranked step.

- [ ] **Step 1: Write the category-dispatch cost-value and build-cheapest wrappers**

Append to `common/scripted_effects/si_selection_effects.txt` (new file):

```
si_category_best_cost_in_province_value = { # $CATEGORY_NUM$ — province scope
	if = { limit = { $CATEGORY_NUM$ = 1 } value = si_economy_best_cost_value }
	else_if = { limit = { $CATEGORY_NUM$ = 2 } value = si_development_best_cost_value }
	else_if = { limit = { $CATEGORY_NUM$ = 3 } value = si_levy_best_cost_value }
	else_if = { limit = { $CATEGORY_NUM$ = 4 } value = si_maa_best_cost_value }
	else_if = { limit = { $CATEGORY_NUM$ = 5 } value = si_fortification_best_cost_value }
	else = { value = si_specials_best_cost_value }
}

si_build_cheapest_in_category_effect = { # $CATEGORY_NUM$ — province scope
	if = { limit = { $CATEGORY_NUM$ = 1 } si_build_cheapest_economy_effect = yes }
	else_if = { limit = { $CATEGORY_NUM$ = 2 } si_build_cheapest_development_effect = yes }
	else_if = { limit = { $CATEGORY_NUM$ = 3 } si_build_cheapest_levy_effect = yes }
	else_if = { limit = { $CATEGORY_NUM$ = 4 } si_build_cheapest_maa_effect = yes }
	else_if = { limit = { $CATEGORY_NUM$ = 5 } si_build_cheapest_fortification_effect = yes }
	else = { si_build_cheapest_specials_effect = yes }
}
```

- [ ] **Step 2: Write the spend loop**

Append to `common/scripted_effects/si_selection_effects.txt`. This is character scope; `$PROVINCE_LIST$` is a caller-supplied block naming which list-builder to search (e.g. `every_in_list = { list = si_domain_provinces }` — Task 10/11 populate that list first). We use CK3's `ordered_in_list` to pick the single cheapest-eligible province for the category, act on it, then loop (CK3 has no native `while`, so the loop is bounded to a fixed number of iterations per call — 20 is comfortably above what a single domain/vassal pass should ever need in one month, and simply stops early via the `si_spend_more` guard once nothing affordable remains):

```
si_spend_on_category_once_effect = { # $CATEGORY_NUM$, $RESERVE_VAR$ — character scope. Returns whether it spent (via si_spent_this_round).
	set_variable = { name = si_spent_this_round value = 0 }

	ordered_in_list = {
		list = si_candidate_provinces
		order_by = {
			si_category_best_cost_in_province_value = { CATEGORY_NUM = $CATEGORY_NUM$ }
		}
		max = 999999 # ascending order (cheapest cost value first)
		count = 1

		limit = {
			si_category_best_cost_in_province_value = { CATEGORY_NUM = $CATEGORY_NUM$ } < 999999
		}

		save_temporary_scope_as = si_target_province
	}

	if = {
		limit = { exists = scope:si_target_province }
		if = {
			limit = {
				trigger_if = {
					limit = { has_treasury = yes }
					treasury - scope:si_target_province.{ si_category_best_cost_in_province_value = { CATEGORY_NUM = $CATEGORY_NUM$ } } >= var:$RESERVE_VAR$
				}
				trigger_else = {
					gold - scope:si_target_province.{ si_category_best_cost_in_province_value = { CATEGORY_NUM = $CATEGORY_NUM$ } } >= var:$RESERVE_VAR$
				}
			}
			si_pay_for_upgrade_effect = {
				PROVINCE = scope:si_target_province
				COST = scope:si_target_province.{ si_category_best_cost_in_province_value = { CATEGORY_NUM = $CATEGORY_NUM$ } }
				CATEGORY_NUM = $CATEGORY_NUM$
			}
			set_variable = { name = si_spent_this_round value = 1 }
		}
	}
}

si_spend_on_category_effect = { # $CATEGORY_NUM$, $RESERVE_VAR$ — character scope. Loops until nothing affordable remains this pass.
	set_variable = { name = si_spend_loop_guard value = 0 }
	while = {
		limit = { var:si_spend_loop_guard < 20 }
		si_spend_on_category_once_effect = { CATEGORY_NUM = $CATEGORY_NUM$ RESERVE_VAR = $RESERVE_VAR$ }
		change_variable = { name = si_spend_loop_guard add = 1 }
		if = {
			limit = { var:si_spent_this_round = 0 }
			change_variable = { name = si_spend_loop_guard value = 20 } # break out — nothing more affordable
		}
	}
}
```

**IMPORTANT — verify `while` and the `scope:X.{ value }` inline-value syntax before relying on this pattern:** this step assumes (a) CK3 script supports a `while = { limit = {...} <effects> }` looping construct, and (b) a saved scope's per-scope value can be re-evaluated inline via `scope:si_target_province.{ some_value }`. Both are plausible (Jomini does have `while` for effects, and scope-qualified value evaluation is a known pattern) but were **not** directly confirmed against the installed game files during planning — this is exactly the kind of syntax risk flagged in the design spec's Section 13. Before writing this step, run:

```bash
grep -rln "^\s*while = {" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | head -3
```

and read one real usage to confirm exact `while` syntax (limit block, no separate "condition" keyword, etc.). If `while` doesn't exist or works differently than assumed, fall back to a fixed-count sequence instead — call `si_spend_on_category_once_effect` a literal 8 times in a row (unrolled, no loop construct needed), which is simpler, has no syntax risk, and only differs behaviorally by capping this pass at 8 purchases per category instead of 20 (acceptable — the next monthly pass picks up anything left over). Prefer the unrolled version unless `while` is confirmed working, since it removes a real source of syntax risk for a purely cosmetic loop-count benefit.

- [ ] **Step 3: Write the pay-for-upgrade effect (deduct gold, hand off to Task 9's delayed grant)**

Append to `common/scripted_effects/si_selection_effects.txt` (this calls `si_schedule_delayed_grant_effect`, defined in Task 9 — safe to reference now and implement next, since both land in the same mod and load order within `common/scripted_effects/` doesn't matter for effect *definitions*, only for effects *called* before they're defined being a real parse issue in some contexts; to avoid any risk, Task 9 is implemented immediately after this task and both are verified together before either is used by Task 10/11):

```
si_pay_for_upgrade_effect = { # $PROVINCE$, $COST$, $CATEGORY_NUM$ — character scope
	trigger_if = {
		limit = { has_treasury = yes }
		remove_short_term_treasury = $COST$
	}
	trigger_else = {
		remove_short_term_gold = $COST$
	}
	$PROVINCE$ = {
		si_schedule_delayed_grant_effect = { CATEGORY_NUM = $CATEGORY_NUM$ }
	}
}
```

**Verify the exact currency-deduction effect names** (`remove_short_term_treasury`/`remove_short_term_gold` are a reasonable guess based on the `budget_short_term`/`war_chest_gold` naming seen in `00_ai_budget_effects.txt` during brainstorming research, but were not directly confirmed as valid standalone effects — the AI budget system moves gold *between* internal budget buckets, which is a different operation from *spending* gold outright):

```bash
grep -rn "^\s*remove_gold\s*=\|^\s*remove_short_term_gold\s*=\|^\s*add_gold\s*=\s*-" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | head -10
grep -rn "^\s*remove_treasury\s*=\|^\s*add_treasury\s*=\s*-" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | head -10
```

Use whichever real effect name that search confirms (most likely `add_gold = { value = $COST$ multiply = -1 }` / `add_treasury = { value = $COST$ multiply = -1 }`, or a direct `remove_gold`/`remove_treasury` if one exists) in place of the placeholder names above before moving on.

- [ ] **Step 4: Verify and commit**

```bash
grep -i "si_selection\|si_spend_on_category\|si_pay_for_upgrade\|si_category_best_cost" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output (this task's effects aren't invoked by anything yet, so this only catches parse errors).

```bash
git add common/scripted_effects/si_selection_effects.txt
git commit -m "Add cross-province cheapest-candidate selection and spend loop"
```

---

### Task 9: Delayed-grant construction mechanic

**Files:**
- Create: `common/scripted_effects/si_construction_effects.txt`
- Modify: `common/scripted_effects/si_building_catalog_effects.txt` (add the six `si_apply_pending_<category>_effect` blocks and the duration-stamping switches)
- Modify: `common/scripted_effects/si_selection_effects.txt` (`si_pay_for_upgrade_effect` calls the scheduler)
- Modify: `common/on_action/si_on_actions.txt`

**Interfaces:**
- Consumes: `si_candidate_building` (the index Task 6/7's `si_build_cheapest_<category>_effect` already leaves set — Task 8 Step 3 already established that `si_pay_for_upgrade_effect` runs immediately after selection, in the same province scope, so this index is still valid when scheduling happens).
- Produces: `si_pending_grant_days_for_economy_value` (and one sibling per category, province scope — looks up the chosen building's real construction_time from `si_candidate_building`), `si_schedule_delayed_grant_effect` (province scope), `si_apply_pending_<category>_effect` ×6 (province scope — the deferred `add_next_building_tier_effect` call, keyed on the same index), `si_apply_pending_grant_effect` (province scope, dispatches to the right category and re-validates), `si_monthly_construction_sweep_effect` (character scope, called by the monthly on_action).

Construction-time data (verified directly against `common/buildings/*.txt`, translated via the `@..._construction_time_base` macros confirmed in `common/script_values/00_building_values.txt`: quick=730, standard=1095, slow=1825):

| Building | Days | Building | Days |
|---|---|---|---|
| outposts | 730 | camel_farms | 1095 |
| logging_camps | 1095 | elephant_pens | 1825 |
| peat_quarries | 1095 | stables | 730 |
| plantations | 730 | smiths | 730 |
| quarries | 730 | regimental_grounds | 1825 |
| hunting_grounds | 730 | wind_furnace | 1095 |
| cereal_fields | 1095 | workshops | 1825 |
| hill_farms | 1095 | hill_forts | 1825 |
| hospices | 1095 | ramparts | 1095 |
| orchards | 1095 | curtain_walls | 1095 |
| farm_estates | 1825 | watchtowers | 1095 |
| paddy_fields | 1095 | monastic_schools | 1095 |
| common_tradeport | 730 | guild_halls | 1095 |
| caravanserai | 1825 | scriptorium | 1095 |
| watermills | 1825 | megalith | 1095 |
| windmills | 1825 | pastures | 730 |
| qanats | 1095 | barracks | 1095 |
| murex_farm | 1095 | military_camps | 730 |
| waterworks | 1095 | horse_pastures | 730 |
| | | hillside_grazing | 730 |
| | | warrior_lodges | 730 |

- [ ] **Step 1: Add the duration-lookup switches to Task 6/7's selection effects**

Edit `common/scripted_effects/si_building_catalog_effects.txt` (Task 6 Step 5). Immediately after each `si_build_cheapest_<category>_effect`'s series of `si_try_economy_candidate_effect` calls (i.e. exactly where the old, now-removed `switch` used to call `add_next_building_tier_effect`), add a duration-stamping switch instead. For `si_build_cheapest_economy_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 730 } }  # outposts
		1 = { set_variable = { name = si_pending_grant_days value = 1095 } } # logging_camps
		2 = { set_variable = { name = si_pending_grant_days value = 1095 } } # peat_quarries
		3 = { set_variable = { name = si_pending_grant_days value = 730 } }  # plantations
		4 = { set_variable = { name = si_pending_grant_days value = 730 } }  # quarries
		5 = { set_variable = { name = si_pending_grant_days value = 730 } }  # hunting_grounds
		6 = { set_variable = { name = si_pending_grant_days value = 1095 } } # cereal_fields
		7 = { set_variable = { name = si_pending_grant_days value = 1095 } } # hill_farms
		8 = { set_variable = { name = si_pending_grant_days value = 1095 } } # hospices
		9 = { set_variable = { name = si_pending_grant_days value = 1095 } } # orchards
		10 = { set_variable = { name = si_pending_grant_days value = 1825 } } # farm_estates
		11 = { set_variable = { name = si_pending_grant_days value = 1095 } } # paddy_fields
		12 = { set_variable = { name = si_pending_grant_days value = 730 } }  # common_tradeport
		13 = { set_variable = { name = si_pending_grant_days value = 1825 } } # caravanserai
		14 = { set_variable = { name = si_pending_grant_days value = 1825 } } # watermills
		15 = { set_variable = { name = si_pending_grant_days value = 1825 } } # windmills
	}
```

For `si_build_cheapest_development_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 1095 } } # qanats
		1 = { set_variable = { name = si_pending_grant_days value = 1095 } } # murex_farm
		2 = { set_variable = { name = si_pending_grant_days value = 1095 } } # waterworks
	}
```

For `si_build_cheapest_levy_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 730 } }  # pastures
		1 = { set_variable = { name = si_pending_grant_days value = 1095 } } # barracks
	}
```

Edit `common/scripted_effects/si_building_catalog_effects.txt` (Task 7 Step 2) the same way, for `si_build_cheapest_maa_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 730 } }  # military_camps
		1 = { set_variable = { name = si_pending_grant_days value = 730 } }  # horse_pastures
		2 = { set_variable = { name = si_pending_grant_days value = 730 } }  # hillside_grazing
		3 = { set_variable = { name = si_pending_grant_days value = 730 } }  # warrior_lodges
		4 = { set_variable = { name = si_pending_grant_days value = 1095 } } # camel_farms
		5 = { set_variable = { name = si_pending_grant_days value = 1825 } } # elephant_pens
		6 = { set_variable = { name = si_pending_grant_days value = 730 } }  # stables
		7 = { set_variable = { name = si_pending_grant_days value = 730 } }  # smiths
		8 = { set_variable = { name = si_pending_grant_days value = 1825 } } # regimental_grounds
		9 = { set_variable = { name = si_pending_grant_days value = 1095 } } # wind_furnace
		10 = { set_variable = { name = si_pending_grant_days value = 1825 } } # workshops
	}
```

`si_build_cheapest_fortification_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 1825 } } # hill_forts
		1 = { set_variable = { name = si_pending_grant_days value = 1095 } } # ramparts
		2 = { set_variable = { name = si_pending_grant_days value = 1095 } } # curtain_walls
		3 = { set_variable = { name = si_pending_grant_days value = 1095 } } # watchtowers
	}
```

`si_build_cheapest_specials_effect`:

```
	switch = {
		trigger = var:si_candidate_building
		0 = { set_variable = { name = si_pending_grant_days value = 1095 } } # monastic_schools
		1 = { set_variable = { name = si_pending_grant_days value = 1095 } } # guild_halls
		2 = { set_variable = { name = si_pending_grant_days value = 1095 } } # scriptorium
		3 = { set_variable = { name = si_pending_grant_days value = 1095 } } # megalith
	}
```

- [ ] **Step 2: Write the six deferred-apply effects (the actual `add_next_building_tier_effect` calls)**

Append to `common/scripted_effects/si_building_catalog_effects.txt`. These reuse the exact same index-to-building mapping as their `si_build_cheapest_*` siblings, but only run once the delay has elapsed (Step 4 calls these, never Task 6-8):

```
si_apply_pending_economy_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = outposts } }
		1 = { add_next_building_tier_effect = { BUILDING = logging_camps } }
		2 = { add_next_building_tier_effect = { BUILDING = peat_quarries } }
		3 = { add_next_building_tier_effect = { BUILDING = plantations } }
		4 = { add_next_building_tier_effect = { BUILDING = quarries } }
		5 = { add_next_building_tier_effect = { BUILDING = hunting_grounds } }
		6 = { add_next_building_tier_effect = { BUILDING = cereal_fields } }
		7 = { add_next_building_tier_effect = { BUILDING = hill_farms } }
		8 = { add_next_building_tier_effect = { BUILDING = hospices } }
		9 = { add_next_building_tier_effect = { BUILDING = orchards } }
		10 = { add_next_building_tier_effect = { BUILDING = farm_estates } }
		11 = { add_next_building_tier_effect = { BUILDING = paddy_fields } }
		12 = { add_next_building_tier_effect = { BUILDING = common_tradeport } }
		13 = { add_next_building_tier_effect = { BUILDING = caravanserai } }
		14 = { add_next_building_tier_effect = { BUILDING = watermills } }
		15 = { add_next_building_tier_effect = { BUILDING = windmills } }
	}
}

si_apply_pending_development_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = qanats } }
		1 = { add_next_building_tier_effect = { BUILDING = murex_farm } }
		2 = { add_next_building_tier_effect = { BUILDING = waterworks } }
	}
}

si_apply_pending_levy_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = pastures } }
		1 = { add_next_building_tier_effect = { BUILDING = barracks } }
	}
}

si_apply_pending_maa_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = military_camps } }
		1 = { add_next_building_tier_effect = { BUILDING = horse_pastures } }
		2 = { add_next_building_tier_effect = { BUILDING = hillside_grazing } }
		3 = { add_next_building_tier_effect = { BUILDING = warrior_lodges } }
		4 = { add_next_building_tier_effect = { BUILDING = camel_farms } }
		5 = { add_next_building_tier_effect = { BUILDING = elephant_pens } }
		6 = { add_next_building_tier_effect = { BUILDING = stables } }
		7 = { add_next_building_tier_effect = { BUILDING = smiths } }
		8 = { add_next_building_tier_effect = { BUILDING = regimental_grounds } }
		9 = { add_next_building_tier_effect = { BUILDING = wind_furnace } }
		10 = { add_next_building_tier_effect = { BUILDING = workshops } }
	}
}

si_apply_pending_fortification_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = hill_forts } }
		1 = { add_next_building_tier_effect = { BUILDING = ramparts } }
		2 = { add_next_building_tier_effect = { BUILDING = curtain_walls } }
		3 = { add_next_building_tier_effect = { BUILDING = watchtowers } }
	}
}

si_apply_pending_specials_effect = {
	switch = {
		trigger = var:si_candidate_building
		0 = { add_next_building_tier_effect = { BUILDING = monastic_schools } }
		1 = { add_next_building_tier_effect = { BUILDING = guild_halls } }
		2 = { add_next_building_tier_effect = { BUILDING = scriptorium } }
		3 = { add_next_building_tier_effect = { BUILDING = megalith } }
	}
}
```

`add_next_building_tier_effect` is the real vanilla scripted_effect from `common/scripted_effects/00_building_effects.txt` (confirmed by direct read during planning) — it walks the chain (`if NOT has_building_or_higher X_01 → add X_01, else_if has X_01 → add X_02, ...`) so this task doesn't need to duplicate tier-selection logic, only trigger it at the right time.

- [ ] **Step 3: Write the scheduler and hook it into the payment flow**

`common/scripted_effects/si_construction_effects.txt`:

```
si_schedule_delayed_grant_effect = { # $CATEGORY_NUM$ — province scope. Assumes si_candidate_building and
	# si_pending_grant_days are already set (Task 6-7's si_build_cheapest_<category>_effect sets both,
	# immediately before this is called by si_pay_for_upgrade_effect in the same scope).
	set_variable = { name = si_pending_grant_pending value = 1 }
	set_variable = {
		name = si_pending_grant_category
		value = $CATEGORY_NUM$
		days = var:si_pending_grant_days
	}
}
```

Edit `common/scripted_effects/si_selection_effects.txt` (Task 8 Step 3), `si_pay_for_upgrade_effect` already calls `si_schedule_delayed_grant_effect` — no change needed there; this step just confirms the contract both sides rely on (candidate index + duration already set before scheduling).

**Verify `set_variable`'s `days` field accepts a variable reference** (`days = var:si_pending_grant_days`) rather than only a literal, since the duration varies per building:

```bash
grep -rn "days = var:" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common" 2>/dev/null | head -5
```

If no results turn up, fall back to passing the resolved day-count directly as a macro parameter instead of via a variable: change `si_schedule_delayed_grant_effect` to take a `$DAYS$` param (`si_schedule_delayed_grant_effect = { CATEGORY_NUM = $CATEGORY_NUM$ DAYS = $DAYS$ }`) and have Task 8's `si_pay_for_upgrade_effect` pass `DAYS = var:si_pending_grant_days` as plain macro text — this still works because macro substitution is textual, so `days = $DAYS$` expands to `days = var:si_pending_grant_days` either way; the two approaches are equivalent unless the grep specifically shows `days` rejects `var:` references entirely, in which case the day-count must be resolved to a literal number before scheduling (i.e. inline each of the ~40 numbers directly into `si_pay_for_upgrade_effect`'s call site instead of via a variable — more verbose, but removes the risk).

- [ ] **Step 4: Write the apply-and-guard dispatcher and the monthly sweep**

Append to `common/scripted_effects/si_construction_effects.txt`:

```
si_apply_pending_grant_effect = { # Province scope. Called once si_pending_grant_category's timer has elapsed.
	if = {
		limit = { has_ongoing_construction = no }
		if = { limit = { var:si_pending_grant_category = 1 } si_apply_pending_economy_effect = yes }
		else_if = { limit = { var:si_pending_grant_category = 2 } si_apply_pending_development_effect = yes }
		else_if = { limit = { var:si_pending_grant_category = 3 } si_apply_pending_levy_effect = yes }
		else_if = { limit = { var:si_pending_grant_category = 4 } si_apply_pending_maa_effect = yes }
		else_if = { limit = { var:si_pending_grant_category = 5 } si_apply_pending_fortification_effect = yes }
		else_if = { limit = { var:si_pending_grant_category = 6 } si_apply_pending_specials_effect = yes }
	}
	# If has_ongoing_construction was yes (the player, or a vassal, manually queued something in this
	# slot in the meantime), the pending grant is dropped silently per spec Section 8's accepted
	# trade-off: the gold was already spent; the next monthly pass will find a new candidate if funds remain.
	remove_variable = si_pending_grant_category
	remove_variable = si_pending_grant_pending
	remove_variable = si_pending_grant_days
	remove_variable = si_candidate_building
}

si_monthly_construction_sweep_effect = { # Character scope. Applies any of this character's tracked
	# provinces whose delayed grant has come due. si_pending_grant_category self-expires after its
	# scheduled `days` (CK3's standard set_variable duration behavior); si_pending_grant_pending does
	# NOT expire, so "pending is set but category is gone" reliably means "the timer just elapsed."
	every_in_list = {
		list = si_candidate_provinces
		limit = {
			has_variable = si_pending_grant_pending
			NOT = { has_variable = si_pending_grant_category }
		}
		si_apply_pending_grant_effect = yes
	}
}
```

- [ ] **Step 5: Hook the sweep into the monthly on_action**

Edit `common/on_action/si_on_actions.txt`, add:

```
on_yearly_playable_pulse = { } # placeholder removed — real hook is on_action below

si_monthly_pulse = {
	events = {}
	effect = {
		si_monthly_construction_sweep_effect = yes
	}
}
```

Verify the correct vanilla on_action name for "fires monthly per playable character" before finalizing (this plan assumes one exists analogous to the confirmed `on_yearly_playable_pulse` referenced in `00_ai_budget_effects.txt` research, but the monthly equivalent's exact name wasn't independently confirmed):

```bash
grep -rln "on_monthly_playable_pulse\|on_playable_pulse" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/on_action" 2>/dev/null
```

If `on_monthly_playable_pulse` exists, hook `si_monthly_construction_sweep_effect` (and, once written, Task 10/11's passes) onto it directly rather than defining a new custom `si_monthly_pulse` on_action — simpler and avoids needing to register a new on_action at all. Use whichever the grep confirms.

- [ ] **Step 6: Verify and commit**

```bash
grep -i "si_construction\|si_pending_grant\|si_monthly" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output.

```bash
git add common/scripted_effects/si_construction_effects.txt common/scripted_effects/si_building_catalog_effects.txt common/scripted_effects/si_selection_effects.txt common/on_action/si_on_actions.txt
git commit -m "Add delayed-grant construction mechanic with manual-build guard"
```

---

### Task 10: Monthly on_action — domain pass

**Files:**
- Modify: `common/scripted_effects/si_pass_effects.txt` (new file)
- Modify: `common/on_action/si_on_actions.txt`

**Interfaces:**
- Consumes: `si_spend_on_category_effect` (Task 8), `si_monthly_construction_sweep_effect` (Task 9), `si_domain_enabled`/`si_domain_reserve`/`si_priority_1..6` (Task 2/3/5).
- Produces: `si_domain_pass_effect` (character scope) — populates the `si_candidate_provinces` list-builder target with the player's directly-held provinces, then spends through all 6 categories in the player's ranked order.

- [ ] **Step 1: Write the domain province list population**

`common/scripted_effects/si_pass_effects.txt`:

```
si_populate_domain_provinces_effect = { # Character scope
	clear_saved_scope = si_candidate_provinces # no-op if not yet defined; ensures a clean list each run
	every_held_title = {
		limit = { tier = tier_county }
		every_county_province = {
			save_scope_value_as = { name = si_candidate_provinces value = this } # see note below
		}
	}
}
```

**Verify the exact mechanism for building an ad-hoc scope list before finalizing this step.** CK3 script has two different "list" concepts: (a) `list = { ... }` blocks used by decision/interaction targeting, and (b) the `save_scope_as`/`variable_list` family for arbitrary runtime-built lists referenced later via `every_in_list = { list = NAME }`. This plan's Task 8 already assumes `every_in_list = { list = si_candidate_provinces }` works as a way to iterate a runtime-populated province list — confirm the actual population mechanism (likely `add_to_variable_list = { name = si_candidate_provinces target = this }` at character scope from within the loop, mirroring vanilla's `variable_list` idiom) before finalizing:

```bash
grep -rn "add_to_variable_list" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | head -5
grep -rn "every_in_list" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | head -5
```

Use the confirmed real syntax (most likely `add_to_variable_list = { name = si_candidate_provinces target = this }` called on the province scope, paired with `every_in_list = { variable = si_candidate_provinces }` or `list = si_candidate_provinces` depending on which the grep confirms) in place of the illustrative `save_scope_value_as` call above, and correspondingly fix Task 8's `every_in_list`/`ordered_in_list` calls to match the same confirmed keyword (`list =` vs `variable =`). Also clear the list at the start of each pass (`remove_variable_list = si_candidate_provinces` on the character, called before repopulating) so stale entries from a previous month don't linger.

- [ ] **Step 2: Write the domain pass**

Append to `common/scripted_effects/si_pass_effects.txt`:

```
si_domain_pass_effect = { # Character scope
	if = {
		limit = { var:si_domain_enabled = 1 }
		si_populate_domain_provinces_effect = yes
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_1 RESERVE_VAR = si_domain_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_2 RESERVE_VAR = si_domain_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_3 RESERVE_VAR = si_domain_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_4 RESERVE_VAR = si_domain_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_5 RESERVE_VAR = si_domain_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_6 RESERVE_VAR = si_domain_reserve }
	}
}
```

**Verify parameter macros accept `var:X` as a value** (i.e. `CATEGORY_NUM = var:si_priority_1` passed into a `$CATEGORY_NUM$`-parameterized effect) before finalizing — this is standard Jomini usage (macro params are plain text substitution, and `var:x` is valid wherever a value is expected), but confirm no quoting/escaping issue exists by checking one real example:

```bash
grep -rn "= var:" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/scripted_effects" | grep -v "limit\|trigger" | head -5
```

- [ ] **Step 3: Hook into the monthly pulse alongside Task 9's sweep**

Edit `common/on_action/si_on_actions.txt`, extend whichever monthly hook Task 9 Step 5 confirmed:

```
	effect = {
		si_domain_pass_effect = yes
		si_monthly_construction_sweep_effect = yes
	}
```

(Sweep runs after the pass so a grant scheduled in a *previous* month that's now due gets applied in the same tick a new one might be scheduled — order between the two doesn't functionally matter here since they touch different variables, but keeping the sweep second means the freshest possible eligibility re-check happens last.)

- [ ] **Step 4: Verify end-to-end**

```bash
grep -i "si_pass\|si_populate_domain\|si_domain_pass" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Then ask the user to: enable domain upgrades with a low reserve floor (e.g. 0) on a character with plenty of gold and at least one holding with an obviously affordable upgrade available, fast-forward a few months in-game, and confirm (a) gold decreases roughly matching a building's cost, (b) the building actually appears after a delay (not instantly), (c) the reserve floor is respected (stop spending once low on gold).

- [ ] **Step 5: Commit**

```bash
git add common/scripted_effects/si_pass_effects.txt common/on_action/si_on_actions.txt
git commit -m "Add monthly domain auto-upgrade pass"
```

---

### Task 11: Monthly on_action — vassal pass

**Files:**
- Modify: `common/scripted_effects/si_pass_effects.txt`
- Modify: `common/on_action/si_on_actions.txt`

**Interfaces:**
- Consumes: `si_spend_on_category_effect` (Task 8), `si_vassal_enabled`/`si_vassal_reserve` (Task 2/4), `si_domain_pass_effect` (Task 10, runs first).
- Produces: `si_vassal_pass_effect` (character scope) — cheap-gates on remaining gold above the vassal reserve, then (if it passes) populates `si_candidate_provinces` with every province held by every vassal (direct and indirect) and runs the same 6-category spend sequence against the vassal reserve floor.

- [ ] **Step 1: Write the vassal province list population**

Append to `common/scripted_effects/si_pass_effects.txt`:

```
si_populate_vassal_provinces_effect = { # Character scope
	every_vassal = {
		every_held_title = {
			limit = { tier = tier_county }
			every_county_province = {
				add_to_variable_list = { name = si_candidate_provinces target = this } # keyword confirmed in Task 10 Step 1
			}
		}
	}
}
```

- [ ] **Step 2: Write the cheap gate + vassal pass**

Append to `common/scripted_effects/si_pass_effects.txt`:

```
si_vassal_pass_effect = { # Character scope. Runs after si_domain_pass_effect.
	if = {
		limit = {
			var:si_vassal_enabled = 1
			trigger_if = { limit = { has_treasury = yes } treasury > var:si_vassal_reserve }
			trigger_else = { gold > var:si_vassal_reserve }
		}
		remove_variable_list = si_candidate_provinces
		si_populate_vassal_provinces_effect = yes
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_1 RESERVE_VAR = si_vassal_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_2 RESERVE_VAR = si_vassal_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_3 RESERVE_VAR = si_vassal_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_4 RESERVE_VAR = si_vassal_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_5 RESERVE_VAR = si_vassal_reserve }
		si_spend_on_category_effect = { CATEGORY_NUM = var:si_priority_6 RESERVE_VAR = si_vassal_reserve }
	}
}
```

The cheap gate (spec Section 10) is the `trigger_if/trigger_else` block checking current gold against the reserve *before* `si_populate_vassal_provinces_effect` runs — the expensive recursive vassal/province iteration is skipped entirely whenever there's nothing to spend anyway.

- [ ] **Step 3: Confirm `every_vassal` is recursive (direct + indirect) before finalizing**

This was inferred during brainstorming from naming-convention consistency (no separate `every_direct_vassal` list-builder was found to exist) rather than directly confirmed via an authoritative source. Confirm before relying on it:

```bash
grep -B3 -A3 "every_vassal = {" "C:/Program Files (x86)/Steam/steamapps/common/Crusader Kings III/game/common/activities/guest_invite_rules/activity_invite_rules.txt" | head -20
```

Look for comments or contextual clues (e.g. a comment mentioning "including sub-vassals" or usage alongside `top_liege`/realm-wide framing) confirming it's recursive. If ambiguous, the safer alternative — costing an extra nesting level but removing any doubt — is nesting a nested nested `every_vassal` nested a fixed number of levels deep (e.g. 3 levels, covering direct + 2 levels of sub-vassals, which covers the overwhelming majority of realistic vassal chains) instead of relying on single-call recursion:

```
every_vassal = {
	every_held_title = { limit = { tier = tier_county } every_county_province = { add_to_variable_list = { name = si_candidate_provinces target = this } } }
	every_vassal = {
		every_held_title = { limit = { tier = tier_county } every_county_province = { add_to_variable_list = { name = si_candidate_provinces target = this } } }
		every_vassal = {
			every_held_title = { limit = { tier = tier_county } every_county_province = { add_to_variable_list = { name = si_candidate_provinces target = this } } }
		}
	}
}
```

Prefer the single recursive call if confirmed; use the nested fallback only if the grep is inconclusive.

- [ ] **Step 4: Hook into the monthly on_action after the domain pass**

Edit `common/on_action/si_on_actions.txt`:

```
	effect = {
		si_domain_pass_effect = yes
		si_vassal_pass_effect = yes
		si_monthly_construction_sweep_effect = yes
	}
```

- [ ] **Step 5: Verify end-to-end**

```bash
grep -i "si_vassal_pass\|si_populate_vassal" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output. Ask the user to enable vassal subsidies with a low reserve, play a character with several vassals, fast-forward a few months, and confirm gold is spent on vassal holdings (visible by checking a vassal's holding view for a building that appeared without the player manually building it) while respecting the reserve floor and running after domain spending per spec Section 10.

- [ ] **Step 6: Commit**

```bash
git add common/scripted_effects/si_pass_effects.txt common/on_action/si_on_actions.txt
git commit -m "Add monthly vassal subsidy pass"
```

---

### Task 12: Localization pass for remaining tooltips

**Files:**
- Modify: `localization/english/si_l_english.yml`

**Interfaces:**
- Consumes: nothing new — this task only adds missing player-facing strings for things Tasks 6-11 introduced that display in tooltips (e.g. the decision's own `is_shown`/effect tooltips once it does real work, and any `custom_tooltip` calls used for clarity).

- [ ] **Step 1: Add a custom tooltip explaining what the decision effect does, since it no longer just opens a menu silently**

Edit `common/decisions/si_settings_decision.txt`, in `effect`, ensure the opening line reads clearly for the tooltip preview players see before confirming the decision:

```
	effect = {
		custom_tooltip = si_settings_decision_effect_tt
		si_ensure_defaults_effect = yes
		trigger_event = si_menu.1
	}
```

- [ ] **Step 2: Add the remaining localization keys**

Append to `localization/english/si_l_english.yml`:

```yaml
 si_settings_decision_effect_tt:0 "Open the Steward's Initiative settings menu."
```

(Remove the now-unused `si_settings_decision_placeholder_tt` key from Task 1 if nothing else references it.)

- [ ] **Step 3: Verify**

```bash
grep -i "si_l_english\|localization" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log"
```

Expected: no output, and no `[MISSING LOCALIZATION]`-style text visible anywhere in the settings menu flow when the user clicks through it once more end-to-end.

- [ ] **Step 4: Commit**

```bash
git add common/decisions/si_settings_decision.txt localization/english/si_l_english.yml
git commit -m "Finish localization pass for settings decision tooltip"
```

---

### Task 13: End-to-end playtest checklist

**Files:** none (verification-only task).

**Interfaces:** Consumes the entire mod. Produces: a confirmation the mod satisfies spec Sections 7, 10, and 11 (monthly pipeline, performance mitigations, edge cases) under real play, not just per-task spot checks.

- [ ] **Step 1: Clean-log baseline**

Ask the user to fully close CK3, delete/rename `C:/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log`, relaunch, and load a save.

```bash
grep -i "^\[" "/c/Users/corwi/Documents/Paradox Interactive/Crusader Kings III/logs/error.log" | grep -i "si_"
```

Expected: no output at all — a fully clean load with the mod active.

- [ ] **Step 2: Domain-only playtest**

Enable domain upgrades only (vassal subsidies off), set a mid-range reserve (e.g. 500), rank categories in a distinctive, memorable order (e.g. Fortification first). Fast-forward 12 in-game months. Confirm: upgrades appear in Fortification-eligible holdings before other categories, gold never drops below 500, and the settings menu re-opened via the decision shows the same settings persisted correctly.

- [ ] **Step 3: Vassal-subsidy playtest**

Additionally enable vassal subsidies with a reserve lower than the domain reserve (e.g. 200). Fast-forward another 12 months on a character with at least 3 vassals of varying wealth. Confirm domain spending still happens first each month (check via save-editor or simply by watching gold deplete toward the domain floor before any vassal holding changes), and vassal holdings do receive upgrades once domain spending settles at its floor.

- [ ] **Step 4: Manual-construction conflict check (spec Section 8's core safety guarantee)**

With domain upgrades enabled, manually queue a build in a holding via the normal UI in the same month the mod is also likely to target that holding (pick a cheap, obviously-eligible building). Fast-forward past both the mod's decision point and the manual construction's completion. Confirm no duplicate/conflicting building state results — the mod should skip that slot silently per Task 9 Step 4's guard.

- [ ] **Step 5: Succession reset check**

Let the playtest character die (or use console `kill` in debug mode) and confirm the heir's settings start fresh (mod effectively off) rather than inheriting the deceased's configuration, per spec Section 11.

- [ ] **Step 6: Record results**

If all five checks pass, the mod is feature-complete per the spec. If any fail, use `superpowers:systematic-debugging` to isolate which task's mechanism is misbehaving before making further changes — do not patch symptoms without first identifying which specific effect/trigger produced the wrong outcome.

- [ ] **Step 7: Final commit**

```bash
git add -A
git commit -m "Complete end-to-end playtest verification"
git push origin main
```

(This is the first `git push` in the plan — confirm with the user before running it, per the standing instruction to check before pushing to the remote.)
