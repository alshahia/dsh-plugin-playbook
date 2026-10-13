> **Universal copy** — source: `deepseek_plugins/docs/best-practices/creating-a-source-plugin.md`
> (commit `4498abd`, `master`; that repository is now frozen at
> `_archive/deepseek_plugins/` in the workspace). Copied 2026-10-13; internal
> path references were rebased to this `guides/` folder on the same day.

# Best practices — creating a source (host-side) plugin

Source-of-truth pointers live in the references column, not here. If this guide
and a referenced document disagree, the referenced document wins; report the
drift instead of quietly editing either side (AGENTS.md §1, §7).

| Claim | Source of truth |
|---|---|
| Package anatomy, relative imports, no root package.json | AGENTS.md §2, §4, §6 |
| Junction farm mechanics | MEMORY.md §1, ENVIRONMENT.md |
| M1 proven behavior (engine/store/trigger/archive) | compaction-handoff/src, its tests, HANDOFF.md M1 entries |
| M2 proven behavior (chat command) | compact-config-command/src, its tests, HANDOFF.md M2 entries |
| Patch-file URL scheme rule | docs/incidents/ (ERR_UNSUPPORTED_ESM_URL_SCHEME record); header comment in cordis.patch.yml |
| `ctx.fs` write surface, `FsWriteIntent`, state-file revision guards (2026-10-12) | target `packages/fs/fs/src/types.ts:76-83,117-125`, `packages/fs/fs/src/index.ts:255-293`, `packages/fs/fs-local/src/fsio.ts:588-598`; proved in `D:\my_deepseek_harness\dsh-vision-bridge\src\uploads.ts` + `docs/DEVIATIONS.md` D-3 |
| `ctx.webServer` routes and the no-Agent handler (2026-10-12) | target `packages/host/webserver/src/index.ts:43-53,334-341`; proved in `dsh-vision-bridge/src/gui.ts` + `docs/GUI.md` |
| `defineTool` parameter DSL vs JSON Schema (2026-10-12) | target `packages/core/tools/src/schema.ts:577,583,625-629` |
| Routes are outside the GUI session gate (2026-10-12) | target `packages/client/connection/src/browser-auth.ts:237-292`; mitigation and honest caveat in `dsh-vision-bridge/src/gui.ts:160-182` |
| Optional services: read with `ctx.get(name)`/`ctx.inject`; a re-apply is not a re-import (2026-10-13) | `dsh-vision-bridge/docs/INCIDENT-INDEX.md` → I-4 and `docs/incidents/2026-10-13-optional-service-read-404-panel.md`; fixed in that plugin's `8352710`, guarded in its `tests/service.spec.ts` + `tests/gui.spec.ts`; harness-side guard "postmortem 0001" in `packages/fs/tool-fs-search/tests/load-path.spec.ts:1-15` |

## 1. Package anatomy (proven: compaction-handoff, compact-config-command)

A host-side plugin is a minimal package that stays a **TS source** — no build
step:

```json
{
  "name": "<plugin-id>",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "exports": { ".": "./src/index.ts", "./src/*": "./src/*" }
}
```

Rules:

- `main`/"exports" point at `src/*.ts`. A client-card package is the only
  exception (see creating-a-client-card-plugin.md).
- Sibling plugins import each other **by relative path**
  (`../../compaction-handoff/src/store.ts`), never through junctions
  (AGENTS.md §6).
- `@deepseek-ai/*` value imports go through the alias facade only. If your
  plugin needs a `@deepseek-ai/*` package not yet in the facade, extend the
  junction farm + `vitest.config.ts` aliases per MEMORY.md §1 — do not import
  by absolute checkout path.
- No root package.json/lockfile exists in the workspace and none may be
  created; dependencies resolve through `node_modules` junctions into the
  target DSH checkout. Re-point with `node scripts/link-node-modules.mjs`
  only (AGENTS.md §3).
- Every entry file carries a JSDoc `@module <name>` header (AGENTS.md §6).
- Keep logic in focused modules (`config.ts` (validator), `store.ts` (single
  store file), `types.ts` (schemas), `index.ts` (entry)) — the M1/M2 layout
  is the pattern; see their sources for the file split.

## 2. Cordis entry pattern (proven: both M1 & M2)

The entry module exports exactly:

- `export const name = '<plugin-id>'` — the loader entry id.
- `export const Config: z<…> = z.object({…})` — composition options schema
  (schemastery). Optional; omit when the plugin takes no options.
- `export const inject = ['<service>', …]` and
  `export function apply(ctx, options): void`.

