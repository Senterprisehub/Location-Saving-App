Detailed Working Description

Location Tracker is a complete, self‑contained web application delivered as a single index.html file. It lets you track your current position on a Google‑tiled map, save named locations locally, and back everything up to your Google Drive as a single JSON file — without any OAuth client ID or Google Identity Services. Instead, Google Drive access is handled by a Google Apps Script Web App that you deploy once in your own account.

---

Architecture Overview

```
┌──────────────────┐        ┌─────────────────────┐        ┌─────────────────┐
│  index.html      │  POST  │  Apps Script Web App│ DriveApp│  Google Drive   │
│  (browser)       │ ─────► │  (runs as "you")    │ ──────► │ Saved address/  │
│  localStorage    │ ◄───── │  JSON response      │ ◄────── │ saved_address.  │
│  Leaflet Map     │        │                     │         │ json            │
└──────────────────┘        └─────────────────────┘        └─────────────────┘
```

· Client side: Leaflet renders Google map tiles, localStorage caches locations, and a simple fetch() talks to the Apps Script.
· Bridge side: Code.gs receives a { action, payload } JSON body and reads/writes a file called saved_address.json inside a Drive folder named Saved address.
· No credentials in the browser: the Apps Script runs under your Google account, so the browser never sees a token, client ID, or OAuth consent screen.

---

Core Features

1. Track & Save Locations

· Click 📍 Get My Location → the browser's Geolocation API returns coordinates.
· A draggable marker appears on the map; drag it to fine‑tune.
· Click 💾 Save This Location → enter a title → the record { id, title, lat, lng, createdAt, updatedAt } is pushed into localStorage.

2. Manage Saved Locations

· Each entry appears as a card in the Locations tab with title and coordinates.
· Click a card → the map flies to that marker and opens its popup (with a link to Google Maps).
· ✏️ Rename and 🗑️ Delete buttons appear on hover.
· Any change marks the app as "dirty" and (if auto‑sync is on) triggers a debounced push to Drive after 1.2 seconds.

3. Two‑Tab Layout

· 📍 Locations — tracking controls + scrollable list of saved spots.
· ⚙️ Settings — Gmail ID, Apps Script setup guide, Bridge URL, sync/import buttons, local data tools.

4. Google Drive Sync & Import

· ☁️ Sync to Drive → POSTs { action: 'save', payload } to the bridge. The Apps Script finds or creates the Saved address folder, then either updates saved_address.json or creates it if missing.
· 📥 Import from Drive → POSTs { action: 'load' } and merges the returned file into local data by id, keeping whichever record has the newer updatedAt. This makes it safe to run repeatedly.
· The JSON payload includes app, version, gmail, updatedAt, count, and the full locations array.

5. Apps Script Bridge (no OAuth)

· A single Code.gs script handles three actions: ping (health check), save, and load.
· The bridge uses DriveApp.getFoldersByName / createFolder and folder.createFile / file.setContent / getBlob().getDataAsString() — nothing else.
· Requests are sent as simple POST with a JSON string body (no custom headers), which avoids CORS preflight and works from any origin.

6. Guided Setup Inside the App

· The Settings tab contains a complete, copy‑ready Apps Script inside a dark IDE‑style code block with a 📋 Copy code button.
· A 9‑step visual stepper walks through deployment:
  1. Open script.google.com
  2. Create a new project
  3. Paste the code
  4. Save the project
  5. Click Deploy → New deployment
  6. Choose Web app as the type
  7. Set Execute as: Me and Who has access: Anyone
  8. Authorize the script (with an info callout explaining the "unsafe" warning)
  9. Copy the /exec Web App URL
· The URL is pasted into the Bridge URL field and saved locally. A 🔌 Test Connection button verifies the bridge is reachable.

7. Auto‑Sync & Status

· A toggle enables automatic pushing of every change after a 1.2‑second debounce.
· A status bar at the top shows connection state with a coloured dot:
  · Green — all changes saved
  · Amber — changes pending sync
  · Indigo (pulsing) — syncing now
  · Grey — bridge URL not configured
· The Last Sync timestamp updates in the Status card after every successful sync.

8. Local Data Tools

· ⬇️ Export Local JSON downloads the current payload as locations_YYYY‑MM‑DD.json.
· 🗑️ Clear All Local Data wipes localStorage — the Drive copy stays intact.
· 🔓 Clear Script URL removes the saved bridge URL without touching your data.

9. Modern UI

· Inter font, design tokens as CSS variables, and automatic dark mode via prefers-color-scheme.
· Gradient primary buttons with a glow‑on‑hover effect, animated tab underline, hover‑lift location cards, custom toggle switches, toast notifications, and an SVG empty state.

---

Data Flow Summary

```
[Geolocation] → [Draggable Marker] → [Save with Title]
      ↓
[localStorage] ←→ [Locations List & Map Markers]
      ↓ (Sync)              ↑ (Import)
[Apps Script Bridge] → [Drive: "Saved address/saved_address.json"]
```

All location data lives in localStorage for instant offline access. Google Drive acts as the durable cross‑device backup, and the Import button reconstructs the local list from that backup — de‑duplicating by id and keeping the newest version of each record.

---

Key Advantages Over the OAuth Version

Aspect Old (OAuth) New (Apps Script)
Setup Google Cloud project + OAuth client ID + consent screen One Apps Script deployment
Credentials in browser Yes (bearer token) None
Cross‑origin issues Frequent CORS / redirect problems None (simple POST)
Token expiration Requires re‑auth every hour Not applicable
Permissions scope drive.file per user Runs as the deployer
Works from file:// No Yes

The result is a portable, zero‑dependency location manager that works from a single HTML file, an HTTPS server, or even an Electron wrapper — with your own Google Drive as the storage backend.