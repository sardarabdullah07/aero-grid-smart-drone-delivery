# Aero-Grid — Smart Drone Delivery System

Aero-Grid is a full-stack visualization of four classical AI techniques cooperating to plan and execute a multi-stop drone delivery mission across a 40x40 city grid. A weather classifier issues a pre-flight go/no-go verdict, a genetic algorithm orders the delivery stops to minimize tour distance, A\* finds the collision-free path for each leg around buildings and no-fly zones, and a tabular Q-Learning agent learns a single leg from scratch so its policy can be compared against A\* and stress-tested under new obstacles.

The backend is a **stateless** FastAPI service: the city is passed in with every request and is never stored server-side, so the frontend owns the source of truth. There is no database, no authentication, and no user or order persistence — this is an algorithm visualization tool, not a production logistics system.

---

## Features

- **City generation and validation** — procedurally generates a 40x40 grid with buildings, no-fly zones, 8 delivery targets, and a depot at `[1, 1]`. Difficulty (`easy` / `medium` / `hard`) controls building density; a seed makes any city reproducible. Validation runs A\* from the depot to every target and reports blocked or unreachable ones.
- **Weather go/no-go classification** — three scikit-learn models (Gaussian Naive Bayes, scaled Logistic Regression, Decision Tree) classify `wind`, `visibility`, and `rainfall` into `Safe to Fly`, `Requires Altitude Drop`, or `Grounded`. A comparison endpoint runs all three and computes a majority verdict, breaking a 3-way tie in favor of the more conservative (higher-severity) label.
- **Model transparency** — per-model accuracy, confusion matrices, and per-class precision/recall/F1, plus feature importance and a sampled scatter of the labeled training set for plotting.
- **Route optimization (Genetic Algorithm)** — solves the delivery ordering as a TSP with tournament selection and order crossover. Returns the full generational history so the frontend can animate convergence frame by frame, alongside the naive-order distance and the percentage improvement.
- **Per-leg pathfinding (A\*)** — 8-directional search with selectable `octile`, `manhattan`, or `euclidean` heuristics. Returns not just the path but the explored set and node count per leg, so the visualization can show the search frontier rather than only the result.
- **Reinforcement learning (Q-Learning)** — trains a tabular agent on a single start→goal leg with configurable learning rate, discount, epsilon decay, episode caps, and potential-shaping coefficient. Returns episode history, periodic Q-table snapshots, the converged greedy path, and A\*'s solution to the same problem for a side-by-side quality comparison.
- **Policy replay and generalization testing** — replays the learned greedy policy on a city, and stress-tests it against a perturbed city. Perturbations can be placed automatically (random off-path cells, keeping the demo solvable) or manually from cells the user clicks — deliberately including cells *on* the trained path, to show where a learned policy breaks and A\* does not.
- **Real weather data pipeline** — `data_pipeline.py` pulls hourly ERA5 observations for Islamabad (2022-01-01 → 2023-12-31) from the Open-Meteo Archive API, derives an aviation-style visibility proxy from dew-point spread, humidity, and precipitation, and labels each hour for flight safety. A synthetic generator is retained as a fallback when the API is unreachable.
- **Interactive frontend** — a 3D landing scene, a canvas grid renderer, an animated phase stepper across the mission, live metrics dashboards, reward curves, confusion matrices, and a results bento. Trained models are cached to disk as joblib artifacts so the server boots instantly after the first run.

---

## Architecture

Two independently-run services communicating over plain JSON REST:

```
┌──────────────────────────┐         HTTP / JSON          ┌──────────────────────────┐
│  frontend (Next.js 15)   │ ───────────────────────────► │  backend (FastAPI)       │
│  localhost:3000          │ ◄─────────────────────────── │  127.0.0.1:8000          │
│  Zustand mission store   │                              │  stateless — no DB       │
└──────────────────────────┘                              └──────────────────────────┘
```

- **Transport is REST only.** There are no websockets, no server-sent events, and no polling. Every animation in the UI (GA convergence, A\* frontier sweep, Q-table evolution) is driven client-side by replaying a full history array that the backend returned in a single response.
- **The backend holds no session state.** Each request carries the complete `CityState`. The one piece of server-side state is the weather classifier, which is trained once at import and is read-only afterward, so it is safe to share across requests.
- **Ports.** Backend on `8000`, frontend on `3000`. The frontend defaults to `http://127.0.0.1:8000` and can be pointed elsewhere with `NEXT_PUBLIC_API_BASE`.
- **CORS** is restricted at the backend to exactly `http://localhost:3000` and `http://127.0.0.1:3000`. If you change the frontend port, update `allow_origins` in `backend/main.py`.
- **Client state** lives in a Zustand store (`frontend/src/lib/store.ts`) that holds the city and every phase result across navigations; `frontend/src/lib/api.ts` is the single typed client wrapping all eleven endpoints.

