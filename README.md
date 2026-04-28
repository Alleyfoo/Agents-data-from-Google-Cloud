# Data Agents Demo

A small AI-style tool that takes messy spreadsheets (Excel or CSV files) and turns them into clean, well-organized data — the kind of cleanup an analyst would otherwise do by hand.

It looks at your file, figures out which row is actually the header (the row with the column names), tidies up the values, and saves a cleaned-up copy along with a clear record of every decision it made. You can use it from the command line, from a simple text-based menu, or from a small web app.

GitHub home: https://github.com/Alleyfoo/Data-agents-demo

## Why it matters

Real-world spreadsheets are rarely clean. Headers are buried a few rows down, columns have stray spaces, numbers are stored as text, and so on. This project automates that first painful pass while still asking a human when something is genuinely ambiguous — and it keeps a paper trail of every choice so the work can be reviewed or repeated later.

## Quickstart (command line)

You'll need Python 3.11 or newer, plus `git` and `pip` (Python's package installer). The commands below assume you're using PowerShell (the Windows terminal) from the project's main folder.

```powershell
python -m venv .venv
.\\.venv\\Scripts\\Activate.ps1
python -m pip install -r demos/requirements-demo.txt

# Try the included sample
python data_agents_cli.py run --input data/samples/sample_mini.csv --run-id demo --interactive

# Or point to your own file (it will ask if it's unsure about the header)
python data_agents_cli.py run --input path\\to\\your.xlsx --run-id demo --interactive

# Or run it in two steps (useful for scripting)
python data_agents_cli.py run --input path\\to\\your.xlsx --run-id demo
python data_agents_cli.py confirm --run-id demo --choice row_1
python data_agents_cli.py resume --run-id demo
```

Results are saved into `artifacts/<run-id>/`. You'll get a clean CSV (`clean.csv`), a description of the columns (`schema_spec.json`), and a detailed activity log (the "shadow log"). Any CSV or Excel file works — just point `--input` at it.

## Other ways to use it

- **Text-based menu (TUI)**: `python demos/tui_app.py --input path\\to\\your.xlsx --interactive`
- **Web demo (Streamlit)**: `streamlit run demos/streamlit_app.py`
- **Web "mapping studio"** (a richer web UI for column mapping): `streamlit run demos/streamlit_mapping_studio.py`
- **One-click launchers (Windows)**: `demos/run_tui_demo.bat`, `demos/run_streamlit_demo.bat`

## Running on your laptop vs. running in the cloud

The same code works locally or on Google Cloud — you just point an environment variable at the right place to store results.

