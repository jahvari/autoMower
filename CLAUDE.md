# Claude project notes for autoMower

This is a Hubitat Elevation integration for Husqvarna Automower® robotic lawn mowers. Three Groovy files run on the hub:

- [automower-connect.groovy](automower-connect.groovy) — the SmartApp (OAuth, polling, WebSocket coordination, child device management)
- [mower-device.groovy](mower-device.groovy) — device driver for each mower
- [websocket-device.groovy](websocket-device.groovy) — device driver for the WebSocket connection

## Important constraints

- **Hubitat runtime, not standalone Groovy.** The files reference Hubitat-injected symbols (`state`, `atomicState`, `settings`, `app`, `device`, `httpGet`, `httpPost`, `sendEvent`, `subscribe`, `runIn`, `runEvery1Hour`, `interfaces.webSocket`, etc.). They cannot be compiled or run with a standalone Groovy installation. Local `groovyc` will fail on every undefined symbol.
- **No tests possible locally.** There is no Hubitat simulator in this repo. All testing must happen by installing on a real hub.
- **API rate limit: 10K requests/month, 1 req/sec.** Per the comment at the top of `automower-connect.groovy`. The default 15-minute polling interval was chosen to stay within budget.
- **Husqvarna Event-format-V1 was phased out 2025-02-28.** All WebSocket event handlers must use V2 event names (`battery-event-v2`, `mower-event-v2`, `planner-event-v2`, etc.). There is no `status-event` / `positions-event` / `settings-event` anymore.

## Code conventions used in this repo

- The author uses `//file:noinspection` directives at the top of each file to silence IntelliJ inspections (`GroovyDoubleNegation`, `GroovyUnusedAssignment`, etc.). Style follows their preferences, not strict CodeNarc defaults — see `.groovylintrc.json` for the local lint config we use.
- **String constants:** log levels (`sDEBUG`, `sERROR`, `sINFO`, `sTRACE`, `sWARN`) and miscellaneous strings (`sNULL`, `sBLANK`, `sSPACE`, `sLINEBR`) at the top of each file. Don't introduce inline string literals where a constant already exists.
- **Connection-state constants:** `cFULL`, `cWARN`, `cLOST` (in automower-connect.groovy) — distinct from log-level `sWARN` even though the underlying string happens to be the same. Use the `c*` constants for anything that reads or writes `state.connected` or compares against `apiConnected()`.
- **Token writes:** always go through `setTokens(access, refresh, expiresEpoch)` and `clearAuthToken()`. The state ↔ atomicState dual-write is centralized there.
- **Log masking:** every LOG site that would interpolate a token, API key/secret, or HTTP headers/body must route the value through one of `maskTok(t)`, `redactedSettings()`, `redactedParams(p)`, or `redactedToken(m)` (defined in `automower-connect.groovy` near `setTokens`). Don't add new log lines that include `state.authToken`, `state.refreshToken`, `state.accessToken`, `settings.apiKey`, `settings.apiSecret`, or a raw `Bearer ...` header.
- **Triplicated helpers:** `formatDt`, `getDtNow`, `mTZ`, `span`, and the per-file log helpers are duplicated between `automower-connect.groovy` and `mower-device.groovy` because Hubitat apps and device drivers run in separate Groovy contexts. **Keep both files in sync when changing any of these helpers.** `websocket-device.groovy` has a different `formatDt` (mdy flag, two format strings) — that one is NOT a duplicate; leave it alone.
- **`getMowerDNI()` was removed** — the device network id is identical to the Husqvarna mower id, no transform needed. Use mower ids directly.

## Things that need a live hub to verify

These changes are in the code but have not been tested against a real mower and Husqvarna's live API:

- The OAuth token response is now parsed via `(Map)resp.data` rather than the older quirky `resp.data.each{ kk=it.key }; JsonSlurper.parseText(kk)` workaround. The change depends on Hubitat's HTTP client honoring `contentType: "application/json"` as the Accept header. Verify on the OAuth callback and on token refresh.
- The defensive cuttingHeight/headlight handling accepts both possible V2 event shapes (top-level `attributes.cuttingHeight` or nested `attributes.settings.cuttingHeight`). Once a real WebSocket event is captured, prune the unused branch.
- The five new V2 device attributes (`isErrorConfirmable`, `inactiveReason`, `workAreaId`, `restrictedReason`, `externalReason`) are read from `srcMap.attributes.{mower,planner}.X` and null-guarded. Confirm they appear in the device's attribute pane when a real mower reports them.

## Other things to remember

- **`enableOauth()` was removed.** The previous code POSTed to `http://localhost:8080/app/edit/update?...oauthEnabled=true` to flip OAuth on automatically. This is not a documented Hubitat API and broke on hubs with non-default admin ports or auth. OAuth must now be enabled manually in the IDE — see the README and the auth-failure page.
- **The websocket child driver reads tokens via `atomicState`**, not `state`. This is why the dual-write exists. If you add new token-related state in the parent, write it to atomicState too if the child needs to read it before the parent's state has flushed.
- **`hubStartupHandler()`** is the modern hub-reboot hook. There's no `subscribe(location, "systemStart", ...)` call — Hubitat invokes the named method automatically on startup.

## Local dev setup

- JDK 21 LTS (Adoptium Temurin) at user-level: `%LOCALAPPDATA%\Programs\Eclipse Adoptium\jdk-21.x...`
- Groovy 5.0.6 at `C:\groovy-5.0.6`
- VS Code with the **Groovy Lint, Format and Fix** extension by Nicolas Vuillamy (uses CodeNarc 3.7)
- `.groovylintrc.json` at repo root tunes which lint rules apply (Hubitat code triggers many false positives — disable noisy style rules but keep real-bug rules active)
- `codenarc-ruleset.xml` and `codenarc-report.json` are local artifacts from running CodeNarc directly — not tracked
