# Genome Firewall

Research prototype: upload an *Escherichia coli* whole-genome assembly (FASTA) and estimate antibiotic resistance for **Ampicillin**, **Ciprofloxacin**, and **Trimethoprim**.

This is **not a clinical diagnostic**. Every result must be confirmed with laboratory susceptibility testing.

The machine-learning core, training contract files, and bundled example genomes come from the public [Biohackers / Genome Firewall](https://github.com/omarjku/Biohackers) repository (MIT as claimed by its README). This workspace adds a FastAPI + React interface, honest demo-mode labeling, and tests around upload/validation.

## What already works

| Path | What happens |
|---|---|
| Bundled example (`562.7624.fna`) | Cached AMRFinderPlus TSV + **trained model** (fitted at runtime from `data/processed/`) |
| Upload that matches a bundled FASTA hash | Same as above, labeled **cached annotation** |
| New FASTA with `amrfinder` on PATH | Live AMRFinderPlus + trained model |
| New FASTA without `amrfinder` | Marker detection **fails**; the model is **not** run; no fake scores |
| Empty / invalid FASTA | Rejected at validation |

AMRFinderPlus identifies known resistance genes and some mutations. It is **not** a complete susceptibility model. Absence of a known marker does **not** prove susceptibility.

## Setup

Requires **Python 3.12** (3.11+ is fine) and Node.js 20+.

```bash
# Python
/opt/homebrew/bin/python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Frontend
cd frontend && npm install && cd ..
```

AMRFinderPlus is **not** required for the bundled-example demo. Installing it (and its database) is multi-gigabyte and was skipped here because `amrfinder` was not on PATH and Docker was unavailable.

## Launch

Terminal 1 — API:

```bash
source .venv/bin/activate
uvicorn backend.main:app --host 127.0.0.1 --port 8000
```

Terminal 2 — UI:

```bash
cd frontend
npm run dev
```

Open http://127.0.0.1:5173

Or one shot: `bash scripts/dev.sh`

Optional: build the UI and let FastAPI serve it:

```bash
cd frontend && npm run build && cd ..
uvicorn backend.main:app --host 127.0.0.1 --port 8000
```

Then open http://127.0.0.1:8000

## Two-minute demo script

See [`docs/DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md).

Short version:

1. Open the app. Note the research-prototype banner and “AMRFinderPlus not on PATH”.
2. Run bundled example **562.7624**. The amber **Demonstration mode** banner must stay visible.
3. Show three cards: model call, calibrated probability, marker vs statistical evidence, no-call if present.
4. Upload an empty `.fasta` — validation fails, no predictions.
5. Export the report.

## Architecture

```
FASTA → validation → AMRFinderPlus (live or cached TSV)
      → name bridge (AMRFinder symbols → NDARO family tokens)
      → per-drug logistic regression (src/predictor.py)
      → Platt calibration + no-call gate (src/calibration.py)
      → FastAPI (backend/) → React dashboard (frontend/)
```

- **Frontend:** React, Vite, TypeScript, Tailwind CSS
- **Backend:** FastAPI wrapping `src/` (the original pipeline)
- **ML:** scikit-learn logistic regression, fitted at process start from committed `data/processed/` (`models/` is empty by design in the upstream project)

## Model and training data (honest)

The original project does **not** ship a pickled model. On startup, `fasta_pipeline.load_engine()` fits:

- per-drug L2-regularized logistic regression
- Platt scaling on a grouped (MLST) calibration split
- a no-call band on calibrated probability **0.30–0.70**, plus an out-of-distribution Hamming-distance check

Training inputs are the contract files in `data/processed/` (`features.csv`, `labels.csv`, `genomes.csv`, `drug_targets.json`), described as BV-BRC *E. coli* genomes with laboratory AST labels. This prototype **did not re-train or re-evaluate** that collection. Held-out numbers quoted in the upstream README live in `reports_real_scaled/` and are **not re-claimed here** unless you re-run `src/evaluation.py` yourself.

Known limitation copied from upstream: features are largely **mutation-blind**. Ciprofloxacin resistance driven by `gyrA`/`parC` point mutations is weakly represented even when AMRFinderPlus reports those mutations.

## Tests (actual results)

```bash
source .venv/bin/activate
pytest tests/ -q
```

**140 passed** (includes upstream Genome Firewall tests plus new API/validation tests). No API key or `amrfinder` required.

Also verified outside pytest:

| Case | Result |
|---|---|
| Bundled `562.7624.fna` via `pipeline.analyze` | Ampicillin `likely_to_fail` (TEM, P=0.993); Ciprofloxacin `likely_to_work` (P=0.259); Trimethoprim `no_call` (P=0.364, ambiguous band) |
| Empty FASTA `POST /api/analyze` | `ok=false`, `error_code=invalid_fasta`, no predictions |
| ~1.2 Mb synthetic FASTA without `amrfinder` | `error_code=missing_amrfinder`; prediction stage skipped |
| Unknown `example_id` | `error_code=unknown_example` |
| Report export | Client-side `.txt` download from the dashboard (no patient data) |

The Ciprofloxacin RuntimeWarning from `predictor.py` is expected: curated gene names do not match NDARO feature columns, so that drug stays `statistical_association`.

## License and attribution

MIT — see `LICENSE` and `NOTICE`. Upstream write-up: `docs/UPSTREAM_README.md`.
