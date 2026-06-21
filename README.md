# Fraud Triage Copilot

An explainable assistant that helps a fraud analyst triage suspicious
payment transactions and ask questions about them in natural language —
grounded in real transaction features and evaluated to avoid hallucination.

## The problem

Fraud analysts review flagged transactions under time pressure. Pure ML
scores are accurate but opaque ("why was this flagged?"), and raw data
exploration is slow. This project pairs a transparent fraud signal with a
grounded language interface, so an analyst can both *trust* the flag and
*interrogate* the data conversationally.

## Who it's for

A payments fraud analyst doing first-line triage.

## What it does

1. Ingests payment transactions and engineers fraud-relevant features.
2. Scores transactions with a transparent baseline (rules + simple ML).
3. Explains a flag in plain language and answers questions about patterns,
   grounded in the actual data and a fraud-policy document.
4. Every answer is checked against an evaluation suite (no invented IDs or
   amounts).

## Architecture

_Diagram to be added in Phase 5._

## Tech stack

- **Data:** DuckDB, dbt-core, pandas
- **ML baseline:** scikit-learn
- **GenAI:** Ollama (local LLM), sentence-transformers, ChromaDB
- **Evaluation:** Ragas, promptfoo
- **Serving:** FastAPI, Streamlit, Docker

All tooling is open source / free tier.

## Project structure

_See the directory tree — folders fill in phase by phase._

## Setup

\`\`\`bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
\`\`\`

## Roadmap

- [x] Phase 0 — Project skeleton
- [ ] Phase 1 — Data & feature engineering (DuckDB + dbt)
- [ ] Phase 2 — Baseline fraud signal (scikit-learn)
- [ ] Phase 3 — RAG + text-to-SQL agent
- [ ] Phase 4 — Evals & guardrails
- [ ] Phase 5 — API, UI & deployment


