> **Universal copy** — source: `deepseek_plugins/docs/best-practices/creating-a-client-card-plugin.md`
> (commit `4498abd`, `master`; that repository is now frozen at
> `_archive/deepseek_plugins/` in the workspace). Copied 2026-10-13; internal
> path references were rebased to this `guides/` folder on the same day.
>
> **⚠ Written against DSH 0.1.6 — the settings seam it describes no longer exists.**
> In 0.2.x, `ctx.settings.register`, `settingsScope`, `settingsNamespace`,
> `SettingsScope`/`SettingsScopeSnapshot`, the `settings.plugin.item` slot and the
> `@deepseek-ai/dsh-settings-file` package are **removed**. They are replaced by
> `ctx.settings` (describe/update/replace), `ctx.configForms` and
> `ctx.settingsSchema`, with a volatile schemastery `Config` giving a native
> settings page for free. Read §2, §7 and §8 as **history**; take the current
> shape from a shipped 0.2.x plugin.

# Best practices — creating a client (web settings card) plugin

Proven example: `web-compact-config` (M3). Everything below states **proven
behavior only**; every section carries its source pointer. If a referenced
document disagrees with this guide, the reference wins — report the drift, do
not silently edit either side (AGENTS.md §7).

| Claim | Source of truth |
|---|---|
| Client package contract, bundle purity gate | web-compact-config/package.json, tsdown.config.ts |
| Host bridge / settings namespace contract | web-compact-config/src/index.ts (head comment), HANDOFF.md M3 entries |
| Client slot registration | web-compact-config/src/client/index.ts |
| Bundle banner contract | web-compact-config/tsdown.config.ts (documented in its head comment) |
| Header-control sibling rule | Card.tsx head block + commit 6c52415; plugin-kit plan K3-1 |
| Patch-row name rule, delivery, cache staleness (corrected 2026-10-13) | plugin-kit spec §4; MEMORY.md §1/§8; docs/incidents/2026-10-13-file-url-row-client-bundle-claim.md |
| Live install = junction + row; `install_bundle` with a `file:` spec injects a snapshot (2026-10-13) | docs/incidents/2026-10-13-install-injection-snapshot.md; MEMORY.md §8 |
| Hand-written no-build-step bundle contract (2026-10-12) | target `packages/preset/agent-preset/skills/cordis-plugin-development/templates/decoration/{client.js,package.json}`; `dsh-vision-bridge/{client.js,package.json}` + its `tests/client.spec.ts`; `dsh-autocompact/docs/PLUGIN-GUIDE.md` |
| Does a package need `./package.json` exported? (2026-10-12) | target `packages/client/modules/src/index.ts:871-893` (`locatePkgJson` fallback) + `:823-833` (negative verdict cached) |
| The no-Agent route handler and per-project `sessionId` (2026-10-12) | target `packages/host/webserver/src/index.ts:43-53`; `dsh-vision-bridge/src/gui.ts` + `docs/GUI.md` |
| Routes are outside the GUI session gate; same-origin mitigation (2026-10-12) | target `packages/client/connection/src/browser-auth.ts:237-292`; `dsh-vision-bridge/src/gui.ts:160-182, 346-348` |
| A config flag must gate the host AND be rendered (2026-10-13) | `dsh-vision-bridge/docs/DEVIATIONS.md` D-5; docs/incidents/2026-10-13-enabled-switch-gap.md; commit `3753bbd` |

## 1. A card plugin is two plugins in one package

- **Host half** (`src/index.ts`): registers the plugin's settings namespace —

  ```ts
  ctx.inject(['settings'], (sctx) => {
    sctx.settings.register(settingsNamespace(NS), CardSchema, { validate: …, … })
  })
  ```

  The namespace's resolved value mirrors the single store file; the file stays
  the lower layer (no composition `base` layer), so an external key removal
  is visible in the resolved value too (web-compact-config/src/index.ts head
  comment).
