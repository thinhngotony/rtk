# OMP (Oh My Pi) Hooks

> Part of [`hooks/`](../README.md) — see also [`src/hooks/`](../../src/hooks/README.md) for installation code

## Design Intent

RTK's OMP extension is a **rewrite-only token optimizer**. It rewrites bash commands to their
`rtk`-prefixed equivalents, saving 60–90% context tokens.

**Permission gating is intentionally out of scope.** RTK does not block, confirm, or audit
commands — that concern belongs to a dedicated permission extension. This separation keeps
RTK's hook fast, predictable, and composable with other OMP extensions.

## Specifics

- TypeScript hook using OMP's `HookAPI` (loaded via the `hooks/pre/` discovery path)
- Subscribes to `tool_call` events and narrows to the `bash` tool via `toolName`
- Calls `rtk rewrite` via `pi.exec`; mutates and returns the revised input for compatibility across OMP versions
- All error paths return `undefined` (pass through); RTK never blocks execution
- Version guard at load time: probes `rtk --version`; if the binary is missing, sets a persistent status-line warning via `session_start` (`ctx.ui.setStatus`) and disables rewrites; if `< 0.23.0`, logs a warning and disables rewrites — the extension never blocks on a missing or stale binary
- Installed to `.omp/hooks/pre/rtk.ts` (project-local) or `~/.omp/agent/hooks/pre/rtk.ts` (global)

## Architecture

OMP loads `hooks/pre/*.ts` files as hooks at startup. Older versions observe in-place input
mutation, while newer versions apply a handler's returned `{ input }`. The hook does both so
the rewritten command reaches execution across supported OMP versions.

## Install

```bash
# Project-local (default)
rtk init --agent omp
# → creates .omp/hooks/pre/rtk.ts

# Global — all projects
rtk init -g --agent omp
# → creates ~/.omp/agent/hooks/pre/rtk.ts
```

OMP auto-discovers extensions from the `hooks/pre/` directory on startup. Set the
`OMP_AGENT_DIR` environment variable to override the global install location.

Preview the install without writing files:

```bash
rtk init --agent omp --dry-run
```

## Uninstall

```bash
# Remove project-local install (run from the project root)
rtk init --uninstall --agent omp
# → removes .omp/hooks/pre/rtk.ts

# Remove global install
rtk init --uninstall --agent omp --global
# → removes ~/.omp/agent/hooks/pre/rtk.ts
```

Uninstall is idempotent — re-running when nothing is installed is a no-op.
Only the extension file is managed by install/uninstall.

## Testing

```bash
# Load the extension directly without installing
# (OMP loads extensions from hooks/pre/ automatically)

# Verify rewrites are active — ask the agent to run a command, then check history
rtk gain --history   # should show rtk-prefixed commands with savings %

# Test RTK_DISABLED passthrough
RTK_DISABLED=1 omp -p --mode text --no-session "Run: git status"
# → commands pass through unchanged; no rewrites in rtk gain --history
```

## Design Notes

- All filtering logic lives in `rtk rewrite` (the Rust registry), not in this file
- Exit codes 0 and 3 both mean "rewrite and allow"; they are handled identically
- Uses `pi.exec` for subprocess management — consistent with OMP's extension API
- Local interfaces (no `import type`) for maximum portability across OMP versions
