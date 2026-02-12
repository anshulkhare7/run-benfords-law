# Benford's Law Web App — Implementation Plan

## Overview

Rebuild the existing Java CLI tool as a general-purpose web application using **FastAPI** (Python backend) and **React** (frontend). The plan is organized into four phases, each delivering a working increment. Phase 1 is a minimal but accurate MVP; subsequent phases layer on richer file formats, statistical tests, and production features.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | Python 3.11+, FastAPI, Uvicorn | Async-native, automatic OpenAPI docs, rich data science ecosystem |
| Frontend | React 18, Vite, Chart.js (via react-chartjs-2) | Fast dev server, lightweight charting, widely known |
| Testing (backend) | pytest, httpx (for async test client) | Standard Python testing stack |
| Testing (frontend) | Vitest, React Testing Library | Vite-native, fast, mirrors Jest API |
| Linting | Ruff (Python), ESLint + Prettier (JS) | Fast, opinionated, low config |

---

## Project Structure (Target)

```
benford-app/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app entry point, CORS config
│   │   ├── routers/
│   │   │   └── analyze.py       # /api/analyze endpoints
│   │   ├── core/
│   │   │   ├── parser.py        # Extract numbers from raw text (Phase 1), CSV, PDF (later)
│   │   │   ├── analyzer.py      # Compute digit distribution, compare to Benford expected
│   │   │   └── benford.py       # Benford constants + formulas (expected distribution, MAD, chi-sq)
│   │   └── models/
│   │       └── schemas.py       # Pydantic request/response models
│   ├── tests/
│   │   ├── test_parser.py
│   │   ├── test_analyzer.py
│   │   └── test_api.py
│   ├── requirements.txt
│   └── pyproject.toml
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── FileUpload.jsx        # Drag-and-drop file upload
│   │   │   ├── ResultsTable.jsx      # Observed vs. expected digit table
│   │   │   ├── BenfordChart.jsx      # Bar chart comparing observed vs. expected
│   │   │   └── AnalysisSummary.jsx   # Overall verdict / conformity banner
│   │   ├── services/
│   │   │   └── api.js                # Axios/fetch wrapper for backend calls
│   │   └── pages/
│   │       └── HomePage.jsx          # Single-page layout combining upload + results
│   ├── tests/
│   ├── package.json
│   └── vite.config.js
└── README.md
```

---

## Phase 1 — Working MVP (Plain Text)

**Goal**: A user can upload a `.txt` file, see the first-digit frequency distribution, and compare it visually against Benford's expected distribution. Results must be correct — leading significant digits only (1–9), not leading zeros.

### Task 1.1 — Backend project scaffolding

- Initialize a Python project in `backend/` with `pyproject.toml`
- Install and configure FastAPI, Uvicorn, pytest, httpx, Ruff
- Create `requirements.txt` with pinned versions:
  - `fastapi`, `uvicorn[standard]`, `python-multipart` (for file uploads)
- Create `app/main.py`:
  - Instantiate `FastAPI()` app
  - Add CORS middleware allowing `http://localhost:5173` (Vite dev server)
  - Include the `analyze` router
- Verify: `uvicorn app.main:app --reload` starts without errors
- Verify: `GET /docs` returns Swagger UI

**Acceptance criteria**: Server starts, `/docs` loads, CORS is configured.

### Task 1.2 — Number extraction module (`core/parser.py`)

- Write function: `extract_numbers(text: str) -> list[float]`
  - Use regex `r'-?\d[\d,]*\.?\d*'` to find all numeric tokens in freeform text
  - Strip commas (e.g., `1,000,000` -> `1000000`)
  - Convert to float
  - **Skip zeros** — Benford's Law doesn't apply to the value 0
  - Return list of floats
- Edge cases to handle:
  - Negative numbers: use absolute value
  - Decimals like `0.005`: the leading significant digit is `5`, not `0` (handled in analyzer, but parser should still extract the value `0.005`)
  - Numbers with commas: `12,345` -> `12345.0`
  - Ignore non-numeric tokens entirely