### Frontend routes

| Route | Purpose |
|---|---|
| `/` | Landing page with the React Three Fiber 3D city scene |
| `/mission` | Main flow — weather, optimize, and fly phases in one stepper |
| `/learn` | Q-Learning training, replay, and generalization testing |
| `/results` | Mission summary and analytics |
| `/weather`, `/optimize`, `/fly` | Legacy paths; redirect to `/mission` |
| `/setup` | Redirects to `/build` |

> **Note:** `/setup` and the mission page both navigate to `/build`, but `frontend/src/app/build/` is not present in this repository, so those links currently 404. The likely cause is the root `.gitignore`: its Python-oriented `build/` rule also matches `frontend/src/app/build/` and excluded the city-builder page from version control. To restore it, add a negation such as `!frontend/src/app/build/` to `.gitignore` and commit the directory.

---

## Tech Stack

**Backend** (`backend/requirements.txt`)

| Package | Version constraint |
|---|---|
| fastapi | >= 0.104.0 |
| uvicorn[standard] | >= 0.24.0 |
| scikit-learn | >= 1.3.0 |
| numpy | >= 1.24.0 |
| pydantic | >= 2.0.0 |
| pandas | >= 2.0.0 |
| joblib | >= 1.3.0 |
| requests | >= 2.31.0 |

**Frontend** (`frontend/package.json`)

| Package | Version constraint |
|---|---|
| next | ^15.1.0 |
| react / react-dom | ^19.0.0 |
| typescript | ^5.0.0 |
| tailwindcss / @tailwindcss/postcss | ^4.0.0 |
| zustand | ^5.0.0 |
| framer-motion | ^12.0.0 |
| recharts | ^2.15.0 |
| three | ^0.184.0 |
| @react-three/fiber | ^9.6.1 |
| @react-three/drei | ^10.7.7 |
| lucide-react | ^0.474.0 |
| @21st-sdk/react / nextjs / agent | ^0.1.4 / ^0.0.11 / ^0.0.18 |
| eslint / eslint-config-next | ^9.0.0 / ^15.1.0 |

---

## Project Structure

```
Aero-Grid-Smart-Drone-Delivery-System/
├── backend/
│   ├── main.py                  FastAPI app, Pydantic models, all 11 route handlers
│   ├── weather_classifier.py    Trains/persists GaussianNB, LogisticRegression, DecisionTree
│   ├── genetic_algorithm.py     GA for delivery-order TSP (tournament selection, order crossover)
│   ├── astar.py                 8-directional A* with octile/manhattan/euclidean heuristics
│   ├── q_learning.py            Tabular Q-Learning agent, greedy policy extraction, replay
│   ├── data_pipeline.py         Builds weather_data.csv from the Open-Meteo Archive API
│   ├── _phase1_verify.py        Standalone sanity-check script for the core algorithms
│   ├── weather_data.csv         Labeled training set consumed by the classifier
│   ├── city.json                Saved sample city fixture (buildings, NFZs, targets, depot)
│   ├── models/                  Persisted joblib artifacts + cached evaluation metrics
│   └── requirements.txt         Python dependencies
├── frontend/
│   ├── src/app/                 App Router pages: /, /mission, /learn, /results, redirects
│   │   ├── layout.tsx           Root layout and global chrome
│   │   └── globals.css          Tailwind v4 entry and design tokens
│   ├── src/components/
│   │   ├── CityCanvas.tsx       Canvas renderer for the grid, paths, and explored cells
│   │   ├── PhaseStepper.tsx     Mission phase progress indicator
│   │   ├── MissionStatusStrip.tsx, DecisionLog.tsx, ValidationPanel.tsx, ToolPalette.tsx
│   │   ├── landing/             3D hero scene (CityScene3D, Drone, Building, HeroText)
│   │   ├── phase-panels/        Weather, Optimize, and Fly phase UIs + weather sub-charts
│   │   ├── learn/               Training controls, reward curve, live metrics, leg selector
│   │   └── ui/                  Scrubber and Skeleton primitives
│   ├── src/lib/api.ts           Typed fetch client for every backend endpoint
│   ├── src/lib/store.ts         Zustand store holding city and all phase results
│   ├── src/hooks/useCountUp.ts  Animated numeric counter hook
│   ├── next.config.ts           Next.js configuration
│   └── package.json             Node dependencies and scripts
├── .gitignore
└── README.md
```

---

## API Reference

