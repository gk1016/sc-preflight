# CLAUDE.md — sc-preflight

Read SPEC.md first. It is the contract.

## Rules
- Read source from disk before proposing changes. Never patch from memory of a file.
- Minimal blast radius. One concern per change. No speculative abstractions, no "cleaner" rewrites.
- `cargo check` before `cargo build --release`. Tests (`cargo test`) must pass on fixtures with no hardware present.
- The simulator and the rewriter are pure functions. DirectInput enumeration sits behind a trait; tests feed it from fixtures.
- Preserve `actionmaps.xml` formatting on rewrite. Users diff this file by eye.
- `fix --apply` must: refuse if StarCitizen.exe is running; write a timestamped backup first; assert reference counts, uniqueness, XML validity; re-simulate and refuse if any flag remains.
- No network, no telemetry, no installer, no service.
- No emojis in code, docs, or CLI output.

## Fixtures are ground truth
`fixtures/observed_assignment_before_fix.txt` is what the game actually did. If the simulator disagrees with it, the simulator is wrong.

## Milestones
1. Parser + simulator passing tests 1, 2, 4 in SPEC.md.
2. Rewriter passing tests 3, 5. `fix --dry-run` end to end on fixtures.
3. Real DirectInput enumeration on Windows. `scan` end to end.
4. `fix --apply` with all safety gates.
