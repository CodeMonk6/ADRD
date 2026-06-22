# ADRD — AI Biomedical Research Scientist

A secure, traceable, **scientist-in-the-loop discovery operating system** for Alzheimer's disease
and related dementias. ADRD turns fragmented neurodegeneration evidence — literature, knowledge
graphs, failed clinical trials, and single-cell data — into **ranked, citation-grounded,
contradiction-aware, experimentally testable hypotheses**. It is deliberately *not* a chatbot.

The first build answers one flagship question end-to-end:

> *Which human-relevant microglial lipid-metabolism mechanisms may explain APOE4-associated
> acceleration of tau pathology, and what is the most efficient experiment to test the top mechanism?*

with the mechanistic spine `APOE4 → lipid transport → microglial lipid-droplet state →
inflammatory signaling → tau propagation → neurodegeneration`.

> **Status:** runs end-to-end on the full Docker stack, grounded in real Alzheimer's genetic
> associations from Open Targets. The flagship workflow produces causal-gated, citation-verified
> hypotheses with executable experiment plans, every step traceable and gated for expert sign-off.

---

## What makes it different

Off-the-shelf tools can't do the part that matters. Knowledge graphs encode association without
causal direction; large language models overclaim causality; single-cell foundation models
underperform classical methods zero-shot. ADRD's defensible core is the **governance and
causal-evidence layer** built around the model:

1. **Tiered T0–T5 causal-evidence scoring** with negative- and failed-trial subtraction, engineered
   onto association-only graphs.
2. **A causal gate** — no hypothesis is labeled *causal* unless it passes a refutation test **or**
   carries a genetic/perturbational anchor (T3+). Enforced in code, not prompts.
3. **Deterministic citation verification** — every surfaced claim maps to a retrievable verbatim
   source span; LLM-emitted citations are never trusted.
4. **Closed-loop reproducible reanalysis** — the system runs computational experiments and records
   full provenance (RO-Crate), rather than only recommending them.

Every output carries a safety label (`evidence` / `inference` / `speculation` / `validation-plan`),
provenance, and an expert-approval gate.

---

## Architecture

```
Next.js workbench  ──SSE──▶  FastAPI + Arq/Redis
        │                          │
        │              custom typed orchestrator (planner → literature/KG/dataset →
        │              mechanism → causal → hypothesis → critic → Bradley-Terry rank →
        │              experiment design → reviewer → expert gate)
        ▼                          │
  evidence map · NeuroKG    OpenRouter LLM gateway (Claude default, provider-agnostic, mock mode)
  hypothesis board                 │
  experiment plans          ┌──────┴───────────────────────────────────────────┐
                            ▼              ▼              ▼              ▼        ▼
                   OpenSearch (BM25)  Qdrant (dense)  Neo4j (NeuroKG)  Postgres  Redis
                            └── RRF + MedCPT rerank + citation verifier ──┘
```

| Layer | Choice |
|---|---|
| Orchestration | Custom typed, auditable state-graph orchestrator (LangGraph is the documented scale path) |
| LLM gateway | OpenRouter (OpenAI-compatible); Claude Opus 4.8 / Sonnet 4.6 / Haiku 4.5 by default; deterministic mock mode |
| Retrieval | OpenSearch BM25 + Qdrant dense (MedCPT) + RRF + MedCPT cross-encoder + deterministic citation verifier |
| Knowledge graph | Neo4j NeuroKG with the engineered T0–T5 / provenance / contradiction layer on every edge |
| Compute | Scanpy + scvi-tools reproducible reanalysis (papermill); foundation models optional/low-confidence |
| Scoring | GRADE-style tiered score + causal gate + Bradley-Terry round-robin ranking |
| Governance | Audit log, RO-Crate provenance, license-gating, safety labels, human-in-the-loop |

See `DEVELOPMENT_PLAN.md` for the full design, data-access matrix, and milestone ladder.

---

## Quickstart

Requires Docker. The pipeline runs with or without an LLM key — set `ADRD_LLM_MOCK=1` to run a
fully deterministic, offline demo on the curated seed corpus.

```bash
make env                 # create .env from template
#   add OPENROUTER_API_KEY  (or set ADRD_LLM_MOCK=1 in .env to run with no key)
make up                  # build + start: postgres · neo4j · qdrant · opensearch · redis · backend · worker · frontend
make bootstrap           # init Postgres tables + Qdrant collection + OpenSearch index
make kg-init             # create NeuroKG schema/constraints
make ingest              # ingest the curated seed corpus for the flagship workflow
make ingest-live         # (optional) add REAL data: Open Targets genetic anchors + bounded PubMed
make demo                # run the flagship workflow end-to-end and print the output package
```

Then open:

- **http://localhost:3000** — scientist workbench (ask a question, watch agents stream, inspect results)
- **http://localhost:8000/docs** — API
- **http://localhost:7474** — Neo4j browser (NeuroKG)

### Run on real, live data

Set `ADRD_LLM_MOCK=0` and an `OPENROUTER_API_KEY`, then ingest from open APIs:

```bash
docker compose exec backend python -c "import asyncio; from app.ingestion.pipeline import ingest; \
  asyncio.run(ingest({'queries':['APOE4 microglia lipid tau'], \
  'sources':['pubmed','europepmc','clinicaltrials'], 'extract': True}))"
```

For real single-cell reanalysis, install the `science` extras and point `ADRD_SCRNA_PATH` at an open
AnnData (e.g. SEA-AD or a CZ CELLxGENE Census export); otherwise the engine runs the same analysis on
clearly-labeled synthetic data so the loop always closes.

---

## Evaluation

```bash
cd backend && pytest -q                 # core governance/causal logic (no services needed)
python eval/run_eval.py                 # gold-suite integration eval against a live stack
```

The gold suite (`eval/gold_suite.json`) checks citation precision, causal-gate consistency,
negative-evidence surfacing, and full traceability. Hypothesis-quality scoring additionally requires
a sampled human-expert audit — by design, the harness does not fake it.

---

## Data & licensing

The MVP uses **open, no-DUA sources only**: SEA-AD, MIT/Tsai processed atlas, CZ CELLxGENE, Open
Targets, Reactome, GWAS Catalog, PubTator 3.0, PubMed/Europe PMC, ClinicalTrials.gov v2, bioRxiv,
and OBO ontologies. A license-gating module blocks restricted sources (UMLS, DrugBank, DisGeNET,
ADNI, AMP-AD/ROSMAP raw) from any redistributable artifact. Restricted/DUA data is deferred to the
governed production phase.

---

## Repository layout

```
backend/app/   config · llm gateway · stores · models · ingestion · kg · retrieval · scoring ·
               agents (+ orchestrator) · experiments · governance · api · scripts
frontend/      Next.js workbench (workspace · run view · NeuroKG explorer · hypotheses)
eval/          gold suite + integration eval
DEVELOPMENT_PLAN.md   full design, data matrix, milestones, risks
```