Use `ctx.effect(fn, cb)` / `ctx.inject([...], (svc) => {…})` so cleanup rides
the context lifecycle; never manage resources outside `ctx`. M1 does this for
the token-meter subscription and the store watcher; M2 for the command — read
their `index.ts` as the working examples.

## 3. Config: one store file + one shared validator

The suite's pattern (proven by M1 and reused by the M3 bridge):

- A single JSON store file is the only mutable state. It is **user-editable**
  (market line in HANDOFF.md) — treat user edits as a supported input path.
- All reads go through `readConfigRaw`/the store module; all mutations go
  through `parseHandoffConfig` (shared validator in `config.ts`) +
  `atomicWriteJson`. Never ad-hoc-validate or direct-write the store.
- Cross-field rules the schema cannot express belong in the validator, not in
  the schema mirror (see web-compact-config/src/index.ts comment lines 38-43).

If your new plugin manages its own config, re-create this split rather than
inventing a second validation path; see Task K3-3 (kit extraction planned) —
until it lands, copy from `compaction-handoff` and keep consumption-gating in
mind (spec 2026-09-13 §2: prove ≥2 users before abstracting).

## 3a. `ctx.fs` is text-only — and its write intents are not optional (2026-10-12)

**Refines §3** (a JSON store file is the pattern) with the service's real
surface, proved on `dsh-vision-bridge`.

- The mounted `Fs` service offers `writeText` (UTF-8) and `editText`; there is
  **no `writeBytes`, no `mkdir`, and no `remove`/`unlink`**. `writeFileAtomic`
  in the backend does create missing parents
  (`packages/fs/fs-local/src/fsio.ts:588-598`, `mkdir(directory, { recursive:
  true })` at `:598`), but that function is **not reachable through the
  service**. Consequence, proved: a GUI that uploads images cannot write them
  into a project folder at all.
- If your plugin must persist bytes, use the component that owns bytes — the
  harness `AttachmentStore` (`ctx.attachments.saveImage()` validates and
  content-addresses). Keep a JSON index of **references**, and re-derive the
  path (`attachments.imageHostPath(ref)`) and the bytes
  (`attachments.readImage(ref)`) per read — never trust a path stored on disk.
  That is what stops a hand-edited index pointing your plugin at an arbitrary
  file and bypassing `read_image`'s route gate. Worked example:
  `dsh-vision-bridge/src/uploads.ts` (head comment) and `docs/DEVIATIONS.md`
  D-3.
- **There is no delete primitive.** Do not ship a route/button that pretends
  to delete. `dsh-vision-bridge` answers an honest permanent **501** with the
  path to remove by hand; a route that claimed to delete and did not would be
  the one place in the plugin that lies about what happened.

### `FsWriteIntent` — the trap that reads as a passing test

```ts
type FsWriteIntent =
  | { kind: 'createIfAbsent' }                       // REJECTS an existing target (FS_NOT_OBSERVED)
  | { kind: 'replaceIfVersion'; version: FsVersion }  // REJECTS absence or mismatch
```

Source: `packages/fs/fs/src/types.ts:117-125`. The rule that matters:

- **Omitting the intent is an unconditional overwrite**, not a third union arm
  (`types.ts:120-121`, `index.ts:255-256, 274-278`).
- Any JSON state file written read-modify-write therefore needs
  `createIfAbsent` **when absent** and `replaceIfVersion` with the `version`
  from a preceding `stat` (**`FsInfo` has `{version, type, size?}` and no
  mtime** — `types.ts:76-83`). Otherwise two concurrent writers silently drop
  one entry, and the loser is data the user believes they saved.
- The guard is testable **only** with a fake that *enforces* the intent: a fake
  that ignores the third argument makes the test pass while the bug is live.
  `dsh-vision-bridge/tests/uploads.spec.ts` has a `recordingFs` that enforces
  both arms (`:124`, `:133-145`) and asserts the exact intent passed (`:399`,
  `:411`).

## 3b. Host HTTP routes: `ctx.webServer`, and what a handler does not have (2026-10-12)

Proved on `dsh-vision-bridge` (`src/gui.ts`, `docs/GUI.md`).

- **`webServer` (capital S) is the HTTP carrier.** `register(route: WebRoute):
  () => void`; `WebRoute = { kind: 'exact' | 'prefix', path, handler }`
  (`packages/host/webserver/src/index.ts:43-53`, `register` at `:334-341`).
  Registering a duplicate `(kind, path)` **throws**
  `webserver: duplicate exact route "<path>"` (`:337`). Register inside
  `ctx.effect` and return a disposer that calls every route disposer, and
  register each route in its **own** try/catch: a duplicate throws, and one
  occupied path should cost that route, not the whole panel
  (`dsh-vision-bridge/src/gui.ts:530-546`).
