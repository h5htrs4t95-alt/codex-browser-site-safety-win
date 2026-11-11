# Changelog

## v1.0.1 — 2026-11-11

- Support Codex Desktop `26.1103.4239.0` (plugin `26.1103.32239`, runtime
  `db1cc9e06a6c48d6`).
- Fix the resolver matcher for minified identifiers containing `$`. The
  `26.1103` build renamed the resolver default to `$R`; the old `\w+` token
  pattern did not match it, so the watcher logged `patched` while only
  inserting the navigation guard.
- Support Codex Desktop `26.1029.7803.0` (plugin `26.1029.61003`, runtime
  `9c26d707817e90a7`); generalise the resolver matcher to any function name
  and any env-reader function (`Nt` / `Yt` / `Jt`).
- Document `upstream_error` — “The encrypted content for item `cmp_…` could
  not be verified”: continuing a thread created under another auth context
  (ChatGPT auth → API-key gateway) cannot be decrypted by the new upstream.
  Use a new thread.
- Docs: full Chinese Windows guide, compatibility matrix, error map,
  auto-repair internals, diagnostics, upgrade playbook.

## v1.0.0 — 2026-10-23

- First release. Patch the security-mode resolver in every `browser-service.mjs`
  copy Codex can load — plugin cache (`browser`, `chrome`) and the `cua_node`
  runtime (`@oai/browser-desktop`, `@oai/cua`) — so an unset environment is
  treated as `disabled-for-local-testing`.
- Watcher (2 s) and scheduled task (5 min) re-apply after Codex rewrites its
  config or replaces the runtime.
- Windows guide (EN/ZH): install, verification, limits, disclaimer.
