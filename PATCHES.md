# Fork patch series

Operational record of what this fork changes and how to carry it forward.
`NOTICE-LGPL.md` is the licence notice and states the *rationale*; this file is
the *maintenance* document. Keep both in sync when the delta changes.

## Base

| | |
| --- | --- |
| Upstream | `https://github.com/semgrep/semgrep-vscode` (LGPL-2.1) |
| Base commit | `ec6a91b` — tag **`v1.17.0`** |
| Fork branch | `fix/legacy-lsp-strict-settings` |
| Fork remote | `origin` → `https://github.com/lcrusan-md/semgrep-vscode` |
| Published version | `1.17.2` as `mdthink-appsec.semgrep-patched` |
| Engine pinned by | `semgrep-version` → `release-1.159.0` |

`git diff --stat v1.17.0..HEAD` is the authoritative delta and should stay small:

```
NOTICE-LGPL.md | 33 +   (new)
PATCHES.md     |        (new — this file)
package.json   | 15 +-
src/extension.ts|       (patch 2)
src/lsp.ts     | 12 +
```

Upstream `develop` is currently **5 commits ahead** of the base tag
(`81d213a`, `307d26b`, `fd645e1`, `74f2a2b`, `14dce33`) — all CI/workflow
hardening, none touching `src/`. Rebasing onto `develop` should therefore be
conflict-free.

## Patch 1 — strip `scan.secrets` from `initializationOptions` (functional)

**File:** `src/lsp.ts`, in `lspOptions()`, immediately after
`initializationOptions` is spread from `env.config.cfg` and before
`initializationOptions.metrics` is assigned.

The legacy LSP server parses the `scan` object with `ppx_deriving_yojson` in
strict mode (`semgrep/semgrep` `src/lsp_legacy/server/Legacy_user_settings.ml`),
which rejects the **entire** object on any key the OCaml record does not declare.
`Legacy_initialize_request.ml` swallows that parse error and falls back to
defaults, where `configuration` is empty. VS Code materializes `scan.secrets`
from its settings-schema default even when the user never sets it, so this
tripped on every activation — silently discarding the user's
`semgrep.scan.configuration` and scanning with the `auto` registry pack instead.

**This is the patch the whole `appsec-ide-toolkit` rollout depends on.** That
project's `ide/vscode/settings.template.json` sets
`semgrep.scan.configuration` to a local rules directory *and* explicitly sets
`semgrep.scan.secrets: false` — precisely the combination that triggers the bug.

**Upstream fix in flight:** `lcrusan-md/semgrep` branch
`fix/legacy-lsp-strict-yojson-settings` (two commits, against
`Legacy_user_settings.ml` and `Legacy_initialize_request.ml`) fixes this
server-side. See "Exit condition" below.

## Patch 2 — remove the hosted-MCP activation nudge

**File:** `src/extension.ts`, end of `afterClientStart()`. Upstream calls
`vscode.commands.executeCommand("semgrep.mcpSetup")` on every activation; this
fork comments it out.

`semgrep.mcpSetup` prompts the user to wire the hosted
`https://mcp.semgrep.ai` remote MCP server into the current repository (writing
`.cursor/mcp.json` and `.cursor/rules/semgrep.mdc`). `appsec-ide-toolkit`'s
stated guarantee is that no code leaves the developer's machine, so the prompt is
removed rather than left to depend on the user declining it.

Scope: the nudge is gated on `vscode.env.uriScheme === "cursor"`, so it was
already inert in plain VS Code. This only affects developers using Cursor. The
`semgrep.mcpSetup` command remains registered in `src/commands.ts` and can still
be invoked deliberately from the palette.

## Patch 3 — identity and build (`package.json`)

- `name` `semgrep` → `semgrep-patched`; `displayName` → `Semgrep (patched, mdthink-appsec)`; `publisher` `Semgrep` → `mdthink-appsec`; `version` → `1.17.2`; `repository` → the fork. This prevents collision with, and any "update" path from, the Marketplace extension.
- `vscode:prepublish` `npm run esbuild-base -- --minify` → `npm run esbuild-extension-only`, a new inline `esbuild.buildSync` for `src/extension.ts` only.

