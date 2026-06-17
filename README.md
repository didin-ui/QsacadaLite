# Q-Scada (Qoalca Scada)

A web-based SCADA (Supervisory Control and Data Acquisition) HMI builder with real-time Modbus TCP data acquisition, role-based authentication, historical trending, reporting, alarm management, and automated data archival.

> **Last updated:** 2026-05-20

---

## Changelog

### 2026-05-20

- **Project renamed** — SCADA2026 is now **Q-Scada** (Qoalca Scada). Docker image tags updated to `scada2026-backend` / `scada2026-frontend` for backward compatibility with existing `prepare-offline.bat` / `load-offline.bat` scripts.
- **Persistent Modbus TCP socket** — the polling engine no longer opens and closes a TCP connection on every read cycle. Each enabled connection now maintains **one persistent socket** that is held open for the lifetime of the poller. This prevents the application from being displaced by concurrent Modbus clients (Node-RED, Modbus Poll, etc.) on devices that allow only one simultaneous connection. If the socket drops (device power-off, network fault), it is cleaned up automatically and re-established on the next poll tick.
- **Unit ID range fixed (0–255)** — corrected a backend bug where unit ID `0` (valid Modbus broadcast address) was silently replaced with `1` due to JavaScript `||` coercion. Backend now validates the full 0–255 range and returns HTTP 400 for values outside it. Frontend form input now shows `(0–255)` hint and enforces `min=0 max=255`.
- **Connection test Modbus exception handling** — the Test Connection button previously failed for devices that respond to register-0 reads with a Modbus exception (e.g. unit ID 150 returning exception code 0x02 IllegalDataAddress or 0x01 IllegalFunction). Any Modbus exception response now correctly resolves as **Connected** — the exception proves the TCP connection and unit ID are valid. The success message now includes the exception code for clarity (e.g. `device responded with Modbus exception (exception code 0x02)`). Previously only `IllegalDataAddress` was handled; all exception codes are now accepted.
- **Persistent socket preserved on Modbus exception** — fixed a secondary bug where a Modbus exception during a regular poll cycle would destroy the persistent socket, forcing an unnecessary reconnect on the next tick. The socket is now only destroyed for genuine TCP failures (timeout, ECONNREFUSED, connection dropped).

### 2026-03-27

- **Offline sensor mitigation** — the Modbus poller now logs `offline` status transitions to `alarm_events` (same transition-tracking logic that prevents duplicates). When the circuit breaker trips (3 consecutive failures), all tags mapped to that connection are immediately broadcast as `offline` via WebSocket and logged to the alarm table.
- **Duplicate / Copy sensor** — in Edit mode, every widget now has an **⧉** button (bottom-left corner) that creates an exact copy of the sensor — same type, Modbus mapping, thresholds, scale factor, and unit — with a new auto-incremented Tag ID (same prefix) and position offset +3%. Live runtime values are reset to `---`/offline so the copy is clean. Works for all widget types: single, combo_th, combo (multi-parameter), and toggle.
- **Offline filter in Alarms page** — new **Offline** tab in the Alarm Management page shows only unacknowledged offline sensor events. The **Active** tab now explicitly excludes offline events (alarm + warning only). Offline rows are styled with a dark-gray background and gray status dot.
- **Alarm stats badge in top nav** — the `🔔 Alarms` navigation tab now shows a live count badge: red for active alarm/warning events, gray for offline events. The count is polled from `/api/alarms/stats` every 15 seconds.
- **Offline sensors indicator on canvas** — in Live mode, a gray pill ("N offline") appears in the top-right of the canvas whenever one or more sensors are reporting `offline` status.
- **Alarm log panel includes offline events** — the in-canvas alarm panel (slide-in from right) now records offline transitions and displays them with gray styling and "OFFLINE" label.
- **GET /api/connections/status endpoint** — new endpoint returns real-time circuit-breaker state (`fails`, `inBackoff`, `backoffRemainingMs`) for all enabled connections.
- **docker-compose.yml image name fix** — added explicit `image: scada2026-backend` and `image: scada2026-frontend` tags to the backend and frontend services so `prepare-offline.bat`'s `docker save` step finds the built images correctly.

### 2026-03-22

- **Edit toolbar moved to left sidebar** — the edit-mode toolbar is now a vertical panel on the left side of the canvas (148 px wide) instead of a horizontal bar at the top. The **+ Sensor** dropdown opens to the right using `position: fixed` so it is never clipped by the canvas.
- **Widget delete confirmation** — clicking the **×** button on any sensor card now shows a confirmation dialog (`"Hapus sensor <ID — Name>?"`) before removing the widget. Applies to all widget types.
- **Toggle button color logic reversed** — ON state = **Red** (danger/active), OFF state = **Green** (safe/idle). Previously ON was green and OFF was dark.
- **Trends & Report tag list fixed** — the `/api/logs/tags` endpoint now returns all tags registered in the layout (single, combo_th, combo, toggle) in addition to tags that already have logged data. Previously only tags with existing `data_log` rows were listed.
- **Backend error logging improved** — Modbus poll failures and DB insert errors now print detailed messages to the Docker log for easier debugging.

---

## Overview

Q-Scada lets you design industrial monitoring and control screens by placing sensor and control widgets on top of a background image (P&ID diagram, machine layout, floor plan), then connecting each widget to a live Modbus TCP register. Data is polled from PLC/slave devices in real time and pushed to all connected browsers via WebSocket.

### Key capabilities

