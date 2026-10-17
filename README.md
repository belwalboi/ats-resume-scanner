# ATS Resume Scanner

Compares a resume (PDF/DOCX) with a job description and returns matched and missing skills, evidence
snippets, an **explainable** compatibility score and truthful improvement suggestions.

> The score is a heuristic text-overlap measure for self-review. It is **not** an official ATS score, a
> hiring probability, or a validated prediction. Weights and the similarity scaling are not calibrated
> against hiring outcomes.

## Architecture

```
User → Streamlit (app/frontend.py) ──HTTP──▶ FastAPI (app/main.py, app/api/routes.py)
                                                └─ services/pipeline.py
                                                     resume_parser → job_analyzer → skill_matcher (taxonomy)
                                                     → semantic_analyzer → scoring_engine → recommender
```

The frontend contains no analysis logic; the backend is the single source of truth. If `ATS_API_URL` is
unset, the frontend starts the backend itself on `127.0.0.1` (single-deployment mode), so only the Streamlit
port is public.

## Features
- PDF/DOCX upload with extension, MIME, magic-byte, size (default 5 MB), page-count and decompressed-size checks;
  empty, malformed, encrypted and scanned (no text layer) files are rejected with clear messages (no OCR).
- Skill taxonomy (`app/data/skills.json`, ~125 skills) with aliases, boundary-aware matching (no `Java`⊂`JavaScript`,
  `SQL`⊂`NoSQL`), case-sensitive ambiguous aliases (`ML`, `Swift`, …) and skill *families*.
- Job-description analysis: required vs preferred (headers and inline cues such as "nice to have", "a plus"),
  experience and qualification lines (shown, not scored).
- **Exact** vs **related** (same family, partial credit) vs **missing**, with resume evidence snippets and whether
  a skill is shown in context or only in a skills list.
- TF-IDF cosine similarity (default); optional sentence-embedding backend behind the same interface.
- Recommendations that never suggest adding skills you do not have.

## Scoring
`overall = 100 × Σ(wᵢ·vᵢ) ÷ Σ(wᵢ)` over the components that are available (weights renormalised if one is missing).

| Component | Default weight | Value |
|---|---|---|
| Skill coverage | 0.60 | Σ(importance × credit) ÷ Σ(importance); exact = 1, related = 0.5, missing = 0; importance: required 1.0, unspecified 0.75, preferred 0.5 |
| Text similarity | 0.25 | TF-IDF cosine ÷ 0.45, capped at 1 (the 0.45 saturation is a heuristic, not calibrated) |
| Evidence in context | 0.15 | Share of exactly matched skills (importance-weighted) found outside a skills list |

If the job description has no recognised skills, skill coverage and evidence are unavailable and the score
reflects text similarity only (a warning is returned). All weights are configurable (below).

## Local setup
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
cp .env.example .env                                   # optional
streamlit run app/frontend.py                          # UI; starts the backend automatically
# or run the backend alone (OpenAPI docs at http://127.0.0.1:8000/docs):
uvicorn app.main:app --port 8000
# separate frontend/backend:   ATS_API_URL=http://127.0.0.1:8000 streamlit run app/frontend.py
```
Developed and tested on Python 3.12 (Linux). The frontend uses 3.10+ syntax.

## Environment variables (prefix `ATS_`, see `.env.example`)
`MAX_UPLOAD_MB` (5), `MAX_PDF_PAGES` (15), `MAX_JD_CHARS` (20000), `SEMANTIC_BACKEND` (`tfidf`|`embeddings`),
`WEIGHT_SKILLS/SEMANTIC/EVIDENCE`, `WEIGHT_REQUIRED/PREFERRED/UNSPECIFIED`, `RELATED_CREDIT`,
`TFIDF_SATURATION`, `API_URL` (frontend only). Streamlit's own upload cap is set in `.streamlit/config.toml`.

## API
Interactive docs: `/docs` (Swagger) and `/openapi.json`.
- `GET /health` → `{status, version, semantic_backend, max_upload_mb}`
- `POST /analyze` (multipart: `file` = PDF/DOCX, `job_description` = text) → score breakdown, `matched_skills`,
  `related_skills`, `missing_skills`, `recommendations`, `job_summary`, `warnings`, `meta`.
  Errors return `{"detail": "..."}`: **413** too large, **415** unsupported type, **422** missing/blank job description,
  empty/malformed/encrypted/scanned file, **500** unexpected (generic message).
```bash
curl -F "file=@resume.pdf" -F "job_description=<jd.txt" http://127.0.0.1:8000/analyze
```

## Testing
```bash
pip install -r requirements-dev.txt && python -m pytest tests -q
```
Tests use synthetic resumes only (generated in `tests/conftest.py`). The optional embedding backend's model path is
not covered (needs a model download); only its fallback-to-TF-IDF behaviour is tested.

## Deployment
Limits below were checked against the providers' docs on 2026-10-17 and can change.

**Recommended: Streamlit Community Cloud (single deployment, free).** Docs list 690 MB minimum / 2.7 GB maximum memory and
sleep after 12 hours without traffic. Measured here: backend ≈ 230 MB, idle Streamlit ≈ 55 MB, so the app should fit.
1. Push this folder to a GitHub repository (requires a GitHub account; a public repo is simplest).
2. At share.streamlit.io choose **Create app**, select the repo/branch, main file `app/frontend.py`.
3. Advanced settings → Python 3.12. No secrets are required.
4. Deploy, then run a real upload through the public URL (not yet done for this project).
The first request after sleep restarts the app and its backend subprocess (cold start). Whether Community Cloud
permits the frontend's backend subprocess has **not** been verified; if it does not, use option 2 or host the
backend separately and set `ATS_API_URL`.

**Fallback: Render (Docker, free).** `Dockerfile` + `render.yaml` are provided (New → Blueprint). Free instances have
512 MB RAM / 0.1 CPU, spin down after 15 minutes idle and take about a minute to wake; this app may be tight on
memory and slow on 0.1 CPU. The Docker image has not been built in this environment.

## Privacy and security
- Uploads are parsed in memory; the app does not store resumes and logs only metadata (type, size, counts, timing),
  never content. Starlette may buffer uploads over 1 MB in a temporary file for the duration of the request.
- The hosting platform can keep its own request logs; do not upload anything you would not share with the host.
- No secrets are needed or committed. In single-deployment mode the API listens on `127.0.0.1` only.
- No authentication or rate limiting: a public instance can be used by anyone and consumes CPU per request.

## Known limitations
- Only skills in the taxonomy are detected; add entries to `app/data/skills.json` (duplicate aliases fail fast on load).
- No negation, recency or proficiency detection ("no Docker experience" still matches Docker).
- "Related" uses coarse families (e.g. PostgreSQL ↔ SQL ↔ MySQL earn partial credit, not full).
- Ambiguous capitalised words at sentence start (Swift, Rust, Ruby, Dart, Excel, Spark) can false-match.
- Required/preferred detection relies on English headers/cues; unusual layouts fall back to "unspecified".
- Multi-column PDFs may extract out of order; DOCX text boxes and headers/footers are not read; scanned PDFs need OCR (unsupported).
- The embedding backend (`requirements-embeddings.txt`, `ATS_SEMANTIC_BACKEND=embeddings`) is implemented but was not
  evaluated or run end-to-end here; whether it improves results enough to justify its memory/startup cost is unknown.
