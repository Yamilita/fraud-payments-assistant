# Fraud Triage Copilot — Product Requirements Document

Version:  1.0 — Initial release
Status:   Draft — pending dev review
Author:   Yamila Assi Peretti (PM)
Stack:    DuckDB · dbt-core · scikit-learn · Ollama · ChromaDB · FastAPI · Streamlit

---

## 1. Problem Statement

Fraud analysts reviewing flagged payment transactions work under high time pressure
with tools that provide scores but not explanations. A transaction marked as suspicious
by a rule engine or ML model offers no immediate answer to "why was this flagged?" —
forcing the analyst to pivot between multiple systems, query raw data manually, and
reconstruct context before making a disposition decision.

This latency in triage increases false-positive review time, delays true-positive
escalation, and erodes analyst trust in the detection system. As transaction volume
scales, the gap between flagged volume and analyst capacity widens.

The Fraud Triage Copilot addresses this by coupling a transparent fraud signal
(rules + interpretable ML) with a grounded natural-language interface that lets
analysts interrogate flagged transactions and query patterns conversationally —
without leaving a single interface and without the system inventing data.

---

## 2. Goals

| #  | Goal                                                          | Metric                                          | Target                                      |
|----|---------------------------------------------------------------|-------------------------------------------------|---------------------------------------------|
| G1 | Reduce analyst time-to-triage per flagged transaction         | Avg. time from flag to disposition              | <= 3 min (baseline TBD in pilot)            |
| G2 | Provide explainable fraud signals                             | % flags with human-readable explanation         | 100% of flagged transactions                |
| G3 | Enable natural-language data interrogation                    | Task completion rate on gold-set queries        | >= 85% correct, grounded answers            |
| G4 | Prevent hallucination in answers                              | Faithfulness score (Ragas) on eval suite        | >= 0.90 on 15-question gold set             |
| G5 | Ship a deployable, documented v1                              | Public URL + passing eval suite + README        | Live before end of Phase 5                  |

---

## 3. Non-Goals

The following are explicitly out of scope for v1.
They are documented here to prevent scope creep during implementation.

- NOT IN SCOPE: Real-time streaming ingestion.
  The system ingests batch CSV data (PaySim).
  Streaming via Kafka or Flink is a v2 consideration.

- NOT IN SCOPE: Production fraud model.
  The ML baseline is a learning artifact, not a production-grade model.
  No SLA, no retraining pipeline.

- NOT IN SCOPE: Multi-user authentication and RBAC.
  The Streamlit UI has no login system.
  Single-analyst, local or single-tenant deployment only.

- NOT IN SCOPE: Fine-tuning the LLM.
  The system uses a pre-trained local model (Mistral / Qwen via Ollama).
  Fine-tuning on fraud data is a stretch goal.

- NOT IN SCOPE: Orchestration and scheduling.
  No Airflow/Prefect pipeline. Data refresh is manual for v1.

---

## 4. User Stories

Single persona for v1: the Fraud Analyst doing first-line triage.

### 4.1 Ingestion & Scoring

- As a fraud analyst, I want the system to automatically score all transactions in
  the loaded dataset so that I do not have to run queries manually to identify suspects.

- As a fraud analyst, I want to see which features drove the fraud score for a
  specific transaction so that I can validate or override the flag with confidence.

- As a fraud analyst, I want the scoring logic to be transparent (rules + interpretable
  model, not a black box) so that I can explain my disposition decisions to a supervisor.

### 4.2 Natural-Language Interrogation

- As a fraud analyst, I want to ask "Why was transaction TXN-00423 flagged?" in plain
  language and receive an explanation grounded in that transaction's actual feature
  values — not a generic description.

- As a fraud analyst, I want to ask "Show me all transfers over 5000 EUR flagged this
  week" and get a result derived from real data, not invented records.

- As a fraud analyst, I want the system to tell me when a question is outside its
  scope or cannot be answered from available data, rather than generating a
  plausible-sounding but false answer.

### 4.3 Evaluation & Trust

- As a fraud analyst, I want to know the system has been tested against known questions
  with verified answers so that I can calibrate my trust in its responses.

- As a QA reviewer, I want a documented eval suite (gold set + scores) so that I can
  reproduce quality checks after any model or prompt change.

---

## 5. Requirements

Priority legend:
  P0 = Must-have. Cannot ship without it.
  P1 = Nice-to-have. Core use case works without it.
  P2 = Future consideration. Out of scope for v1.

---

### P0 — Must-Have

REQ-01  Data ingestion
  Description:  Load PaySim CSV into DuckDB via dbt staging model.
  Acceptance:   Given a valid PaySim CSV in data/raw/, when `dbt run` executes,
                then a staging table exists in DuckDB with correct schema, no nulls
                on key columns, and row count matches the source file.