- Write unit tests in `tests/test_parser.py`:
  - Empty string -> empty list
  - Pure text with no numbers -> empty list
  - `"Revenue was 1,234,567 in 2023"` -> `[1234567.0, 2023.0]`
  - `"Growth of -3.5% from 0.005 to 100"` -> `[3.5, 0.005, 100.0]`
  - `"Page 1 of 50"` -> `[1.0, 50.0]`
  - The sample file `data/reliance-AR.txt` — verify it extracts > 0 numbers

**Acceptance criteria**: All unit tests pass. Function handles commas, decimals, negatives.

### Task 1.3 — Benford analysis module (`core/analyzer.py` + `core/benford.py`)

**`core/benford.py`** — Pure constants, no logic:

```python
# Benford's Law expected distribution for first significant digit
# P(d) = log10(1 + 1/d) for d in 1..9
EXPECTED_DISTRIBUTION = {
    1: 0.30103,
    2: 0.17609,
    3: 0.12494,
    4: 0.09691,
    5: 0.07918,
    6: 0.06695,
    7: 0.05799,
    8: 0.05115,
    9: 0.04576,
}
```

**`core/analyzer.py`**:

- Write function: `get_leading_digit(n: float) -> int | None`
  - Return the first significant digit (1–9) of the number
  - For `0.005` -> `5`, for `1234` -> `1`, for `-42` -> `4`
  - For `0` or `0.0` -> return `None` (skip)

- Write function: `compute_distribution(numbers: list[float]) -> dict`
  - Call `get_leading_digit` on each number
  - Count occurrences of each digit 1–9
  - Return a dict like:
    ```python
    {
      "total_count": 1500,
      "digit_counts": {1: 452, 2: 263, ...},
      "observed_distribution": {1: 0.3013, 2: 0.1753, ...},
      "expected_distribution": {1: 0.30103, 2: 0.17609, ...},
    }
    ```
  - `observed_distribution` values are `digit_count / total_count`

- Write unit tests in `tests/test_analyzer.py`:
  - `get_leading_digit(1234)` -> `1`
  - `get_leading_digit(0.005)` -> `5`
  - `get_leading_digit(-42.7)` -> `4`
  - `get_leading_digit(0)` -> `None`
  - `get_leading_digit(9)` -> `9`
  - `compute_distribution([1, 2, 3, ..., 9])` — with known inputs, verify counts and percentages
  - `compute_distribution([])` — empty list returns zero counts
  - With a dataset that perfectly follows Benford's — verify observed matches expected within tolerance

**Acceptance criteria**: All tests pass. Leading significant digit extraction is correct for decimals, negatives, and zero.

### Task 1.4 — Pydantic schemas (`models/schemas.py`)

- Define response model:
  ```python
  class DigitResult(BaseModel):
      digit: int
      count: int
      observed_pct: float   # e.g., 30.1
      expected_pct: float   # e.g., 30.103
      difference: float     # observed - expected

  class AnalysisResponse(BaseModel):
      filename: str
      total_numbers: int
      digits: list[DigitResult]  # always 9 items, digits 1–9
  ```

**Acceptance criteria**: Models are importable and serializable to JSON.

### Task 1.5 — API endpoint (`routers/analyze.py`)

- `POST /api/analyze`
  - Accepts: `UploadFile` (multipart form)
  - Validates: file is `.txt`, file size < 10MB
  - Flow:
    1. Read file content as UTF-8 string
    2. Call `extract_numbers(text)`
    3. Call `compute_distribution(numbers)`
    4. Map result to `AnalysisResponse`
    5. Return JSON
  - Error cases:
    - No file uploaded -> 422
    - Non-.txt file -> 400 with message `"Only .txt files are supported in this version"`
    - File too large -> 400 with message `"File must be under 10MB"`
    - No numbers found in file -> 200 with `total_numbers: 0` and empty `digits`

- Write integration tests in `tests/test_api.py`:
  - Upload a small .txt file with known numbers, verify response structure and values
  - Upload a non-.txt file, verify 400
  - Upload empty .txt file, verify `total_numbers: 0`

**Acceptance criteria**: Endpoint works end-to-end. Swagger UI can be used to upload a file and see results.

### Task 1.6 — Frontend project scaffolding

- Initialize React project with Vite in `frontend/`:
  ```bash
  npm create vite@latest frontend -- --template react
  ```
