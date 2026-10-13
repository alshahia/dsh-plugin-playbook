> **Universal copy** — source: `note_issue_fix/2026-10-12-update-to-0.2.1-and-plugin-compat.md`
> in the workspace `D:\my_deepseek_harness` (a loose note, never in a repository).
> Copied 2026-10-13; path references to the legacy plugin suite were rebased to
> `_archive/deepseek_plugins/`, its frozen home.
>
> It is the only record of how to move a DSH checkout **and** a plugin suite across
> the 0.2.x break. Verify every command against the checkout you actually have.

# Updating DeepSeek Harness to 0.2.1 + making the auto-compress plugin compatible

Fresh-context agent guide. Written for an agent with **zero memory** of the session
that produced it. Covers four things, in the order you will need them:

1. Update a DSH checkout to a newer release **without losing uncommitted work**.
2. Make the **auto-compress plugin suite** (`_archive/deepseek_plugins/`)
   compile and pass against the newer release.
3. Re-evaluate an **older fix note** (`note_issue_fix\deepseek_issue_solve .md`) —
   two issues: missing `omen-alpha` model, and `400 MissingSessionID`.
4. **Build + run + verify** the updated checkout (and the pitfalls that cost time).

Machine: Windows, PowerShell 5.1. Node v24.11.1, pnpm 11.28.5, TypeScript 6.0.3.

---

## 0. Orientation (read first, 90 seconds)

- **Harness checkout under test:** `D:\deepseek-harness_2`
  - remote `origin` = `https://github.com/deepseek-ai/deepseek-harness.git`,
    default branch **`master`**.
  - It is a pnpm workspace. Run every harness command with `workdir` = this path.
- **Plugin workspace:** `_archive/deepseek_plugins/` — its **own git
  repo**, no root `package.json`, `node_modules` is a **junction farm** into the
  target checkout. See its `AGENTS.md` / `ENVIRONMENT.md` / `MEMORY.md`.
  - **Target lock = `D:\deepseek-harness_2`** (recorded in `scripts/.dsh-target.txt`;
    `tsconfig.json` `paths` all point at `../../deepseek-harness_2/...`).
  - **Never** `pnpm install` in the plugin workspace; never hand-edit its
    `node_modules`. Re-point with `node scripts/link-node-modules.mjs`.
- **The running harness that hosts the user's agent session is a DIFFERENT
  process.** Never kill it. Boot your own instances on a **spare port** and tear
  them down yourself.
- **`gh` is NOT installed.** Use the GitHub REST API via `Invoke-RestMethod`.

Global shell gotcha: `$PID` is a read-only automatic variable in PowerShell — name
your own variable something else (`$pobj`).

---

## 1. Update the harness checkout to the latest release

### 1.1 Find the release

`.../releases/latest` **404s** (every release is a prerelease). List them instead:

```powershell
Invoke-RestMethod "https://api.github.com/repos/deepseek-ai/deepseek-harness/releases?per_page=20" `
  -Headers @{ 'User-Agent'='opencode' } | Select tag_name, name, published_at, prerelease
