# In-game checks through the BYOND MCP

In-game checks use **[byond-mcp](https://github.com/svtn-cc/byond-mcp)**: an MCP server that compiles a separate build with a test harness, runs the world in Dream Daemon, connects up to two real Dream Seeker clients and lets you call code inside the world. Its README is the reference for every tool and argument; this file covers how to use it in this project.

## Setup

If the `byond` tools are not available, the server is not installed or not configured. It needs BYOND 516, Windows (for clients and screenshots) and a Rust toolchain to build:

```
git clone https://github.com/svtn-cc/byond-mcp.git
cd byond-mcp
cargo build --release
```

Then point the MCP client at the binary in a project-level config (`.omp/mcp.json`, `.claude/mcp.json` or `.cursor/mcp.json`):

```json
{
	"mcpServers": {
		"byond": {
			"command": "C:/path/to/byond-mcp/target/release/byond-mcp.exe",
			"args": ["--project", "C:/path/to/crimson-grid"]
		}
	}
}
```

Installing software and editing the user's MCP config is the user's call: tell them what is missing instead of doing it unasked.

Things to know about the server:

- It builds `tgstation.mcp.dmb` from a copy of the `.dme` plus a harness; the regular `tgstation.dmb` and the sources are never modified. After changing DM code, call `build` again — the normal build does not update the `.mcp.dmb`.
- The harness runs any proc for whoever holds the session token. A `.mcp.dmb` build must never be deployed to a public server.
- One machine has two BYOND identities, `guest` and `account`, so at most two real client windows. A `guest` join fails while the user's own BYOND pager is open.
- `enter_game` creates bodies directly instead of going through the lobby. With `own_character: true` the game treats them as the players' own characters and mid-round saves **really modify the player's save file** — leave it off unless the user asks for it.
- `ui_act` drives the server-side handler: it tests game logic, not a window's layout or JavaScript.
- `logs` takes `tail`, `grep` and `since_last`; lists in results are cut at 300 entries and expanded two levels deep unless `depth` is raised.

## The loop

1. `build` — `{"map": "runtimetown", "defines": ["SKIP_LAVALAND", "SKIP_SPACE_LEVELS"], "wait_secs": 300}`. Always the small map. The result is `tgstation.mcp.dmb`; the regular build is not touched.
2. `server_start` — `{"wait_secs": 300}`.
3. `enter_game` — `{"players": 2, "wait_secs": 600}`: connects two clients, starts the round, puts their bodies side by side.
4. Character setup and the checks themselves — `eval_lua`, `call_proc`, `ui_list` / `ui_act`.
5. `screenshot` — what each client actually sees.
6. `logs` with `{"file": "runtime.log"}` — any new runtime errors.
7. `server_stop`. After a code change, start again from step 1: the world does not pick up changes live.

One ckey is the caster, the other the witness. Nearly everything in this game is about what the second player does and does not see, so check **through the second client's eyes**, not only through variables.

## Lua (`eval_lua`)

Lua state persists between calls — define helpers once as globals.

```lua
P = function(s) return dm.global_procs._text2path(s) end
A = dm.global_procs.get_mob_by_ckey('ckey1')
B = dm.global_procs.get_mob_by_ckey('ckey2')
A:make_kindred()
A:give_st_power(P('/datum/discipline/obfuscate'), 5)
A:st_set_stat(P('/datum/st_stat/ability/stealth'), 4)
```

Things to know:

- A BYOND list is not a Lua table: `ipairs`/`pairs` throw. Length is `#list`, an element is `list[i]`, associative access is `list[key]`.
- Global procs are `dm.global_procs.<name>`; built-ins take a leading underscore: `_locate(x, y, z)`, `_get_step(atom, dir)`, `_text2path`. Global variables are `dm.global_vars.GLOB.<name>`, subsystems `dm.global_vars.SSname`. There is no `world` variable.
- Create a datum with `dm.new('/path', args…)`. A datum created by the `spawn` tool with a mob argument may be garbage-collected at once — create it from Lua and keep it in a variable.
- A `nil` inside a returned table truncates it: return `x ~= nil`, not the object.
- A proc that sleeps (an animated roll, a menu) returns from Lua before its result exists. Read the state in a **separate, following call**.
- Calling a player verb through its proc (`A:whisper('…')`) is more reliable than `client_command`.

## Interfaces

A proc that opens a menu blocks until it is answered. The sequence:

1. `call_proc` with `wait_secs: 2` → returns `{"status": "running", "job": N}`.
2. `ui_list` for the ckey — radial menus (`choices`) and tgui windows (`interface`, `static_data`).
3. `ui_act`:
   - radial menu — `{"ckey": …, "choice": "<label>"}`;
   - `tgui_alert` — `{"action": "choose", "params": {"choice": "<button>"}}`;
   - `tgui_input_checkboxes` — `{"action": "submit", "params": {"entry": [["<label>", <1-based index>]]}}`.
4. `job_result` with the job number — wait for it to finish.

A window may not be open yet between steps: on an empty `ui_list`, ask again.

## Screenshots

- `screenshot` with a `ckey` captures that client's main window.
- Separate tgui windows may have no title — take their contents from `ui_list` instead.
- Catch anything brief (a screen flash, a balloon) with a burst of captures right after the action and pick the frame.
- In the report, show the capture of **the one who should see the difference**, and next to it the one who should not.

## What the test map does to results

- Runtime Town has no dark tiles: check anything that needs darkness with the threshold temporarily changed, and restore it.
- Check rare events (a fraction-of-a-percent chance) by temporarily raising the chance variable to 100%, and restore it.
- A character built from the test account's preferences can carry inconsistent data (for example, the DNA gender block not matching the mob's gender). Check appearance copying explicitly.
- NPCs nearby react to teleports and to the supernatural: "Masquerade violations" in chat may come from the harness, not from the code under test — verify against `SSmasquerade.masquerade_breachers`.
- Dice rolls are random: reach an outcome by repeating in a loop or by setting the number of successes directly, and say in the report which you did.

## The report

HTML at `.omp/<topic>_report/report.html`, screenshots beside it. A table of: what was checked, what was expected, what happened, the frame. Separate blocks for:

- how it was tested and what was called around the interface;
- what could **not** be checked, and why;
- bugs found — and whether each was re-checked after the fix;
- what was altered for the test (thresholds, chances) and whether it was restored.

Do not write "works" about anything not observed in game. Do not bend the world to fit the expectation: if a step cannot be done through the game, say so.

## Unit tests

`build.bat dm-test` compiles the test build and runs the world. Locally DreamDaemon shows a "Proceed with trusted mode?" dialog — the user confirms it; do not bypass it. To run one test, temporarily put `TEST_FOCUS(/datum/unit_test/<name>)` on the first line of its file and remove it before handing off. Results land in `data/logs/ci/`.