- **Client half** (`src/client/index.ts`, bundled): mounts a card into the
  shared `settings.plugin.item` slot, keyed by the namespace it edits —

  ```ts
  export const inject = ['slots', 'connection', 'remote', 'settingsScope']
  export function apply(ctx) {
    const controller = new MyCardController(ctx.settingsScope.bind({ namespace: NS }))
    ctx.slots.inject('settings.plugin.item', () => ctx.slots.register({
      name: 'settings.plugin.item', key: NS, inject: () => controller.inject(),
    }, MyCard))
  }
  ```

  Value imports of `@deepseek-ai/*` are forbidden here (purity gate, §3).
  The slot type declaration merges come in as `import type {}` from
  `dsh-client-ui-settings*/client` — type-only imports are erased and never
  reach the gate.

### 1b. Inject-face reactivity — NEVER put async values inside the inject face

ui-renderer **memoizes each slot entry's inject result per binding**
(`cachedRootInject` / `cachedSessionInject` WeakMaps). A static inject face
evaluated before an async handler resolves (e.g. `remote.$mount`) is then
pinned forever — proven live in the Editor tab's "still mounting" bug (0.1.6-
alpha.1, incident docs/incidents/2026-09-16-editor-tab-still-mounting-inject-cache.md).
The renderer-blessed fix, copied from `ui-workspace`:

- The inject face carries **hook sources** — objects with `getSnapshot()` /
  `subscribe(listener)` (the `HostObservable` currency): `inject: () =>
  ({ hooks: { fs: fsSource, workspaces: workspacesSource } })`.
- The renderer mines each name via `standardHookPropName` and hands the
  component synthetic selector hooks `use<Name>` — shape
  `<R>(select: (value: T) => R) => R`:
  `const fs = props.useFs?.(v => v)`.
- **Snapshot identities must be stable** while no notify happened (the uSES
  loop is identity-based): cache the resolved face once; return a shared
  `EMPTY_*` constant instead of a fresh array literal when empty.
- After the async resolution, mutate the snapshot then call every subscribed
  listener (`emitFs()`); the synthesized hooks re-render the tree.

Tests that call the view factory directly with a fully-formed face hide this
bug — exercise the inject face as data (sources present, snapshot flips after
the mount promise resolves) to catch it.
- The card edits go through the settings scope (staged draft → save plan), not
  by writing the store file directly. The host bridge commits them back to the
  file through the settings commit watcher, and external file edits are
  re-adopted behind an echo guard. Keep that direction: file → namespace is
  adoption; namespace → file is commit; the client never touches the file.
- Cross-field rules the wire schema cannot express go into the registration's
  `validate` option (delegating to the SHARED validator of the plugin that
  owns the store) — never re-implement validation in the card.

## 2. package.json client contract

```json
"dsh": {
  "client": {
    "platform": "web",
    "inject": [
      "@deepseek-ai/dsh-client-connection",
      "@deepseek-ai/dsh-client-runtime",
      "@deepseek-ai/dsh-client-ui-settings",
      "@deepseek-ai/dsh-api-remotes",
      "@deepseek-ai/dsh-client-ui-settings-plugins"
    ]
  }
}
```

Plus the exports a card package needs that a source-only plugin does not
(source-only anatomy: creating-a-source-plugin.md):

```json
"exports": {
  ".": "./src/index.ts",
  "./src/*": "./src/*",
  "./client": "./lib/client.js",
  "./package.json": "./package.json"
}
```

Source: web-compact-config/package.json. Extend the `inject` list only after
proving the new requirement on the target checkout.

## 3. Bundle contract (tsdown)

The card ships as a **CJS factory artifact** (`lib/client.js`), not as TS
source. The proven config is web-compact-config/tsdown.config.ts — copy it and
change only `ID`, the entry, and (carefully) the externals list.

