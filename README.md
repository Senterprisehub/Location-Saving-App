Detailed Working Description — Location Tracker

Location Tracker is a complete, single-file (index.html) web application that lets users track their current GPS position on a Google-tiled map, save named locations, edit or delete them, get turn-by-turn directions to any saved place, and back everything up to Google Drive as a single JSON file. It uses no OAuth, no client ID, and no Google Cloud project — Google Drive access is delegated to a Google Apps Script Web App that the user deploys once in their own account.

The entire app runs from one HTML file. No build step, no framework, no server, no dependencies beyond three public CDNs (Leaflet, Google Fonts, and optionally the browser's own APIs). It works from file://, http://localhost, or any HTTPS host.

---

Architecture Overview

```
┌─────────────────────────┐
│       index.html        │
│  ┌───────────────────┐  │
│  │  Leaflet + Map    │  │   ← Google tiles via tile URL
│  ├───────────────────┤  │
│  │  Sidebar (2 tabs) │  │   ← Locations / Settings
│  ├───────────────────┤  │
│  │  localStorage     │  │   ← Offline cache + settings
│  ├───────────────────┤  │
│  │  fetch() bridge   │  │   ← Only network call
│  └───────────────────┘  │
└───────────┬─────────────┘
            │ POST (form-urlencoded)
            ▼
┌─────────────────────────┐
│  Apps Script Web App    │
│  (runs as the user)     │
│  ┌───────────────────┐  │
│  │  doPost/doGet     │  │
│  │  saveData /       │  │
│  │  loadData         │  │
│  └────────┬──────────┘  │
└───────────┼─────────────┘
            │ DriveApp
            ▼
┌─────────────────────────┐
│  Google Drive           │
│  📁 Saved address       │
│    └─ saved_address.json│
└─────────────────────────┘
```

The browser never sees a Google credential. The Apps Script executes under the deployer's identity and is the only thing that touches Drive.

---

Core Features

1. Track & Save Locations

When the user clicks 📡 Get My Location:

· The browser's navigator.geolocation.getCurrentPosition() is called with enableHighAccuracy: true, timeout: 15000, maximumAge: 0.
· On success, a draggable Leaflet marker appears at the returned coordinates and the map flies there at zoom level 16.
· A popup opens saying "You are here — drag to fine-tune".
· If the user drags the marker, its dragend event updates the stored tempLatLng object so the saved coordinates reflect the final position.

Once a location is found, 💾 Save This Location becomes enabled. Clicking it prompts for a title, then pushes a record onto the locations array:

```js
{
  id: uuid(),
  title: "Home",
  lat: 19.0760,
  lng: 72.8777,
  createdAt: "2026-01-15T10:23:00.000Z",
  updatedAt: "2026-01-15T10:23:00.000Z"
}
```

The record is saved to localStorage, plotted on the map, and added to the list. If auto-sync is on, a debounced push to Drive fires 1.2 seconds later.

2. Manage Saved Locations

Every saved location appears as a card in the Locations tab, sorted by most recently updated. Each card shows the title, coordinates (5 decimal places), and three action buttons:

Icon Action
🧭 Opens Google Maps directions from the device's location to the pin
✏️ Prompts to rename the location
🗑️ Confirms and deletes the location

Clicking anywhere else on the card flies the map to that marker and opens its popup. Popups include the title, coordinates, a 🧭 Directions button (filled indigo) and a 📍 View link.

3. Two-Tab Sidebar

· 📍 Locations — "Get My Location" controls and the scrollable list of saved places. A live badge shows the count.
· ⚙️ Settings — Gmail ID, Apps Script preset, deployment guide, bridge URL, sync/import controls, diagnostics, status cards, and local data tools.

Tab switching slides an indigo underline and fades the new panel in via a fadeUp animation. The map is re-measured 60 ms after switching so Leaflet doesn't clip.

4. Google Maps Directions

Two URL helpers are used throughout the app:

```js
function directionsUrl(loc) {
  return 'https://www.google.com/maps/dir/?api=1'
       + '&destination=' + encodeURIComponent(loc.lat + ',' + loc.lng)
       + '&travelmode=driving';
}

function viewUrl(loc) {
  return 'https://www.google.com/maps/search/?api=1'
       + '&query=' + encodeURIComponent(loc.lat + ',' + loc.lng);
}
```

Because only destination is passed (no origin), Google Maps always uses the device's current position as the starting point. On Android and iOS the OS intercepts these URLs and opens the native Google Maps app; on desktop they open a new browser tab with routing ready.

5. Google Drive Sync & Import via Apps Script

The app talks to Drive only through a deployed Apps Script Web App. The protocol is intentionally minimal:

Client (from the app):

```js
const params = new URLSearchParams();
params.append('action', action);
if (payload !== undefined) params.append('payload', JSON.stringify(payload));

fetch(state.scriptUrl, { method: 'POST', body: params });
```

The URLSearchParams body produces an application/x-www-form-urlencoded request — a simple request in CORS terms — so no OPTIONS preflight is ever sent. This is the only request shape that reliably works with Apps Script Web Apps, whose endpoints do not answer preflight requests and often redirect mid-flight.

Server (Apps Script handle(e)):

· Reads e.parameter.action and e.parameter.payload (form-encoded).
· Falls back to e.postData.contents JSON parsing for backward compatibility.
· Dispatches on three actions: ping, save, load.
· save writes to 📁 Saved address/saved_address.json — creating the folder and file if needed, or overwriting the content if both exist.
· load returns the file's parsed JSON.

All responses are wrapped in ContentService.createTextOutput(JSON.stringify(...)) with .setMimeType(ContentService.MimeType.JSON).

6. Merge Strategy on Import

When importing from Drive, the local and remote arrays are merged by id:

```js
const byId = new Map(state.locations.map(l => [l.id, l]));

remote.forEach(loc => {
  if (!loc || loc.id == null || typeof loc.lat !== 'number') return;
  const existing = byId.get(loc.id);
  if (!existing) {
    byId.set(loc.id, loc);
    added++;
  } else {
    const a = new Date(existing.updatedAt || existing.createdAt || 0).getTime();
    const b = new Date(loc.updatedAt      || loc.createdAt      || 0).getTime();
    if (b > a) { byId.set(loc.id, loc); updated++; }
  }
});
```

· New locations from Drive are added.
· Existing locations with an older updatedAt are replaced.
· Records with malformed data are silently skipped.
· Repeated imports are safe — no duplicates can be created.

The final toast reports how many were added and how many were updated.

7. Auto-Sync & Live Status

An auto-sync toggle in the Settings tab enables/disables automatic pushing. When enabled, any mutation — save, rename, delete, or Gmail ID change — schedules a pushToDrive call 1.2 seconds later. Debouncing means rapid edits result in one upload, not many.

The status bar at the top of the sidebar always reflects current state:

Colour Meaning
🟢 Green Connected — all changes saved
🟡 Amber Changes pending sync
🔵 Indigo (pulsing) Actively syncing right now
🔴 Red Last sync failed — see Diagnostics
⚪ Grey Bridge URL not configured

A beforeunload handler warns the user if they try to close the tab while there are unsynced changes and a bridge is configured.

8. Modern UI

· Inter typeface with tight letter-spacing; JetBrains Mono for code.
· Design tokens as CSS variables — every colour, radius, and shadow is centralized in :root.
· Automatic dark mode through prefers-color-scheme: dark, which redefines the tokens.
· Gradient primary buttons with a soft indigo glow that lifts on hover.
· Animated tab underline that scales in from the centre.
· Hover-lift location cards with a sliding gradient accent bar on the left edge.
· Custom toggle switches using appearance: none and an inner knob.
· Toast notifications with spring easing at the bottom centre.
· Empty state with an inline SVG map-pin illustration when no locations are saved.
· Mobile layout — sidebar docks to the bottom as a rounded sheet with a drag handle pseudo-element.

---

The Settings Tab — Three Numbered Steps

Step 1 — Copy the Apps Script Code

A dark IDE-style block shows the Code.gs content with a JetBrains Mono font, an amber file dot, and a 📋 Copy code button. The button:

1. Tries navigator.clipboard.writeText() in a secure context.
2. Falls back to an off-screen <textarea> + document.execCommand('copy') for file:// or older browsers.
3. Flashes green "✅ Copied!" for two seconds.

The code lives in a <script type="text/plain" id="appsScriptSource"> block at the bottom of the file. It is never executed or rendered — JS reads its .textContent and injects it into the viewer on load. This keeps the displayed code always in sync with the app's expectations.

The viewer is collapsible: max-height: 0 → max-height: 640px transition when the "Show / hide the code" bar is clicked, with a rotating chevron.

Step 2 — Deploy & Get the Web App URL (Collapsible)

Collapsed by default. The header row shows the numbered badge, title, and a chevron that rotates 180° when opened. Clicking anywhere on the header toggles the panel. The open/closed state is stored in localStorage under mlt_deploy_open, so returning users see it exactly how they left it.

Inside is a visual stepper — numbered circles connected by a vertical fading line — walking through ten deployment steps:

1. Open Google Apps Script — with a 🚀 Open Apps Script ↗ button that jumps straight to the new-project screen.
2. Create a new project — click + New project.
3. Paste the code — Ctrl/⌘ + A, delete, paste.
4. Save the project — Ctrl+S / ⌘+S.
5. Start a new deployment — Deploy → New deployment.
6. Choose "Web app" as the type — the ⚙ gear icon.
7. Set permissions — Execute as: Me, Who has access: Anyone. A ⚠️ warning callout emphasizes why "Anyone" is critical.
8. Authorize the script — including the standard "Advanced → Go to project (unsafe) → Allow" flow. An ℹ️ info callout explains the warning.
9. Copy the Web App URL — showing the …/macros/s/…/exec format. Another ℹ️ callout warns against the /dev URL.
10. Verify the deployment — a 🚨 danger callout shows exactly what the ping should return and what to do if it returns anything else.

A final step reminds users to re-deploy with "New version" after any code change.

Step 3 — Paste the Bridge URL

An input field with a Save Script URL button that validates the URL format via regex, strips accidental surrounding quotes, and warns if /dev is present. A 🌐 Test URL in Browser button opens {url}?action=ping in a new tab so the user can see the raw JSON response themselves.

Below that:

· The auto-sync toggle.
· ☁️ Sync to Drive
· 📥 Import from Drive
· 🔓 Clear Script URL

---

Diagnostics Panel

A single ▶ Run Diagnostics button runs a six-stage health check and displays results as colour-coded rows (green pass, amber warn, red fail). Each row can include a raw response block in a dark code box.

The six checks:

1. Bridge URL saved — is anything configured?
2. URL format valid — ends in /exec, not /dev?
3. Bridge is reachable — POST ping; measures latency; shows raw response on failure.
4. Payload round-trip — POST echo with {hello:'world'}; verifies the bridge echoed it.
5. Drive read works — POST load; reports how many locations the remote file contains.
6. Payload size — estimates the URL-encoded size of the current locations; warns if approaching Apps Script's ~1 MB form-encoded limit.

Root-cause hints — when the ping fails, the diagnostic inspects the raw response and prints a specific fix:

Raw response Diagnosis Fix shown
action=ping (plain text) Stale or default deployment Re-deploy with "New version"
Google login HTML "Who has access" isn't "Anyone" Change setting and re-deploy
Any HTML Wrong endpoint Verify the /exec URL
Empty Deployment not live Check the deployment list

The raw response is also surfaced in a monospace box under the failing row, so power users can see exactly what the bridge returned.

---

Status Cards

Below diagnostics, three info cards display live status:

Card Shows
Bridge <span style="color:green">Configured</span> / <span style="color:orange">Syncing</span> / <span style="color:red">Error</span> / Not set
Target Saved address / saved_address.json (fixed text)
Last Sync Relative time — "Just now", "5 min ago", "2 hr ago", or full timestamp

A fourth Last Error card appears only when state.lastError is set. The error text is persisted across reloads, so a user can close the tab and come back to see why the last sync failed.

---

Local Data Tools

· ⬇️ Export Local JSON — downloads a locations_YYYY-MM-DD.json file containing the same payload the app would send to Drive.
· 🗑️ Clear All Local Data — with confirmation, wipes localStorage for locations. The Drive file is untouched.

---

Storage Model

Three keys in localStorage:

Key Contents
mlt_locations_cache JSON array of location objects
mlt_gmail The saved Gmail ID (for reference)
mlt_autosync "true" or "false"
mlt_last_sync ISO timestamp of last successful sync
mlt_script_url The Apps Script /exec URL
mlt_last_error The most recent sync error message
mlt_deploy_open Whether the Step 2 panel is expanded

The Drive file saved_address.json mirrors the same payload shape:

```json
{
  "app": "MyLocationTracker",
  "version": 3,
  "gmail": "user@gmail.com",
  "updatedAt": "2026-01-15T10:30:00.000Z",
  "count": 12,
  "locations": [ ... ]
}
```

---

The Apps Script Bridge (v3.1)

A single function handle(e) is called by both doPost and doGet. It:

1. Prefers e.parameter.action + e.parameter.payload (form-encoded).
2. Falls back to parsing e.postData.contents as JSON.
3. Dispatches to ping, echo, save, or load.
4. Wraps all responses in ContentService.createTextOutput(JSON.stringify(...)).
5. Catches every exception and returns {ok: false, error: "..."}.

The critical fix log at the top of the script notes:

v3.1 — replaced MimeType.JSON (does not exist in Apps Script) with the string 'application/json'

This was the source of the "Argument cannot be null: mimeType" error. Apps Script's MimeType enum only contains document types like CSV, PDF, and TEXT — it has no JSON member, so MimeType.JSON evaluated to undefined and folder.createFile(name, content, undefined) threw.

The corrected line is:

```js
folder.createFile(FILE_NAME, content, 'application/json');
```

The third argument accepts any MIME type string, so 'application/json' works perfectly, and getBlob().getDataAsString() reads the file back identically.

---

Data Flow Summary

```
┌──────────────────┐   save    ┌──────────────────┐   DriveApp   ┌──────────────────┐
│  Geolocation API │ ────────► │  localStorage    │ ───────────► │  Saved address/  │
└──────────────────┘           │                  │              │  saved_address.  │
                               │  (source of      │              │  json            │
       ┌──────────────────────►│   truth offline) │◄─────────────│                  │
       │                       └──────────────────┘   import     └──────────────────┘
       │                                │
       │                                │
       ▼                                ▼
┌──────────────────┐           ┌──────────────────┐
│  Map markers     │           │  Sidebar list    │
│  (Leaflet)       │           │  (cards)         │
└──────────────────┘           └──────────────────┘
```

Everything flows through commit() — a single funnel that saves to localStorage, re-renders the list, refreshes the map markers, updates status, and (if enabled) schedules an auto-push. This keeps state consistent no matter which action triggered the change.

---

Why This Architecture Was Chosen

Problem Solution
Users don't want to create a Google Cloud project Apps Script Web App requires only a browser login
OAuth tokens expire hourly and need refresh logic No token — Apps Script runs as the deployer permanently
Google's /exec endpoint redirects cross-origin Form-encoded POST is a simple request — no preflight
Apps Script can't answer OPTIONS Same as above
Users forget to re-deploy after editing code Built-in verification step and diagnostics
Users don't know where to paste the URL Step 3 is directly below Step 2
Silent sync failures are hard to debug Raw response inspector in diagnostics
Payload might exceed Apps Script limits Size check with warning at ~400 KB

The result is a portable, self-contained location manager that works offline, syncs through a Google account the user already has, and tells the user exactly what's wrong when something breaks.