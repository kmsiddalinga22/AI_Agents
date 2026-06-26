# AI Test Result Analyzer

An AI-powered web tool that uploads Playwright test result JSON files, sends them to a **Langflow + Groq llama-3.3-70b** flow, and renders a structured failure report — with summary stats, classified error buckets, and a per-test failure table.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Quick Start — Local Dev](#quick-start--local-dev)
- [Environment Variables](#environment-variables)
- [How the AI Analysis Works](#how-the-ai-analysis-works)
- [Error Bucket Categories](#error-bucket-categories)
- [Required Input Format](#required-input-format)
- [UI Walkthrough](#ui-walkthrough)
- [Vercel Deployment](#vercel-deployment)
- [Screenshots](#screenshots)
- [Git Check-In Guidance](#git-check-in-guidance)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Install and verify these before you start:

| Requirement | Minimum Version | How to check |
|-------------|-----------------|--------------|
| Node.js | 18 | `node -v` |
| npm | 9 | `npm -v` |
| Python | 3.10+ | `python --version` |
| Langflow | latest | `langflow --version` |

### Install Langflow

```bash
pip install langflow
langflow run
```

Langflow starts at `http://localhost:7860`. You need a Langflow flow running with the prompt from `AI_TestResult-Analysis_Prompt.md`. Once the flow is created, copy its **flow ID** from the Langflow API panel — you'll need it for your `.env`.

---

## Tech Stack

| Layer | Technology | Details |
|-------|-----------|---------|
| Frontend | React 18 + Vite 8 | Port 5171 in dev |
| Backend (local) | Node.js + Express 4 | Port 3002, handles file upload via Multer |
| Backend (Vercel) | Vercel Serverless Function | `api/analyze.js` using Formidable |
| AI orchestration | Langflow | Runs the prompt flow, proxies to Groq |
| AI model | Groq — llama-3.3-70b | Fast inference via Langflow's Groq component |
| File upload | Multer (local) / Formidable (Vercel) | JSON only, 10 MB max |

---

## Architecture

```
┌─────────────────────────────┐
│   Browser  (Vite :5171)     │
│  - Drag/drop or browse JSON │
│  - Click "Run AI Analysis"  │
└────────────┬────────────────┘
             │  POST /api/analyze  (multipart/form-data)
             ▼
┌─────────────────────────────┐
│  Express API  (:3002)       │
│  - Saves file → temp/       │
│  - Forwards to Langflow     │
│  - Deletes temp file after  │
└────────────┬────────────────┘
             │  POST to Langflow REST API
             ▼
┌─────────────────────────────┐
│  Langflow  (:7860)          │
│  - Runs AI_TestResult       │
│    Analysis_Prompt.md       │
│  - Calls Groq llama-3.3-70b │
└────────────┬────────────────┘
             │  Structured text response
             ▼
┌─────────────────────────────┐
│  React UI parses & renders  │
│  - Summary cards            │
│  - Error bucket grid        │
│  - Failed test table        │
│  - Raw AI output (toggle)   │
└─────────────────────────────┘
```

**Vercel production** replaces the Express server with `api/analyze.js` (Vercel Serverless Function). The React build is served as a static site; `vercel.json` rewrites all non-API routes to `index.html` for SPA routing.

---

## Project Structure

```
TestResult_Analysis_AI_Agent/
│
├── src/                              # React frontend
│   ├── main.jsx                      # ReactDOM entry — mounts <App />
│   ├── App.jsx                       # All UI logic: upload, fetch, parse, render
│   ├── App.css                       # Component styles (cards, table, buckets)
│   └── index.css                     # Global reset + body styles
│
├── api/
│   └── analyze.js                    # Vercel serverless function (POST /api/analyze)
│                                     # Uses Formidable for multipart parsing
│
├── screenshots/
│   ├── LangFlow_Diagram.png          # Langflow flow diagram
│   └── Test_Result_Analysis.png      # Example analyzer UI output
│
├── server.js                         # Local Express dev server
│                                     # - Multer file upload → temp/
│                                     # - Proxies to Langflow
│                                     # - Serves built React app from dist/
│
├── index.html                        # Vite HTML shell — loads /src/main.jsx
├── vite.config.js                    # Vite: port 5171, /api proxy → :3002
├── vercel.json                       # Build config + SPA rewrite rules
├── package.json                      # Scripts: dev, build, start, preview
│
├── AI_TestResult-Analysis_Prompt.md  # System prompt sent to Groq via Langflow
│                                     # Defines 9 error buckets + output format
├── Langflow_Postman_Collection_v2.json  # Postman collection for API testing
├── TestResult_Sanity.json            # Sample Playwright JSON — use to verify setup
│
├── .env                              # Local secrets — DO NOT COMMIT
├── .env.example                      # Template — commit this, not .env
└── .gitignore                        # Excludes node_modules, dist, temp, .env
```

> `temp/` is created automatically by Express to store uploads; it is cleaned up after each request. `dist/` is created by `npm run build`.

---

## Quick Start — Local Dev

### Step 1 — Clone and install

```bash
cd TestResult_Analyzer/TestResult_Analysis_AI_Agent
npm install
```

### Step 2 — Configure environment

```bash
cp .env.example .env
```

Open `.env` and fill in your Langflow details:

```env
LANGFLOW_URL=http://localhost:7860/api/v1/run/<your-flow-id>?stream=false
LANGFLOW_API_KEY=your-langflow-api-key-here
```

> See [Environment Variables](#environment-variables) for exactly where to find these in Langflow.

### Step 3 — Start Langflow

```bash
langflow run
```

Open `http://localhost:7860`, import your flow (or build one using `AI_TestResult-Analysis_Prompt.md`), and confirm it is running.

### Step 4 — Start the app

```bash
npm run dev
```

This runs **two servers in parallel** via `concurrently`:

| Server | URL | Role |
|--------|-----|------|
| Express API | `http://localhost:3002` | Handles `/api/analyze` — file upload + Langflow call |
| Vite dev server | `http://localhost:5171` | React UI — proxies `/api/*` to Express |

Open `http://localhost:5171` in your browser.

### Step 5 — Test the setup

Upload `TestResult_Sanity.json` (included in the repo) and click **Run AI Analysis**. You should see summary cards and a failure table within 10–30 seconds.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `LANGFLOW_URL` | Yes | Full Langflow run endpoint with your flow ID |
| `LANGFLOW_API_KEY` | Yes | API key from your Langflow instance |

**Where to find these in Langflow:**

1. Open `http://localhost:7860`
2. Open your flow → click **API** (top-right)
3. Copy the run URL — it contains your flow ID, e.g.:
   `http://localhost:7860/api/v1/run/fea6bd7a-42bc-41ea-a85e-5dd48087dc19?stream=false`
4. Go to **Settings → API Keys** → create or copy your API key

**For Vercel production:** add both variables in **Vercel Dashboard → Settings → Environment Variables**. The `api/analyze.js` serverless function reads from `process.env` and falls back to hardcoded values only if the env vars are missing.

> **Security:** Never commit real credentials. The `.env` file is excluded by `.gitignore`. Only commit `.env.example`.

---

## How the AI Analysis Works

The analysis follows this pipeline:

1. **Upload** — React sends the JSON file via `POST /api/analyze` (multipart form)
2. **Save** — Express writes the file to `temp/<timestamp>-filename.json`
3. **Prompt** — Express calls Langflow, passing the file path and a session ID
4. **Langflow** — runs `AI_TestResult-Analysis_Prompt.md` through Groq llama-3.3-70b
5. **Response** — Langflow returns a structured text response at:
   `response.outputs[0].outputs[0].results.message.text`
6. **Parse** — `App.jsx` parses the AI text into three sections using regex:
   - `SUMMARY` → counts and pass rate
   - `ERROR_BUCKET_SUMMARY` → bucket name + count per line
   - `FAILED_TEST_DETAILS` → numbered blocks with test name, error, retry, bucket
7. **Render** — React displays the three UI sections
8. **Cleanup** — Express deletes the temp file after the request completes

### AI prompt output format

The AI returns six labeled sections in this order:

```
**FLAKY_TESTS**
**CONSISTENT_FAILURES**
**RERUN_RECOMMENDATION**
**ERROR_CLASSIFICATION**
**SUMMARY**
**ERROR_BUCKET_SUMMARY**
```

The UI reads `SUMMARY` and `ERROR_BUCKET_SUMMARY` for cards, then parses inline `FAILED_TEST_DETAILS` for the table.

---

## Error Bucket Categories

The AI classifies each failed test error into one of 9 buckets using keyword matching (defined in `AI_TestResult-Analysis_Prompt.md`):

| # | Bucket | Triggered by error containing |
|---|--------|-------------------------------|
| 1 | **TestScript_UIChange** | `Invalid syntax`, `NoSuchElementException`, `Did not find element`, `Failed - negative check criteria` |
| 2 | **Assertion** | `Log:Assertion`, `Failed to integrate` |
| 3 | **Performance** | `TimeoutError`, `TimeoutException`, `Timeout 60000ms`, `Alert did not appear within`, `Page did not load within`, `Unable to navigate to` |
| 4 | **Build** | `Service Unavailable`, `Internal Server Error`, `Temporarily down for maintenance` |
| 5 | **UnexpectedAlertMsg** | `UnhandledAlertException`, `Found an unexpected alert`, `unexpected alert open` |
| 6 | **Application** | `Did not find record`, `Unable to locate element`, `Element not found`, `No records found`, `ElementNotVisibleException` |
| 7 | **UIFrameWork** | `WebDriverException occurred`, `File download location` |
| 8 | **Compare_PDF_Excel_EMail** | `Expected and actual files are not same`, `Error while fetching mail from Webmail` |
| 9 | **Unknown** | No keyword above matched |

The bucket assigned by the AI is what appears in the UI. The frontend also has a local fallback classifier in `App.jsx` for cases where the AI output is missing a bucket field.

---

## Required Input Format

The analyzer expects a **Playwright JSON reporter** output. Key fields the AI prompt reads:

```
stats.expected    → passed count
stats.unexpected  → failed count
stats.flaky       → flaky count
stats.skipped     → skipped count
stats.duration    → run time in ms

root.suites[].suites[].specs[]
  spec.ok                              → false = failed
  spec.title                           → test name
  spec.tests[].results[].status        → "failed" | "passed" | "unexpected"
  spec.tests[].results[].errors[].message  → error text
  spec.tests[].results[].retry         → retry count
```

**Minimal valid example:**

```json
{
  "stats": {
    "expected": 47,
    "unexpected": 8,
    "flaky": 0,
    "skipped": 0,
    "duration": 12345
  },
  "root": {
    "suites": [
      {
        "specs": [
          {
            "title": "redirects to dashboard after successful login",
            "ok": false,
            "tests": [
              {
                "results": [
                  {
                    "status": "failed",
                    "retry": 0,
                    "errors": [
                      { "message": "TimeoutError: page.waitForURL: Timeout 60000ms exceeded." }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

Use `TestResult_Sanity.json` (included) as a working reference file.

---

## UI Walkthrough

### Upload panel

- Click to browse or **drag and drop** a `.json` file onto the drop zone
- File name and size are shown after selection; click **✕** to remove
- Only `.json` files are accepted (validated client-side and server-side)
- Max file size: **10 MB**

### Running analysis

1. Click **▶ Run AI Analysis**
2. A spinner shows while Langflow processes (typically 10–30 seconds)
3. On error, a red alert bar shows the error message

### Output sections

| Section | What it shows |
|---------|--------------|
| **Summary** | Total / Passed / Failed / Skipped counts + visual pass-rate progress bar + failure rate sentence |
| **Error Bucket Summary** | Color-coded cards per bucket (icon + count + category name) |
| **Failed Test Details** | Table: # / Test Name / Error Message / Bucket pill |
| **Raw AI Output** *(collapsible)* | Full AI response text — useful for debugging |

### Bucket color coding

| Bucket | Color |
|--------|-------|
| Performance | Orange |
| Assertion | Yellow/Amber |
| Application | Red/Pink |
| Unknown | Purple |
| Others | Indigo |

---

## Vercel Deployment

### 1. Understand `vercel.json`

```json
{
  "name": "testresutlanalyzer",
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "rewrites": [
    { "source": "/((?!api/).*)", "destination": "/index.html" }
  ]
}
```

All routes except `/api/*` are rewritten to `index.html` for SPA routing. The `api/analyze.js` file is automatically deployed as a Vercel Serverless Function.

### 2. Set environment variables

In **Vercel Dashboard → Settings → Environment Variables**, add:

| Key | Value |
|-----|-------|
| `LANGFLOW_URL` | Your hosted Langflow run endpoint |
| `LANGFLOW_API_KEY` | Your Langflow API key |

### 3. Deploy via CLI

```bash
cd TestResult_Analyzer/TestResult_Analysis_AI_Agent
vercel --prod
```

Or connect via Vercel Dashboard: import the repository and set **Root Directory** to `TestResult_Analyzer/TestResult_Analysis_AI_Agent`.

### 4. Verify

After deployment:
- Open your Vercel URL — React app should load
- `POST <vercel-url>/api/analyze` should return JSON
- Upload `TestResult_Sanity.json` to confirm end-to-end output

---

## Screenshots

**Langflow flow diagram** — shows how the Langflow flow is wired up:

![Langflow flow screenshot](screenshots/LangFlow_Diagram.png)

**Analyzer UI output** — example output after uploading a test result file:

![Analyzer UI output screenshot](screenshots/Test_Result_Analysis.png)

---

## Git Check-In Guidance

### Commit these files

```
src/
api/
screenshots/
index.html
server.js
vite.config.js
vercel.json
package.json
package-lock.json
AI_TestResult-Analysis_Prompt.md
Langflow_Postman_Collection_v2.json
TestResult_Sanity.json
.env.example
.gitignore
README.md
```

### Do NOT commit these

```
node_modules/     ← auto-installed by npm install
dist/             ← auto-generated by npm run build
temp/             ← runtime upload folder, auto-created
.vercel/          ← Vercel CLI metadata
.env              ← contains real API keys and secrets
```

All exclusions are already in `.gitignore`.

---

## Troubleshooting

**`ECONNREFUSED` or `Failed to contact Langflow`**
- Confirm Langflow is running: visit `http://localhost:7860` in your browser.
- Confirm the flow ID in `LANGFLOW_URL` matches an active flow in Langflow.
- Check that `LANGFLOW_API_KEY` is valid (Langflow → Settings → API Keys).

**`Only JSON files are allowed`**
- Only `.json` files are accepted. Rename or re-export your file with a `.json` extension.

**`Unexpected response structure from Langflow`**
- The UI expects: `response.outputs[0].outputs[0].results.message.text`
- Open the **Raw AI Output** toggle in the UI to inspect the actual response.
- Check that your Langflow flow outputs a Chat Message component as its final node.

**Blank page or API not reachable**
- Use `npm run dev` in local development — this starts **both** Vite (`:5171`) and Express (`:3002`). Running `npm start` alone starts only Express; the React UI will not be available.
- Vite proxies `/api/*` to `http://localhost:3002`. If port 3002 is already in use, change `PORT` in `server.js` and update `target` in `vite.config.js` to match.

**Analysis takes too long or times out**
- The server waits up to 120 seconds for Langflow (`timeout: 120000` in `server.js` and `api/analyze.js`).
- Large JSON files and heavy Langflow flows can approach this. Try trimming the input or simplifying the flow.

**Port conflicts on startup**
- Vite port: change `--port 5171` in the `dev` script in `package.json`.
- Express port: change `PORT = 3002` in `server.js` and update `vite.config.js` → `server.proxy['/api'].target`.
