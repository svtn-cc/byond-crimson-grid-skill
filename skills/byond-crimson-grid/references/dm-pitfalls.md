# Engine and codebase traps

Everything below has already broken work in this project.

## Compile order

- `modular_corvax` is included in `tgstation.dme` before `modular_darkpack`. For a proc defined in both, the darkpack version wins. Darkpack behaviour cannot be overridden from corvax that way.
- `subtypesof()` returns types in declaration order. When ranks depend on that order (a Discipline's powers), declare the subtypes in one file with a leading `_`, with no intermediate abstract subtypes, and pin the order with a unit test.

## Helpers and defines

- Do not trust memory of tgstation: `CONSCIOUS` and `random_unique_name()` do not exist here (`STABLE` and `generate_random_name_species_based()` do). `rg` every helper before using it.
- Darkpack proc signatures differ from tg (`get_visible_name(…, mob/examiner)`, `copy_dna` with a blood-type flag). Read the definition.

## Visibility and images

- Invisibility is an atom's `invisibility` against a mob's `see_invisible`. Raising `see_invisible` for "our side" is wrong: they would see every invisible thing at that level.
- To show something to **chosen clients**, put an `image` in `client.images`. An image attached to an invisible atom is visible to that client.
- An image on an atom **inherits the atom's pixel offset** but **does not inherit its direction**. A copy of an atom's `appearance` that has an offset (`pixel_x = -32` on cars) is shifted twice — zero the offset; set the direction yourself and update it on `COMSIG_ATOM_DIR_CHANGE` (the signal fires before `dir` changes; the new value is in the argument).
- HUD images (auras, health bars) are drawn for every HUD user regardless of the atom's invisibility, and the image's own `invisibility` does not hide them. To hide them, take the atom out of the HUD (`remove_atom_from_hud`) and put it back later.
- To replace an atom's look for some players, use `add_alt_appearance` with `override = TRUE` (`one_person`, `one_person/reversed`, `everyone`). It is a snapshot: refresh it when clothing, posture or facing changes.
- An invisible but closed door stays opaque and dense.
- Light from an invisible atom stays visible.

## Character data

- Chat names and recognition go through the guestbook by `real_name`. Changing DNA changes `real_name`, and with it who the character is seen as.
- The TTS voice is stored in DNA (`tts_seed_dna`) and is copied along with it.
- The DNA gender block may not match the mob's `gender`. Carry gender over explicitly when copying an appearance.
- An NPC's Willpower defaults to its Courage (which starts at 1); its other stats are at starting values. Comparisons against NPC stats almost always "pass".

## Signals and waiting

- A signal handler must not sleep: `SSroll.storyteller_roll_datum`, `show_radial_menu`, `tgui_alert`, `do_after` — only through `INVOKE_ASYNC`.
- One click can raise several watched signals (an item interaction and `COMSIG_DO_AFTER_BEGAN`). If a roll hangs on them, guard against firing twice in the same tick.
- Calling `qdel` on a datum from a handler invoked by that datum's own deletion gives a CRASH "destroy proc was called multiple times". Keep "forget it" and "delete it" as separate procs.
- Status effects: when `on_apply()` returns FALSE the engine calls `on_remove()` straight away. Set the fields `on_remove()` uses to know what to undo at the very end of `on_apply()`.

## Rolls

- A roll's visibility is its type's `roll_output_type`: `ROLL_PUBLIC` shows the roller's name to everyone around. For hidden actions use `ROLL_FLAG_ROLLER` (the roller only) or `NONE` (nobody). Define a `/datum/storyteller_roll/<name>` subtype.
- A roll whose result must not reveal that an attempt happened is silent, and writes nothing on failure.

## Interface and admin tools

- The number of dots a Discipline offers in the menu comes from its number of powers; there are separate hard caps in `admin_preference_editor.dm` and `discipline_pref_middleware.dm` — check both when adding ranks.
- The ban on powers above the fifth dot for Generations weaker than the seventh is enforced in the character menu, not in the power.

## Building and resources

- A new `.dmi` or sound while a server is running: `cannot find file`. The server holds the `.rsc`; stop it and rebuild.
- `dreamchecker` may not be installed locally — only CI verifies `SIGNAL_HANDLER` correctness and types.