REQ-02  Feature engineering
  Description:  Compute fraud-relevant features via dbt model:
                velocity, amount deviation, balance delta, step-hour,
                origin/destination frequency.
  Acceptance:   Given the staging table, when the features model runs,
                then all 5+ feature columns are populated with no nulls;
                dbt not-null and unique tests pass.

REQ-03  Baseline fraud signal
  Description:  Rule-based threshold + scikit-learn model (logistic regression
                or isolation forest). Precision-recall evaluated; result persisted.
  Acceptance:   Given the features table, when scoring runs:
                (a) a score and binary flag column exist for every transaction;
                (b) precision on the test set is reported;
                (c) confusion matrix is logged;
                (d) model artifact is saved to disk.

REQ-04  Transaction explanation
  Description:  LLM generates a plain-language explanation for any flagged
                transaction, grounded exclusively in that transaction's real
                feature values.
  Acceptance:   Given a flagged transaction ID, when the user requests an explanation:
                (a) the explanation references only feature values present in that
                    database row;
                (b) no transaction IDs, amounts, or dates appear that are not in
                    that row;
                (c) Ragas faithfulness >= 0.90 on the gold set.

REQ-05  Text-to-SQL agent
  Description:  Translate natural-language questions into validated SELECT queries
                on DuckDB and return results.
  Acceptance:   Given a natural-language question about transaction data:
                (a) only SELECT statements are executed;
                (b) query references only existing tables/columns;
                (c) result is returned as a formatted table or summary;
                (d) invalid queries return an error message, not a hallucinated result.

