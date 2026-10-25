# WindWise

WindWise is a national UK wind generation forecast accuracy dashboard for January 2026 day-ahead purchasing decisions. It uses Elexon BMRS FUELHH actual generation (filtered to `fuelType=WIND`) and WINDFOR forecast publication history. Codex/AI tools assisted with development.

## Requirements and data flow

The UI requests `/api/accuracy`; FastAPI fetches Elexon `datasets/WINDFOR` and `datasets/FUELHH`, validates their JSON records, filters intervals, chooses the eligible forecast, matches by UTC `startTime`, then returns metrics and intervals. Live queries confirmed Elexon responds with JSON objects containing `data` arrays. FUELHH accepts `settlementDateFrom`/`settlementDateTo`; WINDFOR accepts `publishDateTimeFrom`/`publishDateTimeTo`. Historical multi-day FUELHH requests are rejected, so the backend safely fetches daily windows with bounded concurrency and retries HTTP 429 responses. Generation is reported in MW, and timestamps are UTC ISO 8601.

The decision-time assumption is configurable: by default 11:00 Europe/London on the day before delivery. For each delivery interval, the selected forecast is the one with the greatest `publishTime` not later than that cutoff. This is a transparent approximation because the brief does not specify an exact trading cutoff. MAE and RMSE are in MW; MAPE is a percentage over non-zero actuals. Only matched intervals contribute to metrics; missing values remain null and coverage is reported.

## Structure

`backend/app` contains the API and data logic; `backend/tests` contains synthetic fixtures for selection and metric tests; `frontend/src` contains the React dashboard; Docker files and Compose configure local running.

## Local setup

Python 3.11+ and Node 20+ are needed. Configure environment variables from `.env.example` as needed.

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --reload
```

In a second terminal:

```powershell
cd frontend
npm install
npm run dev
```

Open http://localhost:5173. API docs are at http://localhost:8000/docs.

## Docker

From the project root run `docker compose up --build`, then open http://localhost:5173. API docs remain on port 8000.

## API

- `GET /api/health` service check
- `GET /api/accuracy?start_date=2026-01-01&end_date=2026-01-31&cutoff_hour=11` metrics, interval series, coverage, decision rule
- `GET /api/forecasts` selected forecast intervals, same filters
- `GET /api/actuals` wind actuals, same filters

Date filters are validated within January 2026. FastAPI provides OpenAPI docs.

## Verification

Backend: `cd backend; pytest`. Frontend: `cd frontend; npm run build`.

## Deployment and known limits

### Live deployments

- Frontend: [https://windwise-seven.vercel.app/](https://windwise-seven.vercel.app/)
- Backend health check: [https://windwise-one.vercel.app/api/health](https://windwise-one.vercel.app/api/health)
- Backend API documentation: [https://windwise-one.vercel.app/docs](https://windwise-one.vercel.app/docs)
- Source repository: [https://github.com/whyabhisheek/WindWise](https://github.com/whyabhisheek/WindWise)

Vercel environment variables are configured separately for each project:

| Vercel project | Name | Value |
| --- | --- | --- |
| Frontend (`windwise-seven`) | `VITE_API_URL` | `https://windwise-one.vercel.app` |
| Backend (`windwise-one`) | `CORS_ORIGINS` | `https://windwise-seven.vercel.app` |

Redeploy each project after changing its environment variables. Add any custom frontend domain to the backend's `CORS_ORIGINS` value as well. The frontend container serves the Vite build with Nginx; the FastAPI service is deployed separately.

The live January 2026 request returned 1,488 FUELHH intervals (30-minute actuals) and 744 exact-time forecast matches (WINDFOR forecasts occur on hourly start times). Forecast coverage is 50% because of this interval cadence difference; only exact UTC `startTime` matches feed the metrics. Interpret the error metrics within this coverage limit. The app surfaces API failures and never substitutes fabricated metrics. The January daily queries returned complete windows in live verification; pagination metadata was not present in the observed responses.

## Demo script (under 10 minutes)

**Client overview (4 minutes):** Introduce the January day-ahead decision question; show forecast versus actual and absolute error; explain the cutoff rule; walk through matched count and MAE/RMSE/MAPE and coverage; emphasize that metrics describe only matched records.

**Technical review (4 minutes):** Trace React → FastAPI → Elexon; explain UTC interval matching, WIND filter, publish-time eligibility, deduplication, and zero-actual MAPE handling; show `/docs`, tests, Docker Compose, configuration, and known historical-data limitation.