**Why:** the build machine's only Node.js is 32-bit, and `lightningcss` (pulled
in by `esbuild-css-modules-plugin`, used for the webview's CSS modules) publishes
no `win32-ia32` binary, so `build.mjs` cannot build `src/webviews/index.tsx`.

**Accepted consequence:** `out/webview.js` and `out/webview.css` are never
produced, so the **Code Search** panel and **Policy Configuration** tree render
blank. Diagnostics — the reason the extension is deployed — are unaffected. To
restore the panels, build on 64-bit Node and revert `vscode:prepublish` to
`esbuild-base -- --minify`.

## Rebasing onto a newer upstream

```bash
git fetch upstream
git rebase upstream/develop        # or a newer tag
npx tsc --noEmit -p tsconfig.json  # must exit 0
npm run lint
```

Then rebuild, repackage, and reinstall:

```bash
npm ci
npx vsce package                   # runs vscode:prepublish
code --uninstall-extension mdthink-appsec.semgrep-patched
code --install-extension semgrep-patched-<version>.vsix
```

Bump `version` in `package.json` on every repackage so VS Code actually replaces
the installed build, and update the VSIX committed at
`appsec-ide-toolkit/ide/vscode/` — `Install-DevKit.ps1` installs whatever single
`.vsix` it finds there.

**Check patch 1 is still needed and still correct** after any rebase: confirm
`initializationOptions` is still built by spreading `env.config.cfg` in
`lspOptions()`, and confirm `scan.secrets` still exists in
`contributes.configuration` with a schema default.

## Verifying the patches actually work

1. Set `semgrep.scan.configuration` to a local rules directory and leave `semgrep.scan.secrets` unset.
2. Open the **Semgrep (Client)** output channel.
3. Confirm the logged `Semgrep Initialization Options` block contains your `scan.configuration` paths and **no** `scan.secrets` key.
4. Open a file with a known finding; confirm squiggles come from your local rules, not registry rule IDs.

For patch 2: in Cursor, activate the extension on a repo with no
`.cursor/rules/semgrep.mdc` and confirm no MCP setup prompt appears.

## Exit condition

This fork exists only because of patch 1. Retire it when **all** of the
following hold:

1. `lcrusan-md/semgrep@fix/legacy-lsp-strict-yojson-settings` is merged upstream into `semgrep/semgrep`.
2. A released Semgrep version containing that fix is available, and `appsec-ide-toolkit/engine/requirements.txt` pins at or above it.
3. A scan with the stock Marketplace `semgrep.semgrep` extension is confirmed to honour `semgrep.scan.configuration` with `scan.secrets` present.

Then: point `appsec-ide-toolkit/ide/vscode/` at the Marketplace extension,
delete the VSIX, and update `Install-DevKit.ps1` and `README.md`. Patch 2 is a
policy decision, not a bug workaround — decide separately whether to keep the
fork solely for it, or handle the hosted-MCP concern another way.

**Status as of 2026-09-09:** this fork's branch **is** published — `origin`
carries `fix/legacy-lsp-strict-settings` at `974bcf9`, matching local `HEAD`, so
the source-availability statement in `NOTICE-LGPL.md` is satisfied. The
engine-side branch is published on `lcrusan-md/semgrep` as well. What has **not**
happened is the upstream pull request against `semgrep/semgrep` — that is the
outstanding task, and until it merges this fork stays load-bearing.

Note when checking this yourself: `git rev-parse @{u}` reports "no upstream
configured" whenever a branch simply lacks tracking config, and
`lcrusan-md/semgrep` has a narrowed fetch refspec
(`+refs/heads/develop:refs/remotes/origin/develop`) that creates no
remote-tracking refs for feature branches. Neither is evidence that a branch is
unpublished. Use `git ls-remote origin '<branch>'`.

## Known issues carried by this fork

- **The test suite will not run** without an edit: `src/test/suite/basic.test.ts` resolves the extension by the hard-coded ID `vscode.extensions.getExtension("Semgrep.semgrep")` (non-null asserted), which is `mdthink-appsec.semgrep-patched` here.
- **No webview bundle** — see patch 3.
- **`semgrep-version` (`release-1.159.0`) is decorative in this build.** No engine binary is bundled, so the version file only affects `findSemgrep()`'s reporting when `dist/` exists. The engine actually used is whatever `semgrep.path` points at — for toolkit installs, the pinned venv from `engine/requirements.txt`.