- **On your own machine**: leave `ARTIFACTS_ROOT` (the setting that says where to save outputs) unset; it defaults to a local `./artifacts` folder. Install with `pip install -r demos/requirements-demo.txt` and run `streamlit run demos/streamlit_app.py`.
- **Saving to Google Cloud Storage** (Google's online file storage): set `ARTIFACTS_ROOT=gs://your-bucket/artifacts` and run as usual; outputs go to the cloud bucket instead of a local folder.
- **Running on Google Cloud Run** (a service that hosts the app online): build the included `Dockerfile` (a recipe for packaging the app) and deploy with `gcloud run deploy ... --set-env-vars ARTIFACTS_ROOT=gs://your-bucket/artifacts`. The app listens on whatever network port Cloud Run assigns it.
- **Quick demo mode in the cloud** (without a storage bucket): set `ARTIFACTS_ROOT=/tmp/artifacts` and `UPLOADS_DIR=/tmp/uploads`, and keep Cloud Run's max instances set to 1 so a single run doesn't get split across multiple servers.
- **File handling**: uploaded files are copied into the artifact store, so resuming a run doesn't depend on where the original file lived.
- **Buildpacks support** (an alternative way to deploy without writing a Dockerfile): the root `requirements.txt` and `Procfile` are pre-configured to start Streamlit correctly.
- **Step-by-step manual**: `CloudRun_MappingStudio_Manual_v0_2.docx` walks through deployment and usage with screenshots, including header mapping, outputs, and troubleshooting.

## What it actually does

- Spots the most likely header row, and asks you to confirm if it isn't sure.
- Cleans up column names and the values inside them.
- Saves a complete, reproducible record of the run (schema, evidence, log, clean CSV).
- Keeps the user interface separate from the engine, so new front-ends can be added without touching the core logic in `runtime/`.

## How a run flows

```
Your CSV/XLSX
   │
   ▼
runtime.excel_flow.puhemies_run_from_file
   ├─ Find candidate header rows → evidence_packet.json
   ├─ If unclear → header_spec.json → CLI/TUI/Streamlit asks you
   ├─ You confirm → human_confirmation.json
   ├─ The orchestrator picks back up → cleans and normalizes data
   └─ Writes outputs:
        • clean.csv
        • schema_spec.json (tidy column names + types)
        • shadow.jsonl (full trace log)
```

Why this is more than "just read an Excel file":

- Picking the header row is treated as a real decision, not a hidden guess buried inside a parser.
- Human answers are saved, so runs can be replayed and audited later.
- The user interfaces are intentionally thin; the real work happens in `runtime/`, so you can swap interfaces without changing the core.
- Outputs are structured files (JSON and CSV), ready for downstream tools — not screenshots or one-off printouts.

## Where to look in the code

- **Orchestration** (the main flow): `runtime/excel_flow.py` — `puhemies_run_from_file` (the first pass), `puhemies_continue` (after the user confirms), and the helpers that write artifacts and the audit log.
- **Header detection**: `_normalize_header`, `_header_looks_like_data`, and the candidate-generation logic inside `excel_flow.py`; it scores possible header rows, flags ambiguous ones, and saves them to `header_spec.json`.
- **Human-in-the-loop bits** (where the app pauses to ask you): `data_agents_cli.py` (command line), `demos/tui_app.py` (text menu), `demos/streamlit_app.py` and `demos/streamlit_mapping_studio.py` (web). They all surface the same question from `header_spec.json` and call `write_human_confirmation`.
- **Data cleaning**: `runtime/data_janitor.py` — `clean_value` and `clean_series` strip whitespace, fix numbers stored as text, and handle blanks before writing `clean.csv`.
- **Schema + evidence files**: `schema_spec.json` and `evidence_packet.json` capture the cleaned column info, confidence scores, and the decisions made — so a run can be replayed or debugged.

### How the pieces connect

```
CLI/TUI/Streamlit
   │ calls
   ▼
runtime.excel_flow.puhemies_run_from_file
   ├─ _build_header_candidates → header_spec.json
   ├─ _append_shadow           → shadow.jsonl (event log)
   ├─ write_human_confirmation → human_confirmation.json (after you pick a header)
   ├─ puhemies_continue        → re-loads your choice and cleans the data
   ├─ _infer_dtype/clean_series → schema_spec.json, clean.csv
   └─ _write_json              → evidence_packet.json, save_manifest.json
```

## Design principles

- **Decisions are explicit**: when the tool isn't confident, it asks — and saves your answer so the run is reproducible.
- **Clear separation**: the engine (header detection, cleaning, orchestration) lives in `runtime/`; the user interfaces are thin layers that just prompt and display.
- **Traceable by default**: every step writes to `shadow.jsonl` and structured spec files, so nothing important is hidden.
- **Pluggable agent roles**: standard agent and skill definitions live under `agent-base/` (mirrored in `.github/`) so this can drop into a larger multi-agent workflow.

## Project layout

- `data_agents_cli.py` / `data-agents.ps1` — the command-line entry point and a PowerShell wrapper.
- `runtime/` — header detection, cleaning ("janitor"), and orchestration code.
- `demos/` — text-menu and web front-ends, plus demo dependencies.
- `tests/` — basic coverage for the main flow.
- `agent-base/` and `.github/` — shared agent definitions and templates.

## Example output

- A cleaned-up CSV produced from the bundled sample: `docs/example-output/sample_mini_clean.csv`

## Links

- Profile: https://github.com/Alleyfoo
- Related project: https://github.com/Alleyfoo/Data-tool-demo
