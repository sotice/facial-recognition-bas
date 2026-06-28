# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Facial Recognition Based Attendance System — a Streamlit web app that uses FaceNet (MTCNN + InceptionResnetV1) for face detection/embedding, stores data across Supabase (PostgreSQL), Qdrant (vector DB), and Databricks (data warehouse), and logs attendance to Google Sheets in real-time.

## Commands

```bash
# Run the Streamlit app
streamlit run app.py

# Install dependencies
pip install -r requirements.txt

# Create and activate venv (Windows)
python -m venv venv
.\venv\Scripts\activate

# Clear Streamlit cache
streamlit cache clear
```

There are no tests, linter configs, or CI pipelines in this repository.

## Configuration

All secrets live in `.streamlit/secrets.toml` (git-ignored). Sections: `[supabase_api]`, `[supabase_db]`, `[qdrant]`, `[databricks]`, `[gcp_service_account]`, `[email]`. Access via `st.secrets["section"]["key"]` in code.

## Architecture

### Routing (Single-page Streamlit app)

`app.py` is the entry point. It uses `st.session_state.current_page` for client-side routing — each "page" is a function in a `UI/*.py` module. Navigation is done via `FUNC/navigation.py:go_to()`, which sets the session state key and calls `st.rerun()`. Admin pages check `st.session_state.logged_in` before rendering.

### Three-layer structure

| Layer | Directory | Role |
|-------|-----------|------|
| UI | `UI/` | Streamlit page functions (one per screen). Each module exports a `show()` or named function. |
| Business Logic | `BACKEND/` | DB operations (`*_OP.py` naming convention). Student CRUD, department CRUD, report generation, admin auth. |
| Utilities | `FUNC/` | Shared services — face embedding generation, Google Sheets I/O, email sending, navigation helper. |

### Key backend modules

- **`BACKEND/RDB_connection_OP.py`** — Initializes the Supabase client once, caches in `st.session_state.supabase`.
- **`BACKEND/VDB_connection_OP.py`** — Initializes the Qdrant client, auto-creates the `student_face_encodings_` collection (512-dim cosine vectors) and `S_id` payload index on first run.
- **`BACKEND/student_OP.py`** — Core student logic: batch registration (`process_and_upload_students`), face search (`find_student_by_embedding` with 0.75 threshold), update/delete with dual RDB+VDB writes.
- **`BACKEND/databricks_connection_OP.py`** — Triggers and monitors Databricks ETL jobs via REST API.
- **`BACKEND/report_OP.py`** — Queries Databricks Gold layer for attendance analytics, sends HTML reports to HODs via email.

### Face recognition pipeline (`FUNC/face_embedding_OP.py`)

MTCNN detects and aligns faces to 160×160, then InceptionResnetV1 (pretrained on VGGFace2) produces a 512-dimensional embedding. Models are loaded once via `@st.cache_resource`. Device auto-selects CUDA if available.

### Data flow

1. **Student registration**: Google Sheets → admin uploads → generate embeddings → store in Supabase (student record) + Qdrant (face vectors) → send confirmation email.
2. **Attendance**: webcam capture → MTCNN → FaceNet embedding → Qdrant cosine similarity search → identity confirmation → append row to Google Sheets.
3. **Reporting**: admin triggers → query Databricks Gold layer (star schema: `fact_attendance`, `dim_students`, `dim_departments`) → generate HTML → email to HODs.

### Databricks ETL (`PIPELINE/`)

Notebooks follow medallion architecture (Bronze → Silver → Gold). `01_CREATE_CATALOG_AND_SCHEMA.SQL` sets up the Unity Catalog. Pipelines 02-05 are Databricks notebooks that can be triggered from the admin Job Monitor UI.

### Student ID format

Generated as `YYMMDD-XXXX` (e.g., `260401-0001`), where the date prefix is the registration date and XXXX is a zero-padded sequential counter queried from Supabase.

## Conventions

- Module naming: `*_OP.py` suffix for backend operation modules (e.g., `student_OP.py`, `admin_OP.py`).
- UI modules are imported in `app.py` and called via their exported function (e.g., `Home.show()`, `Admin_login.login_page()`).
- All database clients are module-level singletons, initialized at import time.
- The Supabase client is cached in `st.session_state` to survive Streamlit reruns.
- Qdrant collection name is hardcoded as `student_face_encodings_` in `VDB_connection_OP.py`.
- Face matching threshold is `0.75` (cosine similarity) in `student_OP.py:find_student_by_embedding()`.