- **Role-based login** — JWT authentication with three roles: admin, supervisor, operator
- **Operator restriction** — operators are locked to Live mode only; Edit mode and layout save are disabled
- Drag-and-drop sensor and control widgets onto a page image
- Four widget types: **Single value**, **Combo T+H** (temperature + humidity), **Custom Multi-Parameter combo** (up to 6 user-defined parameters in one card, each with its own Tag ID), and **ON/OFF Toggle** (Modbus coil write)
- **Duplicate sensor** — one-click copy of any widget preserving full Modbus config, thresholds, and scale; new Tag ID assigned automatically
- Per-sensor Modbus register mapping — connection, register type, address, data type, scale factor, offset
- Per-sensor alarm and warning thresholds with colour-coded status (green / yellow / red / gray for offline)
- **Offline sensor mitigation** — circuit-breaker pattern (3 failures → 30 s backoff); offline transitions logged to `alarm_events`; immediate WebSocket broadcast of offline status to all affected widgets
- **Batch read mode** per connection — groups tags into contiguous block reads (up to 125 registers per request)
- **Sensor Settings** — per-sensor metadata: location, brand, serial number, installation year, sample interval
- **Trends page** — multi-tag time-series chart (recharts); ranges: Last Hour, Day, Week, Month, Year, Custom date/time; CSV export
- **Report page** — raw data table for any tag selection + date range; export as CSV, Excel, or PDF
- **Alarm Management** — list alarm/warning/offline events; filter by Active, Offline, or All History; live badge in top nav; acknowledge individually or all at once
- **Users Management** (admin only) — create, edit, delete users and assign roles
- **Auto-Archival** (admin only) — scheduled CSV export + deletion of old data; retention 30 days to 5 years; partition-aware (instant DROP TABLE)
- Multi-page layout with named tabs and prev/next navigation
- App opens in Live mode by default; Edit mode available to admin and supervisor
- Auto-save to PostgreSQL; localStorage fallback when backend is offline
- Export / import layout as JSON
- Full Docker Compose deployment (PostgreSQL + Node.js backend + nginx-served React frontend)
- **Offline deployment** — `prepare-offline.bat` / `load-offline.bat` for air-gapped sites

---

## Architecture

```
Browser
  |
  |-- HTTP GET /        --> nginx --> React SPA (built static files)
  |-- HTTP /api/*       --> nginx --> Node.js Express  :3001
  |-- WS   /ws          --> nginx --> Node.js WebSocket :3001
                                        |
                                        +-- PostgreSQL :5432
                                        |     layout (JSONB)
                                        |     modbus_connections
                                        |     data_log (partitioned by month)
                                        |     alarm_events (partitioned by month)
                                        |     users
                                        |     archival_config / archival_runs
                                        |
                                        +-- Modbus TCP polling engine
                                        |     jsmodbus (pure Node.js)
                                        |     persistent socket per connection
                                        |     sequential or batch read
                                        |     logs readings to data_log
                                        |     broadcasts live_data via WS
                                        |
                                        +-- Archival scheduler
                                        |     hourly tick, runs at schedule_hour
                                        |     exports CSV, drops old partitions
                                        |
                                        +-- Modbus TCP write endpoint
                                              POST /api/modbus/write
```

### Services (docker-compose.yml)

| Service    | Image / Build            | Port (host)       | Role                               |
|------------|--------------------------|-------------------|------------------------------------|
| `db`       | postgres:16-alpine       | 5433              | Persistent storage                 |
| `backend`  | scada2026-backend        | —                 | REST API + WebSocket + Modbus poll |
| `frontend` | scada2026-frontend       | 80 (configurable) | nginx: SPA + reverse proxy         |

> The database port is exposed on host port **5433** (not 5432) to avoid conflicts with any locally installed PostgreSQL instance.

---

## Tech Stack

| Layer         | Technology                                                    |
|---------------|---------------------------------------------------------------|
| Frontend      | React 19.2, Vite 7.3, recharts 3.8, plain CSS (no framework) |
| Backend       | Node.js, Express 4.19, ws 8.17 (WebSocket), bcrypt 6, jsonwebtoken 9 |
| Modbus        | jsmodbus 4.0.10 — pure JS, no native deps                    |
| Report export | xlsx (Excel), jsPDF + jspdf-autotable (PDF)                  |
| Database      | PostgreSQL 16 — JSONB layout, monthly-partitioned data_log   |
| Proxy / Serve | nginx 1.27-alpine                                             |
| Container     | Docker + Docker Compose v2 (BuildKit)                         |

---

## User Roles

| Role           | SCADA (edit) | SCADA (live) | Sensor Settings | Trends | Report | Alarms | Users | Archival |
|----------------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **admin**      | ✅  | ✅  | ✅  | ✅  | ✅  | ✅  | ✅  | ✅  |
| **supervisor** | ✅  | ✅  | ✅  | ✅  | ✅  | ✅  | —   | —   |
| **operator**   | —   | ✅  | —   | ✅  | ✅  | ✅  | —   | —   |

Default admin credentials (seeded on first run):
- **Username:** `admin`
- **Password:** `Admin123!`

> Change the default password immediately after first login.

---

## Database Schema