REQ-06  Out-of-scope handling
  Description:  System refuses or redirects questions it cannot answer from
                available data.
  Acceptance:   Given a question outside the data scope (e.g., "What is the CEO's
                email?"), when submitted, then the system responds with a clear
                refusal and does not generate a fabricated answer.

REQ-07  Evaluation suite
  Description:  10-15 gold-set Q&A pairs with verified answers; Ragas + promptfoo
                scores computed and stored.
  Acceptance:   Given the gold set in evals/, when the eval script runs:
                (a) faithfulness, relevancy, and grounding scores are printed;
                (b) results are saved to evals/results.json;
                (c) overall faithfulness >= 0.90.

REQ-08  FastAPI endpoints
  Description:  POST /score returns fraud flag + top features.
                POST /explain returns LLM explanation.
  Acceptance:   Given a valid transaction payload:
                - POST /score returns {flag: bool, score: float, top_features: [...]};
                - POST /explain returns {explanation: str} grounded in data;
                - HTTP 422 on invalid input.

REQ-09  Streamlit UI
  Description:  Analyst can select a transaction, view its score + explanation,
                and submit a free-text question.
  Acceptance:   Given the app is running:
                (a) selecting a transaction ID displays score, flag, and explanation;
                (b) submitting a question returns a grounded answer within 30 seconds
                    on local Ollama.

---

### P1 — Nice-to-Have

REQ-10  Docker image
  Description:  Single `docker build` produces a runnable container for the API + UI.
  Acceptance:   Given the repo root, when `docker build -t fraud-copilot .` runs,
                then the image builds without error and `docker run` serves the API
                on port 8000.

REQ-11  Public deployment
  Description:  Live URL on Streamlit Community Cloud or Hugging Face Spaces.
  Acceptance:   Given the GitHub repo, when deployed, then the app is accessible
                via a public URL with no authentication required.

REQ-12  Architecture diagram
  Description:  System diagram included in README.
  Acceptance:   Given the README rendered on GitHub, a diagram showing data flow
                CSV -> DuckDB -> dbt -> ML model -> RAG -> API -> UI is visible.

---

### P2 — Future Considerations (out of scope for v1)

REQ-13  Analyst feedback loop
  Description:  Analyst can mark a flag as false positive; feedback stored for
                future retraining.
  Note:         Tracked in backlog. Not blocking v1.

REQ-14  Prefect orchestration
  Description:  Scheduled dbt run + model refresh pipeline.
  Note:         Tracked in backlog. Not blocking v1.

REQ-15  LLM fine-tuning
  Description:  Domain-adapted model on synthetic fraud explanations.
  Note:         Tracked in backlog. Not blocking v1.

---

## 6. Success Metrics

### 6.1 Leading Indicators (measurable at launch)

| Metric                                    | Target                                          | Measurement method                        |
|-------------------------------------------|-------------------------------------------------|-------------------------------------------|
| Ragas faithfulness score                  | >= 0.90                                         | evals/results.json — run post Phase 4     |
| Task completion rate (gold set)           | >= 85% of questions answered correctly          | Manual review of gold-set outputs         |
| API response time (/explain, local Ollama)| <= 30 s on Mistral 7B                           | Manual timing; logged in test run         |
| dbt test pass rate                        | 100% (not-null + unique tests)                  | `dbt test` output                         |
| Hallucinated entity rate                  | 0 invented IDs/amounts in gold-set answers      | Ragas grounding + manual spot-check       |

### 6.2 Lagging Indicators (post-deployment)

- Portfolio signal: project linked from LinkedIn profile and CV within 30 days of deployment.
- Technical interview utility: able to walk through architecture and eval methodology
  in a 45-min interview without notes.
- Energy diagnostic: personal learning-log energy score >= 3/5 average across
  Phases 1-4 (confirms technical track fit).

---

## 7. Open Questions

| #  | Question                                                                                                     | Owner          | Blocking?                                   |
|----|--------------------------------------------------------------------------------------------------------------|----------------|---------------------------------------------|
| Q1 | Which Ollama model to use: Mistral Small (Apache 2.0) or Llama 4 Scout? Mistral preferred for licensing.     | Engineering    | No — default to Mistral; swap if needed     |
| Q2 | Text-to-SQL: LangChain/LlamaIndex or custom prompt chain? Custom simpler for v1.                             | Engineering    | No — decide at Phase 3 start               |
| Q3 | Acceptable latency SLO for /explain on deployment (Streamlit Cloud is CPU-only). May need Groq free tier.    | Engineering    | YES — needed before Phase 5                 |
| Q4 | How to validate gold-set ground truth: self-annotation or cross-check against raw DuckDB queries?            | PM + Data      | No — resolve before Phase 4                |
| Q5 | Streamlit Community Cloud vs Hugging Face Spaces for v1 demo?                                                | Engineering    | No — decide at Phase 5 start               |

---

## 8. Timeline

| Phase   | Duration    | Deliverable                                                            | Requirements covered         |
|---------|-------------|------------------------------------------------------------------------|------------------------------|
| Phase 0 | 0.5 weeks   | Repo skeleton, README, .gitignore, .env, requirements.txt              | —                            |
| Phase 1 | 1 week      | PaySim in DuckDB; dbt staging + features models; dbt tests passing     | REQ-01, REQ-02               |
| Phase 2 | 1 week      | Baseline rule + ML scorer; precision-recall report; scoring function   | REQ-03                       |
| Phase 3 | 1.5 weeks   | Ollama connected; RAG over fraud_policy.md; text-to-SQL agent; QA      | REQ-04, REQ-05, REQ-06       |
| Phase 4 | 1 week      | Gold set; Ragas + promptfoo scores >= targets; guardrails live         | REQ-07                       |
| Phase 5 | 1 week      | FastAPI; Streamlit UI; Dockerfile; public deployment; arch diagram     | REQ-08 to REQ-12             |

Total estimated effort: 4-6 weeks at part-time pace (~2-3 hours/day).

Hard dependency: Phase 3 cannot start until Phase 2 baseline exists.
The ML model's feature values are required to generate grounded explanations.

---

## 9. Technical Stack Reference

| Layer              | Tool                        | License       | Notes                                          |
|--------------------|-----------------------------|---------------|------------------------------------------------|
| Data warehouse     | DuckDB                      | MIT           | In-process; no server required                 |
| Transformations    | dbt-core v1.11              | Apache 2.0    | Use stable 1.11; v2.0 is alpha as of June 2026 |
| ML baseline        | scikit-learn                | BSD-3         | Logistic regression or isolation forest        |
| LLM runtime        | Ollama                      | MIT           | Local inference; no API cost                   |
| Default LLM model  | Mistral Small / Qwen 3.5    | Apache 2.0    | Cleanest license; strong multilingual          |
| Embeddings         | sentence-transformers       | Apache 2.0    | all-MiniLM-L6-v2 as default                    |
| Vector store       | ChromaDB                    | Apache 2.0    | Local persistent store                         |
| Eval — RAG         | Ragas                       | Apache 2.0    | faithfulness, answer_relevancy metrics         |
| Eval — prompts     | promptfoo                   | MIT           | Regression testing across prompt versions      |
| API                | FastAPI + uvicorn           | MIT           | /score and /explain endpoints                  |
| UI                 | Streamlit                   | Apache 2.0    | Single-page analyst interface                  |
| Container          | Docker                      | Apache 2.0    | Docker Desktop free for personal use           |
| Hosting            | Streamlit Community Cloud   | Free tier     | CPU only; may need Groq for LLM inference      |

---

END OF DOCUMENT
