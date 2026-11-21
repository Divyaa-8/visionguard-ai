# VisionGuard AI

**Multi-stream video intelligence: ask questions in plain language, get evidence, track people and vehicles across cameras, and receive alerts in real time.**

VisionGuard AI is a working prototype of a surveillance-analytics dashboard. You can search what happened across many cameras ("red car near the gate last night"), see the supporting evidence, follow the same object from one camera to the next, and get live alerts pushed to the browser.

---

## Table of contents

1. [Features](#features)
2. [Project structure](#project-structure)
3. [Quick start](#quick-start)
4. [Using the app](#using-the-app)
5. [Database](#database)
6. [Datasets (you must download the real ones)](#datasets-you-must-download-the-real-ones)
7. [Sample footage](#sample-footage)
8. [Training and evaluation](#training-and-evaluation)
9. [What is real and what is not](#what-is-real-and-what-is-not)
10. [Roadmap](#roadmap)
11. [Troubleshooting](#troubleshooting)

---

## Features

| Feature | What it does |
|---|---|
| **Conversational search** | Type a question such as "white van near the entrance this morning". It is converted into structured filters (object, colour, place, time) and sent to `/api/search`. |
| **Evidence view** | Every result links back to the camera, timestamp and detection that produced it. |
| **Clarify-once memory** | If a query is ambiguous, the system asks one clarifying question and remembers your answer for later searches. |
| **Cross-camera tracking** | Links the same identity seen on different cameras (re-identification). |
| **Live alerts** | Alerts are broadcast over a WebSocket (`/ws/alerts`) to every open browser. |
| **Standing queries** | Save a search so it keeps watching for new matches. |

---

## Project structure

```
visionguard-ai/
├── frontend/
│   └── index.html                 React UI (single file; loads React from a CDN, so it needs internet)
├── backend/
│   ├── app/
│   │   ├── main.py                FastAPI app: REST API + WebSocket + serves the frontend
│   │   ├── db.py                  Database connection (SQLAlchemy; SQLite default, PostgreSQL optional)
│   │   └── seed_data.json         Demo cameras, events, alerts and memory loaded on first start
│   ├── pipeline/
│   │   └── process_video.py       OPTIONAL: YOLO + ByteTrack on a video file, posts results to the API
│   └── data/
│       └── visionguard.db         Created automatically on first run
├── datasets/
│   ├── synthetic/
│   │   ├── train/                 detections.csv (6000 rows), reid_pairs.csv (4000 rows)
│   │   └── test/                  detections.csv (1500), reid_pairs.csv (1000), retrieval_queries.json
│   └── scripts/
│       ├── generate_synthetic.py          Regenerate the synthetic data
│       ├── train_models.py                Train and evaluate the alert classifier and re-ID linker
│       ├── eval_retrieval.py              Precision/recall of /api/search against test queries
│       └── download_real_datasets.md      Links and notes for real public datasets
├── scripts/
│   ├── run.sh                     Start the server (macOS/Linux)
│   └── run.bat                    Start the server (Windows)
└── requirements.txt
```

---

## Quick start

**Requirements:** Python 3.10 or newer, and an internet connection (the UI loads React from a CDN).

### 1. Open the project
In VS Code: **File → Open Folder** and select `visionguard-ai`.

### 2. Create a virtual environment and install dependencies
Open a terminal (**Terminal → New Terminal**) and run:

```bash
python -m venv .venv

# Activate it:
#   Windows:       .venv\Scripts\activate
#   macOS/Linux:   source .venv/bin/activate

pip install -r requirements.txt
```

### 3. Start the server
```bash
uvicorn backend.app.main:app --reload --port 8000
```
Or use the helper scripts: `scripts/run.sh` (macOS/Linux) or `scripts/run.bat` (Windows).

### 4. Open the app
- App: <http://localhost:8000>. The top bar should show **Live backend**.
- Interactive API docs: <http://localhost:8000/docs>

### Running without the backend
Open `frontend/index.html` directly in a browser. It falls back to built-in mock data and the top bar shows **Mock data**. If you serve the frontend separately (for example with VS Code Live Server on port 5500), the page will still try to reach the backend at `http://localhost:8000`.

---

## Using the app

1. **Check the status badge** in the top bar: *Live backend* means data is read from and written to the database; *Mock data* means the built-in demo data is being used.
2. **Search** using plain language. Parsing of your text into filters happens in the browser using rules (see [What is real and what is not](#what-is-real-and-what-is-not)).
3. **Open a result** to see the evidence (camera, time, detection details).
4. **Watch the alerts panel**: new alerts arrive live over the WebSocket.

---

## Database

By default the app uses **SQLite**. The file `backend/data/visionguard.db` is created and filled with demo data the first time you start the server. To reset everything, stop the server and delete that file.

**Tables:** `cameras`, `events`, `detections`, `alerts`, `memory`, `standing_queries`, `tracks`

### Using PostgreSQL (optional)
```bash
pip install psycopg2-binary

# Windows
set DATABASE_URL=postgresql+psycopg2://user:password@localhost:5432/visionguard

# macOS/Linux
export DATABASE_URL=postgresql+psycopg2://user:password@localhost:5432/visionguard
```
Set the variable in the same terminal before starting the server.

---

## Datasets (you must download the real ones)

There are two kinds of data, and it is important not to confuse them.

### 1. Synthetic datasets: included in the repo
Found in `datasets/synthetic/`. These are **generated from rules, with about 3% label noise**, and contain **no video**. They exist so you can run the training and evaluation code end to end.

| File | Columns (features → label) |
|---|---|
| `detections.csv` | hour, confidence, dwell_sec, speed_px_s, in_restricted_zone, persons_in_frame, box size, object_type, camera → `event_type`, `is_alert` |
| `reid_pairs.csv` | time_gap_s, camera_hops, color_match, size_ratio, embedding_cosine → `same_identity` |

> **Warning:** Because the data follows the rules used to generate it, models score very high (alert classifier ≈ 0.97 accuracy, re-ID linker ≈ 0.99 on the test split). **These numbers do not show real-world accuracy.**

### 2. Real public datasets: NOT included, you must download them
Real datasets are large and each has its own licence terms, so they are **not bundled** with this project. To get real results, download them yourself by following `datasets/scripts/download_real_datasets.md`.

| Dataset | Useful for |
|---|---|
| UA-DETRAC | Vehicle detection and tracking |
| MOT (MOT16/17/20) | Multi-object tracking |
| CityFlow | Cross-camera vehicle re-identification |
| VIRAT | Surveillance activity recognition |
| UCF-Crime | Anomaly / crime event detection |
| COCO | General object detection |

Before using any of them, read and accept the licence of each dataset. Make sure you have enough disk space, as some are tens of gigabytes.

---

## Sample footage

The demonstration of this project used **sample footage** (short example videos) rather than a live camera network or a full public dataset. The footage is **not included in this repository**, because of file size and because of the licence terms of the original videos.

To try the video pipeline yourself:

1. Put your own sample video(s) in a folder of your choice (for example `samples/`).
2. Install the pipeline dependencies:
   ```bash
   pip install ultralytics opencv-python requests
   ```
3. Start the backend (see [Quick start](#quick-start)), then run:
   ```bash
   python backend/pipeline/process_video.py --help
   ```
   to see the available options for pointing the pipeline at your video.

Use only footage you have the right to use, and be aware that recordings of people may be subject to privacy laws in your country.

---

## Training and evaluation

Run these from the project root, with the virtual environment active.

```bash
# 1. (Optional) regenerate the synthetic data
python datasets/scripts/generate_synthetic.py

# 2. Train and evaluate the alert classifier and the re-ID linker
pip install scikit-learn numpy
python datasets/scripts/train_models.py

# 3. Evaluate search quality (the backend must be running)
python datasets/scripts/eval_retrieval.py
```

`eval_retrieval.py` sends the queries from `datasets/synthetic/test/retrieval_queries.json` to `/api/search` and reports precision and recall. It scores **1.00 on the 8 seed queries**, which is a small, easy test set, so treat it as a sanity check rather than a benchmark.

---

## What is real and what is not

Being clear about the limits of the prototype:

**Tested and working**
- Database creation and seeding
- All REST endpoints
- WebSocket alert broadcast
- Frontend in both live-backend and mock-data modes
- Retrieval evaluation on the 8 seed queries

**Simplified or placeholder**
- **Text-to-filter parsing** (natural language → object / colour / place / time) uses simple rules in the browser, **not an LLM**. `/api/search` accepts structured filters only. To get true natural-language understanding, put an LLM or a CLIP text encoder in front of it.
- **Camera pictures in the UI** are drawn placeholder scenes, not real video feeds.
- **Analytics numbers** are placeholders (the `MEASURED` values in `frontend/index.html`). Replace them with real output from `eval_retrieval.py`.
- **Video pipeline** (`backend/pipeline/process_video.py`, YOLO + ByteTrack) is **experimental**. Nothing analyses real video until you run it yourself.

**Not implemented yet**
- Zone logic (restricted areas, line crossing)
- CLIP/SigLIP re-identification embeddings
- Vector search with FAISS or Qdrant
- A production alert engine

---

## Roadmap

- [ ] Replace rule-based query parsing with an LLM or CLIP text encoder
- [ ] Run and validate the video pipeline on real datasets (UA-DETRAC, MOT, CityFlow)
- [ ] Add embedding-based re-ID with a vector index (FAISS or Qdrant)
- [ ] Add zone definitions and a real alert engine
- [ ] Replace placeholder analytics with measured results

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Top bar shows **Mock data** | The backend is not running or not reachable. Start it with `uvicorn backend.app.main:app --reload --port 8000` and refresh. |
| Blank page or UI will not load | The UI loads React from a CDN, so check your internet connection. |
| `ModuleNotFoundError` | Activate the virtual environment and run `pip install -r requirements.txt`. |
| Demo data looks wrong or stale | Stop the server, delete `backend/data/visionguard.db`, and start again. |
| `eval_retrieval.py` fails to connect | The backend must be running on port 8000 first. |
| Port 8000 already in use | Start on another port, e.g. `--port 8001`, and open that port in the browser. |