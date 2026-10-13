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
| [`guides/shortest-path-to-a-plugin.md`](guides/shortest-path-to-a-plugin.md) | The funnel: shape → shortest order → worst traps. **Read first.** | archived plugin suite |
| [`guides/creating-a-source-plugin.md`](guides/creating-a-source-plugin.md) | Deep guide — the host half: services, `ctx.fs`, `ctx.webServer`, `defineTool`, route security | archived plugin suite |
| [`guides/creating-a-client-card-plugin.md`](guides/creating-a-client-card-plugin.md) | Deep guide — the client card and the hand-written bundle contract. **Carries a 0.1.6 banner: §2/§7/§8 describe removed seams** | archived plugin suite |
| [`guides/plugin-checklist.md`](guides/plugin-checklist.md) | Pre-flight gates before claiming a plugin is done | archived plugin suite |
| [`guides/plugin-template-protocol.md`](guides/plugin-template-protocol.md) | Copy-and-rename protocol + the contract list for a new host-side plugin | archived plugin suite |
| [`guides/toolchain-and-test-traps.md`](guides/toolchain-and-test-traps.md) | Launcher flag order, `--dump-config` composition proof, the boot-adopt store-write race | archived plugin suite |
| [`guides/dsh-upgrade-to-0.2.1.md`](guides/dsh-upgrade-to-0.2.1.md) | Moving a checkout **and** a plugin suite across the 0.2.x break | workspace note |
| [`INCIDENT-INDEX.md`](INCIDENT-INDEX.md) | One-page catalog of the 18 core records: issue, root cause, fix, status + a verification matrix and live-application semantics | `dsh-autocompact` |
| [`RETRO.md`](RETRO.md) | Nine **process** failures and the rule each produced (not technical bugs — the meta-lessons) | `dsh-autocompact` |
| [`incidents/`](incidents/) | The 18 core records the catalog tables, verbatim | `dsh-autocompact` |
| [`incidents/workspace-suite/`](incidents/workspace-suite/) | 6 further records from the earlier plugin suite (junction farm, inject cache, target drift, the settings-seam retirement) | `deepseek_plugins` |
| [`projects/`](projects/) | Per-**project** entry cards: what a plugin is, its shape, its trap list, and a pointer to its own authoritative docs. The project keeps its own notes; this folder only points. | multiple |

The deep guides **are copied here** as of 2026-10-13. They used to live in the plugin suite's
workspace and were merely pointed at; that repository is now archived, and a guide that does not
travel with this repository cannot be read from another machine. The originals stay authoritative
and the copies carry a provenance header.

| Need | Read |
|---|---|
| Host-side (engine, hook, watcher, command) | `guides/creating-a-source-plugin.md` (§3a = `ctx.fs`, §3b = `ctx.webServer`, §3c = `defineTool`, §7 = route security) |
| Client card / Web-UI surface | `guides/creating-a-client-card-plugin.md` (§7 = the 0.2.x seam, §8 = the hand-written bundle + contract assertions) — **read its 0.1.6 banner first** |
| Pre-flight checklist | `guides/plugin-checklist.md` |
| New-plugin copy/rename protocol | `guides/plugin-template-protocol.md` |
| Launcher / composition / store-write traps | `guides/toolchain-and-test-traps.md` |
| Upgrading a checkout and a suite to 0.2.1 | `guides/dsh-upgrade-to-0.2.1.md` |
| The fully worked guide + 14-entry trap list | `D:\my_deepseek_harness\dsh-autocompact\docs\PLUGIN-GUIDE.md` |

## Projects (per-project entry cards)

The records above are **plugin-agnostic** experience. A project also gets a thin
**entry card** here so the next agent finds it, while its authoritative notes
stay in the project's own repo — the card points, it does not duplicate.