- Dispatch does `await route.handler(req, res)` and **consumes no return
  value**; there is **no JSON helper**. Write your own `send(res, status,
  payload)` that sets `content-type` + `cache-control: no-store` and guards
  `res.writableEnded`.
- **A route handler has no Agent.** It receives a raw `IncomingMessage`: no
  session, no cwd, no `Agent`. A per-project panel must pass its `sessionId`
  from the client slot's `standardProps` (query param on GET, body field on
  POST) and the host resolves `ctx.agents.get(id)` → `session.header.cwd` →
  project dir, falling back to the launch directory. That is the difference
  between a **per-project** panel and a **per-process** one.
- **An optional service must be read with `ctx.get(name)` or consumed through
  `ctx.inject([...], …)` — never as a bare property, and never behind a
  `catch { return undefined }` (corrected 2026-10-13).** This bullet previously
  recommended reading optional services structurally in a `try/catch`; that is
  exactly the bug. A service that is **not** declared in `inject` **throws**
  `Error: cannot get property "webServer" without inject` on property access — it
  does not return `undefined`. A `safeGet` that swallowed the throw turned "host
  rule violated" into "no web server in this composition": the panel registered
  **no routes** and answered 404 while Config, its tools and its client occupant
  all looked healthy, and the same swallow also hid `agents` (the per-project
  inbox silently fell back to the launch dir) and `attachments` (uploads
  unreadable). Shipped code consumes this view through a child scope —
  `ctx.inject(['webServer'], scope => install(scope))`
  (`packages/client/modules/src/index.ts:653`,
  `packages/client/connection/src/index.ts:137`,
  `packages/api/gateway/src/index.ts:240`) — keeping a direct read only as the
  structural fallback for contexts without `inject`, and a `safeGet` helper
  prefers `ctx.get(name)` **with an undefined check** before the property read.
  The harness's own tests guard this trap as **"postmortem 0001"**
  (`packages/fs/tool-fs-search/tests/load-path.spec.ts:1-15`; also
  `packages/mcp/mcp-client/tests/load-path.spec.ts`,
  `packages/subagent/tool-subagent/tests/tool-subagent.spec.ts:565`,
  `packages/experimental/hooks-claude-code/tests/bridge.spec.ts:467`). Record:
  `docs/incidents/2026-10-13-optional-service-read-404-panel.md` (I-4).
- **A panel that must report *itself* absent** should register from the plugin's
  base context (or a scope that exists headless) so its own routes exist even
  when a dependency is missing — but it reports absence by reading the optional
  service the sanctioned way (previous bullet) and logging, not by throwing. Note
  a `ctx.inject` callback may fire asynchronously, so a closure reading a `let` at
  request time sees the latest value (that is how the panel reports which
  children got mounted).
- **Delivery: a re-apply is not a re-import.** `client.js` takes effect on the
  next page load; a **newly applied** junction + row on a `patchReload: live`
  profile goes live entirely on save (module, tools, and the client bundle in an
  open page). But a GUI re-apply (disable → enable) re-runs `apply()` on the
  **cached** module — it does **not** re-import it — so a source change to an
  already-applied host `src/*.ts` needs the **next harness start** (measured
  2026-10-13: after editing the source and toggling off/on, the panel route still
  answered 404 while an independent probe in the same profile proved the pattern
  the plugin now uses answers 200). The same incident showed a `file:///` row is
  only for a **self-contained** module: mounting this plugin by path made the
  entry *disappear from the live composition* (no `Config.listConfigs` entry,
  tools gone) while the same form worked for the import-free `.mjs` probe — a
  package with bare imports must be mounted by **package name** through the
  profile junction.
- **Write routes need a same-origin check.** See §7.

## 3c. `defineTool` turns the `parameters` DSL into a JSON Schema (2026-10-12)

Proved on `dsh-vision-bridge` (`src/index.ts`, `tests/index.spec.ts`).

- `defineTool(...)` compiles the DSL once:
  `parameterSchemaSpecToJsonSchema(options.parameters)`, stored as
  `tool.parameters` (`packages/core/tools/src/schema.ts:577, 583`). The DSL
  annotations are **non-validating**; assertions must target
  `parameters.properties.<name>.type` and `parameters.required`, **not**
  `parameters.<name>.type`.
