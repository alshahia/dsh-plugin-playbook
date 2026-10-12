# dsh-plugin-playbook — the universal step stone for DSH plugins

Everything here is **plugin-agnostic experience** collected from building real plugins for the
DeepSeek Harness (DSH): the issue→fix records, their one-page catalog, the process retro and the
shortest-path guide. A new plugin starts by reading this folder instead of re-deriving traps that
already have a recorded root cause, a fix and evidence.

Two rules of use:

1. **Find the shipped implementation of the same thing and copy its shape** before writing anything.
2. **Grep this folder before theorizing.** Every trap below cost someone hours once; the record
   exists so it costs you nothing.

## Start here (the order that works)

1. Read [`guides/shortest-path-to-a-plugin.md`](guides/shortest-path-to-a-plugin.md) — the funnel:
   pick the shape (30 s), then the shortest order, then the traps that cost the most.
2. Decide the shape. If the answer is "a config form", **write nothing extra**: a volatile
   schemastery `Config` gets a native, validated settings page for free.
3. Before designing, scan the [symptom table](#symptom--record-find-your-problem) below. If your
   problem is there, read that record first — it holds the root cause, the fix and the evidence.
4. Copy the shipped implementation (or the nearest existing plugin) and adapt. Never invent a UI
   the platform already renders.
5. Wire it and prove it in this order: patch row (disable anything you replace) → junction →
   `--dump-config` (boot-free wiring proof) → suite → live check → user visual checkpoint for
   anything geometric.
6. When you prove a **new** trap, record it in your own project (issue + fix + evidence) **and**,
   if it is not specific to your plugin, copy the record into `incidents/` and add a row to
   [`INCIDENT-INDEX.md`](INCIDENT-INDEX.md). That is how this folder stays worth reading.

## Layout

| Path | What it is | Source project |
|---|---|---|
| [`guides/shortest-path-to-a-plugin.md`](guides/shortest-path-to-a-plugin.md) | The funnel: shape → shortest order → worst traps. **Read first.** | `deepseek_plugins` |
| [`INCIDENT-INDEX.md`](INCIDENT-INDEX.md) | One-page catalog of the 14 core records: issue, root cause, fix, status + a verification matrix and live-application semantics | `dsh-autocompact` |
| [`RETRO.md`](RETRO.md) | Nine **process** failures and the rule each produced (not technical bugs — the meta-lessons) | `dsh-autocompact` |
| [`incidents/`](incidents/) | The 14 core records the catalog tables, verbatim | `dsh-autocompact` |
| [`incidents/workspace-suite/`](incidents/workspace-suite/) | 6 further records from the earlier plugin suite (junction farm, inject cache, target drift, the settings-seam retirement) | `deepseek_plugins` |

Companion deep guides are **not** copied here (they carry project-specific wiring); they stay with
the workspace and are pointed to:

| Need | Read |
|---|---|
| Host-side (engine, hook, watcher, command) | `D:\my_deepseek_harness\deepseek_plugins\docs\best-practices\creating-a-source-plugin.md` |
| Client card / Web-UI surface | `D:\my_deepseek_harness\deepseek_plugins\docs\best-practices\creating-a-client-card-plugin.md` (§7 = the 0.2.x seam) |
| Pre-flight checklist | `D:\my_deepseek_harness\deepseek_plugins\docs\best-practices\plugin-checklist.md` |
| The fully worked guide + 14-entry trap list | `D:\my_deepseek_harness\dsh-autocompact\docs\PLUGIN-GUIDE.md` |
| Updating a checkout to a new release + plugin compat | `D:\my_deepseek_harness\note_issue_fix\2026-10-12-update-to-0.2.1-and-plugin-compat.md` |

## Symptom → record (find your problem)

| Symptom you are looking at | Read |
|---|---|
| Tests and `tsc` are green but the harness cannot load the plugin (`Cannot find package`) | [`runtime-resolution-alias-only`](incidents/2026-10-12-runtime-resolution-alias-only.md) — test resolution ≠ runtime resolution; the junction farm is the fix |
| The suite breaks right after a harness upgrade; a package or API vanished | [`settings-seam-redesign-retires-m3-card`](incidents/workspace-suite/2026-10-12-settings-seam-redesign-retires-m3-card.md), [`0.1.6 compatibility contrast`](incidents/workspace-suite/2026-09-15-dsh-0.1.6-alpha.1-compatibility-contrast.md) — a removed seam means *retire or port*, and an alias to a nonexistent path hides the real error |
| You are about to hand-write a settings form | [`native-settings-surface-already-serves-us`](incidents/2026-10-12-native-settings-surface-already-serves-us.md) — a volatile `Config` already gets a native, revision-safe form |
| `.volatile()` fails or behaves unexpectedly | [`schemastery-volatile-containment`](incidents/2026-10-12-schemastery-volatile-containment.md) — volatility is a *containment* rule (one node per section) |
| A clamp/drop notice is logged twice for one config generation | [`warn-once-probe-pass`](incidents/2026-10-12-warn-once-probe-pass.md) — keep the decision pass silent, the reporting pass noisy |
| A schema default silently reinterprets every configuration | [`trigger-default-regression`](incidents/2026-10-12-trigger-default-regression.md) — "unset" must be expressible |
| A client card never mounts, or a removed plugin still renders | [`client-inject-package-names`](incidents/2026-10-12-client-inject-package-names.md), [`editor-tab-still-mounting-inject-cache`](incidents/workspace-suite/2026-09-16-editor-tab-still-mounting-inject-cache.md) — `dsh.client.inject` takes **package names**, and the client graph caches |
| A panel pushes the composer bar up / is clipped | [`panel-floats-above-composer`](incidents/2026-10-12-panel-floats-above-composer.md) — portal to `<body>`, clamp, re-anchor, dismiss on outside press + Escape |
| Save is disabled, `capabilities.write: false` | [`config-editor-entries-are-loader-nodes`](incidents/2026-10-12-config-editor-entries-are-loader-nodes.md) — `configEditor` hands out Loader nodes (`options.id`) |
| Two engines both claim the same service, or an `insert:` row mounts without its patch | [`bundle-mount-composition-traps`](incidents/2026-10-12-bundle-mount-composition-traps.md) — disable the replaced row in the same edit |
| The bundle's externals list, or a test comment asserting a platform rule | [`externals-list-test-premise`](incidents/2026-10-12-externals-list-test-premise.md) — assert against the frozen `PLATFORM_MODULES` table |
| Live progress has no client-visible signal | [`gui-without-client-compaction-event`](incidents/2026-10-12-gui-without-client-compaction-event.md) — serve the phase from a host route and poll |
| An undocumented loader event looks like the right hook | [`loader-volatile-update-not-typed`](incidents/2026-10-12-loader-volatile-update-not-typed.md) — re-read live config at decision time |
| Archives/files land in the wrong project tree | [`archive-root-project-scope`](incidents/2026-10-12-archive-root-project-scope.md) — the session's own creation cwd is the authority |
| The tuner's client half is registered but the page looks stale / a card renders without its styles | [`live-profile-install`](incidents/2026-10-12-live-profile-install.md), [`card-mount-border-and-stale-http-cache`](incidents/workspace-suite/2026-09-13-card-mount-border-missing-and-stale-http-cache.md) — host source changes need a harness restart, bundles need a page load |
| An external CLI or executor resolves to the wrong path | [`agent-browser-cli-and-executor-path-drift`](incidents/workspace-suite/2026-09-13-agent-browser-cli-and-executor-path-drift.md) |
| A plugin "does not work" and you are not sure it is even loaded | [`auto-compact-not-working`](incidents/workspace-suite/2026-09-12-auto-compact-not-working.md) — prove what is actually loaded before theorizing |
| Your tests are green and the UI is still broken for the user | [`RETRO.md`](RETRO.md) rule 1 — structural proof is not layout proof; state the check the user must run |
| Two of your own records/README contradict the code | [`RETRO.md`](RETRO.md) rule 9 — run a consolidation pass; writing the index is what exposes contradictions |

## Provenance

| Copied from | At | Copied on |
|---|---|---|
| `dsh-autocompact/docs/{INCIDENT-INDEX.md, RETRO.md, incidents/*.md}` | commit `e8cf2e0` (`main`, worktree clean) | 2026-10-12 |
| `deepseek_plugins/docs/incidents/*.md` | commit `4498abd` (`master`) | 2026-10-12 |
| `deepseek_plugins/docs/best-practices/shortest-path-to-a-plugin.md` | commit `4498abd` (`master`) | 2026-10-12 |

Copies are verbatim except for a one-line **"Universal copy"** provenance header at the top of the
index, the retro and the guide. The source repos remain authoritative: each project keeps its own
copy of its notes/guide and **points here** for the shared set — this folder is a copy, not a
replacement for a project's own record keeping.

## Caveats (read before quoting numbers)

- **Counts in the records are "as of" their own moment** (test/file counts, line numbers). The
  catalog states the current baseline once, at the top of `INCIDENT-INDEX.md`; do not treat an
  "as of" number as today's truth.
- The records describe target DSH **`0.2.1-alpha.2`** (and, in the workspace set, `0.1.x` drift
  history). Line numbers and APIs move between releases — re-verify against source before relying
  on a specific line.
- Several records end in **user checkpoints** (a page refresh, one harness restart, a visual check).
  Those were the honest next step at the time, not necessarily completed.
- This folder is a **local git repo with no remote** (deliberately — no push without authorization).
  Renaming the folder is safe; update the pointers listed in the deep guides if you do.