- Install dependencies: `react-chartjs-2`, `chart.js`, `axios`
- Configure Vite proxy to forward `/api/*` to `http://localhost:8000`
- Create basic `App.jsx` that renders `<HomePage />`
- Verify: `npm run dev` starts, shows a blank page without errors

**Acceptance criteria**: Dev server starts, proxies to backend, renders without errors.

### Task 1.7 — File upload component (`FileUpload.jsx`)

- Build a file upload area:
  - A dashed-border drop zone with text: "Drop a .txt file here, or click to browse"
  - On file select or drop: validate it's `.txt`, show filename, show "Analyze" button
  - On "Analyze" click: POST to `/api/analyze` as multipart form data
  - Show a loading spinner/state while waiting for response
  - On error: show the error message from the API
  - On success: pass the response data up to the parent via a callback prop

- Keep it simple — no drag-and-drop library needed; native HTML `<input type="file" accept=".txt">` plus basic `onDrop` handler is enough.

**Acceptance criteria**: User can select a .txt file, click Analyze, and the API is called. Errors are shown. Loading state is visible.

### Task 1.8 — Results table (`ResultsTable.jsx`)

- Render a table with columns: **Digit**, **Count**, **Observed %**, **Expected %**, **Difference**
- 9 rows, one per digit (1–9)
- Format percentages to 1 decimal place (e.g., "30.1%")
- Color the Difference column: green if |difference| < 2%, yellow if 2–5%, red if > 5%
- Show `total_numbers` and `filename` above the table

**Acceptance criteria**: Table renders correctly from API response data. Difference coloring works.

### Task 1.9 — Bar chart (`BenfordChart.jsx`)

- Use Chart.js (via `react-chartjs-2`) to render a grouped bar chart:
  - X-axis: digits 1–9
  - Two bars per digit: blue = Observed %, orange = Expected %
  - Y-axis: percentage (0–35%)
  - Legend: "Observed" and "Expected (Benford's Law)"
- Chart should be responsive (fill container width)

**Acceptance criteria**: Chart renders and visually matches the expected Benford curve shape when given a conforming dataset.

### Task 1.10 — Assemble the home page (`HomePage.jsx`)

- Layout (top to bottom):
  1. Header: app name "Benford's Law Analyzer" + one-line description
  2. `<FileUpload />` component
  3. Once results are returned: `<ResultsTable />` and `<BenfordChart />` side by side (or stacked on mobile)
- State management: `useState` in HomePage holds the API response; passed down as props
- No routing needed — single page

**Acceptance criteria**: Full flow works end-to-end: upload file -> see table + chart. Page is usable on desktop and doesn't break on mobile.

### Task 1.11 — End-to-end test with sample data

- Use the existing `data/reliance-AR.txt` as the test file
- Manually verify:
  - Total numbers extracted is reasonable (hundreds to thousands)
  - Digit 1 has the highest count (~30%)
  - Digit 9 has the lowest count (~4-5%)
  - The chart visually shows the classic Benford logarithmic decline
- Write one backend integration test that uploads `reliance-AR.txt` and asserts digit 1 has the highest observed percentage

**Acceptance criteria**: The sample data produces a distribution that visually and numerically resembles Benford's Law.

### Phase 1 Deliverable

A working web app where a user can:
1. Open `http://localhost:5173` in a browser
2. Upload any `.txt` file containing numbers
3. See a table and chart comparing observed first-digit distribution vs. Benford's expected distribution
4. Visually assess whether the data conforms to Benford's Law

---

## Phase 2 — CSV Support + Conformity Scoring

**Goal**: Support structured data (CSV), let the user pick which column to analyze, and provide a simple conformity verdict.

### Task 2.1 — CSV parsing (`core/parser.py`)

- Add function: `extract_numbers_from_csv(file_bytes: bytes, column: str | None) -> list[float]`
  - Use `pandas.read_csv()`
  - If `column` is specified: extract numbers from that column only
  - If `column` is None: extract numbers from all numeric columns
  - Return list of floats (same format as `extract_numbers`)
- Return the list of available column names alongside the numbers so the frontend can offer column selection

### Task 2.2 — Column selection API