| Project | One line | Entry card |
|---|---|---|
| `dsh-vision-bridge` | Text-only model + real vision: per-project image inbox, `describe_image`, a pinned vision sub-agent, upload panel | [`projects/dsh-vision-bridge.md`](projects/dsh-vision-bridge.md) |

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
| A GUI must write, upload, or delete **image bytes** into a project folder | [`projects/dsh-vision-bridge.md`](projects/dsh-vision-bridge.md) traps 1-2, then `guides/creating-a-source-plugin.md` §3a — `ctx.fs` has no binary write and no delete |
| Two writers silently drop one entry from a JSON state file | same card, trap 2 / source-plugin guide §3a — `FsWriteIntent` semantics; omitting the intent overwrites |
| `webserver: duplicate exact route "…"` on mount; or a route handler needs JSON output | same card, trap 3 / source-plugin guide §3b — `ctx.webServer.register`, no JSON helper, register inside `ctx.effect` |
| A per-project panel behaves per-process | same card, trap 4 / source-plugin + client-card guides — a route handler has no Agent; pass `sessionId` |
| A panel is absent exactly when it should say a service is missing | same card, trap 5 / source-plugin guide §3b — an optional service read as a bare property **throws** (`cannot get property "…" without inject`); use `ctx.get(name)` or `ctx.inject([...])`, and never let a `catch { return undefined }` hide it (I-4) |
| A panel/route answers 404 (while `/` answers 401) and every health check looks green | [`optional-service-read-404-panel`](incidents/2026-10-13-optional-service-read-404-panel.md) — the routes were **never registered**: a bare optional-service read threw and a `try/catch` turned it into "service absent" (I-4) |
| An upload/write route answers without the GUI session cookie | same card, trap 6 / source-plugin guide §7 + checklist — a real local-only host gap; mitigate same-origin, do not call it solved |
| A hand-written `client.js` never mounts, or a top-level `return` is a SyntaxError | same card, trap 7 / client-card guide §8 — IIFE, `__ModuleLoader__.load`, the `exports` map |
| Client tests are green and the registration is still broken | same card, trap 8 / client-card guide §8 — assert the literal contract from source |
| A `defineTool` parameter assertion fails, or `isConcurrencySafe()` is false with no args | same card, trap 9 / source-plugin guide §3c — the DSL becomes a JSON Schema |
| A fake `ctx.inject` fires for every dependency list and explodes on a missing service | same card, trap 10 / source-plugin guide §5 — availability-gate the fake |

## Provenance

| Copied from | At | Copied on |
|---|---|---|
| `dsh-autocompact/docs/{INCIDENT-INDEX.md, RETRO.md, incidents/*.md}` | commit `e8cf2e0` (`main`, worktree clean) | 2026-10-12 |
| `deepseek_plugins/docs/incidents/*.md` | commit `4498abd` (`master`) | 2026-10-12 |
| `deepseek_plugins/docs/best-practices/shortest-path-to-a-plugin.md` | commit `4498abd` (`master`) | 2026-10-12 |
| `deepseek_plugins/docs/best-practices/{creating-a-source-plugin,creating-a-client-card-plugin,plugin-checklist}.md` | commit `4498abd` (`master`) | 2026-10-13 |
| `deepseek_plugins/_template-plugin/README.md` → `guides/plugin-template-protocol.md` | commit `4498abd` (`master`) | 2026-10-13 |
| `deepseek_plugins/MEMORY.md` §1/§3 lessons → `guides/toolchain-and-test-traps.md` | commit `4498abd` (`master`) | 2026-10-13 |
| `note_issue_fix/2026-10-12-update-to-0.2.1-and-plugin-compat.md` → `guides/dsh-upgrade-to-0.2.1.md` | workspace file (never in a repo) | 2026-10-13 |
| `projects/dsh-vision-bridge.md` — a new entry card, not a copy | `D:\my_deepseek_harness\dsh-vision-bridge` (dirty tree at hand-over) | 2026-10-12 |

Copies are verbatim except for a one-line **"Universal copy"** provenance header at the top of the
index, the retro and the guide. The `deepseek_plugins` repository named in this table was frozen
at `_archive/deepseek_plugins/` in the workspace on 2026-10-13; the paths quoted above are
historical, and everything still worth reading was copied into this folder first. The source repos remain authoritative: each project keeps its own
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