- **Banner contract**: the bundle opens with a one-line loader banner
  (`window.__ModuleLoader__.load({ id: '<pkg>', factory: (require) => {`),
  closes with its matching footer (`return module.exports; } });`) and carries
  a `var module = { exports: {} }; var exports = module.exports;` intro.
  Copy the exact three strings from `outputOptions` (banner / footer / intro)
  and change only the id. The built-in card's bundle is the reference
  implementation (packages/client/tsdown.client.ts in the checkout); our
  tsdown.config.ts head comment documents why it is a standalone
  reproduction of the preset's client face.
- **externals = the platform module table**: `react`, `react/jsx-runtime`,
  `react-dom`, `react-dom/client`, and the `@deepseek-ai` client packages the
  shell shares into the frozen module table (see PLATFORM_MODULES in the
  config). Everything else — including your own files — is inlined.
- **Bundle purity gate**: a `resolveId` plugin throws for any non-externals
  `@deepseek-ai/*` value import. Keep it. It is the machine check for the
  rule "cross-plugin value imports are forbidden; collaborate through cordis
  services".
- Rebuild after client changes (tsdown; lib/ is gitignored) — the delivery
  junction points at the package on disk (§5), so an in-place rebuild is
  invisible to the browser until the staleness rule is respected (§5).

## 4. Card chrome rules (learned on 6c52415; moves to plugin-kit chrome via K3-1)

- The card head is a **disclosure** (`<button>` toggling the body); pills and
  banners derive from state, not from ad-hoc markup. Read
  web-compact-config/src/client/Card.tsx and the built-in mirror
  (packages/client/ui-settings-plugins/src/client/PluginCard.tsx in the
  checkout).
- **Never nest a control inside the disclosure button.** A header toggle must
  be a SIBLING of the button, wrapped together in a flex-row container — a
  control inside `<button>` is invalid HTML and breaks activation.
  Proven in 6c52415.
- State is drawn from the controller projection (loading / error / value
  states included); row components never edit the store file — writes go
  `face.edit(key, value)` on the staged draft.
- Every control carries an accessible name (`aria-label` / associated label);
  the master enable/disable switch uses `role="switch"` + `aria-checked`.
- **A user-facing config flag must gate the HOST and be RENDERED.** A flag that
  gated only the tools left a fully operational-looking panel whose upload still
  committed. Gate the write route too (403 before decoding the body) and render the
  state (disabled controls + a "… is disabled." row); absent listing data is not
  "disabled". Record: `docs/incidents/2026-10-13-enabled-switch-gap.md`.

## 5. Delivery: junction, patch-row name, cache staleness

1. **Patch-row `name` rule — package name OR path; the manifest is what matters.** The proven
   live pattern is a bare-name row over a **profile junction** (`name: web-compact-config`) —
   a preference, not a law: a `file://`/relative/absolute row also
   carries the client half (`locatePkgJson`, `packages/client/modules/src/index.ts:871-905`).
   What a card actually needs is the owning `package.json` reachable from the row's module
   path with `dsh.client` + `exports['./client']` present (§8), mounted one way only.
   Record: `docs/incidents/2026-10-13-file-url-row-client-bundle-claim.md` (it refutes the
   former "a file URL silently suppresses the bundle" claim recorded here).
2. **Delivery junction**: the running profile loads client packages from
   `C:\Users\<user>\.dsh\profiles\node_modules\<pkg-name>`, which is a
   junction into this workspace's package directory — create it when a new
   card package first ships (never recopy files through it).
3. **Cache staleness after in-place rebuilds**: client bundles are fetched
   with `?rev=<boot-hash>`; after an in-place tsdown rebuild the browser can
   keep serving the OLD bundle. A hard reload of the Web GUI (127.0.0.1:3080)
   is required before declaring a visual fix "not applied" — and whether a
   change rendered is ALWAYS a user checkpoint, never agent-verified.
