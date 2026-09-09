# Notice — modified LGPL-2.1 work

This repository is a fork of [`semgrep/semgrep-vscode`](https://github.com/semgrep/semgrep-vscode),
licensed under LGPL-2.1 (see `LICENSE`).

- **Base:** tag `v1.17.0`
- **Fork:** https://github.com/lcrusan-md/semgrep-vscode
- **Branch:** `fix/legacy-lsp-strict-settings`
- **Modification:** `src/lsp.ts` strips the `scan.secrets` key from the LSP
  `initializationOptions.scan` object before sending it to the language server.
  VS Code materializes `scan.secrets` from its settings-schema default even when the user
  never sets it. The legacy LSP server (`semgrep/semgrep`
  `src/lsp_legacy/server/Legacy_user_settings.ml`) parses that object with
  `ppx_deriving_yojson` in strict mode, which rejects the entire object on any key the
  OCaml record doesn't declare — and `Legacy_initialize_request.ml` swallows that parse
  error and silently falls back to defaults with an empty `configuration`. The practical
  effect: the user's `semgrep.scan.configuration` (a locally-hosted rule set) was always
  discarded in favour of the full registry rule pack, with no error surfaced anywhere.
- **Also modified:** `src/extension.ts` no longer runs `semgrep.mcpSetup` on activation.
  Upstream prompts the user to wire the hosted `https://mcp.semgrep.ai` remote MCP server
  into the open repository; this build is distributed under a guarantee that no code leaves
  the developer's machine, so the prompt is removed rather than left to depend on the user
  declining it. The nudge was already gated on `vscode.env.uriScheme === "cursor"`, so this
  affects Cursor users only. The command itself remains registered and can still be invoked
  deliberately.
- **Also changed:** `package.json` — `name`/`publisher`/`version` changed to
  `semgrep-patched` / `mdthink-appsec` / `1.17.2` so this build cannot collide with, or be
  offered as an "update" from, the official Marketplace extension. `vscode:prepublish` uses
  an extension-only esbuild invocation rather than upstream's `build.mjs`, because this
  build machine's only Node.js install is 32-bit and `lightningcss` (used by the webview's
  CSS-modules plugin) has no `win32-ia32` binary — the webview/policy panels are not built
  as a result; core diagnostics are unaffected.

See `PATCHES.md` for the full patch series, the rebase and repackage runbook, and the
condition under which this fork is retired.

**Source availability:** the exact modified source is the `fix/legacy-lsp-strict-settings`
branch above, which is public. No engine binary is bundled in the built VSIX; it depends
entirely on the user's `semgrep.path` setting pointing at a separately-installed Semgrep CLI.

**To revert to the unmodified upstream extension:**
`code --uninstall-extension mdthink-appsec.semgrep-patched`, then install
`semgrep.semgrep` from the Marketplace.
