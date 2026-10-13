# Project — `dsh-vision-bridge` (let a text-only model see images)

A **project entry**, not an incident. It says what the plugin is, the shape it
proved, and where its authoritative notes live. The notes themselves stay in the
plugin's own repo — this file **points**; it does not duplicate. When this card
and the plugin's docs disagree, the plugin's docs win (report the drift).

- **Repo:** `D:\my_deepseek_harness\dsh-vision-bridge\`
- **Target:** DSH `0.2.1-alpha.2` (`D:\deepseek-harness_2`)
- **Status at hand-over:** committed and green — `3753bbd`
  (`fix(gui): make the master switch hold at the upload route and in the panel`) and
  `8885d3b` (docs: the guide, the incident index and the two records), then `71c4477`
  (`docs(install): a junction + row is the live path; install_bundle injects a snapshot`).
- **Install: DONE** (2026-10-13) — a **junction + bare-name patch row**, live-verified on the
  `patchReload: live` `web` profile with **no restart and no page refresh**: Config
  `status:"schema"`, both tools in the Host tool list, the panel occupant at
  `conversation.composer.dock` order 55, and the GUI toggle `schema`→`inactive`→`schema`.
  Not `install_bundle` — a `file:` spec there installs an injected snapshot copy (I-3, trap 12).

## What it is

A DSH plugin that gives a text-only routed model real vision **without changing
its own route**. Images live in a per-project inbox; either a one-off
`describe_image` tool or a pinned **vision sub-agent** (a second model that
genuinely declares image input) reads them and returns **text**. The
conversation model only ever receives text.

Three model-facing surfaces plus one browser surface:

| Surface | Job |
|---|---|
| `describe_image` | one image + one question → bounded text (one-off convenience) |
| `vision_agent` / `vision_agent_fresh` | a vision **sub-agent** holding many images over many steps (`fork` inherits the conversation, `spawn` starts fresh) |
| `vision_inbox` | listing/diagnostic: inbox path, configured route + whether it declares `image`, mounted children, images |
| composer-dock panel | the user uploads images per project, sees the listing, and reads the honest delete refusal |

## The four gates it does **not** remove

Four independent host gates refuse images on a text-only route, all correct, all
fatal to "let my agent look at this mockup". The plugin leaves every one in
place and moves the pixels to a route that is allowed to carry them:

1. prompt admission (browser image attachment → `MODEL_DOES_NOT_SUPPORT_IMAGES`);
2. silent image→text projection of a text-only route (keeps a truncated 8-hex id);
3. subagent delivery refused for a text-only route;
4. `read_image` refusing a text-only route by design.

The one deliberate divergence: `describe_image` does **not** call
`assertImageCapableRoute`, because it returns text to the caller while a
*different*, explicitly image-capable route consumes the image inside the host.
The mirror obligation replaces it (assert the **vision** route declares `image`).
Rationale and evidence: `docs/SPEC.md` §3.1.

## Architecture, in one glance

```
user uploads ──► <project>/.dsh/vision/reference/*.png   (user-authored truth)
agent capture ─► <project>/.dsh/vision/capture/*.png     (agent evidence)
                     │
        describe_image (one image)   vision_agent (many images, many steps)
                     │
        ctx.attachments.saveImage ──► image ContentBlock
                     │
        ctx.llm.stream({ provider, model })  ← the user-configured vision route
                     │
        BlockAssembler → text → UNTRUSTED-DATA envelope → caller
```

Host modules: `config.ts` (volatile `Config` + clamp-and-warn reader),
`workspace.ts` (project dir from `session.header.cwd`), `inbox.ts` (layout +
containment + magic-byte sniff), `prompts.ts` (versioned tasks + envelope),
`vision.ts` (the nested call), `cache.ts` (content-addressed),
`uploads.ts` (the reference index), `gui.ts` (host routes), `child.ts` (the
sub-agent mount), `index.ts` (entry), `client.js` (hand-written bundle).

## Deviations from its approved spec/plan (read these before "fixing" code back)

| # | Deviation | Where |
|---|---|---|
| D-1 | The vision sub-agent is mounted **from the plugin** (calls the stock `tool-subagent` `apply`), not from a patch row — a row's `agentOptions` is static, and `modelSelectionSettings` on a standing row throws. | `docs/DEVIATIONS.md` D-1 |
| D-2 | `toolFilter` is **not validated at mount**; the allow-list is asserted against the live `Tool.listTools` instead. | `docs/DEVIATIONS.md` D-2 |
| D-3 | An upload is committed to `ctx.attachments.saveImage()` with a JSON index of **references**, not written into a zone folder — `ctx.fs` has no binary write. "Delete" is an honest permanent 501. | `docs/DEVIATIONS.md` D-3 |
| D-4 | The host route contract is `/inbox` + `/upload` + `/remove`, not the spec's first-draft `/status` + `/list` + `/upload`. | `docs/DEVIATIONS.md` D-4 |
| D-5 | The `enabled` switch also gates the upload route with **403** (before the body is decoded or anything is committed), and the panel renders a "The bridge is disabled." state with its controls disabled — the switch is a host check *and* a rendered state, not one or the other. | `docs/DEVIATIONS.md` D-5 |
| D-6 | Installed as a junction + bare-name patch row, **not** with `install_bundle`; neither half needed a restart — the plan's A1.3 "one restart" was an assumption about the delivery table, not a requirement of the mechanism. `install_bundle` with a `file:` spec installs a pnpm-**injected hard-linked snapshot copy**, so a later source edit would never have reached the harness. | `docs/DEVIATIONS.md` D-6 |

## Trap list (short; full records in the guides)

Each trap below cost real debugging time on this plugin and is written up, with
its source, in the best-practices guides (scope: `deepseek_plugins/docs/`):

1. **`ctx.fs` cannot write or delete binary** — `writeText`/`editText` only; no
   `writeBytes`, `mkdir`, `remove`. A GUI that uploads images cannot write them
   into a project folder. Commit bytes to the attachment store; store
   *references*; re-derive the path per read. → *source-plugin guide*.
2. **`FsWriteIntent` semantics** — omitting the intent is an unconditional
   overwrite; read-modify-write JSON state needs `createIfAbsent` /
   `replaceIfVersion` from a preceding `stat`. `FsInfo` has no mtime. A fake
   that ignores the intent makes the test pass while the bug is live. →
   *source-plugin guide*.
3. **`webServer` (capital S) is the HTTP carrier** — `WebRoute = {kind, path,
   handler}`, duplicate `(kind, path)` **throws**, dispatch consumes no return
   value, and there is no JSON helper. → *source-plugin guide*.
4. **A route handler has no Agent** — pass `sessionId` from the client slot's
   `standardProps`; the host resolves `agents.get(id)` → `session.header.cwd`.
   That is the difference between a per-project and a per-process panel. →
   *source-plugin + client-card guides*.
5. **Read optional services structurally; register the panel from the BASE
   context** — a panel mounted from a nested `ctx.inject` is absent exactly when
   it must report what is missing. → *source-plugin guide*.
6. **Plugin routes are not behind the GUI's session gate** — a real local-only
   host gap. Mitigation used here: a same-origin check on every write route.
   Recorded as a security note with a mitigation, **not** a solved problem. →
   *source-plugin guide + checklist*.
7. **The hand-written client bundle contract** — classic `<script src>` (no
   top-level `return`), `__ModuleLoader__.load({id: <package name>, factory})`,
   `require` against the frozen table, the `./client` + `./package.json`
   exports, `dsh.client.inject` naming **bundle package names**. The patch row
   may name the **package** (the proven install) *or* a path — a `file://` row
   carries the client half too (`locatePkgJson` walks up to the owning
   `package.json`), so what matters is a reachable manifest with `dsh.client` +
   `exports['./client']`, and one package must never be mounted both ways. →
   *client-card guide §5.1/§8* + [2026-10-13-file-url-row-client-bundle-claim](../incidents/2026-10-13-file-url-row-client-bundle-claim.md).
8. **A client half should assert its contract from source** — rendering cannot
   be tested in a node environment; what a regression breaks is the literal
   registration, route paths, `cache: 'no-store'`, portal call, and the package
   `exports`/`files`/`dsh` shape. → *client-card guide*.
9. **`defineTool` turns the `parameters` DSL into a JSON Schema** — assert
   against `parameters.properties.<name>.type` / `parameters.required`;
   `isConcurrencySafe` soft-validates and a bare call returns `false`. →
   *source-plugin guide*.
10. **Testing a plugin's `apply`** — the fake `ctx.inject` must be
    **availability-gated**; a fake `subagents` needs `resolveMaxDepth`. →
    *source-plugin guide*.
11. **A user-facing config flag must gate the host AND be rendered** — `enabled`
    gating only the tools left the panel looking fully operational while an
    upload still committed. Fixed in `3753bbd`: the upload route answers **403**
    before any work, and the panel renders "The bridge is disabled." and disables
    its controls. → *client-card guide §4* +
    [2026-10-13-enabled-switch-gap](../incidents/2026-10-13-enabled-switch-gap.md).
12. **`install_bundle` with a `file:` spec installs a hard-linked snapshot copy, not a
    link.** pnpm *injects* a `file:` dependency (`.modules.yaml` → `injectedDeps`), so
    the harness resolves the copy, a later source edit never reaches it, and a re-install
    answers `Already up to date` — a successful-looking install can freeze your code.
    Install a **junction + bare-name patch row** to develop; reserve `install_bundle` for
    a frozen release. On a `patchReload: live` profile the junction + row applies entirely
    on save (module, tools, and the client bundle in an open page); a source change to an
    **already-applied** module needs a re-apply (GUI disable → enable) or a restart. →
    *both funnel guides + client-card guide §5 (item 5)* +
    [2026-10-13-install-injection-snapshot](../incidents/2026-10-13-install-injection-snapshot.md).

## Authoritative docs (read these, not a summary of them)

| Doc | What it is authoritative for |
|---|---|
| `dsh-vision-bridge/docs/SPEC.md` | design, the four gates, acceptance evidence classes |
| `dsh-vision-bridge/docs/PLAN.md` | the implementation plan |
| `dsh-vision-bridge/docs/DEVIATIONS.md` | D-1..D-6 and why the built system differs |
| `dsh-vision-bridge/docs/VISION-AGENT.md` | the sub-agent, the path-as-medium trick, injection containment |
| `dsh-vision-bridge/docs/GUI.md` | slots, the portal placement, the frozen route contract, browser checkpoint steps |

Evidence trail: `tests/` (11 files, the suite the plugin reports as **283 tests**
green at the time of writing), `--dump-config` exit 0 with the row, `tsc --noEmit` exit 0,
and the live install checks recorded in I-3.

## Shared-experience pointers

- The generic funnel and the plugin-agnostic trap catalog:
  [`../README.md`](../README.md) and
  [`../guides/shortest-path-to-a-plugin.md`](../guides/shortest-path-to-a-plugin.md).
- The deep guides that carry the traps above:
  `D:\my_deepseek_harness\deepseek_plugins\docs\best-practices\creating-a-source-plugin.md`
  and `...\creating-a-client-card-plugin.md`, with the pre-flight gates in
  `...\plugin-checklist.md`.