4. `dsh-client-store` and `dsh-client-ui-renderer` are absent on the RUN
   checkout (fork-era packages) — if your card seems to need them, re-check
   what actually provides the seam on the target first (AGENTS.md §4.5).
5. **Never develop against an `install_bundle`d copy.** `install_bundle` with a `file:` spec
   installs by pnpm **injection**, not by link: `.modules.yaml` gains `injectedDeps` and the
   profile's `node_modules\<pkg>` is a hard-linked **snapshot copy** (a real directory,
   `LinkType` empty — not a junction). A later source edit never reaches the harness, and a
   re-install returns `changed:false` / `Already up to date` — a successful-looking install
   can freeze your code. Develop through the junction + row; reserve `install_bundle` for a
   frozen release. On a `patchReload: live` profile the junction + row applies entirely on
   save; a source change to an **already-applied** module needs the **next harness start** —
   a GUI re-apply (disable → enable) re-runs `apply()` on the **cached** module and does
   **not** re-import it (measured 2026-10-13). Record:
   `docs/incidents/2026-10-13-optional-service-read-404-panel.md` (I-4); install injection:
   `docs/incidents/2026-10-13-install-injection-snapshot.md`.

## 6. Before you claim done

Read plugin-checklist.md — a card plugin additionally needs the junction
reachable and the bundle rebuilt; that checklist owns it.


## 1c. Client ctx is inject-gated; mounted Remote namespaces are 'remote.<ns>' services

- Declare every ctx key the client module touches in the module's `inject`
  array AND the plugin's `dsh.client.inject` feature list. Undeclared
  `ctx.<key>` reads throw 'cannot get ... without inject' at RUNTIME (the
  whole slot entry blanks, console: "slot entry crashed in ...").
- A plugin that mounts its own Remote namespace: declare `remote` only, call
  `ctx.remote.$mount(...)`, then read the namespace via
  `ctx.get('remote.' + ns)` once the mount resolves. Never inject-declare
  '`remote.<ns>`' — you create it yourself, later; declaring it parks the
  plugin at boot.
- TS-over-tsdown will NOT fail on an unimported bare identifier; it emits the
  unbound name. Keep the import list exact; grep `lib/client.js` for bare
  module-local names after bundling.
Record: docs/incidents/2026-09-16-editor-tab-still-mounting-inject-cache.md
(follow-up §"blank after reload").

## 7. Target drift — the 0.2.x seam, and the portal trap (2026-10-12)

§1/§1b/§1c above describe the **0.1.x** seam and stay valid for those targets. On
DSH **0.2.1-alpha.2** (`D:\deepseek-harness_2`, verified 2026-10-12 by reading the
checkout) the settings seam changed — do not copy §1 onto 0.2.x:

| 0.1.x (this guide) | 0.2.1-alpha.2 |
|---|---|
| `ctx.settings.register(ns, schema, …)` | Host service `ctx.settings` (`SettingsForms`) projects each **active** entry's schemastery `Config`; a volatile Config needs no registration to get a page |
| client `ctx.settingsScope.bind({namespace})` | client `ctx.configForms` (`get` / `describe` / `whileServed`) + `ctx.settingsSchema` |
| card into `settings.plugin.item` | auto-generated page per entry, or a hand-written card into `plugins.item` gated by `configForms.whileServed([ns], …)` |
| — | tabs come from `settings.plugins.tab` (`configurable` order 0, `all` order 10); the Plugins nav is `settings.section` id `plugins`, order 15 |

Consequences for new work:

- **A volatile `Config` gets a validated, revision-safe form for free.**
  `volatileForm()` projects only the volatile subtree; the write path
  (`update`/`replace`/`mutate`) refuses non-volatile paths and stale revisions
  (`expectedRevision`), restores inherited values on unset, and writes through the
  same `configEditor.edit` a custom route would use. Hand-roll UI only for what
  the schema cannot express, and suppress the generated page with
  `configure({ auto: false })` if you ship your own.