- Modify or add endpoint: `POST /api/preview`
  - Accepts CSV upload
  - Returns: list of column names + first 10 rows of data (for preview)
- Modify `POST /api/analyze`:
  - Accept optional `column` query parameter
  - Accept `.csv` files in addition to `.txt`

### Task 2.3 — Data preview component (`DataPreview.jsx`)

- After uploading a CSV, show:
  - A preview table (first 10 rows)
  - A dropdown to select which column to analyze
  - An "Analyze" button that sends the column choice to the API
- For .txt files: skip preview, go straight to analysis (same as Phase 1)

### Task 2.4 — MAD conformity score (`core/analyzer.py`)

- Implement Mean Absolute Deviation (MAD):
  ```
  MAD = (1/9) * Σ |observed_pct(d) - expected_pct(d)| for d in 1..9
  ```
- Add conformity bands to the response:
  - MAD < 0.006 → "Close conformity"
  - 0.006 ≤ MAD < 0.012 → "Acceptable conformity"
  - 0.012 ≤ MAD < 0.015 → "Marginal conformity"
  - MAD ≥ 0.015 → "Nonconformity"
- Add to `AnalysisResponse`: `mad_score: float`, `conformity: str`

### Task 2.5 — Conformity banner (`AnalysisSummary.jsx`)

- Display a colored banner above the results:
  - Green: "Close conformity" / "Acceptable conformity"
  - Yellow: "Marginal conformity"
  - Red: "Nonconformity — this dataset deviates significantly from Benford's Law"
- Show the MAD score
- Add a small info tooltip explaining what MAD means and that nonconformity ≠ fraud

### Phase 2 Deliverable

Users can upload `.txt` or `.csv` files, optionally select a column, and get a clear conformity verdict (not just raw numbers).

---

## Phase 3 — Statistical Tests + More File Formats

**Goal**: Add rigorous statistical testing and support PDF/Excel uploads.

### Task 3.1 — Chi-squared test (`core/analyzer.py`)

- Implement chi-squared goodness-of-fit test:
  ```
  χ² = Σ (observed_count(d) - expected_count(d))² / expected_count(d)
  ```
  - Use `scipy.stats.chisquare` or implement manually
  - Degrees of freedom = 8 (9 digits minus 1)
  - Return: chi-squared statistic, p-value, pass/fail at α = 0.05
- Add to response: `chi_squared: float`, `p_value: float`, `chi_squared_result: str`

### Task 3.2 — First-two-digit analysis

- Extend `analyzer.py` with `compute_first_two_digit_distribution(numbers)`
  - For each number, extract the first two significant digits (10–99)
  - Benford's expected: `P(d) = log10(1 + 1/d)` where d ranges from 10 to 99
  - Return same structure as first-digit but with 90 buckets instead of 9
- New endpoint or query param: `POST /api/analyze?mode=first_two_digits`
- Frontend: add a toggle between "First digit" and "First two digits" views

### Task 3.3 — PDF file support

- Add `pdfplumber` dependency
- Add function: `extract_text_from_pdf(file_bytes: bytes) -> str`
- Wire into the `/api/analyze` endpoint: accept `.pdf`, extract text, then run through `extract_numbers()`
- File size limit: 50MB for PDFs

### Task 3.4 — Excel file support

- Add `openpyxl` dependency
- Add function: `extract_numbers_from_excel(file_bytes: bytes, column: str | None) -> list[float]`
  - Use `pandas.read_excel()`
  - Same column-selection logic as CSV
- Wire into `/api/analyze`: accept `.xlsx`/`.xls`

### Task 3.5 — Enhanced results page

- Add tabs: "First Digit" | "First Two Digits"
- Show statistical test results (chi-squared, p-value) below the chart
- Add an "Interpretation" section explaining what each test result means in plain language

### Phase 3 Deliverable

A statistically rigorous tool that handles common file formats and gives users actionable test results with explanations.

---

## Phase 4 — Production Features

**Goal**: Persistence, export, batch analysis, and deployment.

### Task 4.1 — Database + analysis history