```sql
-- Modbus TCP connections
CREATE TABLE modbus_connections (
  id            SERIAL  PRIMARY KEY,
  name          VARCHAR(100) NOT NULL,
  host          VARCHAR(255) NOT NULL,
  port          INTEGER      NOT NULL DEFAULT 502,
  unit_id       INTEGER      NOT NULL DEFAULT 1,
  poll_interval INTEGER      NOT NULL DEFAULT 2000,  -- ms
  enabled       BOOLEAN      NOT NULL DEFAULT true,
  batch_read    BOOLEAN      NOT NULL DEFAULT false,
  max_gap       INTEGER      NOT NULL DEFAULT 10,
  created_at    TIMESTAMP             DEFAULT NOW()
);

-- Full layout as a single JSONB document
CREATE TABLE layout (
  id         INTEGER PRIMARY KEY DEFAULT 1,
  data       JSONB   NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Sensor data log — range-partitioned by month (logged_at)
-- Monthly child tables are auto-created: data_log_YYYY_MM
CREATE TABLE data_log (
  id        BIGSERIAL,
  tag_id    VARCHAR(50)  NOT NULL,
  value     FLOAT,
  status    VARCHAR(20),
  logged_at TIMESTAMP    NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, logged_at)
) PARTITION BY RANGE (logged_at);

-- Alarm events — range-partitioned by month (occurred_at)
-- Monthly child tables are auto-created: alarm_events_YYYY_MM
CREATE TABLE alarm_events (
  id           BIGSERIAL,
  tag_id       VARCHAR(50)  NOT NULL,
  tag_name     VARCHAR(100),
  page_name    VARCHAR(100),
  field        VARCHAR(20),
  value        FLOAT,
  unit         VARCHAR(20),
  status       VARCHAR(20),
  occurred_at  TIMESTAMP    NOT NULL DEFAULT NOW(),
  acknowledged BOOLEAN      NOT NULL DEFAULT false,
  ack_by       VARCHAR(100),
  ack_at       TIMESTAMP,
  PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

-- Users and authentication
CREATE TABLE users (
  id            SERIAL PRIMARY KEY,
  username      VARCHAR(50) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(20) NOT NULL DEFAULT 'operator',  -- admin | supervisor | operator
  full_name     VARCHAR(100),
  created_at    TIMESTAMP DEFAULT NOW()
);

-- Archival configuration (single row)
CREATE TABLE archival_config (
  id              INTEGER PRIMARY KEY DEFAULT 1,
  retention_days  INTEGER   NOT NULL DEFAULT 365,   -- 30–1825 (up to 5 years)
  enabled         BOOLEAN   NOT NULL DEFAULT false,
  schedule_hour   INTEGER   NOT NULL DEFAULT 2,      -- 0–23
  last_run_at     TIMESTAMP,
  last_run_rows   INTEGER   DEFAULT 0,
  last_run_status VARCHAR(20)
);

-- Archival run history
CREATE TABLE archival_runs (
  id            SERIAL PRIMARY KEY,
  started_at    TIMESTAMP DEFAULT NOW(),
  finished_at   TIMESTAMP,
  rows_exported INTEGER   DEFAULT 0,
  rows_deleted  INTEGER   DEFAULT 0,
  file_name     VARCHAR(255),
  file_size     BIGINT    DEFAULT 0,
  status        VARCHAR(20),   -- running | success | error
  error_msg     TEXT
);
```

Tables, columns, and monthly partitions are created/migrated automatically on backend startup — no manual SQL needed. The backend pre-creates partitions for 1 month back + current month + 4 months ahead, and runs a daily check to ensure upcoming partitions exist.

---

## Quick Start

### Option A — Full Docker stack (recommended)

```bash
# 1. Copy the project to the target machine
cd MPscada

# 2. Set credentials in .env
cp .env.example .env      # edit POSTGRES_PASSWORD and FRONTEND_PORT

# 3. Build and start all services
docker compose up --build

# Frontend: http://localhost  (or FRONTEND_PORT)
# Login:    admin / Admin123!
```

> **First run note:** if you change `POSTGRES_PASSWORD` after the volume is already initialised, run `docker compose down -v` first to reset the volume.

### Option B — Development mode (hot reload)

```bash
# Terminal 1 — start only the database
docker compose up db -d

# Terminal 2 — backend with auto-reload
cd scada-backend
cp .env.example .env      # adjust PGPASSWORD to match root .env
npm install
npm run dev               # http://localhost:3001

# Terminal 3 — frontend dev server
cd scada-builder
npm install
npm run dev               # http://localhost:5173
```

#### scada-backend/.env

```env
PGHOST=localhost
PGPORT=5432
PGDATABASE=scada_db
PGUSER=postgres
PGPASSWORD=your_password_here   # must match POSTGRES_PASSWORD in root .env
PORT=3001
JWT_SECRET=qscada_jwt_secret    # optional; defaults to built-in value
ARCHIVE_DIR=./archives          # optional
```

### Option C — Offline / Air-gapped deployment

Run once on a machine with internet:
```bat
prepare-offline.bat
```
This pulls all Docker base images, builds the project, and saves everything to `scada2026-images.tar`.

Copy `scada2026-images.tar` + the project folder to the offline machine, then:
```bat
load-offline.bat
```
This loads the pre-built images and starts all services — no internet required.

---

## REST API

All endpoints except `/api/auth/login` and `/api/health` require a JWT Bearer token:
```
Authorization: Bearer <token>
```

### Authentication

| Method | Path              | Role | Description                    |
|--------|-------------------|------|--------------------------------|
| POST   | /api/auth/login   | —    | Login → returns JWT (8h)       |
| GET    | /api/auth/me      | any  | Current user info              |

```json
POST /api/auth/login
{ "username": "admin", "password": "Admin123!" }

Response: { "token": "eyJ...", "user": { "id": 1, "username": "admin", "role": "admin" } }
```

### Users

| Method | Path           | Role  | Description              |
|--------|----------------|-------|--------------------------|
| GET    | /api/users     | admin | List all users           |
| POST   | /api/users     | admin | Create user              |
| PATCH  | /api/users/:id | admin | Update user              |
| DELETE | /api/users/:id | admin | Delete user              |

### Layout

| Method | Path        | Role               | Description       |
|--------|-------------|-------------------|-------------------|
| GET    | /api/layout | any               | Load full layout  |
| PUT    | /api/layout | admin, supervisor | Save full layout  |

### Modbus Connections

| Method | Path                      | Role               | Description                               |
|--------|---------------------------|--------------------|-------------------------------------------|
| GET    | /api/connections          | any                | List connections                          |
| GET    | /api/connections/status   | any                | Real-time circuit-breaker state per conn  |
| POST   | /api/connections          | admin, supervisor  | Create connection                         |
| PATCH  | /api/connections/:id      | admin, supervisor  | Update connection                         |
| DELETE | /api/connections/:id      | admin, supervisor  | Delete connection                         |
| POST   | /api/connections/:id/test | admin, supervisor  | Test connectivity                         |
| POST   | /api/modbus/write         | admin, supervisor  | Write coil / register                     |