```

At time of writing the newest tag was **`dsh-v0.2.1-alpha.2`** (commit `d743267388`),
which equals `origin/master`'s tip.

### 1.2 Fetch (the refspec is crippled)

This clone's `remote.origin.fetch` is limited to a single tag, so
`git fetch origin --tags --prune` brings **tags but no remote branches**:

```powershell
git fetch origin --tags --prune
```

That is enough here: the target tag exists locally after the fetch.

### 1.3 Preserve uncommitted work FIRST (this is the trap)

`git status` showed **48 uncommitted files** (~1286 insertions), and **41 of them
also changed upstream** between the two releases. A blind checkout would clobber
or conflict with that work. Never proceed until it is saved.

```powershell
# commit the WIP onto its own branch (hooks are bypassed on purpose — it is a
# local safety checkpoint, never pushed)
git switch -c wip/local-changes
git add -A
git commit --no-verify -m "wip: local changes before updating to dsh-v0.2.1-alpha.2"
```

- The repo's **pre-commit hook is `lefthook` + `lint-staged`** and it can fail
  mid-run with `patch does not apply` / `conflict while merging unstaged changes`.
  On a throwaway WIP checkpoint that is fine — the changes are already staged and
  preserved in the index/worktree; finish with `--no-verify`.
- There was **no local `master` branch** — HEAD was *detached* at `dsh-v0.1.6-alpha.1`.
  Confirm ancestry before updating: `git merge-base --is-ancestor HEAD <tag>`.

### 1.4 Point master at the release, then install

```powershell
git checkout -B master dsh-v0.2.1-alpha.2   # tag == origin/master tip
git describe --tags HEAD                     # -> dsh-v0.2.1-alpha.2
pnpm install                                 # lockfile was bumped upstream
```

Your WIP is safe on `wip/local-changes`; rebase it later when you want it back.

---

## 2. Make the auto-compress plugin compatible with 0.2.1

Plugin = **`compaction-handoff`** (M1 engine) + **`compact-config-command`** (the
`/compact-config` chat command). `plugin-kit` also passes and is used by other
plugins. Run the suite from the plugin workspace:

```powershell
# workdir = _archive/deepseek_plugins/
& 'D:\deepseek-harness_2\node_modules\.bin\vitest.CMD' run
```

Two REAL drifts surfaced (both in `compaction-handoff/src`). Everything else that
fails (`web-compact-config/tests/bridge.spec.ts` cannot resolve
`@deepseek-ai/dsh-settings-file`; `web-compact-config`/`web-file-editor`/`plugin-kit`
tsc errors) is **out of scope — other plugins.**

### Drift 2.1 — tool results are now a first-class `tool` role message

**Root cause.** In 0.2.x, `createToolResultMessage`
(`packages/llm/llm/src/message.ts`) produces

```ts
{ role: 'tool', source: { kind: 'tool', callId }, toolCallId, content: [<text blocks>], isError }
```

The `ContentBlock` union is now
`text | image | file | reasoning | tool-call | tool-addition | tool-removal` —
there is **no `'tool-result'` block anymore**. `archive.ts`'s `renderMessage` only
detected the old shape (a `tool-result` block inside a user message), so tool
bodies rendered without their ```` ```tool-result ```` fence.

**Fix** (`compaction-handoff/src/archive.ts`): key the tool header/fence on
`message.role === 'tool'`, and delete the obsolete `tool-result` block branch in
`renderBlock` (keeping it makes TS flag a **dead comparison `TS2367`**, because the
block type no longer exists):

```ts
function renderMessage(message: Message): string[] {
  const isToolResult = message.role === 'tool'
  const lines = ['## ' + (isToolResult ? 'tool' : message.role), '']
  if (isToolResult) {
    lines.push('```tool-result', ...renderBlocks(message.content), '```', '')
  } else {
    for (const block of message.content) lines.push(...renderBlock(block))
  }
  lines.push('')
  return lines
}
// renderBlock: keep text + tool-call branches, drop the 'tool-result' branch.
```

### Drift 2.2 — subclass `static Config` fails against the parent (TS2417)

