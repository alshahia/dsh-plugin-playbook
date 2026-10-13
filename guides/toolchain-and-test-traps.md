# Toolchain and test traps — launcher order, composition proof, the boot-adopt race

> **Universal copy** — distilled from `deepseek_plugins/MEMORY.md` §1/§3
> (commit `4498abd`, `master`), harvested 2026-10-13 before that repository was
> frozen at `_archive/deepseek_plugins/`. The two traps below were **not** in this
> playbook before that date.

These are traps in the *harness toolchain*, not in plugin code. Each one produced a
false conclusion before it was understood.

## Trap 1 — Launcher flag order decides whether your patch reaches the app

The DSH launcher consumes its own flags **positionally**. Everything after the
first token it does not recognise is handed verbatim to the app subcommand, and
the app knows only `--host`, `--port` and `--trusted-host`.

So this works:

    node --import tsx/esm apps/cli/src/bin.ts --profile web --patch <file> --port 3081

and this fails with `unknown option '--patch'`:

    node --import tsx/esm apps/cli/src/bin.ts --profile web --port 3081 --patch <file>

**Rule:** launcher-owned flags (`--profile`, `--patch`) come **before** any app
flag. Put the port and host last.

## Trap 2 — `--dump-config` is the only cheap proof that a patch composes

Do not read a patch file and conclude what will mount. Compose it without booting:

    # workdir = the DSH checkout
    node --import tsx/esm apps/cli/src/bin.ts --profile web --patch '<abs-path-to-patch.yml>' --dump-config

PASS is **exit 0 plus the expected rows present in the dump** — not exit 0 alone.
This is boot-free, side-effect-free, and it is the check to run after every wiring
edit (see `plugin-checklist.md` §8 for how to read pre-existing profile noise).

## Trap 3 — A test that edits a plugin's watched JSON store races the boot adopt

When a plugin adopts a store file at boot, it writes a **schema-normalized** shape
(for example `models: []`) roughly 200–300 ms after start — a ~150 ms `fs.watch`
debounce plus the provider chain.

A test that immediately `writeFileSync`s its own fixture is **clobbered** by that
write, and a `matchObject` against the raw boot file passes too early (the file
still has the un-normalized shape). Both failures are intermittent and look like
plugin bugs.

**Rule:** wait for the normalized shape to appear before editing the store, and
assert against the normalized shape, not the boot file.

## Checklist

- [ ] Launcher flags precede app flags in every command you copy.
- [ ] Wiring changes are proven with `--dump-config`: exit 0 **and** the rows.
- [ ] A test that writes a watched store waits for the normalized shape.
- [ ] A claim about what mounted cites the dump, never the patch file.