`GET /api/connections/status` response:
```json
[
  { "id": 1, "name": "PLC-01", "host": "192.168.1.100", "port": 502,
    "fails": 0, "inBackoff": false, "backoffRemainingMs": 0 },
  { "id": 2, "name": "PLC-02", "host": "192.168.1.101", "port": 502,
    "fails": 3, "inBackoff": true,  "backoffRemainingMs": 18500 }
]
```

### Data Logs

| Method | Path             | Role | Description                                    |
|--------|------------------|------|------------------------------------------------|
| GET    | /api/logs        | any  | Time-series data (bucketed) for Trends chart   |
| GET    | /api/logs/tags   | any  | List all known tag IDs                         |
| GET    | /api/logs/report | any  | Raw data for Report page (multi-tag, max 10K)  |

Query params for `/api/logs`:
- `tagId` — tag ID to query
- `range` — `hour` | `day` | `week` | `month` | `year` | `custom`
- `from`, `to` — ISO datetime strings (required when `range=custom`)

Query params for `/api/logs/report`:
- `tags` — comma-separated tag IDs
- `from`, `to` — ISO datetime strings

### Alarms

| Method | Path                        | Role | Description                           |
|--------|-----------------------------|------|---------------------------------------|
| GET    | /api/alarms/stats           | any  | Count unacknowledged events by status |
| GET    | /api/alarms                 | any  | List alarm events                     |
| POST   | /api/alarms/:id/acknowledge | any  | Acknowledge an alarm                  |

Query params for `/api/alarms`:
- `status` — `active` (unacknowledged alarm + warning only) | `offline` (unacknowledged offline events) | `alarm` | `warning` | `all` (full history)
- `limit` — max rows to return (default 200, max 1000)

`GET /api/alarms/stats` response:
```json
{ "alarm": 2, "warning": 1, "offline": 3 }
```

### Archival

| Method | Path                        | Role  | Description                    |
|--------|-----------------------------|-------|--------------------------------|
| GET    | /api/archival/config        | admin | Get archival configuration     |
| PUT    | /api/archival/config        | admin | Update configuration           |
| POST   | /api/archival/run           | admin | Trigger manual archival run    |
| GET    | /api/archival/runs          | admin | List archival run history      |
| GET    | /api/archival/files         | admin | List archive CSV files         |
| GET    | /api/archival/files/:name   | admin | Download a CSV archive file    |
| DELETE | /api/archival/files/:name   | admin | Delete a CSV archive file      |

### System

| Method | Path        | Role | Description                    |
|--------|-------------|------|--------------------------------|
| GET    | /api/health | —    | Health check — DB connectivity |

### WebSocket messages (server → browser)

```json
{ "type": "live_data",
  "connectionId": 1,
  "data": [
    { "tagId": "TT-01", "value": "24.3",  "status": "normal" },
    { "tagId": "TH-01", "value": "22.1",  "status": "normal", "field": "temp" },
    { "tagId": "TH-01", "value": "65",    "status": "normal", "field": "hum"  },
    { "tagId": "CM-01", "value": "1.23",  "status": "normal", "field": "0"    },
    { "tagId": "CM-01", "value": "4.56",  "status": "warning","field": "1"    },
    { "tagId": "SW-01", "value": "1",     "status": "normal", "field": "coil" }
  ]
}
{ "type": "connection_status", "connectionId": 1, "status": "connected" }
{ "type": "connection_status", "connectionId": 1, "status": "error", "message": "ECONNREFUSED" }
```

---

## How to Use

### 1. Login
Open the app URL in a browser. Enter username and password. The JWT token is stored in `localStorage` and expires after 8 hours. After expiry, the login page appears again automatically.

### 2. SCADA canvas (admin / supervisor)
After login, the app opens in **Live mode** by default. Switch to **Edit** mode to design the canvas.

In **Edit mode**, a vertical toolbar appears on the **left side** of the canvas containing all editing controls. The toolbar is 148 px wide and does not overlap the canvas.

#### Upload a background image
Click **📷 Image** in the left toolbar and select a P&ID diagram, machine photo, or floor plan. The image is compressed to JPEG 70% quality (max 1600 px wide) and stored in PostgreSQL. **Recommended size: 1600 × 900 px (16:9).**

#### Add widgets
Click **+ Sensor** in the left toolbar and choose a preset (dropdown opens to the right):

| Preset              | Tag prefix | Type     | Description                                              |
|---------------------|------------|----------|----------------------------------------------------------|
| Multi-Parameter     | CM-        | combo    | Up to 6 custom parameters in one card — `MULTI` badge    |
| Temp & Humidity     | TH-        | combo_th | Temperature + humidity in one card                       |
| Temperature         | TT-        | single   | Single value display                                     |
| Pressure            | PT-        | single   | Single value display                                     |
| Flow Rate           | FT-        | single   | Single value display                                     |
| Humidity            | HT-        | single   | Single value display                                     |
| Level               | LT-        | single   | Single value display                                     |
| Power               | WT-        | single   | Single value display                                     |
| Switch              | SW-        | toggle   | ON/OFF button — writes Modbus coil; ON = Red, OFF = Green |

- **Drag** a widget to reposition it
- **Drag the bottom-right corner handle** to resize
- **Double-click** a widget (Edit mode) to edit Tag ID and display name inline
- **Click ×** (top-right) to delete a widget — a confirmation dialog appears before removal
- **Click ⧉** (bottom-left, blue) to **duplicate** the widget — creates an exact copy with a new auto-incremented Tag ID (same prefix), all Modbus mappings and thresholds preserved, position offset by +3%

Each widget in Edit mode has four corner controls:

| Corner       | Button | Action                        |
|--------------|--------|-------------------------------|
| Top-left     | ⚙      | Open Modbus register mapping  |
| Top-right    | ×      | Delete widget (with confirm)  |
| Bottom-left  | ⧉      | Duplicate widget              |
| Bottom-right | ◢      | Resize widget                 |

#### Configure Modbus TCP connections
Click **⚙ Modbus** → **+ Add Connection**:

| Field      | Description                                                            |
|------------|------------------------------------------------------------------------|
| Name       | Friendly label (e.g. "Boiler PLC")                                     |
| Host / IP  | IP address of the Modbus TCP slave                                     |
| Port       | TCP port (default 502; range 1–65535)                                  |
| Unit ID    | Modbus unit/slave ID — **0–255** (Modbus TCP MBAP 1-byte field; default 1) |
| Poll (ms)  | Polling interval in milliseconds (default 2000; min 100)               |
| Enabled    | Pause polling without deleting the connection                          |
| Batch Read | Group tags into block reads                                            |
| Max Gap    | Max address gap (registers) to merge into one batch                    |

#### Map a widget to a Modbus register
Click the **⚙ gear badge** on a widget (Edit mode):

**Single / combo_th / toggle widgets:**
- **SENSOR INFO** — display name, unit, sensor metadata
- **VALUE REGISTER** — connection, register type (holding/input/coil/discrete), address, data type, scale factor, offset, decimals
- **HUMIDITY REGISTER** (combo_th only) — second register for humidity; includes **Humidity Tag ID** field (default `<id>-HUM`) used for logging and Trends
- **WRITE COIL** (toggle only) — connection + coil address for write
- **COIL READBACK** (toggle, optional) — address to sync ON/OFF state from PLC
- **ALARM THRESHOLDS** — alarm high/low, warning high/low

**Multi-Parameter combo widget (COMBO CONFIG modal):**
- **Card Title** — displayed at top of the combo card
- **Parameters** (up to 6) — click **+ Add Parameter** to add more; each has:
  - **Tag ID** — unique identifier for logging and Trends (auto-uppercased, highlighted blue)
  - **Label** — parameter display name
  - **Unit** — e.g. °C, bar, m³/h
  - **Connection**, **Register Type**, **Address**, **Data Type**
  - **Scale Factor**, **Offset**, **Decimals**
  - **Alarm High/Low**, **Warning High/Low** thresholds
- Click **✕** beside a parameter to remove it

### 3. Sensor Settings (admin / supervisor)
Navigate to **⚙ Sensor Settings** to set per-sensor metadata:
- Location, Brand, Serial Number, Installation Year
- **Sample Interval** (seconds) — controls how frequently the poller logs this tag's value to `data_log`

The settings table expands composite widgets into individual rows:
- **combo (Multi-Parameter)** — one row per parameter showing its own Tag ID (e.g. `CM-01_0`, `CM-01_1`) and label; all parameters share the card's Location/Brand/Serial/InstallYear/SampleInterval
- **combo_th** — two rows: one for Temperature (Tag ID = label `id`) and one for Humidity (Tag ID = `humTagId` or `<id>-HUM`)

### 4. Trends
Navigate to **📈 Trends** to view historical data as a line chart:
- Select one or more tags using the dropdown (multi-select with colored pills)
- Choose a time range: **Last Hour**, **Day**, **Week**, **Month**, **Year**, or **Custom** (date/time from–to)
- Each selected tag gets a unique colored line
- **Export CSV** downloads the current chart data

### 5. Report
Navigate to **📊 Report** to query raw logged data:
- Select tags and a from/to date range, then click **Generate**
- Export as **CSV**, **Excel (.xlsx)**, or **PDF**
- Raw data table shows up to 10,000 rows per query

### 6. Alarms
Navigate to **🔔 Alarms** to review alarm events:
- The `🔔 Alarms` tab in the top navigation shows a live count badge — **red** for active alarm/warning events, **gray** for unacknowledged offline events
- Filter by **Active** (unacknowledged alarm/warning), **Offline** (unacknowledged sensor-offline events), or **All History**
- Offline events are displayed with a dark-gray background and gray status dot
- Click **Acknowledge** on individual events to mark them as handled (also resets the circuit breaker so the poller resumes immediately)
- Each event shows tag ID, name, page, field, value, status, timestamp, and who acknowledged it

### 7. Users (admin only)
Navigate to **👥 Users** to manage user accounts:
- Create users with username, full name, password, and role
- Edit existing users (change role, reset password)
- Delete users (cannot delete your own account or the last admin)

### 8. Archival (admin only)
Navigate to **🗄 Archival** to manage data retention:
- **Retention** — slider from 30 days to 5 years (driven by disk space, not performance)
- **Auto-run hour** — hour of day (0–23) for the daily scheduled run
- **Enable/Disable** auto-archival
- **Run Now** — trigger an immediate archival run
- **Archive Files** — list, download, or delete exported CSV files
- **Run History** — table of past archival runs with row counts, file sizes, and status

### 9. Multiple pages
Click **+ Page** to add a new page. Navigate with the tab bar or prev/next arrows. **Double-click** a tab to rename it. Click **× Page** to delete the current page (requires 2+ pages).

### 10. Export / Import layout
- **Export** saves the entire layout (all pages, widget positions, Modbus mappings) as `scada_layout.json`
- **Import** loads a previously exported file

---

## Persistent Modbus TCP Connection

By default, many Modbus TCP client libraries open a new TCP connection for each read request and close it immediately after. This causes Q-Scada to lose its connection slot on devices that only allow **one simultaneous client** (common on low-cost PLCs, energy meters, and sensor gateways), making it unable to read while tools like Node-RED or Modbus Poll are running.

Q-Scada solves this by holding **one persistent TCP socket per connection** open for the entire lifetime of the poller.

### How it works