- `isConcurrencySafe` is soft-validated first
  (`schema.ts:625-629`: `if (validate(args).length > 0) return false`), so a
  bare call with no args returns `false`. A test that calls it with no args and
  expects your classifier's answer is testing the wrapper, not your code.
- The same soft-validation wraps `presentCall`/`presentResult`
  (`schema.ts:609-624`) because presentation may replay obsolete logged args.

## 4. Wiring: cordis.patch.yml

The workspace composition overlay (`cordis.patch.yml` at the repo root) is
how a plugin gets mounted:

```yaml
- insert:
    - id: '<plugin-id>'
      name: 'file:///<plugin-suite>/<pkg>/src/index.ts'   # Windows abs path, file:// URL
      config:
        configFile: '<plugin-suite>/<your-config>.json'    # or an absolute path
```

Hard rules (both proven by incidents — see the header comment in
cordis.patch.yml):

1. **Windows absolute paths MUST be `file://` URLs.** The loader's
   `tree.import()` does a raw `import(name)`; `import('D:/…')` fails with
   `ERR_UNSUPPORTED_ESM_URL_SCHEME`.
2. A mounted plugin may instead be referenced **by package name**
   (`name: <pkg-name>`) — this is how `web-compact-config` is mounted, and it
   is REQUIRED for a client-card package (patch-row `name` = package name;
   a file URL there silently suppresses the client bundle). Source-only
   plugins may use either form.
3. To replace a base-suite plugin, list `- id: <base-id>` with
   `disabled: true` before the `insert` block.
4. Configuration passed via the patch row `config` lands as the export-loaded
   `Config` schema values described in §2.

## 5. Verification loop (see also plugin-checklist.md)

1. `& '<target-checkout>\node_modules\.bin\vitest.CMD' run` from the workspace
   root — unit tests through the target checkout's bin (target is recorded in
   scripts/.dsh-target.txt).
2. `node --import tsx/esm apps/cli/src/bin.ts --profile web --patch
   'D:/…/<plugin-suite>/cordis.patch.yml' --dump-config` **with workdir =
   the target checkout** — wiring check without booting. Expect exit 0 and
   your plugin's row in the dump. Harmless pre-existing profile-patch noise is
   expected; the row being composed is the evidence (2026-10-12, proved on
   `dsh-vision-bridge`).
3. Never boot a long-lived harness server yourself; GUI/live verification is a
   user checkpoint (AGENTS.md §3, §4.2).

### 5a. Testing `apply` — gate the fake `inject` (2026-10-12)

A fake `ctx.inject` that always fires its callback will fire for **every**
dependency list and explode on a service the fake does not carry. Make it
**availability-gated**: run the callback only when every requested name is
present on the fake (`dsh-vision-bridge/tests/index.spec.ts:70-76`).

A fake `subagents` needs `resolveMaxDepth`: `'provider-managed'` → `undefined`
(no cap), otherwise `configured ?? 0` — matching the real service
(`packages/subagent/subagent/src/index.ts:237-241`; asserted in
`packages/subagent/subagent/tests/continuation.spec.ts:278-288`). Without it the
stock `tool-subagent` `apply` throws on the depth check
(`tool-subagent/src/index.ts:166,177`).

## 6. Before you claim done

That is a separate checklist — read plugin-checklist.md. This guide ends
before "done" and does not restate it.

## 7. Security: routes are outside the GUI's session gate (2026-10-12)

**A real, documented local-only gap in the host, not a solved problem.**
`ctx.webServer` routes registered by a plugin are **not** behind the GUI's
browser-auth gate. The gate itself authorizes an **index** request — it mints
the cookie on `/` and serves `index.html`
(`packages/client/connection/src/browser-auth.ts:237-292`); it is not general
middleware over every route. So a plugin's `/your-plugin/...` route answers
without the session cookie. The `dsh-autocompact` work re-verified this live
(200, no cookie) and it is recorded there as an open risk
(`dsh-plugin-playbook/INCIDENT-INDEX.md`, "Open or deferred").

What a plugin **can** do, and what `dsh-vision-bridge` does:

- Refuse a cross-origin **write** with 403 when `Origin` is present and its
  host differs from `Host`; allow a missing `Origin`, because a non-browser
  client sends none (`dsh-vision-bridge/src/gui.ts:167-182`, applied at
  `:346-348` and `:482-484`).
- Say so in the plugin's own limitations, next to the mitigation — an
  unauthenticated write surface that is documented is still an
  unauthenticated write surface.

Do not describe this as fixed. Keep mutating routes local, and prefer a
read-only surface plus the native settings form over a hand-rolled write
route.