**Root cause.** schemastery moved **3.18.2 → 3.18.5-alpha.1** and its
`Schema<T>.meta.default` is **contravariant** (`Partial<T> | undefined`). A
subclass that declares `static Config: z<HandoffPluginConfig>` (whose type shares
**no keys** with the parent's `z<BasicCompactionConfig>`) makes the class **static
side** incompatible with the base → `error TS2417: Class static side ... incorrectly
extends base class static side`.

**Minimal repro** (proves it is real, not a facade artifact):

```ts
import z from 'D:/deepseek-harness_2/vendor/schemastery/src/index.ts'
interface BaseConf { x?: number }
interface SubConf { configFile?: string }
class Base { static Config: z<BaseConf> = z.object({ x: z.number() }) }
class Sub extends Base { static Config: z<SubConf> = z.object({ configFile: z.string() }) } // TS2417
```

**Fix** (`compaction-handoff/src/index.ts`): cast the static side to the
**intersection** so `Partial<...>` has keys in common with the base:

```ts
import type { BasicCompactionConfig } from '@deepseek-ai/dsh-compaction-basic'
// ...
static Config = z.object({ configFile: z.string() })
  as z<BasicCompactionConfig & HandoffPluginConfig>
```

The runtime schema still parses only `configFile`; the cast is purely to satisfy
the class static side.

### Verify the plugin

```powershell
& 'D:\deepseek-harness_2\node_modules\.bin\vitest.CMD' run          # -> 136/136 pass (19/20 files; the one fail is out-of-scope)
node node_modules/typescript/bin/tsc -p tsconfig.json               # -> NO errors in compaction-handoff/src or compact-config-command/src

# composition check (workdir = harness checkout): exit 0, plugin ids present
node --import tsx/esm apps/cli/src/bin.ts --profile web `
  --patch '_archive/deepseek_plugins/cordis.patch.yml' --dump-config
```

The tsc facade output is **noisy**: it compiles the whole checkout into one program
and reports ~214 errors inside `../../deepseek-harness_2/packages/**` because its
hand-maintained path list is incomplete for 0.2.x. Only errors whose path starts
with your plugin folder **matter**; everything under `deepseek-harness_2/` is facade
noise. Filter: `... | Select-String 'error TS' | Where-Object { $_ -notmatch 'deepseek-harness_2' }`.

---

## 3. Re-check `note_issue_fix\deepseek_issue_solve .md`

That note covered the `@earendil-works/pi-ai` wrapper in
`packages/llm/llm-pi-ai`. Where to look:

- Dependency decl: `packages/llm/llm-pi-ai/package.json` → `@earendil-works/pi-ai`.
- Installed copy: `node_modules/.pnpm/@earendil-works+pi-ai@<ver>_*/node_modules/@earendil-works/pi-ai/dist`.

### Issue 1 — `omen-alpha` model missing → **ALREADY SOLVED, no action**

- The new release already ships `"@earendil-works/pi-ai": "^1.0.2"` — far past the
  `0.85.1` bump the old note asked for. The fix *mechanism* is in the release.
- Caveat: the **built-in** `opencode-go` catalog (`dist/providers/data/opencode-go.json`,
  29 models nested by protocol) does **not** list `omen-alpha`. That is harmless
  because the user's `%USERPROFILE%\.dsh\settings.yaml` supplies an explicit
  `models:` list, which **replaces the whole catalog**. (`modelOverrides` cannot add
  ids the catalog does not describe; a `models:` list can.) `omen-alpha` is therefore
  still selectable.
- Base URLs to expect in that catalog: `anthropic-messages` → `https://opencode.ai/zen/go`,
  `openai-completions`/`openai-responses` → `https://opencode.ai/zen/go/v1`.

### Issue 2 — `400 MissingSessionID` → **PARTIALLY solved upstream; OUR FIX IS STILL NEEDED**

**What changed upstream (pi-ai 1.0.2).** `dist/providers/opencode-go.js` and
`dist/providers/opencode.js` wrap each protocol API with
`withOpenCodeSessionHeader` (`dist/providers/opencode-headers.js`), which sets
`x-opencode-session: options.sessionId` before dispatch (unless the header is
already present). So for **built-in** routes the header is now automatic — provided
the request carries a `sessionId`.

**Why that is not enough here.** The harness builds a provider in
`packages/llm/llm-pi-ai/src/provider.ts`:
- a **catalog** route with **no `api` override** → `reuseCatalogProvider` → delegates
  `streamSimple` to pi-ai's built-in provider → **header injected**.
- a route with an **explicit `api:`** override, or a **non-catalog id** → `createProvider`
  over the **raw** protocol factory (`PROTOCOLS[api]()`) → **NOT wrapped → header missing**.

The user's default route is **`opencode2`** — a custom route with
`api: openai-completions` + a `baseURL` + its own `models:` list — i.e. the
unwrapped case. So Issue 2 recurs for the user's default.

**Our fix (re-applied to `D:\deepseek-harness_2`, source plane):**

`packages/llm/llm-pi-ai/src/config.ts`
```ts
// interface PiAiProviderProfile
/** Header name the adapter sets per request to String(options.sessionId). Empty disables. */
sessionHeader?: string
// zod `profile`
sessionHeader: z.string().default(''),
```

`packages/llm/llm-pi-ai/src/adapter.ts`
```ts
function requestHeaders(
  headers: Readonly<Record<string, string>> | undefined,
  dynamic?: Readonly<Record<string, string>> | undefined,
): Record<string, string> {
  const attribution = attributionHeaders()
  const reserved = new Set(Object.keys(attribution).map(n => n.toLowerCase()))
  return {
    ...Object.fromEntries(Object.entries(headers ?? {}).filter(([n]) => !reserved.has(n.toLowerCase()))),
    ...Object.fromEntries(Object.entries(dynamic ?? {}).filter(([n]) => !reserved.has(n.toLowerCase()))),
    ...attribution,
  }
}
// ...single streamSimple call site:
headers: requestHeaders(
  profile.headers,
  profile.sessionHeader && options.sessionId !== undefined
    ? { [profile.sessionHeader]: String(options.sessionId) }
    : undefined,
),
```

**Coverage** (repo gate is per-file 100%): two cases added in
`packages/llm/llm-pi-ai/tests/adapter.spec.ts` — header present when
`sessionHeader` is configured, absent when not (both drive `assemble(...,
{ sessionId: 'sess-1' as never })` against the local `mockServer`).

**Harmlessness / interaction.** For built-in routes pi-ai's wrapper skips adding the
header when it is already present, so our injection is a no-op there. The harness
config schema (`profile = z.object({...})`) is **non-strict (no `.strict()`), so a
left-over `sessionHeader:` key in settings is silently stripped — it never breaks
loading.** Verified: `pnpm exec tsc -b packages/llm/llm-pi-ai` clean, and
`pnpm exec vitest run packages/llm/llm-pi-ai --exclude '**/*.e2e.ts'` → 388/388 pass.

**Do the real A/B when a key is available** (gate on `$env:OPENCODE2_API_KEY`):
with `sessionHeader: x-opencode-session` the request completes; without it the
gateway answers `400 MissingSessionID` (surfaced as a `finish` reason with
`kind: 'error'`, not a throw).

---

## 4. Build + run + verify (the part that consumes the most time)

### 4.1 After ANY cross-version checkout: `clean` then `build`

**Symptom.** `pnpm run build` failed with
`[MISSING_EXPORT] "SettingsProvider" is not exported by "../settings/src/index.ts"`
pointing at a `lib/types/index.js` line. **No source imports `SettingsProvider`** —
it is a **stale 0.1.6 build artifact** left in gitignored `lib/` by the old checkout.
Same class of residue also leaves missing/garbled `lib/typert.host.js` and client
bundles.

```powershell
pnpm run clean     # removes build outputs + residue from deleted packages (356 paths)
pnpm run build     # exit 0; ... "recorded 357 client artifact(s)"
```

The **client bundles must exist** before the `web` profile boots (see 4.3).

### 4.2 Headless one-shot (fastest end-to-end; needs a valid key)

The `headless` profile patch is empty, so pass a **host-only overlay** to mount the
plugin. Launcher flags (`--profile`, `--patch`) come **FIRST**; app flags after.

```powershell
# host-only overlay file (example): disable base engine, insert the plugin
#   - id: compaction-basic
#     disabled: true
#   - insert:
#       - id: compaction-handoff
#         name: 'file:///_archive/deepseek_plugins/compaction-handoff/src/index.ts'
#         config: { configFile: '_archive/deepseek_plugins/handoff-config.json' }
#       - id: compact-config-command
#         name: 'file:///_archive/deepseek_plugins/compact-config-command/src/index.ts'

node --import tsx/esm apps/cli/src/bin.ts --profile headless `
  --patch D:/deepseek-harness_2/my-overlay.yml --json "Reply with exactly: pong"
```

Observed on this machine: boot ran clean (empty stderr after the build), session
created, one turn — and the **only** error was
`401 AUTH: api key ****097b is invalid` on the default route `opencode2`.
That is a **bad user credential**, not code; the request reached the gateway, so
routing/header wiring is fine.

### 4.3 Web GUI

The **web profile patch is permanent** at
`C:\Users\<user>\.dsh\profiles\web\cordis.patch.yml` (mounts the whole plugin
suite on every boot; no `--patch` needed). To scope a check to the auto-compress
plugin, pass an overlay that disables the unrelated client plugins:

```powershell
# overlay:  - id: web-compact-config / disabled: true ; - id: web-file-editor / disabled: true
node --import tsx/esm apps/cli/src/bin.ts --profile web `
  --patch D:/deepseek-harness_2/scope.overlay.yml --no-open --port 3097
```

Readiness line (printed after the loader settles):
`dsh web: http://127.0.0.1:3097/?token=...`.
Fetch it with the token → **HTTP 200, title `DSH Local Build`**.

Rules that keep you out of trouble:
- **Never** touch the host harness; boot on a **spare port**, keep the PID, and
  `Stop-Process -Id <your-pid> -Force` when done. Verify the port is freed.
- `Start-Process ... -RedirectStandardOutput/-Error` is the reliable way to launch
  a background boot; poll the log files instead of blocking the shell.
- **Startup FAILURE logs** land in `C:\Users\<user>\.dsh\logs\startup-<ts>-<id>.log`.
  A **clean** boot writes **no** such log; a boot whose log count did not increase
  is a clean boot.
- Known non-fatal noise (ignore): repeated
  `settings-forms ... cannot get required service "configEditor" in inactive context`
  from `SettingsForms.importLegacyDocument` while importing the user's legacy
  settings — unrelated to the plugin.

---

## 5. Pitfalls / time-sinks (memorize)

- PowerShell is fighting you constantly:
  - `$PID` is read-only — use another variable name.
  - `Select-String -List` / `Format-Table` output is unreliable here; prefer the
    **Grep** tool for code search.
  - **Never** `Get-ChildItem -Recurse` over `node_modules` / junction farms — it
    explodes on broken junctions and times out. Scope the path.
- `gh` is absent → GitHub REST (`Invoke-RestMethod`), and `releases/latest` 404s.
- The clone's `git fetch` refspec only covers one tag → remote branches don't appear.
- lefthook/lint-staged can fail a WIP commit; stage everything and use `--no-verify`
  for the checkpoint only.
- The plugin **target is a path lock** (`scripts/.dsh-target.txt` + `tsconfig.json`
  paths → `D:\deepseek-harness_2`). Junctions point at that path, so updating the
  checkout **in place** (checkout master) is what makes the junctions see 0.2.1.
- `createProvider` vs `reuseCatalogProvider` is the whole story of Issue 2: the
  built-in opencode wrapper only applies to catalog routes **without** an `api:`
  override.
- Always `clean` before `build` after a version jump; stale `lib/` is invisible
  until it poisons a bundle.

---

## 6. One-paragraph checklist

Branch WIP → `git checkout -B master dsh-v0.2.1-alpha.2` → `pnpm install` →
`pnpm run clean && pnpm run build` → in the plugin workspace run the target vitest
(expect the auto-compress plugin green: 136/136) → fix `archive.ts` (tool role) and
`index.ts` (static-Config cast) if not already → confirm pi-ai is `^1.0.2` and the
catalog has no `omen-alpha` (fine — the user's `models:` list replaces it) →
re-apply the `sessionHeader` fix in `packages/llm/llm-pi-ai` (needed for custom
routes like `opencode2`) with its two tests → `--dump-config` exit 0 → boot
`--profile headless --patch <host-overlay>` (one tiny task) and
`--profile web --no-open --port <spare>` (expect the URL line + HTTP 200), then kill
your own PIDs. The two things that WILL bite a bad key and stale build artifacts:
`401 AUTH` (user credential) and `client bundles not found` / `SettingsProvider`
(stale `lib/` → `pnpm run clean`).
