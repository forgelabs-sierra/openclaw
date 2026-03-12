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

**Fix (Part A — Container IP resolution):** After the sandbox browser container is created, resolve its internal IP via `docker inspect` and connect directly on the container network using the internal CDP port (default 9222) instead of `127.0.0.1:<host-mapped-port>`.

**Fix (Part B — HTTP proxy bypass):** OpenClaw sets a global `EnvHttpProxyAgent` dispatcher (for `HTTP_PROXY`/`HTTPS_PROXY` support). This causes all `fetch()` calls — including CDP health checks and reachability probes to container-network IPs (e.g. `10.89.x.x`) — to be routed through the HTTP proxy (squid), which cannot reach those IPs. The fix uses a direct `undici.Agent` dispatcher for CDP fetch calls, bypassing the global proxy entirely. The existing `withNoProxyForCdpUrl` mechanism only handles loopback addresses and `NO_PROXY` with `EnvHttpProxyAgent` does not support CIDR notation (`10.89.0.0/16` is silently ignored).

**Backward compatible:** Falls back to `127.0.0.1` + host-mapped port if `docker inspect` fails (e.g. when the gateway runs directly on the host). The proxy bypass is safe for all environments since CDP connections are always to local or container-network endpoints.

### Files changed

| File                            | Change                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| `src/agents/sandbox/browser.ts` | `waitForSandboxCdp()` — added `cdpHost` parameter                                          |
| `src/agents/sandbox/browser.ts` | `buildSandboxBrowserResolvedConfig()` — added `cdpHost` parameter, dynamic `cdpIsLoopback` |
| `src/agents/sandbox/browser.ts` | `ensureSandboxBrowser()` — resolves container IP after `readDockerPort()`                  |
| `src/agents/sandbox/browser.ts` | `waitForSandboxCdp()` — use direct `undici.Agent` dispatcher to bypass HTTP proxy          |
| `src/agents/sandbox/docker.ts`  | Added `readDockerContainerIp()` helper                                                     |
| `src/browser/cdp.helpers.ts`    | `fetchCdpChecked()` — use direct `undici.Agent` dispatcher to bypass HTTP proxy            |

### Use case

```
┌──────────────────────────────────────────────────┐
│  Host (macOS / Linux)                            │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │ Gateway container (openclaw-agent)         │  │
│  │                                            │  │
│  │  global dispatcher = EnvHttpProxyAgent     │  │
│  │  HTTP_PROXY = http://squid:3128            │  │
│  │                                            │  │
│  │  fetch(10.89.1.4:9222)                     │  │
│  │    ├─ via EnvHttpProxyAgent → squid  ✗     │  │  ← proxy can't reach container IP
│  │    └─ via direct Agent()            ✓     │  │  ← bypass proxy, connect directly
│  │                                            │  │
│  │  127.0.0.1:xxxxx (host port)        ✗     │  │  ← loopback = container's own loopback
│  │  10.89.1.4:9222  (container IP)     ✓     │  │  ← resolved via docker inspect
│  └───────────────┬────────────────────────────┘  │
│                  │  openclaw-sandbox-browser net  │
│  ┌───────────────▼───────────┐                   │
│  │ Sandbox browser           │                   │
│  │ (openclaw-sbx-browser-*)  │                   │
│  │ CDP on :9222              │                   │
│  └───────────────────────────┘                   │
└──────────────────────────────────────────────────┘
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