```
Startup
  └─ scheduleConnection()
       └─ getOrCreateSocket()  →  TCP connect  →  socket OPEN (held)

Poll tick 1  →  reuse socket  →  read registers  →  socket stays open
Poll tick 2  →  reuse socket  →  read registers  →  socket stays open
Poll tick N  →  reuse socket  →  read registers  →  socket stays open

Device power-off / network fault
  └─ socket 'close' event  →  entry removed from pool
  └─ next poll tick  →  getOrCreateSocket()  →  reconnect automatically

Connection disabled / deleted in UI
  └─ unscheduleConnection()  →  destroyPersistentSocket()  →  socket closed
```

### Connection slot ownership

| Scenario | Result |
|----------|--------|
| Q-Scada starts first, then Node-RED connects | Device rejects Node-RED — Q-Scada holds the slot |
| Node-RED starts first, Q-Scada starts later | Q-Scada connects once Node-RED releases (or device allows 2nd client) |
| Device supports multiple clients (e.g. 4 slots) | All clients coexist without conflict |

### Socket lifecycle states

| State | Meaning |
|-------|---------|
| `connecting` | TCP handshake in progress — concurrent poll ticks share the same connect Promise |
| `ready` | Socket is open and usable — reads go directly without reconnecting |
| *(not in pool)* | Socket closed or never created — next `getOrCreateSocket()` call reconnects |

### Reconnect behaviour

There is no explicit reconnect timer. Reconnection happens naturally: whenever a poll tick calls `modbusRead()` and the socket is gone (closed by remote, network error, or read timeout), `getOrCreateSocket()` creates a new socket. The circuit-breaker still applies — after 3 consecutive failures the poller backs off for 30 seconds before retrying.

---

## Batch Read Mode

When **Batch Read** is enabled on a connection, the poller groups all tags into contiguous address ranges and reads them in a single Modbus request.

```
Sequential (default):
  TT-01 addr 100 → open TCP → read 2 reg → close   ┐
  PT-01 addr 102 → open TCP → read 2 reg → close   ├ 3 TCP connections
  FT-01 addr 104 → open TCP → read 2 reg → close   ┘

Batch (Max Gap ≥ 4):
  open TCP → read addr 100–105 (6 registers) → close → split to 3 tags
  = 1 TCP connection, 1 round-trip
```

| Mode       | 10 tags on LAN | TCP connections |
|------------|----------------|-----------------|
| Sequential | ~800 ms        | 10              |
| Batch      | ~80 ms         | 1–3             |

---

## Modbus Data Types

All multi-register types use **Big-Endian (AB CD)** byte order — the most common default for industrial PLCs.

| Data type   | Registers | Bytes   | Range                                |
|-------------|-----------|---------|--------------------------------------|
| `int16`     | 1         | 2       | −32,768 … 32,767                     |
| `uint16`    | 1         | 2       | 0 … 65,535                           |
| `int32`     | 2         | 4       | −2,147,483,648 … 2,147,483,647       |
| `uint32`    | 2         | 4       | 0 … 4,294,967,295                    |
| `float32`   | 2         | 4       | IEEE 754, ±3.4 × 10³⁸                |

Display formula: `displayed = raw × scaleFactor + offsetVal`, rounded to `decimals` places.

---

## Project Structure

```
MPscada/
├── docker-compose.yml           # Orchestrates db + backend + frontend + archives volume
├── .env                         # Credentials and port (git-ignored)
├── .env.example                 # Configuration template
├── prepare-offline.bat          # Pull images + build + save tar (run once, online)
├── load-offline.bat             # Load tar + docker compose up (offline machine)
├── SAT-checklist.md             # Site Acceptance Test checklist (125 test items)
│
├── scada-backend/               # Node.js backend
│   ├── server.js                # Express, WebSocket, DB init, partition manager, archiver
│   ├── db.js                    # PostgreSQL connection pool (pg)
│   ├── middleware/
│   │   └── auth.js              # JWT verify + role check middleware
│   ├── routes/
│   │   ├── auth.js              # POST /login, GET /me
│   │   ├── users.js             # CRUD users (admin only)
│   │   ├── layout.js            # GET/PUT layout JSONB
│   │   ├── connections.js       # CRUD + test + status Modbus TCP connections
│   │   ├── write.js             # POST /api/modbus/write
│   │   ├── logs.js              # GET /logs (trends), GET /logs/tags, GET /logs/report
│   │   ├── alarms.js            # GET /alarms/stats, GET /alarms, POST /alarms/:id/acknowledge
│   │   └── archival.js          # Config, manual run, file list/download/delete
│   ├── modbus/
│   │   └── poller.js            # Polling engine: sequential + batch read, data logging, WS broadcast
│   ├── archival/
│   │   └── archiver.js          # Partition-aware archiver: CSV export + DROP TABLE
│   ├── archives/                # CSV archive files (bind-mounted Docker volume)
│   ├── package.json
│   └── Dockerfile               # Multi-stage: deps (BuildKit npm cache) + runtime
│
└── scada-builder/               # React frontend (single-file SPA)
    ├── src/
    │   ├── App.jsx              # All UI: login, SCADA canvas, trends, report, alarms,
    │   │                        #         users, archival, sensor settings
    │   ├── main.jsx             # React entry point
    │   ├── index.css            # Theme variables (dark/light) + base styles
    │   └── App.css              # Component-specific styles
    ├── public/                  # Static assets
    ├── vite.config.js           # Dev proxy + manualChunks (vendor/charts)
    ├── nginx.conf               # SPA routing + reverse proxy to backend
    ├── package.json
    └── Dockerfile               # Multi-stage: Vite build → nginx serve
```

---

## Widget Label Fields (stored in layout JSONB)