- Add SQLite (or PostgreSQL for production) via SQLAlchemy
- Store each analysis: timestamp, filename, results JSON, conformity verdict
- `GET /api/analyses` — list past analyses
- `GET /api/analyses/:id` — retrieve a specific analysis
- Frontend: add a `/history` page showing past analyses with links to results

### Task 4.2 — Export results

- `GET /api/analyses/:id/export?format=pdf` — generate a PDF report (use `reportlab` or `weasyprint`)
  - Include: chart image, table, statistical test results, conformity verdict
- `GET /api/analyses/:id/export?format=csv` — export raw digit counts as CSV
- Frontend: "Download Report" button on results page

### Task 4.3 — Drill-down: numbers behind each digit

- Extend the analysis response to include `sample_numbers` per digit — e.g., the first 20 numbers that contributed to each digit's count
- Frontend: clicking a bar in the chart or a row in the table opens a panel showing those actual numbers
- Helps users verify data quality and understand what's driving the distribution

### Task 4.4 — Multiple dataset comparison

- Allow uploading 2+ files and comparing their distributions on one chart
- Useful for comparing year-over-year reports or different companies
- New endpoint: `POST /api/compare` accepting multiple files
- Frontend: overlay chart with one color per dataset

### Task 4.5 — Docker + deployment

- Create `Dockerfile` for backend (Python + Uvicorn)
- Create `Dockerfile` for frontend (Node build -> Nginx static serve)
- Create `docker-compose.yml` wiring both together
- Add a `Makefile` or `justfile` with common commands: `make dev`, `make build`, `make test`
- Document deployment to a cloud provider (e.g., Fly.io, Railway, or AWS ECS)

### Phase 4 Deliverable

A production-ready application with persistence, reporting, and easy deployment.

---

## Key Design Principles

1. **Separation of concerns**: The analysis engine (`core/`) has zero knowledge of HTTP or React. It takes numbers in and returns results out. This makes it testable, reusable, and swappable.

2. **Accuracy first**: The leading significant digit extraction must correctly handle decimals (`0.005` -> `5`), negatives, and zeros. Get this wrong and every result is wrong. This is why Task 1.3 has the most unit tests.

3. **No false authority**: The app must clearly state that Benford nonconformity is an indicator for further investigation, not proof of fraud. This disclaimer should appear on the results page.

4. **Progressive complexity**: Phase 1 has no database, no auth, no complex file parsing. Each phase adds one layer of complexity. A junior developer should be able to complete any single task in 1–3 days.

---

## Estimated Task Sizing

| Size | Description | Examples |
|------|-------------|---------|
| **S** (half day) | Single function + tests, or single UI component | Task 1.2, 1.4, 2.4 |
| **M** (1–2 days) | Module with multiple functions, or component with API integration | Task 1.5, 1.7, 1.9 |
| **L** (2–3 days) | End-to-end feature spanning backend + frontend | Task 2.2+2.3, 3.2, 4.4 |

Phase 1: ~10–12 working days for one developer (parallelizable: backend tasks 1.1–1.5 and frontend tasks 1.6–1.10 can run concurrently with two developers).

---

## Dependencies Between Tasks

```
Phase 1:
  1.1 (backend scaffold) ──┬── 1.2 (parser) ── 1.3 (analyzer) ── 1.4 (schemas) ── 1.5 (API endpoint)
                           │                                                              │
  1.6 (frontend scaffold) ─┴── 1.7 (upload) ───────────────────────────────────── 1.10 (home page) ── 1.11 (e2e test)
                                    │                                                  │
                                    ├── 1.8 (results table) ──────────────────────────┘
                                    └── 1.9 (bar chart) ──────────────────────────────┘

Phase 2: Depends on Phase 1 completion
  2.1 (CSV parser) ── 2.2 (column selection API) ── 2.3 (data preview UI)
  2.4 (MAD score) ── 2.5 (conformity banner)

Phase 3: Depends on Phase 2 completion
  3.1 (chi-squared)  ]
  3.2 (two-digit)    ] ── 3.5 (enhanced results page)
  3.3 (PDF support)  ]
  3.4 (Excel support) ]

Phase 4: Depends on Phase 3 completion
  4.1 (database) ── 4.2 (export) ── 4.5 (Docker)
  4.3 (drill-down)
  4.4 (comparison)
```
