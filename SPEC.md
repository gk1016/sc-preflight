# PREFLIGHT — Specification v0.1

Star Citizen joystick instance-collision detector and `actionmaps.xml` repinner.

## Problem

Star Citizen stores joystick bindings against an **instance number** (`js1_`, `js2_`, ...). It maps physical devices to instances at launch using a table in `actionmaps.xml`:

```xml
<options type="joystick" instance="3" Product=" VKBSim Space Gunfighter   {0126231D-0000-0000-0000-504944564944}">
```

Matching is by `Product` string (DirectInput product name + product GUID). Two devices of the same model have **identical** Product strings, so the table cannot tell them apart. Observed resolution behaviour (see `fixtures/observed_assignment_before_fix.txt`):

1. Devices are processed in DirectInput enumeration order (`Connected joystickN` lines in `Game.log`).
2. A device whose Product matches a pin takes that pin's instance.
3. If that instance is already occupied (duplicate-name device got there first), the device **falls through to the lowest free instance** — it does *not* look for a second pin with the same name.
4. Any later device whose pin is now occupied is displaced the same way.

Enumeration order depends on USB topology (hub tier, port). Changing a hub reorders enumeration; a duplicate-name device (two SEMs, two identical sticks, two throttle modules) can then land in a slot pinned for a different device, silently stealing its binds.

The user-visible symptoms are flight axes firing from the wrong device, orphan binds, and `i_DumpDeviceInformation` omitting the displaced device's row.

## What PREFLIGHT does

```
preflight scan   [--actionmaps <path>] [--json]
preflight fix    [--actionmaps <path>] [--dry-run | --apply] [--layout <name>]
```

### `scan`

1. Enumerate DirectInput game controllers **in the order DirectInput reports them** (`IDirectInput8::EnumDevices(DI8DEVCLASS_GAMECTRL, ..., DIEDFL_ATTACHEDONLY)`). Record: `tszProductName`, `guidProduct`, `guidInstance`, enumeration index.
2. Build the SC Product string for each device exactly as SC writes it: `tszProductName` as-is (leading/trailing/internal spaces preserved) + two spaces + `{GUID}` where the GUID is `guidProduct` formatted uppercase. Verify against fixtures: `" VKBSim Space Gunfighter   {0126231D-0000-0000-0000-504944564944}"`. The GUID in SC's format is PID+VID (`0126` `231D`) followed by `-0000-0000-0000-504944564944` ("PIDVID"); this is the standard DirectInput `guidProduct` layout.
3. Locate and parse `actionmaps.xml` (default `<SC install>\LIVE\user\client\0\Profiles\default\actionmaps.xml`; discover install via the RSI launcher registry/log or `--actionmaps`). Extract the `<options type="joystick" instance=N Product=...>` table and all `<rebind input="jsN_...">` references.
4. **Simulate** SC's assignment per the four rules above. Output the predicted instance for every enumerated device.
5. Flag:
   - `COLLISION` — a device fell through because its pin was occupied.
   - `DISPLACED` — a device did not get its pinned instance.
   - `ORPHAN_BINDS` — `jsN_` references where instance N is predicted to hold a device other than the one the pin names, or no device at all.
   - `UNPINNED` — an enumerated device with no matching pin.
6. Print a table: device, enumeration index, pinned instance, predicted instance, Input N (UI number = instance), status. Exit code 0 if clean, 2 if any flag.

### `fix`

Compute a new pin table that is **order-independent** under the resolution rules:

- Group pinned devices by Product string.
- Any group with count > 1 (duplicate names) must occupy a **contiguous block of the lowest instances** not used by devices that enumerate before them and are unique. In practice: sort groups so duplicate-name groups come first and are contiguous; unique-name devices follow. Preserve the user's relative order within a group.
- Devices that always enumerate first and are unique (e.g. vJoy at instance 1) may keep their slot if that leaves the duplicate block contiguous immediately after. Prefer the minimal change that satisfies the invariant.
- Produce an old→new instance map, then rewrite every `<rebind input="jsOLD_...">` to `jsNEW_` **atomically** (temporary tokens so chains like 3→4, 4→5, 5→3 do not cascade).
- Also rewrite the `instance` attributes in the `<options>` table. Leave `<deviceoptions>` untouched (keyed by name, not instance).

Safety:

- `--dry-run` (default) prints the diff and the before/after predicted table and writes nothing.
- `--apply` refuses if `StarCitizen.exe` is running (the game rewrites the file on exit).
- `--apply` writes `actionmaps.xml.bak_<timestamp>` beside the original before touching it.
- Assert: count of `jsN_` references per N before == count per mapped N after; no temp tokens remain; file parses as XML; `<options>` instance numbers unique.
- Re-run the simulation on the result and refuse to write if any flag remains.
- `--layout <name>` additionally writes `Controls/Mappings/layout_<name>_exported.xml` with the same content so `pp_rebindkeys <name>` restores it.

### Out of scope for v0.1

- Editing binds themselves (which axis does what).
- Disambiguating two identical devices by serial. SC cannot do it, so neither can the file.
- HIDHide / vJoy / Gremlin integration.
- GUI. CLI only; a tray watchdog is a later milestone.

## Verification targets

`fixtures/` holds a real case from 2026-11-16:

| file | meaning |
|---|---|
| `actionmaps_before.xml` | pins that collided (SEM 2, R 3, L 4, SEM 5, vJoy 1/6) |
| `game_log_connected.txt` | the enumeration order that broke it |
| `observed_assignment_before_fix.txt` | what SC actually did (ground truth for the simulator) |
| `actionmaps_after.xml` | the hand-fixed file that resolved it (vJoy 1, SEM 2, SEM 3, R 4, L 5, vJoy 6) |

Required tests:

1. Simulator on `before` + `game_log_connected` order reproduces `observed_assignment_before_fix` exactly.
2. Simulator on `after` + same order yields every device on its pin, no flags.
3. `fix --dry-run` on `before` produces a file equivalent to `after` (same pin table; same `jsN_` reference counts per device).
4. Simulator on `after` under **every permutation** of the six-device enumeration order yields no flags (order independence).
5. Rotation safety: a synthetic file with chain 3→4→5→3 rewrites without cascading.

## Stack

- Rust, single static binary, Windows x64 only.
- `windows` crate (dinput8 via `Win32::Devices::HumanInterfaceDevice`), `quick-xml` (preserve formatting — SC's file is hand-readable and users diff it), `clap`, `anyhow`.
- No installer, no service, no network, no telemetry.
- `cargo check` before `cargo build --release`. Tests must pass on the fixtures without hardware (simulator and rewriter are pure functions over parsed input; DirectInput is behind a trait so fixtures can feed it).

## Known unknowns (hardening backlog)

- Rule 3 (lowest-free fall-through) was inferred from one rig and four launches. Confirm on a second rig and by deliberately reordering a hub.
- Whether SC consults `guidInstance` anywhere. Nothing observed suggests it does.
- Whether SC ever rewrites the `<options>` table on its own (it rewrote a `<rebind>` after an in-UI probe; the table appeared untouched).
- Install-path discovery for non-default RSI launcher locations and PTU/EPTU/HOTFIX channels.
