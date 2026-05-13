# autoMower

A Hubitat Elevation integration for Husqvarna Automower® robotic lawn mowers, using the official Automower® Connect API (REST + WebSocket).

Forked from [imnotbob/autoMower](https://github.com/imnotbob/autoMower).

## What it does

- Connects to your mower via Husqvarna's cloud API
- Receives live state updates via WebSocket (Event-format-V2)
- Exposes mower status, battery, error codes, planner state, work-area info, and statistics as Hubitat attributes
- Lets you send commands (start, pause, park, resume schedule, set cutting height, set headlight mode, set calendar, reset blade usage time) from Hubitat rules and dashboards
- Speaks/notifies you when API connectivity is lost

## Files

| File | Purpose |
| --- | --- |
| `automower-connect.groovy` | Main SmartApp: OAuth, polling, WebSocket coordination, child device management |
| `mower-device.groovy` | Device driver for each mower |
| `websocket-device.groovy` | Device driver wrapping the persistent WebSocket connection |
| `HE/packageManager.json` | Hubitat Package Manager manifest |

## Install

### Prerequisites
- A Hubitat Elevation hub running firmware 2.2.4 or newer
- A Husqvarna developer account (free) at https://developer.husqvarnagroup.cloud

### Get a Husqvarna API key
1. Sign in to https://developer.husqvarnagroup.cloud with your Husqvarna AutoConnect credentials
2. Create a new application
3. Connect it to both the **Authentication API** and the **Automower® Connect API**
4. Set the redirect URI to: `https://cloud.hubitat.com/oauth/stateredirect`
5. Note your Application Key and Application Secret

### Install the code on Hubitat
The recommended path is via [Hubitat Package Manager](https://community.hubitat.com/t/release-hubitat-package-manager/94471). Search for "Husqvarna AutoMower Manager" (or install from this repo's `HE/packageManager.json` URL).

If installing manually:
1. **Apps Code → New App** → paste `automower-connect.groovy` → Save
2. **Drivers Code → New Driver** → paste `mower-device.groovy` → Save
3. **Drivers Code → New Driver** → paste `websocket-device.groovy` → Save
4. **Enable OAuth on the app** — this step is required and must be done manually in the Hubitat IDE:
   - Open **Apps Code** → click **Husqvarna AutoMower Manager**
   - Click the **OAuth** button at the top
   - Choose **Enable OAuth in App** → click **Update**

### Configure
1. **Apps → Add User App → Husqvarna AutoMower Manager**
2. Enter the Application Key and Application Secret from the developer portal
3. Click the authorization link, sign in to your AutoConnect account, click Allow
4. Select which mowers you want to expose as Hubitat devices
5. Save

## Rate limit notes

Husqvarna's API allows **10,000 requests per month** per application key (and at most 1 request per second). When the WebSocket is connected, the integration polls the REST endpoint less aggressively. The default 15-minute polling interval is well within the monthly budget for typical use. If you have multiple mowers, lengthen the interval rather than shortening it.

## Configuration options

- **Polling interval** — 6 / 10 / 15 / 30 / 60 minutes. Default 15.
- **Notification devices** — capability.notification devices to alert when API connectivity is lost
- **Speech / music** — speak the alert via capability.speechSynthesis / capability.musicPlayer
- **Do not disturb** — restrict speech notifications to specific Modes or times of day
- **Debug log level** — 0 (silent) to 5 (extremely verbose)

## Differences vs upstream

This fork extends [imnotbob/autoMower](https://github.com/imnotbob/autoMower) with:

- Several real bug fixes (typo that disabled the API cache, undefined-variable crash in the OAuth error path, NPE risk in the child-device event handler, more)
- Full Event-format-V2 compliance (Husqvarna phased out V1 on 2025-02-28)
- New device attributes exposing V2 fields: `isErrorConfirmable`, `inactiveReason`, `workAreaId`, `restrictedReason`, `externalReason`
- Cleaner OAuth flow with clearer error messaging when OAuth isn't enabled
- Token-refresh on HTTP 401/403 (was 500-only)
- WebSocket `connection-event` logged for diagnostics
- Multiple code-quality cleanups (dead code, unused locals, redundant `.toString()`, undocumented `displayed:` fields, helper consolidation)

See the commit log for the full list.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
