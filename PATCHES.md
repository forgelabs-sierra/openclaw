# Forgelabs Patches

Patches applied on the `forgelabs` branch, on top of upstream `openclaw/openclaw` `main`.

---

## Patch 1: CDP WebSocket Wildcard Fix

**Status:** Merged upstream (PR #17760) — no longer carried on this branch.

Treated `0.0.0.0` and `[::]` as wildcard bind addresses in CDP WebSocket URL normalization. Without this, remote CDP connections to containerized browsers (e.g. browserless) failed because the browser reports `ws://0.0.0.0:<port>` which OpenClaw tried to connect to literally.

---

## Patch 2: Sandbox Browser CDP for Containerized Gateways

**Status:** Active — not yet submitted upstream.

**Problem:** When the OpenClaw gateway runs inside a Docker/Podman container, the per-sandbox browser's CDP endpoint is unreachable via `127.0.0.1` (host port mapping). The gateway container cannot reach host-mapped ports on `127.0.0.1` because that refers to its own loopback, not the host.

**Fix:** After the sandbox browser container is created, resolve its internal IP via `docker inspect` and connect directly on the container network using the internal CDP port (default 9222) instead of `127.0.0.1:<host-mapped-port>`.

**Backward compatible:** Falls back to `127.0.0.1` + host-mapped port if `docker inspect` fails (e.g. when the gateway runs directly on the host).

### Files changed

| File                            | Change                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| `src/agents/sandbox/browser.ts` | `waitForSandboxCdp()` — added `cdpHost` parameter                                          |
| `src/agents/sandbox/browser.ts` | `buildSandboxBrowserResolvedConfig()` — added `cdpHost` parameter, dynamic `cdpIsLoopback` |
| `src/agents/sandbox/browser.ts` | `ensureSandboxBrowser()` — resolves container IP after `readDockerPort()`                  |
| `src/agents/sandbox/docker.ts`  | Added `readDockerContainerIp()` helper                                                     |

### Use case

```
┌─────────────────────────────┐
│  Host (macOS / Linux)       │
│                             │
│  ┌───────────────────────┐  │
│  │ Gateway container     │  │
│  │ (openclaw-agent)      │  │
│  │                       │  │
│  │  127.0.0.1:xxxxx  ✗   │  │  ← host-mapped port unreachable from inside container
│  │  10.89.1.4:9222   ✓   │  │  ← container IP on shared network works
│  └───────────┬───────────┘  │
│              │              │
│  ┌───────────▼───────────┐  │
│  │ Sandbox browser       │  │
│  │ (openclaw-sbx-browser)│  │
│  │ CDP on :9222          │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

---

## Maintenance

### Syncing with upstream

```bash
git fetch upstream
git checkout forgelabs
git merge upstream/main
# Resolve any conflicts in browser.ts / docker.ts
npm test
git push origin forgelabs
```

### Checking if patches are still needed

After merging upstream, if the `forgelabs` branch has no diff from `upstream/main`, all patches have been merged upstream — switch back to `openclaw@latest`.