```js
// Single sensor widget
{
  id:              "TT-01",       // Tag ID (unique per page)
  type:            "single",
  name:            "Temperature",
  unit:            "°C",
  x: 0.45, y: 0.30,              // position (0–1 fraction of page size)
  scale: 1.2,                     // widget display scale
  // Modbus read
  connectionId:    1,
  registerType:    "holding",     // holding | input | coil | discrete
  registerAddress: 100,           // 0-based
  dataType:        "float32",     // float32 | int16 | uint16 | int32 | uint32
  scaleFactor:     0.1,           // displayed = raw × scaleFactor + offsetVal
  offsetVal:       0,
  decimals:        1,
  // Sensor metadata
  location:        "Boiler Room",
  brand:           "Yokogawa",
  serialNumber:    "YG-001",
  installYear:     2026,
  sampleInterval:  15,            // seconds between data_log entries
  // Thresholds
  alarmHigh: 80, alarmLow: null, warningHigh: 70, warningLow: null
}

// Combo T+H widget — temperature + humidity, each with its own Tag ID
{
  ...singleFields,
  type:                "combo_th",
  humTagId:            "TH-01-HUM",  // Tag ID for humidity logging (default: <id>-HUM)
  humConnectionId:     1,
  humRegisterType:     "holding",
  humRegisterAddress:  101,
  humDataType:         "int16"
}

// Multi-Parameter combo widget — up to 6 parameters in one card
{
  id:    "CM-01",
  type:  "combo",
  name:  "Compressor Unit 1",        // card title
  x: 0.3, y: 0.4, scale: 1.2,
  // Sensor metadata (shared across all params)
  location: "Machine Room", brand: "", serialNumber: "", installYear: null,
  sampleInterval: 15,
  params: [
    {
      tagId:           "CM-01-TEMP",  // unique Tag ID for this parameter
      label:           "Temperature",
      unit:            "°C",
      connectionId:    1,
      registerType:    "holding",
      registerAddress: 200,
      dataType:        "float32",
      scaleFactor:     0.1,
      offsetVal:       0,
      decimals:        1,
      alarmHigh: 90, alarmLow: null, warningHigh: 80, warningLow: null,
      value: "---", status: "offline"
    },
    {
      tagId:           "CM-01-PRES",
      label:           "Pressure",
      unit:            "bar",
      connectionId:    1,
      registerType:    "holding",
      registerAddress: 202,
      dataType:        "float32",
      scaleFactor:     0.01,
      offsetVal:       0,
      decimals:        2,
      alarmHigh: 10, alarmLow: null, warningHigh: 8, warningLow: null,
      value: "---", status: "offline"
    }
    // ... up to 6 params total
  ]
}

// ON/OFF Toggle widget
{
  id:    "SW-01",
  type:  "toggle",
  name:  "Pump Start",
  x: 0.5, y: 0.5, scale: 1,
  coilState: false,
  writeConnectionId: 1,
  writeAddress:      0,
  readConnectionId:  1,           // optional readback
  readRegisterType:  "coil",
  readAddress:       0
}
```

---

## Environment Variables

| Variable            | Default        | Description                                |
|---------------------|----------------|--------------------------------------------|
| `POSTGRES_DB`       | scada_db       | Database name                              |
| `POSTGRES_USER`     | postgres       | Database user                              |
| `POSTGRES_PASSWORD` | scada1234      | Database password (change in production!)  |
| `FRONTEND_PORT`     | 80             | Host port for the frontend                 |
| `PORT`              | 3001           | Backend HTTP/WS port                       |
| `JWT_SECRET`        | —              | JWT signing secret (set in production!)    |
| `ARCHIVE_DIR`       | ./archives     | Directory for exported CSV archive files   |

---

## Polling Capacity & Limitations

### Sequential Mode (Batch Read OFF)

| Network condition   | Time per tag | Safe tags at 2 s interval |
|---------------------|--------------|---------------------------|
| LAN (ideal)         | ~10 ms       | ~200 (theoretical)        |
| LAN (real PLC)      | ~30–50 ms    | **10–20 recommended**     |
| WAN / slow PLC      | ~100–200 ms  | 5–10                      |

### Batch Mode (Batch Read ON)

| Data type           | Registers/tag | Tags per batch (125-reg limit) |
|---------------------|---------------|-------------------------------|
| `uint16` / `int16`  | 1             | **125 tags**                  |
| `float32` / `int32` | 2             | **62 tags**                   |

### Multi-Connection Scaling

| Connections | Mode       | Estimated total tags |
|-------------|------------|----------------------|
| 1           | Sequential | 10–20                |
| 1           | Batch      | 62–125               |
| 5           | Batch      | 310–625              |
| 10          | Batch      | 620–1,250+           |

### Recommendations

| Scenario                   | Recommended config                            |
|----------------------------|-----------------------------------------------|
| < 20 tags, one PLC         | Sequential mode, 2000 ms interval             |
| 20–125 tags, one PLC       | Batch mode ON, contiguous register addresses  |
| > 125 tags, one PLC        | Batch mode ON, multiple connections per PLC   |
| Multiple PLCs              | One connection per PLC, all in batch mode     |
| Fast data logging (< 1 s)  | Reduce poll interval to 500 ms (batch mode)   |

---

## Database Capacity Analysis

### Scenario Reference: 26 Sensors × 4 Parameters @ 15-Second Interval

- **26 sensors**, each with **4 float32 parameters**
- **Logging interval: 15 seconds**
- **Total tags: 104**

### Row Generation Rate

| Metric              | Calculation | Result           |
|---------------------|-------------|------------------|
| Rows per minute     | 104 × 4     | **416 rows/min** |
| Rows per hour       | 416 × 60    | **~25,000**      |
| Rows per day        | 25K × 24    | **~600,000**     |
| Rows per month      | 600K × 30   | **~18,000,000**  |

### Performance with Table Partitioning

`data_log` and `alarm_events` are **range-partitioned by month**. PostgreSQL prunes queries to scan only the relevant month's partition — so query performance is fast regardless of total row count.

