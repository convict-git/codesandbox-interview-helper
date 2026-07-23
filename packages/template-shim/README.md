# @interview-helper/template-shim

Telemetry bootstrap for CodeSandbox interview sandboxes. Captures **preview interactions** and **app console output**, then relays them to the interview collector service.

This package is consumed by interview problem templates as an npm dependency. **Do not copy its source into your template repo** — install and import it as described below.

---

## What it captures

| Signal | Scope |
|--------|-------|
| Preview clicks, inputs, focus, scroll | Inside the preview iframe only |
| `console.log` / `warn` / `error` / `info` / `debug` | App runtime in the preview |
| Telemetry health | Reports if bootstrap fails to initialize |

**What it does not capture:**

- Editor keystrokes or file navigation (handled by the candidate browser extension)
- Native Chrome DevTools (F12)
- Screen recording

---

## Installation

Add the package to your interview sandbox template:

```bash
yarn add @interview-helper/template-shim
```

Or in `package.json`:

```json
{
  "dependencies": {
    "@interview-helper/template-shim": "^1.0.0"
  }
}
```

---

## Setup

### 1. Import the bootstrap in your entry file

Add this as the **first import** in your app entry (e.g. `src/main.tsx` or `src/index.tsx`):

```ts
import "@interview-helper/template-shim/bootstrap";
```

The bootstrap must load before your app renders so console hooks and preview listeners are active from the start.

### 2. Configure the telemetry session

The shim needs a `sessionId` and collector URL to send events. Set these as **environment variables in the CodeSandbox sandbox** (Sandbox → Settings → Environment variables, or your template's `.env`).

```env
TELEMETRY_SESSION=<sessionId-from-dashboard>
TELEMETRY_COLLECTOR_URL=https://collector.example.com
TELEMETRY_INGEST_TOKEN=<ingestToken-from-dashboard>
```

| Variable | Required | Description |
|----------|----------|-------------|
| `TELEMETRY_SESSION` | Yes | Session UUID from the interviewer dashboard |
| `TELEMETRY_COLLECTOR_URL` | Yes | Base URL of the collector service |
| `TELEMETRY_INGEST_TOKEN` | Yes | Ingest token returned when the session was created |

Interview templates are **React** apps. The shim reads these via `process.env` (standard for webpack-based React sandboxes on CodeSandbox). Ensure your template's bundler inlines env vars at build time (e.g. webpack `DefinePlugin` or CodeSandbox's built-in env support).

The interviewer provides `sessionId` and `ingestToken` when creating a session in the dashboard. The candidate sandbox URL should also include `?telemetrySession=<sessionId>` so the browser extension joins the same session.

### 3. Verify before the interview

**Interviewer checklist:**

- [ ] `@interview-helper/template-shim` is in `package.json` dependencies
- [ ] Bootstrap import is the first line in the entry file
- [ ] Env vars are set in the sandbox (or `.env` committed for template defaults)
- [ ] Candidate has installed the browser extension
- [ ] Sandbox link includes `?telemetrySession=<sessionId>`

---

## How events are relayed

The shim uses two paths to reach the collector:

1. **Primary:** `window.parent.postMessage` → candidate browser extension content script on `codesandbox.io` → collector API
2. **Fallback:** direct `fetch()` to `POST /api/sessions/:id/events` if the extension relay is unavailable

Both paths require `TELEMETRY_SESSION` and `TELEMETRY_INGEST_TOKEN`.

---

## Event types emitted

| Type | Description |
|------|-------------|
| `preview_interaction` | Click, input, change, focus, blur, scroll in preview |
| `console_log` | App `console.*` output (serialized safely) |
| `telemetry_tamper` | Bootstrap removed or env vars missing at runtime |

Example `preview_interaction` payload:

```json
{
  "action": "click",
  "element": {
    "tag": "button",
    "role": "button",
    "ariaLabel": "Submit",
    "dataTestId": "submit-btn",
    "text": "Submit",
    "rect": { "x": 100, "y": 200, "width": 80, "height": 32 }
  }
}
```

---

## Troubleshooting

### No preview events in the log

- Confirm the bootstrap import is present and loads first.
- Check that env vars are set in the sandbox and available at runtime (e.g. log `process.env.TELEMETRY_SESSION` in dev).
- Ensure the candidate browser extension is installed and the sandbox URL has `?telemetrySession=`.
- Open the collector dashboard and verify the extension shows as connected.

### Console logs missing

- Confirm the bootstrap runs before any `console.log` calls in your app.
- Check the browser console for shim initialization errors.

### Events only from SDK, not from shim

- The extension may be missing — shim fallback `fetch()` requires CORS to be enabled on the collector for the sandbox preview origin.
- Verify `TELEMETRY_COLLECTOR_URL` points to the correct host.

### Candidate removed the bootstrap import

- The collector will not receive preview or console events.
- If detected, a `telemetry_tamper` event may be logged by the extension when expected events stop.
- Interviewers should verify the template before sharing.

---

## Privacy

- Only interactions **inside the preview iframe** are captured.
- Password fields and sensitive input values are not logged.
- No keystroke logging in the editor.
- Console arguments are truncated and sanitized before transmission.

---

## Development (monorepo contributors)

From the repository root:

```bash
yarn workspace @interview-helper/template-shim build
```

Build uses `tsgo` (`@typescript/native-preview`). Output is written to `dist/` and included in the published package via the `"files"` field in `package.json`.

---

## Related documentation

- [Project README](../../README.md) — full architecture, telemetry catalog, and deployment
- [CodeSandbox SDK](https://codesandbox.io/docs/sdk) — server-side collector that complements this package
