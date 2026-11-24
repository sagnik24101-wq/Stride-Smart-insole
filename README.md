# Stride — Smart Insole Activity Platform

Responsive React + TypeScript dashboard for activity records uploaded by an ESP32 smart insole. The app supports a clearly marked demo mode and an optional Firebase Realtime Database connection.

## Run locally

```sh
npm install
cp .env.example .env.local
npm run dev
```

Set Firebase web app values in `.env.local` to use **Live** mode. Without those values the application starts normally in demo mode. Build with `npm run build`.

## Firebase setup

Create a Firebase Web App in project `piezo-based-step-counter`, enable Realtime Database in `asia-southeast1`, and enable an authentication provider (Email/Password or Google) in Firebase Authentication. Add the web app configuration to `.env.local`. The dashboard listens to `walks/` and cleans up the listener when Live mode is left.

Firebase client configuration is public by design; protect database access with Authentication and Realtime Database rules. Do not use open test-mode rules in production. Example authenticated read-only dashboard rules (adapt the path to your user/device authorization model):

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": false
  }
}
```

These rules intentionally do not authorize ESP32 writes: an embedded device needs a secure write design (for example, a trusted backend or scoped device credentials). Never ship service account credentials in the browser or firmware.

## Supported record shape

Legacy layout:

```text
walks/
  walk1/
    2026-11-24T12:00:00Z/
      lat: 22.3145
      lng: 87.3102
      steps: 125
```

Extended record fields are accepted when present: `timestamp`, `lat`, `lng`, `steps`, `distance`, `distanceSource` (`gps` / `estimated`), `gpsStatus`, `gpsAccuracy`, `sessionId`, `deviceId`, and `syncStatus`. `timestamp` can be Unix milliseconds or a date string. The adapter tolerates missing fields and invalid GPS coordinates. Cumulative counters should be session scoped; decreases are treated as counter resets. Existing legacy records without a session identifier are conservatively treated as independent counters. Device online state is never inferred from old records; heartbeat, Wi-Fi, storage, and firmware details show Unknown unless explicitly reported.

## Distance and hardware integration

GPS distance is derived with Haversine distance for consecutive valid fixes in a session, rejecting implausible speeds over 8 m/s. If a firmware record supplies a distance and source, the provided interval is respected. Estimated distance uses a 0.72 m per detected footfall fallback and is separately identified; this default is an uncalibrated placeholder and should be replaced by a reliable per-user calibration in firmware. Separated GPS points across a gap are not bridged by a fabricated route. In firmware, preserve outage intervals and session/counter epochs to improve this distinction.

Piezo sensor values, per-sensor peaks, and thresholds are not inferred from step totals. The sensor page reports missing telemetry when raw values are absent. Its calibration interface is a local workflow and does not change ESP32 settings. Dashboard notifications and profile preferences are local UI actions; authentication sign-in and account management require a configured project and production identity flow.

Expected hardware telemetry additions for full integration:

- `timestamp`, `sessionId`, monotonic cumulative `steps`, and reset/session epoch.
- GPS validity/status, accuracy or HDOP, and explicit outage boundaries.
- Interval `distance` plus `distanceSource`, with units in metres documented.
- Device heartbeat/last-seen, firmware version, Wi-Fi state, pending upload count, storage metrics, and upload result.
- Sensor raw samples or per-footfall heel/ball/toe peak summaries, sensor fusion result, and applied calibration thresholds/profile identifier.

The three piezo discs occupy different regions of one insole. Fuse activations within the firmware debounce window so a single physical footfall is counted once; do not multiply the count by three. Piezo discs detect changing stress/impact and are not static weight sensors.

## Pages

Landing, dashboard, activity history with CSV export, analytics, sensor monitoring/calibration workflow, device monitoring, and settings. Open `/` for the landing page or `/dashboard` for the dashboard.

## Location, history, and assistant

The dashboard supports browser-based live GPS tracking when opened over HTTPS (or localhost) and after the user grants location permission. The browser route distance is calculated between accepted GPS fixes for display; it is not saved to Firebase and does not provide step counts. Destination directions open Google Maps using the current browser fix. Historical coordinates come from uploaded records and are shown as coordinates because place names are not included in the record schema.

Activity history can be filtered to today, yesterday, the past 7 days, or the past 30 days. The Stride assistant page calculates summaries from records currently loaded in the dashboard. It is a local summary helper and does not call a generative AI or reverse-geocoding service.

Email/password sign-up and sign-in use Firebase Authentication when the web app configuration is present. The existing firmware data contract uploads under `walks/`; per-account data separation requires a matching firmware/database path and Firebase security rules. Device Wi-Fi provisioning also depends on a provisioning interface implemented by the ESP32 firmware. The Device page describes the required setup steps without pretending the website can provision firmware that exposes no such interface.

## Local accounts

When “This device” is selected, sign-up stores the account and display name in this browser's IndexedDB. Passwords are stored as salted PBKDF2 hashes, and a local sign-in session is kept in browser storage. These accounts do not sync to another device and can be lost if browser site data is cleared. Select “Firebase cloud” to use Firebase Authentication; Live activity records require that Firebase sign-in and the existing database rules.

## Deploying the static site

This Vite app builds to `dist/` and includes SPA rewrites for its React Router paths. Choose one host:

- **Vercel:** Import the Git repository. Use `npm run build` and `dist` as the output directory; the repository's `vercel.json` handles SPA deep links.
- **Render:** Create a Static Site from the Git repository. The root `render.yaml` defines the build command, `dist` publish directory, and rewrite to `index.html`.

Local browser accounts work without Firebase configuration. To enable the optional Firebase sign-in and live records after deployment, configure the `VITE_FIREBASE_*` variables from `.env.example` in the host's environment settings, then trigger a fresh deployment. These are client-side Firebase web-app settings, not server secrets; protect the database with Firebase Authentication and restrictive database rules. A custom domain can be connected in the selected host after the first deploy.