Base URL: `http://127.0.0.1:8000`. Interactive OpenAPI docs are served at `/docs`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/city/random` | Generate a reproducible random city. Query: `seed` (int, default `7`), `difficulty` (`easy`\|`medium`\|`hard`, default `medium`). |
| POST | `/city/validate` | Check a city is playable — depot and targets unblocked, every target reachable by A\*. |
| POST | `/weather` | Classify `wind` / `visibility` / `rainfall` with Gaussian Naive Bayes; returns label, class probabilities, accuracy. |
| POST | `/weather/compare` | Run all three models on the same input; returns each prediction, the majority verdict, agreement flag, and feature importance. |
| GET | `/weather/metrics` | Per-model accuracy, confusion matrix, and per-class precision/recall/F1, plus feature importance and label names. |
| GET | `/weather/training-data` | Sample labeled training rows for the scatter plot. Query: `n` (1–5000, default `200`). |
| POST | `/optimize` | Run the GA over the city's targets. Body: `city`, `mutation_rate` (0.01–0.20), `population_size` (50–200), `seed`. Returns best route, distances, improvement %, and full generational history. |
| POST | `/fly` | Run A\* across each leg of a route. Body: `city`, `route`, `heuristic`. Returns per-leg path, explored cells, path length, node count, and delivery flag. |
| POST | `/learn/train` | Train a Q-Learning agent on one `start`→`goal` leg. Body: `city`, `start`, `goal`, `hyperparameters`, `seed`. Returns episode history, Q-table snapshots, final Q-table and path, plus A\*'s solution for comparison. |
| POST | `/learn/replay` | Greedy walk of a trained policy. Body: `city`, `q_full`, `start`, `goal`. Returns path, length, and success flag. |
| POST | `/learn/generalize` | Test a trained policy on a perturbed city. Body: `original_city`, `q_full`, `start`, `goal`, `trained_path`, `num_perturbations` (0–20), optional `manual_cells`, `seed`. Returns the perturbed city, added buildings, and both Q-Learning and recomputed A\* results. |

**Q-Learning hyperparameters** accepted by `/learn/train` (with defaults and validated ranges):

| Field | Default | Range |
|---|---|---|
| `alpha` | 0.15 | 0.01 – 1.0 |
| `gamma` | 0.95 | 0.5 – 0.999 |
| `epsilon_start` | 1.0 | 0.0 – 1.0 |
| `epsilon_end` | 0.05 | 0.0 – 1.0 |
| `epsilon_decay_episodes` | 500 | 10 – 5000 |
| `max_episodes` | 1000 | 10 – 5000 |
| `max_steps_per_episode` | 200 | 20 – 2000 |
| `shaping_coefficient` | 0.5 | 0.0 – 2.0 |

---

## Setup

### Prerequisites

- Python 3.12
- Node.js 20 or newer
- npm

No database is required — the backend is stateless and stores nothing between requests.

### 1. Backend

```bash
cd backend
python -m venv venv
venv\Scripts\activate           # Windows
# source venv/bin/activate      # macOS / Linux
pip install -r requirements.txt
```

The classifier trains on first import and caches the fitted models to `backend/models/*.joblib`; those artifacts are already committed, so startup is immediate. `backend/weather_data.csv` is also committed and is required by `/weather/training-data` — the endpoint returns HTTP 503 if it is missing.

To regenerate the dataset from the live Open-Meteo Archive API (optional, requires network access):

```bash
python data_pipeline.py
```

Delete `backend/models/*.joblib` afterward so the classifier retrains on the new data.

### 2. Frontend

```bash
cd frontend
npm install
```

### Environment variables

The codebase reads exactly one environment variable:

| Variable | Read by | Default | Purpose |
|---|---|---|---|
| `NEXT_PUBLIC_API_BASE` | `frontend/src/lib/api.ts` | `http://127.0.0.1:8000` | Base URL of the FastAPI backend. |

The backend reads no environment variables; host, port, and CORS origins are set in code.

To override the API base, create `frontend/.env.local`:

```bash
NEXT_PUBLIC_API_BASE=http://127.0.0.1:8000
```

---

## Running in Development

Start both services in separate terminals.

**Terminal 1 — backend** (from `backend/`, with the virtualenv activated):

```bash
uvicorn main:app --reload --port 8000
```

Running `python main.py` also works and serves on `0.0.0.0:8000` without auto-reload.

**Terminal 2 — frontend** (from `frontend/`):

```bash
npm run dev
```

Open `http://localhost:3000`. Keep the frontend on port 3000 — any other origin is rejected by the backend's CORS policy unless you edit `allow_origins` in `backend/main.py`.

### Available frontend scripts

| Script | Command | Purpose |
|---|---|---|
| `npm run dev` | `next dev` | Start the development server on port 3000 |
| `npm run build` | `next build` | Production build |
| `npm run start` | `next start` | Serve the production build |
| `npm run lint` | `next lint` | Run ESLint |

### Backend sanity check

`backend/_phase1_verify.py` exercises the core algorithms directly, without the HTTP layer:

```bash
python _phase1_verify.py
```

There is no automated test suite in this repository.