- **Volatility is a containment rule** — a volatile node cannot nest inside
  another volatile node or an array element.
- **`react-dom` is a legal platform external** — the frozen `PLATFORM_MODULES`
  table (`platform.ts`, seeded by `seed.ts`) includes `react`,
  `react/jsx-runtime`, `react-dom`, `react-dom/client` and a few `@deepseek-ai`
  client packages. The `@deepseek-ai/*` **value**-import ban is unchanged.
- **Layout trap (cost a live round trip):** a panel rendered as an in-flow child
  of a dock row reflows the composer; `position:fixed`/`absolute` inside the
  composer is trapped by `backdrop-filter`/`transform` containing blocks and
  clipped by overflow. Mirror the shipped `DialogPill`/`useStatDialog`:
  `createPortal(panel, document.body)`, anchor to the trigger's rect, clamp to a
  12 px viewport margin, cap the height with internal scroll, re-anchor with a
  `ResizeObserver` on the panel, the trigger **and its row**, and dismiss on
  `pointerdown` + capture-phase `click` outside plus Escape with focus return.
  Keep status indicators inline; only panels float.

Worked example, evidence and the trap list: `D:\my_deepseek_harness\dsh-autocompact`
— `docs/PLUGIN-GUIDE.md`, `docs/incidents/` (14 records), `docs/RETRO.md`.

## 8. The hand-written, no-build-step bundle (proved 2026-10-12, `dsh-vision-bridge`)

**This refines §3, it does not contradict it.** §3 describes the **tsdown CJS
factory** path (a `lib/client.js` artifact) used by the 0.1.x-era card. A
plugin whose client half is plain JavaScript can skip the build step entirely:
`client.js` is the package's `./client` export, served raw and evaluated as a
**classic `<script src>`**. This is the shape the on-target template ships
(`packages/preset/agent-preset/skills/cordis-plugin-development/templates/decoration/client.js`)
and the shape `dsh-vision-bridge` and `dsh-autocompact` both proved. The
contract, in the order it bites:

- **A top-level `return` is a SyntaxError** in a classic script. The
  `typeof window === 'undefined'` guard must sit **inside an IIFE**
  (`(function () { … })()`). `tests/client.spec.ts` pins both the IIFE and the
  absence of a top-level `return`.
- **Register through the frozen facade:**
  `window.__ModuleLoader__.load({ id: '<package name>', factory: function (require) { … } })`.
  The `id` **must equal the package name**; the shell's diagnostic is
  `bundle <url> loaded without registering "<id>" via __ModuleLoader__.load`
  (`packages/client/modules/src/client/system.ts:222,374`).
- **`require` resolves against the shell's frozen module table.** A
  hand-written bundle here needs only `react` (elements/hooks) and `react-dom`
  (the portal). Any other specifier implies a build step — assert the requested
  set literally (see §8a).
- The factory returns `{ inject: ['slots'], apply(ctx) { ctx.slots.inject('<slot>', () => ctx.slots.register({ name: '<slot>', id: '<package name>', order: N }, Component)) } }`.
  Slot id and bundle id are the same string; a mismatch is the
  "loaded without registering" failure.
- **`package.json` must export `./client`, `./locale/*.json`, `./icon`, and
  `./package.json`.** The last one is the non-obvious one: `locatePkgJson` can
  fall back to `createRequire(baseUrl).resolve('<name>/package.json')`
  (`packages/client/modules/src/index.ts:886`); an exports map without that
  subpath throws `ERR_PACKAGE_PATH_NOT_EXPORTED` there, the `catch` returns
  `undefined`, and the **negative client verdict is cached until restart**
  (`:823-833`). Honest caveat: that arm only runs when the loader has no Node
  internals to resolve the row, and the on-target preset template omits
  `./package.json` and works — so treat it as the defensive export shipped
  client packages carry, not a law that every target enforces. It is cheap to
  add.
