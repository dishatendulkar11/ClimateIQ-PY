[README.md](https://github.com/user-attachments/files/34305278/README.md)

# Climate Intelligence & Heatwave Early Warning System for India — Python (Flask) version

This is the **same website** as the original DBMS project: same pages, colours, layout, map, charts and features. The backend is now written in **Python**, using **Flask** and an **SQLite** database.

* Live weather, a 7-day forecast and history for all **28 States + 8 Union Territories** (from Open-Meteo)
* A transparent **heat-risk score (0–100)**, calculated in Python
* An interactive India map (MapLibre), charts (Chart.js), safety tips and emergency numbers
* Accounts, favourite states and search history, stored in SQLite

Local address once the server is running: **http://127.0.0.1:8000/**

> `127.0.0.1` means "this computer". The site works only while `app.py` is running on **your** laptop. It is not a public website link.

---

## 1. Quick start on Windows (easiest)

1. **Install Python** (skip this if you already have it).
   * Download Python 3.10 or newer from <https://www.python.org/downloads/>.
   * During installation, **tick "Add python.exe to PATH"**.
2. **Extract the ZIP.** Right-click `Heatwave_Python_Project.zip` → **Extract All…** → choose a folder (for example `Documents`).
3. Open the extracted `Heatwave_Python_Project` folder and **double-click `run.bat`**.
   * The first run creates a virtual environment and installs the packages. This takes 1–2 minutes and needs internet.
   * Chrome (or your default browser) opens **http://127.0.0.1:8000/** automatically.
4. **To stop the server:** click the black window and press **Ctrl + C**, or simply close the window.
5. **To start again later:** double-click `run.bat` again. Later starts take only a few seconds.

## 2. Manual start (Command Prompt), step by step

1. Check Python: open **Command Prompt** (Start menu → type `cmd` → Enter) and run
   ```bat
   python --version
   ```
   It should print `Python 3.10` or newer. If it says *not recognized*, install Python (step 1 above).
2. Go to the project folder. In File Explorer, open `Heatwave_Python_Project`, click the address bar, type `cmd` and press Enter. A Command Prompt opens in that folder.
3. Create and activate a virtual environment (do this once):
   ```bat
   python -m venv .venv
   .venv\Scripts\activate
   ```
   `(.venv)` now appears at the start of the line.
   *PowerShell users:* run `.venv\Scripts\Activate.ps1` instead. If PowerShell blocks it, use Command Prompt.
4. Install the packages:
   ```bat
   python -m pip install -r requirements.txt
   ```
5. Start the server:
   ```bat
   python app.py
   ```
   You will see `Climate Intelligence India (Python / Flask) -> http://127.0.0.1:8000/`.
6. Open **Chrome** and go to **http://127.0.0.1:8000/**.
7. To stop, press **Ctrl + C** in the Command Prompt window.
8. To restart, open Command Prompt in the folder again, then run `.venv\Scripts\activate` and `python app.py`.

macOS / Linux: `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt && python app.py`

**Port already in use?** Start on another port with `set PORT=8001` then `python app.py`, and open http://127.0.0.1:8001/.

**Nothing to configure:** no API key, no PostgreSQL installation and no `.env` file are needed. The database file `database/heatwave.db` is created automatically on the first start. `.env.example` lists optional settings.

**Internet:** the laptop needs internet for live weather (Open-Meteo), the map background (CARTO), the map and chart libraries (unpkg / jsdelivr) and the fonts (Google Fonts), exactly like the deployed site.
*Demo tip:* run the site once **before** the presentation while online. Every refresh is saved in SQLite, so if Open-Meteo is unreachable later, the site shows the last saved data with a "Showing cached data" note instead of failing.

---

## 3. How the Python implementation works (for the presentation)

**What Flask does.** Flask is a Python web framework. In this project it does two jobs:
1. It **serves the website**: `index.html`, CSS, JavaScript and the map boundary file in `static/`.
2. It provides a **JSON API** under `/api/...`. The website's JavaScript calls this API, and Python does all the work behind it.

**The main file, `app.py`.** It creates the Flask app (`create_app`), creates and seeds the SQLite database on first start, and registers the three route files. It also adds security headers, rate limiting and JSON error messages, then runs the server on `127.0.0.1:8000`.

**How the frontend talks to Python.** The browser runs `static/js/api.js`, which calls, for example, `fetch('/api/states/MH/weather')`. Flask sends that request to the Python function `weather()` in `backend/routes/weather_routes.py`. Python then:
1. gets the weather for all 36 states with **one** bulk request to Open-Meteo (`backend/services/open_meteo.py`);
2. turns the raw data into rows and works out the heat-risk score (`backend/services/heat_risk.py`);
3. keeps the result in memory for 30 minutes (`backend/services/cache.py`), so pages load fast and Open-Meteo is not overloaded;
4. **saves** each refresh to SQLite (`weather_records`, `forecasts`, `historical_weather` and `heat_risk` tables);
5. returns JSON, which the JavaScript draws as cards, charts and the map.

**What is done in Python**

| Task | File |
|---|---|
| Calling Open-Meteo (forecast + historical archive APIs), with timeouts and errors handled | `backend/services/open_meteo.py` |
| Converting API columns into daily rows, deriving the mean temperature if it is missing | `backend/services/open_meteo.py` → `daily_rows()` |
| Splitting past vs forecast days, building each state's snapshot, falling back to the database if offline | `backend/services/weather_service.py` |
| **Heat-risk score** (5 factors, 0–100) and risk level | `backend/services/heat_risk.py` |
| History: last 7 / 30 days or a custom range back to 1940, summary statistics, date validation | `backend/services/history_service.py` |
| Matching India boundary polygons to states for the map | `backend/services/geo_service.py` |
| Register / login / logout, bcrypt password hashing, sessions, input validation | `backend/services/auth_service.py`, `backend/routes/auth_routes.py` |
| Favourites, search history (create, read, delete) and live table row counts | `backend/routes/user_routes.py` |
| Database schema, connection and seeding | `backend/schema.sql`, `backend/db.py` |

**Heat-risk calculation** (rule-based and transparent; *not* a trained AI model and *not* an official IMD warning):

| Component | Points | Rule |
|---|---|---|
| Maximum temperature | 0–40 | linear from 30 °C to 46 °C |
| Feels-like maximum | 0–25 | linear from 32 °C to 52 °C |
| Humidity stress | 0–10 | only if Tmax ≥ 32 °C; RH from 40 % to 80 % |
| Persistence | 0–15 | 5 points per day ≥ 40 °C in a 3-day window |
| Anomaly | 0–10 | Tmax above the mean of the previous 30 days (+6 °C gives full points) |

Bands (project-defined): 0–19 Very Low · 20–34 Low · 35–49 Medium · 50–64 Elevated · 65–79 High · 80–100 Extreme.

**Files to open while demonstrating the Python code:** `app.py` → `backend/services/heat_risk.py` → `backend/services/weather_service.py` → `backend/routes/weather_routes.py` → `backend/schema.sql`. Then run `python -m pytest` to show the tests passing.

**Database (SQLite, 11 tables):** `users`, `sessions`, `states`, `weather_records`, `forecasts`, `historical_weather`, `heat_risk`, `user_favorites`, `user_search_history`, `safety_tips`, `emergency_contacts`. The schema has primary keys, foreign keys with `ON DELETE CASCADE`, `UNIQUE(userId, stateId)`, `UNIQUE(stateId, date)` and indexes. The dashboard's "Database (live row counts)" card reads the live counts. You can open `database/heatwave.db` with the free "DB Browser for SQLite" to show the tables.

## 4. API endpoints (all implemented in Python)

```
POST /api/auth/register      POST /api/auth/login        POST /api/auth/logout
GET  /api/auth/me            PUT  /api/auth/profile      PUT  /api/auth/password
DELETE /api/auth/account
GET  /api/states             GET  /api/states/<id or code>
GET  /api/states/<id>/weather    GET /api/states/<id>/forecast
GET  /api/states/<id>/history?range=7|30|custom&start=YYYY-MM-DD&end=YYYY-MM-DD
GET  /api/heatwave/<id>      GET  /api/map/risk          GET  /api/map/geo
GET  /api/safety?level=HIGH  GET  /api/emergency
GET/POST /api/favorites      DELETE /api/favorites/<id>
GET/POST /api/search-history DELETE /api/search-history/<id>   DELETE /api/search-history
GET  /api/db/stats           GET  /api/health
```
Example: open http://127.0.0.1:8000/api/states/MH/weather in Chrome to see the raw JSON that Python produces.

## 5. Tests

```bat
.venv\Scripts\activate
python -m pytest
```
There are 28 tests. `tests/test_logic.py` contains the original project's logic tests, rewritten in Python (risk score, data parsing, boundary matching). `tests/test_api.py` tests every endpoint (auth, favourites, history, validation, the database fallback). Open-Meteo is replaced by fake data in the tests, so they run without internet.

## 6. Project structure

```
Heatwave_Python_Project/
├── app.py                  main Flask application (start here)
├── requirements.txt        Python packages (Flask, requests, bcrypt, pytest)
├── run.bat                 one-click start for Windows
├── .env.example            optional settings (no secrets needed)
├── backend/
│   ├── db.py, schema.sql   SQLite connection, tables, seeding
│   ├── security.py         login_required, rate limiting, API errors
│   ├── data/               states.py (36 states/UTs), reference.py (tips, contacts, weather codes)
│   ├── routes/             auth_routes.py, weather_routes.py, user_routes.py
│   └── services/           open_meteo, weather_service, history_service, heat_risk,
│                           geo_service, auth_service, state_service, cache, utils
├── static/                 the ORIGINAL website (unchanged design)
│   ├── index.html, css/styles.css
│   ├── js/                 app.js, api.js, store.js, ui.js, charts.js, heatmap.js, pages/*.js
│   └── data/india_states.geojson   India state/UT boundaries for the map
├── database/               heatwave.db is created here automatically
└── tests/                  pytest tests
```

## 7. What changed from the original project

* **Backend:** Node.js / Express / Prisma / PostgreSQL was replaced by **Python / Flask / SQLite**, keeping the same API URLs and JSON format, so the original frontend works without changes.
* **Frontend:** copied unchanged, with three edits:
  * Three labels that said "PostgreSQL" / "Prisma" now say "SQLite" / "Python", so the page describes the real system.
  * One small bug fix in `static/js/app.js`: the "Add to Favorites" button no longer stays disabled after the first click.
* **Map boundaries:** `static/data/india_states.geojson` is now included in the project, so there is no separate download step. It was built from the open [india-maps-data](https://github.com/udit-001/india-maps-data) dataset, with district polygons merged into the 36 states and UTs.

## 8. Limitations

* Live weather needs internet. Weather is measured at one **representative city per state** (labelled on every page), not averaged over the whole state.
* The heat-risk score is a project-defined estimate. Always follow official **IMD** warnings.
* Map boundaries are for visualisation only. Refer to the Survey of India for official maps.
* This is a local development server for demonstration. It is not configured for public internet hosting.

## 9. Uploading to GitHub (account: shriya21-2107)

1. Sign in to <https://github.com> as **shriya21-2107**.
2. Click **+** → **New repository** → name it `Heatwave_Python_Project` → **Create repository**.
3. On the new repository page, click **"uploading an existing file"**.
4. Drag in **everything inside** the extracted `Heatwave_Python_Project` folder, *except* `.venv` and `database/heatwave.db`, if they exist.
5. Click **Commit changes**.

Or, with Git installed:
```bat
cd Heatwave_Python_Project
git init
git add .
git commit -m "Heatwave early warning system - Python/Flask version"
git branch -M main
git remote add origin https://github.com/shriya21-2107/Heatwave_Python_Project.git
git push -u origin main
```
(`.gitignore` already excludes `.venv`, caches and the database file.)

## 10. Data sources

Weather: [Open-Meteo](https://open-meteo.com/) (CC BY 4.0), Forecast API and Historical Weather API (ERA5) · Basemap © OpenStreetMap contributors, © CARTO · Safety guidance adapted from NDMA heat-wave Do's & Don'ts · Emergency numbers: national services (112, 108, 102, 100, 101, 1070, 1077).