| Total data | Query performance | Limiting factor           |
|------------|-------------------|---------------------------|
| 1 month    | Sub-second        | Only scans 1 partition    |
| 1 year     | Sub-second        | Only scans 1 partition    |
| 5 years    | Sub-second        | Only scans 1 partition    |

> **Without partitioning** the limit was ~50M rows (~2.7 months) before queries degraded.
> **With partitioning**, years of data can be retained with zero performance impact.

### Disk Space Estimate

Each `data_log` row ≈ 120 bytes on disk with index overhead:

| Period  | Rows          | Size    | 500 GB disk |
|---------|---------------|---------|-------------|
| 1 month | 18,000,000    | ~2.2 GB | 0.4%        |
| 1 year  | 216,000,000   | ~26 GB  | 5%          |
| 3 years | 648,000,000   | ~78 GB  | 16%         |
| 5 years | 1,080,000,000 | ~130 GB | 26%         |

### Mitigation Strategies

#### 1. Auto-Archival (built-in)
Configure via **🗄 Archival** (admin only). The archiver:
1. Finds full expired monthly partitions → exports to CSV → `DROP TABLE` (instant, no VACUUM)
2. Handles partial months → chunked `SELECT` → CSV → `DELETE`

Recommended retention by scenario:

| Scenario                      | Disk/year | Recommended retention |
|-------------------------------|-----------|-----------------------|
| 26 sensors × 4 param @ 15 s  | ~26 GB    | 1–3 years             |
| 26 sensors × 4 param @ 30 s  | ~13 GB    | 2–5 years             |
| 10 sensors × 2 param @ 30 s  | ~2.5 GB   | 5 years               |
| 5 sensors × 2 param @ 60 s   | ~0.6 GB   | 5 years               |

#### 2. Increase logging interval
Configure per-sensor in **⚙ Sensor Settings** → Sample Interval:

| Interval | Rows/day (104 tags) | Disk/year |
|----------|---------------------|-----------|
| 5 s      | 1,800,000           | ~78 GB    |
| 15 s     | 600,000             | ~26 GB    |
| 30 s     | 300,000             | ~13 GB    |
| 60 s     | 150,000             | ~6.5 GB   |
| 300 s    | 30,000              | ~1.3 GB   |

#### 3. Partition management queries

```sql
-- List all data_log partitions with sizes
SELECT
  c.relname                                      AS partition,
  pg_size_pretty(pg_total_relation_size(c.oid))  AS size,
  to_char(s.n_live_tup, 'FM999,999,999')         AS live_rows
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_stat_user_tables s ON s.relname = c.relname
WHERE n.nspname = 'public'
  AND c.relname ~ '^data_log_\d{4}_\d{2}$'
ORDER BY c.relname;

-- Total size across all partitions
SELECT pg_size_pretty(pg_total_relation_size('data_log'));

-- Drop an old partition instantly (export via Archival page first)
DROP TABLE data_log_2026_01;
```

### Recommended Maintenance Schedule

| Frequency          | Action                                                                  |
|--------------------|-------------------------------------------------------------------------|
| **Daily**          | Auto-archival runs at configured hour (if enabled)                      |
| **Monthly**        | Check disk: `SELECT pg_size_pretty(pg_total_relation_size('data_log'))` |
| **Quarterly**      | Download CSV archives for off-site backup                               |
| **Yearly**         | Review retention period vs disk space                                   |
| **Before upgrade** | Full `pg_dump` backup                                                   |

---

## Notes

- **No native dependencies** — `jsmodbus` is pure JavaScript; works on Windows, Linux, macOS without build tools
- **Persistent TCP socket** — one socket is held open per Modbus connection; no connect/disconnect overhead per poll. If the device closes the socket, it is automatically re-established on the next poll tick
- **Connection slot priority** — because Q-Scada connects at startup and never releases, it takes the first available slot on single-client devices. Tools like Node-RED or Modbus Poll connecting later will be rejected by the device until Q-Scada stops
- **Unit ID range: 0–255** — the Modbus TCP MBAP header defines the Unit Identifier as a **1-byte field (0–255)**. Unit ID `0` is the valid broadcast address. Values above 255 are outside the Modbus TCP specification and are rejected with HTTP 400. If a slave device does not respond to a unit ID within this range (e.g. ignores IDs > 127), that is a device-side restriction — not a Q-Scada limit
- **Default Live mode** — the app opens in Live mode for all roles; Edit mode is accessible to admin and supervisor via the mode toggle
- **Operator access** — operators can view SCADA (Live only), Trends, Report, and Alarms; they cannot access Edit mode, Sensor Settings, Users, or Archival
- **Offline mode** — if the backend is unreachable, the frontend loads from and saves to `localStorage` automatically
- **Large images** — background images are compressed to JPEG 70% quality, max 1600 px wide; recommended aspect ratio 16:9 (1600 × 900 px)
- **Register addressing** — 0-based (Modbus register 40001 = holding address 0)
- **Float32 byte order** — Big-Endian (AB CD) — register[0] = high word, register[1] = low word
- **JWT expiry** — tokens expire after 8 hours; the login page is shown automatically on expiry
- **Password volume reset** — if `POSTGRES_PASSWORD` changes after the Docker volume exists, run `docker compose down -v` first
- **Tag discovery** — `/api/logs/tags` returns all tags from the layout JSONB (regardless of whether data has been logged yet) merged with distinct tags already in `data_log`
- **Backend rebuild required** — the backend runs from a Docker image (not a volume mount). After any change to `scada-backend/` files, rebuild before restarting:
  ```bash
  docker compose build backend && docker compose up -d backend
  ```
- **Toggle color convention** — ON state is shown in **Red** (active/running), OFF state in **Green** (safe/stopped), following industrial safety color standards
- **Database port** — PostgreSQL is exposed on host port `5433` (not `5432`) to avoid conflicts with any local PostgreSQL installation
