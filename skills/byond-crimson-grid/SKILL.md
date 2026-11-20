---
name: byond-crimson-grid
description: BYOND/DM development in the crimson-grid project (tgstation → DarkPack → Corvax fork, Vampire the Masquerade V20). Use for any edit to .dm/.dme/.dmi files, for writing in-game text and descriptions, and for building and testing through the BYOND MCP.
---

# BYOND: crimson-grid

The project is tgstation → DarkPack13/SecondCity (`modular_darkpack`) → Corvax (`modular_corvax`). Upstream arrives as merge commits, so the first rule of code here is: **our changes live in `modular_corvax` and leave other people's files alone**.

**Language.** Every string a player can see — chat messages, balloon alerts, names, descriptions, button labels, window text — is written in **Russian**. Code, identifiers, code comments, the module's `readme.md` and this skill are in English.

Details live in the references. Read the relevant one before that kind of work:

- `references/writing.md` — how to write any text a player sees. **Required before writing any string for `to_chat`, `desc`, `balloon_alert` or `visible_message`.**
- `references/mcp-testing.md` — building and in-game checks through [byond-mcp](https://github.com/svtn-cc/byond-mcp), the MCP server this project uses for development and testing; includes its setup.
- `references/dm-pitfalls.md` — engine and codebase traps this project has already hit.

## Workflow

1. Find the existing mechanism (`rg`) before writing a new one. Verify every helper, define, signal and proc by search: a "familiar name from tgstation" may not exist here.
2. Mechanics, balance and departures from the rulebook are the project owner's call. Put each fork to them as a question with options and a recommendation; do not pick silently. List whatever you did decide yourself in the report.
3. Write code in `modular_corvax`, and player text by `references/writing.md`.
4. Build the **small** build and check in game through the MCP (`references/mcp-testing.md`). "It compiles" is not "it works": report only behaviour you observed in game.
5. Update the module's `readme.md`.

## Where files go

```
modular_corvax/modules/<module>/
  code/…            all of the module's DM code
  icons/…           the module's .dmi files
  sounds/…
  readme.md         English, filled in from modular_darkpack/module_template.md
code/__DEFINES/~~~corvax/…   defines needed before the rest of the code (traits, signals)
modular_darkpack/modules/unit_tests/<name>.dm   unit tests (included from _darkpack_unit_tests.dm)
```

- Add every new `.dm` to `tgstation.dme`, alphabetically inside the `modular_corvax` block (CI checks that all files are included and sorted). A file that must compile first in its folder gets a leading `_`.
- A local `#define` in a `.dm` file must be closed with `#undef` at the end of that file.
- Declare traits in `code/__DEFINES/~~~corvax/traits/declarations.dm` and register them in `code/_globalvars/traits/_traits.dm` and `admin_tooling.dm`.
- Icons are `.dmi` files in the module's `icons/`. Do not draw icons in code.

## Changing existing code

In order of preference:

1. **New code in `modular_corvax`**: a subtype, a component, a status effect, a signal subscription.
2. **Your own type instead of theirs.** To rewrite a darkpack class's behaviour wholesale, define new types in corvax and repoint the entry point at them (example: `/datum/discipline/obfuscate/New()` swaps `power_type` to `/datum/discipline_power/corvax_obfuscate`). The upstream file stays untouched.
3. **A spot edit in someone else's file** — only when nothing else works (a hard-coded check inside a proc, a missing signal). Mark every such line:

```dm
var/x = 2 // CORVAX EDIT CHANGE - <why>. ORIGINAL: var/x = 1
do_thing() // CORVAX EDIT ADD - <why>
// CORVAX EDIT ADD START - <why>
…
// CORVAX EDIT ADD END
```

Never: comment upstream code out instead of removing it with a marker, reorder or reformat upstream lines, edit an upstream file without a marker.

**A darkpack proc cannot be overridden from a corvax file**: `modular_corvax` is included in the `.dme` before `modular_darkpack`, so on a clash the darkpack definition wins. Use one of the options above.

Every module has a `readme.md`, written in English from the template `modular_darkpack/module_template.md` (copy it into the module root and fill in each section: Module ID, Description, TG Proc/File Changes, Modular Overrides, Defines, Included files not contained in this module, Credits). List every upstream file you touched under the matching section. In Description, also say which upstream revision the code was ported from and what to do on a merge conflict, when that applies.

## Code style

- Tabs for indentation. Absolute type paths. `///` comments above vars and procs; a comment says why, it does not restate the code.
- Express behaviour as variables on the type, not branches on the type: `var/breaks_on_move = TRUE` on a subtype instead of `if(istype(src, …))`. One rule lives in one place.
- Signal handlers start with `SIGNAL_HANDLER` and must not sleep. Rolls, windows and `do_after` go through `INVOKE_ASYNC`.
- After anything that waits (a radial menu, `tgui_alert`, an animated roll), re-check `QDELETED` and the preconditions: the world has moved on.
- An effect on a mob is a `/datum/status_effect` (the engine gives uniqueness and cleanup on mob deletion), not a trait plus hand-rolled `COMSIG_QDELETING` subscriptions.
- References to mobs in long-lived lists are `WEAKREF`s. Whatever you attach (signals, images in `client.images`, traits), detach in the reverse proc.
- Player-facing strings are Russian; inflect names with `declent_ru(CASE)`.
- No checks, retries or abstractions "for later". No stubs and no `TODO`s in code.

## Building

For checks, build **only the small map** — Runtime Town, without the extra levels. The full map loads several times slower and is not needed to test mechanics.

- Through the MCP (preferred): `build` with `{"map": "runtimetown", "defines": ["SKIP_LAVALAND", "SKIP_SPACE_LEVELS"]}` → `tgstation.mcp.dmb`; the regular build is not touched.
- For the user to run by hand — the VS Code task "Build All (low memory mode)", or the vendored bun directly:
  `tools/bootstrap/.cache/bun-*/bun.exe tools/build/build.ts -DSKIP_LAVALAND -DSKIP_SPACE_LEVELS '-DFORCE_MAP="runtimetown"'`
  Through `build.bat` from bash the quoted define gets mangled.
- Pass criterion: `0 errors, 0 warnings`.
- If the build reports `cannot find file` for a `.dmi` or sound you just added and the file is there, a running server is holding `tgstation.rsc`. Stop the server and rebuild; do not work around it.
- Build the full map only when the user asks, or to test the map itself.

## Player text in brief

Full rules and examples are in `references/writing.md`. The core:

1. **A narrator, not a manual.** Text describes what the character senses and sees, not how the mechanic works. No seconds, percentages, stat names, or words like "roll", "cooldown", "effect", "ability".
2. **Do not leak what is hidden.** The victim does not learn the power's name or who used it unless the character could have worked it out.
3. **An RP directive with no mechanic behind it** (the character forgot, believed, was terrified — and the code does not enforce it) goes out on three channels at once: large conspicuous chat text, a `balloon_alert` on screen, and a screen flash. It is phrased in a game master's voice: it states what the character feels and believes, and that tells the player how to play it.
4. Mechanical reference (exact conditions, numbers) goes into the module's `readme.md`, not into the game.
5. All of it in Russian.

## Before handing off

- [ ] Upstream files are untouched, or every edit carries a `CORVAX EDIT` marker and is listed in `readme.md`.
- [ ] New `.dm` files are in `tgstation.dme` in alphabetical order; every `#define` has its `#undef`.
- [ ] Small build: 0 errors, 0 warnings.
- [ ] Behaviour checked in game through the MCP with two clients; no new runtime errors in the log.
- [ ] Every player-facing string is Russian and passes the check in `references/writing.md`.
- [ ] The report separates: what was observed in game, what was not checked and why, what you decided yourself.
