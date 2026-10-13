> **Universal copy** — source: `deepseek_plugins/_template-plugin/README.md`
> (commit `4498abd`, `master`; that repository is now frozen at
> `_archive/deepseek_plugins/` in the workspace, and the skeleton code stayed
> there with it). Copied 2026-10-13.
>
> **This is the *protocol*, not the code.** The copy-and-rename skeleton itself
> remains in the archive; to start a new plugin, follow the steps below and write
> the files fresh (or copy them from `_archive/deepseek_plugins/_template-plugin/`).

# _template-plugin — copy-and-rename skeleton for a host-side plugin

Proven example of the smallest useful plugin shape (a minimal chat command),
adapted from compact-config-command and named TODO-PLUGIN throughout. It is
NOT mounted anywhere — the leading underscore marks it "not a real plugin";
the loader never sees it, but its tests still run in the workspace suite like
any other package (spec 2026-09-13 §5).

Read before copying: creating-a-source-plugin.md (anatomy
and wiring) and plugin-checklist.md (claim-done gates).
This template does NOT cover the client-card half — for that, read
creating-a-client-card-plugin.md and start from the archived `web-compact-config` instead.

## Copy protocol

1. Copy the directory as your real package (a real name, no leading
   underscore). Git-add nothing until step 6 is green.
2. Rename every TODO-PLUGIN marker; afterwards grep the copy for
   TODO-PLUGIN and expect zero hits. Places to sweep (see the marker
   comments in the files):
   - package.json: name + description.
   - src/index.ts: @module, "export const name", the command name (the verb
     users type after the slash), the usage strings, the ctx.effect label.
   - src/parse.ts: @module, the request type name, the grammar itself.
   - tests/parse.spec.ts: assertions updated to YOUR grammar.
3. Edit the example logic out. Keep the CONTRACT, not the demo words:
   - parse returns a closed union incl. usage { error? };
   - the handler maps usage to an error result plus a usage line and never
     throws outward; failures inside are caught and rendered as
     { kind: "error" } with no partial state change;
   - cleanup rides ctx.effect (registration is lifecycle-managed;
     never unsubscribe manually);
   - all state mutations go through one shared validator + one store file
     (creating-a-source-plugin.md §3), and every write to that file passes a
     **write intent** — `createIfAbsent` or `replaceIfVersion` (§3a). If the
     copy ever persists **bytes** (an image, a PDF), it must not use `ctx.fs`:
     `ctx.fs` is text-only and has no delete (§3a).
4. Wire the copy: add an insert row to cordis.patch.yml with the absolute
   path as a file:// URL (the ERR_UNSUPPORTED_ESM_URL_SCHEME trap is
   documented in that file header). **A `file://` row is for a source-only
   plugin only** — the moment the copy grows a client half, change the row to
   the package name or the client bundle is silently dropped
   (creating-a-client-card-plugin.md §5.1, §8).
5. Run the gates from plugin-checklist.md §1-2: package
   tsconfig-level typecheck, full vitest suite via the TARGET checkout bin,
   and the --dump-config composition check (workdir = the RUN checkout) to
   confirm the new row. PASS for the composition check is **exit 0 plus the
   copy's row composed**; pre-existing profile-patch noise is not a failure
   (plugin-checklist.md §8).
6. Update the docs chain (TASKS row, HANDOFF entry, AGENTS §2 layout row) and
   commit with a feat(<plugin>) conventional subject.

## Kit pointers

Shared field-tested logic lives in _archive/deepseek_plugins/plugin-kit/ (Task K3, consumer-gated per
spec §2: ≥2 proven users before extraction). Until a module exists there,
copy this template and keep the contracts; do not invent a second validation
or mutation path alongside the suite's existing shared validator pattern.

## Contracts the copy must keep (added 2026-10-12)

These were proved on `dsh-vision-bridge`; the skeleton is compatible with all
of them, but a copy grows into them quickly. Full records in the guides cited.

1. **`ctx.fs` is text-only.** `writeText`/`editText` only — no `writeBytes`,
   no `mkdir`, no delete. Bytes go through the attachment store; a "delete"
   surface is an honest refusal, never a pretend success
   (creating-a-source-plugin.md §3a).
2. **Every JSON store write passes a write intent.** Omitting it overwrites
   unconditionally; a read-modify-write needs `createIfAbsent` when absent and
   `replaceIfVersion` after a `stat`. The test fake must **enforce** the intent
   or the test proves nothing (§3a).
3. **Host HTTP routes are `ctx.webServer.register`** inside `ctx.effect`, with
   every route disposer returned; there is no JSON helper, so a copy writes its
   own `send`; a duplicate `(kind, path)` throws on purpose. A route handler has
   **no Agent** — pass a `sessionId` for anything per-project (§3b).
4. **Register optional-service surfaces from the plugin's BASE context**, not a
   nested `ctx.inject`, so the surface can report what is missing instead of
   disappearing (§3b).
5. **`defineTool` asserts against the compiled JSON Schema**
   (`parameters.properties.<name>.type`, `parameters.required`); give
   `isConcurrencySafe` valid args when asserting the classifier's answer (§3c).
6. **A fake `ctx.inject` is availability-gated**, and a fake `subagents`
   implements `resolveMaxDepth` — otherwise the fake fires where the real
   composition would not, or the stock sub-agent apply throws (§5a).
7. **A route the copy adds is outside the GUI session gate.** If it mutates
   anything, add the same-origin check and write the limitation into the copy's
   README — mitigated, not solved (§7).