- **`dsh.client = { platform: 'web', immediately: true, inject: [<client bundle package names>] }`.**
  The `inject` array names client **bundle package names** (graph rows), never
  Cordis service keys — service keys there block the client half from ever
  activating (`packages/client/modules/src/client/manifest.ts:44-50`; incident
  `client-inject-package-names`).
- **A `file://` patch row does carry the client bundle** — the former claim is
  refuted. `locatePkgJson` walks up from a path-like row to the owning `package.json`
  (`packages/client/modules/src/index.ts:871-905`; target tests
  `packages/client/modules/tests/node-half.client.spec.ts:326-338`, `:340-359`), so what
  matters is that manifest being reachable and carrying `dsh.client` + `exports['./client']`
  — and never mounting one package both by name and by path (`:391-409`). Record:
  `docs/incidents/2026-10-13-file-url-row-client-bundle-claim.md`.
- **Display metadata is required:** a localized `meta.title` + `meta.description`
  and an original vector icon inside the package (≤ 256 KiB).
- **Delivery:** `client.js` takes effect on the next page load; on a `patchReload: live`
  profile a **newly applied** junction + row brings the host module, its tools and the client
  bundle live on save (no restart, no refresh — verified 2026-10-13); a source change to an
  **already-applied** host `src/*.ts` needs the **next harness start** — a GUI re-apply
  (disable → enable) re-runs `apply()` on the **cached** module and does **not** re-import it,
  so it cannot deliver new source (measured 2026-10-13); a volatile `Config` and the patch row
  are live. Record: `docs/incidents/2026-10-13-optional-service-read-404-panel.md` (I-4) and
  `docs/incidents/2026-10-13-install-injection-snapshot.md`.

### 8a. Assert the client contract from source (the only channel you have)

This workspace's vitest runs `environment: 'node'` with no react/jsdom, so
rendering and layout cannot be tested — say so and call it a user checkpoint.
What **is** worth pinning, because a regression breaks it, is the literal
contract. `dsh-vision-bridge/tests/client.spec.ts` is a good model:

- the literal `window.__ModuleLoader__.load(` call, the `id`, the factory shape;
- the IIFE guard and no top-level `return`;
- the externals list (`require(...)` occurrences) and `cache: 'no-store'`;
- the registered slot name/id and the exact route path strings (and the absence
  of retired routes);
- the portal call (`createPortal(panel, document.body)`), the aria/dialog
  wiring, the height cap, the observers, the dismissal keys;
- the package `exports` / `files` / `dsh` shape, the patch row (package name,
  no `file://`), the locale `meta`, the icon.

Source: `dsh-vision-bridge/tests/client.spec.ts`; the `client.js` it asserts.
Platform facts still come from the frozen table — never from a test comment
(§7, `externals-list-test-premise`).

### 8b. The panel's host routes: a handler has no Agent (2026-10-12)

A slot entry gets its `sessionId` from the client slot's `standardProps`; a host
`WebRoute.handler` gets only a raw `IncomingMessage`. So send the id
(query param on GET, body field on POST) and resolve
`ctx.agents.get(id)` → `session.header.cwd` → project dir, falling back to the
launch directory — otherwise the panel is per-process, not per-project
(`dsh-vision-bridge/src/gui.ts:78, 327, 385`; host-side resolve in
`src/index.ts:99, 109-116` → `src/workspace.ts:51-54, 67`; host-plugin guide §3b).

Register the panel from the plugin's **base** context, not behind the
attachment/LLM injection, or the panel is missing exactly when it should say
which service is missing. And because plugin routes are **outside the GUI
session gate** (`packages/client/connection/src/browser-auth.ts:237-292` is an
*index* gate), every write route needs a same-origin check — 403 when `Origin`
is present and differs from `Host`, allow a missing `Origin` — recorded as a
security note with a mitigation, never as fixed (host-plugin guide §7).
