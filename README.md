# PREFLIGHT

**Star Citizen joystick instance-collision detector and `actionmaps.xml` repinner.**

Part of the ED HUMINT family of tools.

## The problem it solves

You add a hub, move a port, plug in one more device — and suddenly your right stick is "Input 5", pitch fires off a throttle module, and `i_DumpDeviceInformation` doesn't even list the stick. Nothing is broken. Star Citizen matched your devices to binding slots by product name, two of your devices share a name, the USB enumeration order changed, and the second one landed in the slot your stick was pinned to.

PREFLIGHT reads the live DirectInput enumeration, predicts exactly how Star Citizen will assign instance numbers against your `actionmaps.xml`, tells you what will collide before you launch, and — if you ask it to — rewrites the pin table so the assignment is correct regardless of enumeration order.

It does not touch what your axes and buttons do. It only fixes which physical device each `jsN_` refers to.

## Status

Pre-alpha. Spec in [SPEC.md](SPEC.md). The resolution model was derived from one rig; see *Known unknowns* there before trusting `--apply` on yours. `scan` and `--dry-run` are read-only and safe anywhere.

## Usage (planned)

```
preflight scan
preflight fix --dry-run
preflight fix --apply --layout preflight
```

## Diagnosing by hand (what this tool automates)

If you want to see it yourself before trusting a binary:

1. In-game console (`~`): `i_DumpDeviceInformation`. Note the `(N)` after each device; the keybinding UI shows `N+1` as "Input N+1". A missing `(N)` in the sequence is a device the dump omitted.
2. `LIVE\Game.log`: grep `Connected joystick`. That is the enumeration order SC processed.
3. `LIVE\user\client\0\Profiles\default\actionmaps.xml`: the `<options type="joystick" instance=N Product=...>` lines are the pins.
4. Walk the enumeration order against the pins: name match takes the pin; if the pin is occupied, the device falls to the lowest free instance.

Do **not** diagnose with the keybinding UI's double-click-and-deflect trick. It commits the rebind faster than you can press Esc.

## License

MIT.
