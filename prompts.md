Here is the adapted step-by-step build prompt series, fully integrated with the new "Enhanced Idea Generation with Long-Context LLMs" system.

# Research Co-Pilot: Step-by-Step Build Prompts (Enhanced Long-Context Edition)

## 🎯 Overview
This document contains a series of prompts to guide an LLM through building the Research Co-Pilot application. Each prompt builds on the previous one, creating a complete system incrementally. This version incorporates the **Long-Context Synthesis** architecture.

---

## **PROMPT 0: Initial Context Setting** (Start Here)

```
I want to build a Research Co-Pilot application - an AI-powered research assistant that helps scientists discover publishable research ideas and generate working prototypes.

**Core Concept:**
The system ingests academic papers from multiple sources (arXiv, PubMed, Elsevier, etc.). Instead of generating simple ideas paper-by-paper, it processes **large batches of 50-200 papers** using **long-context LLMs** (like Gemini 1.5 Pro) to synthesize a smaller number of high-quality, creative, cross-paper ideas. These ideas are then ranked using a hybrid scoring system, validated selectively via the FutureHouse API, and turned into working code prototypes.

**Key Principles:**
1. **Batch Synthesis over Templates**: Use long-context models to find patterns and gaps across many papers simultaneously, prioritizing quality and creativity over quantity.
2. **Selective API Usage**: Generate the idea bank locally (using the LLM API), rank them, and only send the top-20 candidates to the expensive FutureHouse API for final novelty checks.
3. **Hybrid & Explainable Ranking**: Combine the initial scores provided by the idea generation LLM with deterministic local signals (bibliometrics, data availability checks) for a transparent final score. Avoid opaque embeddings.
4. **Human-in-the-Loop**: Researchers approve ideas and refine prototypes.
5. **Sandboxed Execution**: Never run untrusted LLM-generated code on the host.
6. **Full Provenance**: Audit trail of all papers, prompts, API responses, and code artifacts.

**Tech Stack:**
- Backend: Python FastAPI
- Database: PostgreSQL
- Object Storage: MinIO (S3-compatible)
- Queue: Redis + Celery
- Frontend: React + Tailwind
- LLMs: Gemini 1.5 Pro (Idea Synthesis), Gemini Pro 2.5 (Code Gen), GPT-5 (fallback)
- External APIs: FutureHouse (Crow/Owl), Elsevier, arXiv, PubMed
- Deployment: Docker Compose (local) → VPS (production)

**Target Output:**
Q1-Q2 journal quality ideas (Impact Factor 1-4) that are well-grounded, novel, and come with validated, runnable Colab notebooks.

**Architecture Pattern:**
Ingest → **Batch Synthesis (Long-Context LLM)** → Hybrid Ranking → FutureHouse Validation (selective) → Code Generation → Human Review → Publication Draft

I'll provide detailed specifications in follow-up prompts. For now, acknowledge understanding and ask any clarifying questions about the scope, constraints, or technical approach.
```

---

## **PROMPT 1: Project Structure & Foundation**

```
Let's start building the foundation. Create the initial project structure with:

**Requirements:**
1. Project directory structure following best practices for Python microservices
2. Docker Compose setup with services: FastAPI, PostgreSQL, Redis, MinIO, Celery worker
3. Base configuration management (environment variables, secrets handling)
4. Database models for core entities. **Note the structured nature of IdeaCandidate**:
   - `Paper` (id, title, authors, venue, year, abstract, pdf_path, source, metadata_json)
   - `IdeaCandidate` (id, paper_ids[], title, description, novelty_rationale, methodology, required_datasets[], estimated_compute, risks, gen_timestamp, llm_initial_scores_json, raw_llm_output)
   - `Ranking` (idea_id, score, component_scores_json, rank_timestamp)
   - `FHResponse` (idea_id, task_type, response_json, status, job_id, created_at)
   - `PrototypeArtifact` (idea_id, notebook_path, code_archive, test_logs, sandbox_status)
   - `AuditLog` (entity_type, entity_id, action, prompt_text, response_text, timestamp, user_id)

5. SQLAlchemy models with proper relationships
6. Alembic migrations setup
7. Basic FastAPI app structure with health check endpoint
8. Logging configuration (structured JSON logs)

**Deliverables:**
- Complete directory tree
- docker-compose.yml with all services
- .env.example file
- requirements.txt with pinned versions
- Database models in models.py
- Alembic migration for initial schema
- main.py with FastAPI app skeleton

**Constraints:**
- Use Python 3.11+
- Follow PEP 8 and type hints everywhere
- Include docstrings for all classes/functions
- PostgreSQL 15+, Redis 7+, MinIO latest

Please provide the complete file structure first, then I'll ask for specific file contents.
```

---

## **PROMPT 2: Ingest Service - arXiv Connector**

```
Now let's build the first data ingestion connector for arXiv.

**Requirements:**
Create a production-ready arXiv ingestion service that:

1. **arXiv API Integration:**
   - Uses arxiv Python package (or direct HTTP API)
   - Searches by category, keyword, date range
   - Handles rate limiting (max 1 req/3sec as per arXiv guidelines)
   - Retry logic with exponential backoff
   - Parses: title, authors, abstract, categories, publication date, PDF URL, arXiv ID

2. **PDF Download:**
   - Downloads PDFs to MinIO storage (path: `papers/arxiv/{year}/{arxiv_id}.pdf`)
   - Skips if already exists (idempotent)
   - Handles download failures gracefully
   - Extracts text from PDF using PyMuPDF or pdfplumber (store in Paper.full_text field - add this to model)

3. **Database Persistence:**
   - Checks if paper exists (by arXiv ID) before inserting
   - Stores normalized metadata in Paper table
   - Logs ingest events to AuditLog

4. **Celery Task:**
   - Background task: `ingest_arxiv_batch(categories: list, days_back: int, max_results: int)`
   - Configurable batch size
   - Progress tracking (how many processed/failed)
   - Error handling per paper (don't fail entire batch)

5. **API Endpoint:**
   - POST /api/ingest/arxiv with body: `{categories: [...], days_back: 7, max_results: 100}`
   - Returns job_id for tracking
   - GET /api/ingest/status/{job_id} for progress

**Deliverables:**
- `services/ingest/arxiv_connector.py` with ArxivClient class
- `tasks/ingest_tasks.py` with Celery task
- `api/routes/ingest.py` with endpoints
- Unit tests for ArxivClient (mock HTTP calls)
- Configuration for arXiv categories in config.yaml

**Example Usage:**
```python
from services.ingest.arxiv_connector import ArxivClient

client = ArxivClient()
papers = client.search(
    categories=["cs.AI", "cs.LG"],
    days_back=7,
    max_results=50
)
for paper in papers:
    client.download_pdf(paper.pdf_url, paper.arxiv_id)
    client.save_to_db(paper)

Provide the complete implementation with error handling and logging.
```
---

## **PROMPT 3: Idea Bank Generator (Long-Context Synthesis)**

```
Build the Idea Bank generator - the core "heavy researcher" component. This uses long-context LLMs to synthesize ideas from large batches of papers.

**Requirements:**

1. **Batch Preparation & Compression:**
   Create a `PaperBatchProcessor` class that:
   - Takes a list of 50-200 `Paper` objects.
   - For each paper, extracts key info into a compressed text format: Title, Venue, Contribution (1-2 sentences from abstract), Methods (list), Datasets (list), Limitations (extracted from full text or abstract).
   - Optionally performs lightweight clustering (e.g., simple keyword overlap) to group the papers thematically in the output text.
   - Combines this into a single `PAPER_BATCH_COMPRESSED` string.

2. **Long-Context LLM Synthesis:**
   - Integrate with **Gemini 1.5 Pro** (capable of 1M+ tokens).
   - Implement a prompt that directs the LLM to:
     - Analyze the compressed batch.
     - Generate 20-30 creative ideas that cross-pollinate concepts, bridge gaps, or synthesize methods from multiple papers.
     - **Crucially, mandate the output be a JSON array** where each object has the following structure:
       ```json
       {
         "title": "Concise research title",
         "description": "2-3 sentence description",
         "novelty_rationale": "Why this is novel",
         "source_papers": ["P1_ID", "P5_ID"], 
         "methodology": "High-level approach (3-5 steps)",
         "required_datasets": ["Dataset A", "Dataset B"],
         "estimated_compute": "LOW | MEDIUM | HIGH",
         "risks": "Potential feasibility issues",
         "impact_score": 1-10,  
         "feasibility_score": 1-10
       }
       ```
   - Implement robust error handling and retries for the large LLM call. Cost tracking is important.

3. **Output Parsing & Post-Generation Filter:**
   - Parse the JSON response from the LLM. Handle potential JSON formatting errors from the LLM (use a repair library or retry).
   - Implement `PostGenerationFilter`:
     - **Deduplication:** Use fuzzy matching on titles/descriptions to remove near-duplicates within the batch.
     - **Novelty Pre-Check:** perform a quick arXiv API search on the generated title. Flag if >3 exact matches exist.
   - Save the parsed results as `IdeaCandidate` records in the database. Map the LLM's `impact_score` and `feasibility_score` to the `llm_initial_scores_json` field.

4. **Celery Task:**
   - `generate_ideas_for_batch(paper_ids: list[int])`
   - Should handle fetching the papers, running the processor, calling the LLM, parsing, filtering, and saving.

**Deliverables:**
- `services/idea_generation/batch_processor.py` - PaperBatchProcessor class
- `services/idea_generation/synthesis_engine.py` - SynthesisEngine class (LLM integration)
- `services/idea_generation/post_filter.py` - PostGenerationFilter class
- `tasks/idea_tasks.py` - Celery task
- `config/prompts.yaml` - The main synthesis prompt template
- API endpoint: POST /api/ideas/generate-batch with `{paper_ids: [...]}`

Provide the complete implementation, especially the prompt template and the JSON parsing logic.
```

---

## **PROMPT 4: Ranking Engine (Hybrid Scoring)**

```
Implement the hybrid ranking system. This system **calibrates and augments** the initial scores provided by the LLM with deterministic local checks.

**Requirements:**

1. **Feature Computation:**
   Build a FeatureExtractor class that computes scores for each IdeaCandidate.

   **A. LLM-Provided Scores (Calibration Base):**
   - Extract `llm_impact_score` and `llm_feasibility_score` (1-10) from the IdeaCandidate.
   - Normalize them to [0, 1].

   **B. Bibliometrics (Augmentation) (0-1):**
   - `paper_venue_score`: Average venue quartile of the `source_papers`. (Q1=1.0, Q2=0.8, etc.)
   - `citation_signal`: Average log-citations of the `source_papers`, with recency decay.

   **C. Novelty Signal (Augmentation) (0-1):**
   - `synthesis_degree`: Score based on the number of `source_papers`. (e.g., 1 paper = 0.5, 2 papers = 0.8, 3+ papers = 1.0). Higher synthesis implies higher inherent novelty.

   **D. Feasibility Calibration (0-1):**
   - `data_availability`: Parse the `required_datasets` list.
     - All Public/Known → 1.0
     - Mix or Unknown → 0.7
     - Requires Proprietary/Paid → 0.3
   - `compute_requirement`: Map `estimated_compute` (LOW/MEDIUM/HIGH) to (1.0/0.7/0.4).
   - Combine these to calibrate the LLM score:
     `final_feasibility = 0.4 * normalized_llm_feasibility + 0.6 * (data_availability * compute_requirement)`

   **E. Impact Calibration (0-1):**
   - `industry_relevance`: Check `description` for applied terms (healthcare, finance, etc.) → 0 or 1.
   - `final_impact = 0.7 * normalized_llm_impact + 0.3 * industry_relevance`

   **F. Risk (0-1, penalized):**
   - `copyright_risk`: Based on `required_datasets` (proprietary data) and keywords in the `risks` field.

2. **Composite Score:**
   ```python
   publishability_score = (
       0.25 * bibliometrics_score +
       0.20 * synthesis_degree +
       0.20 * final_feasibility +
       0.20 * final_impact -
       0.15 * risk_score
   )
   ```

3. **Ranking Logic & Explainability:**
   - Compute scores, sort descending, and filter by a configurable threshold (e.g., > 0.6).
   - Store `Ranking` records.
   - Crucially, store **both** the initial LLM scores and the final calculated components in the `component_scores_json` so the UI can show: "LLM rated Feasibility 8/10, but we downgraded it because 'Dataset X' is private."

**Deliverables:**
- `services/ranking/feature_extractor.py` - FeatureExtractor class
- `services/ranking/scorer.py` - CompositeScorer class (implements hybrid formula)
- `config/venue_rankings.json` - Venue quartile mappings
- `config/scoring_weights.yaml` - Configurable weight parameters
- API endpoints:
  - POST /api/ranking/score - Score a list of ideas
  - GET /api/ranking/top?limit=20 - Get top-ranked ideas

**Example Output:**
```json
{
  "idea_id": "123",
  "publishability_score": 0.72,
  "components": {
    "llm_initial": {"impact": 8, "feasibility": 9},
    "bibliometrics": 0.85,
    "synthesis_degree": 0.8,
    "final_feasibility": 0.75, 
    "final_impact": 0.78,
    "risk": 0.3,
    "notes": ["Feasibility downgraded: 'Hospital-X-Data' identified as proprietary."]
  }
}
```
Provide complete implementation with example scoring.
```
```

---

## **PROMPT 5: FutureHouse API Integration**

```
Implement the selective FutureHouse integration layer that validates only top-ranked ideas.

**Requirements:**

1. **FutureHouse Client Wrapper:**
   - Install: `pip install futurehouse-client`
   - Support for both Crow (summaries) and Owl (novelty checks)
   - Rate limiting & Budget tracking (respect API quotas, e.g., 100 calls/day)
   - Retry logic with exponential backoff
   - Response caching (store in `FHResponse` table)

2. **Crow Integration (Structured Summaries):**
   - Task: Get a high-quality structured summary relating the idea to its source papers.
   - Input: Pass the idea's `title`, `description`, and `methodology`. Pass the abstracts of the `source_papers`.
   - Output: Structured JSON with related work, key findings, connections. Store in `FHResponse`.

3. **Owl Integration (Novelty Check):**
   - Task: "Has anyone done X?" query.
   - Input: Pass the idea's `description` and the LLM-generated `novelty_rationale` as context.
   - Output: `novelty_verdict` (NOVEL/PARTIAL/EXISTS), confidence (0-1), related_papers[].
   - Update `IdeaCandidate` with a final `novelty_score` based on verdict:
     - NOVEL → 1.0
     - PARTIAL → 0.6
     - EXISTS → 0.2

4. **Selective Submission Logic:**
   - Function: `evaluate_top_k_ideas(ranking_ids, k=20)`
   - Gets top K ideas. For each, submit Owl job. If NOVEL/PARTIAL, submit Crow job.
   - Re-rank after FH responses (the `novelty_score` should now heavily influence the final rank).

5. **API Endpoints:**
   - POST /api/futureHouse/evaluate - Trigger FH validation for top-K ideas
   - GET /api/futureHouse/status/{job_id} - Check FH job status

**Deliverables:**
- `services/external/futurehouse_client.py` - FHClient class
- `services/external/budget_tracker.py` - BudgetTracker class
- `tasks/fh_tasks.py` - Celery tasks for async FH calls
- `api/routes/futurehouse.py` - API endpoints
- Configuration: `config/futurehouse.yaml` (API key, budget limits)

Include error handling for API failures and rate limits.
```

---
```
## **PROMPT 6: Code Generation Orchestrator (Gemini + GPT-5)**

```
Build the code generation system that creates validated, runnable prototypes.

**Requirements:**

1. **Task Specification:**
   Create `TaskSpec` dataclass. This should be easy to populate from our structured `IdeaCandidate`:
   - `idea_description`: From `IdeaCandidate.description` + `IdeaCandidate.methodology`
   - `datasets`: From `IdeaCandidate.required_datasets`
   - `constraints`: Max lines of code, allowed libraries, compute limits (from `estimated_compute`)
   - `output_format`: Jupyter notebook (.ipynb)

2. **Gemini Client:**
   - Use **Gemini Pro 2.5** API (via google-generativeai SDK) for code generation.
   - Prompt template:
     ```
     You are a research engineer. Generate a production-quality, runnable Python Jupyter notebook for this research idea.
     
     Description & Methodology: {idea_description}
     Required Datasets: {datasets}
     
     Output a complete Jupyter notebook (.ipynb JSON format) with:
     1. Environment setup (pip installs)
     2. Data loading (use placeholders or public URLs if datasets aren't standard)
     3. Model implementation
     4. Training loop / Core experiment logic
     5. Evaluation and visualization
     6. A final cell with simple smoke tests (assert statements) to verify the code ran correctly.
     
     Ensure code is runnable in Google Colab (e.g., T4 GPU assumed).
     ```
   - Retry policy: MAX_RETRIES=2.

3. **Code Validation & Sandbox:**
   - Extract the generated .ipynb JSON.
   - Run linter (ruff) on the code cells.
   - Execute the notebook in a **Docker sandbox**.
     - Docker image: `python:3.11-slim` + common ML libs (torch, transformers, scikit-learn, pandas).
     - Resource limits: 2 CPU, 8GB RAM, 15min timeout.
     - No network access (except pre-approved PyPI/HuggingFace if necessary, or preload libs).
   - Check if the final smoke test cell executed successfully.

4. **GPT-5 Fallback:**
   - Trigger if Gemini fails retries or the sandbox execution fails.
   - Enhanced prompt includes Gemini's failed code + the error logs from the sandbox.
   - GPT-5 gets 1 attempt.

5. **Artifact Storage:**
   - Save the final validated notebook to MinIO: `prototypes/{idea_id}/notebook.ipynb`.
   - Create `PrototypeArtifact` record linking to the notebook, test logs, and which model succeeded.

**Deliverables:**
- `services/codegen/gemini_client.py`
- `services/codegen/gpt5_client.py`
- `services/codegen/orchestrator.py`
- `services/codegen/sandbox.py` - DockerSandbox runner
- `tasks/codegen_tasks.py` - Celery task: `generate_prototype(idea_id)`
- API endpoint: POST /api/codegen/generate/{idea_id}
- Dockerfile for sandbox environment

Include safety checks: never execute code outside the sandbox.
```

---
```
## **PROMPT 7: Dashboard UI (React Frontend)**

```
Build the React + Tailwind dashboard. It must display the rich, structured data from our new idea generator.

**Requirements:**

1. **Project Setup:**
   - Create React app with Vite. Install: react-router-dom, axios, tailwindcss, lucide-react, recharts, shadcn/ui.

2. **Pages/Views:**

   **A. Home/Inbox:**
   - Stats: Papers ingested, Batches run, Ideas synthesized, FH credits used, **LLM API Cost (this batch/total)**.

   **B. Ideas Inbox (Main View):**
   - Table of IdeaCandidates.
   - Columns: **Title**, Source Papers (count), **Risks (summary)**, Rank, Total Score, Status.
   - Status badges: NEW, FH_PENDING, FH_VALIDATED, PROTOTYPE_READY, ARCHIVED.
   - Actions: "View Details", "Evaluate via FH", "Generate Prototype", "Archive".

   **C. Idea Detail (CRITICAL):**
   - **This page must show all structured fields clearly.**
   - Header: Title, generated date, status.
   - Sections (Collapsible cards):
     - **Description**: `IdeaCandidate.description`
     - **Novelty Rationale**: `IdeaCandidate.novelty_rationale`
     - **Methodology**: `IdeaCandidate.methodology` (formatted as a list/steps)
     - **Requirements & Risks**: `required_datasets`, `estimated_compute`, `risks`.
     - **Source Papers**: List linked papers.
   - **Score Breakdown**:
     - Radar chart or bar chart showing component scores.
     - **Tooltip/Popover MUST show the calibration**: "Initial LLM Feasibility: 9/10. Final Feasibility: 0.75. Reason: Proprietary data required."
   - **FH Responses** (if available): Owl verdict badge, Crow summary.
   - Actions: "Generate Prototype", "Promote to Draft", "Archive".

   **D. Prototype Viewer:**
   - Show sandbox test results (PASSED/FAILED).
   - Button: "Open in Colab" (link to MinIO notebook).
   - Show test logs. If FAILED, offer "Escalate to GPT-5".

   **E. Settings:**
   - API Keys.
   - **Idea Generation Config**:
     - `llm_provider` dropdown (Gemini 1.5 Pro, Claude 3.5 Sonnet).
     - `batch_size` (slider: 50-200).
     - `ideas_per_batch` (slider: 20-50).
   - FH Budget limits.

**Deliverables:**
- `frontend/` directory with Vite React app
- `frontend/src/pages/` - All page components
- `frontend/src/components/` - Reusable components (IdeaCard, ScoreBreakdown)
- `frontend/src/api/` - API client

Provide complete code for the **Idea Detail** page, as it's the most complex and important view.
```

---
```
## **PROMPT 8: Integration & End-to-End Workflow**

```
Integrate all components and build the complete end-to-end workflow orchestrator.

**Requirements:**

1. **Workflow Orchestrator:**
   Create a master `WorkflowOrchestrator` service.
   Method: `run_research_batch(config: BatchConfig)`

   Steps:
   1. **Ingest**: Harvest papers from configured sources until a sufficient number (e.g., `batch_size`=100) is collected.
   2. **Synthesize**: call `generate_ideas_for_batch(paper_ids)`. This is a single, long-running Celery task that calls the long-context LLM.
   3. **Rank**: Once synthesis is complete, run the Hybrid Ranking Engine on the new ideas.
   4. **FH Validation**: Select top-K ideas and kick off parallel FH Owl/Crow tasks.
   5. **Re-rank & Notify**: Re-rank with FH scores and notify the user that the batch is ready for review.

2. **Batch Configuration:**
   YAML config:
   ```yaml
   batch:
     name: "Vision Transformers Week 12"
     sources:
       - type: arxiv
         categories: ["cs.CV"]
         days_back: 7
     
     idea_generation:
       llm_model: "gemini-1.5-pro"
       batch_size: 100
       ideas_per_batch: 30
     
     ranking:
       top_k_for_fh: 15
     
     futurehouse:
       budget_limit: 30
   ```

3. **API Endpoints for Workflow:**
   - POST /api/workflow/batch - Start new batch
   - GET /api/workflow/batch/{batch_id} - Status (Ingesting [55/100] → Synthesizing → Ranking → FH Validation)

4. **Error Recovery:**
   - The Synthesis step is expensive. If it fails (e.g., LLM API timeout), the retry logic must be robust. Don't discard the ingested papers; allow re-triggering just the synthesis step.

5. **Integration Test:**
   - End-to-end test with mock data:
     - Insert 100 mock papers.
     - Mock the `SynthesisEngine` to return a fixed JSON list of 3 ideas.
     - Run ranking.
     - Mock FH responses.
     - Generate prototype (mock Gemini).
     - Verify all artifacts are created and the IdeaCandidate state transitions are correct.

**Deliverables:**
- `services/orchestration/workflow.py` - WorkflowOrchestrator class
- `api/routes/workflow.py` - Workflow endpoints
- `tests/integration/test_e2e_workflow.py`

Provide the complete orchestrator implementation.
```

---
```
## **PROMPT 9: Testing, Documentation & Deployment**

```
Finalize the application with comprehensive testing, documentation, and deployment setup.

**Requirements:**

1. **Testing Suite:**
   - Unit tests for all services (>80% coverage).
   - **Performance Test for Batch Processor**: Ensure `PaperBatchProcessor` can handle 200 papers and produce the compressed prompt in a reasonable time (<30s).
   - **Mock Test for Long-Context Call**: Ensure the `SynthesisEngine` correctly handles the large prompt construction and JSON response parsing, using a mocked LLM client.

2. **Monitoring & Observability:**
   - **CRITICAL: LLM Cost Tracking**. Log the token usage (input/output) for every call to the expensive long-context LLM (Gemini 1.5 Pro).
   - Grafana Dashboard:
     - Add a panel for **Cumulative LLM API Cost ($)**.
     - FH credit usage.
     - Pipeline progress.
   - Alerting:
     - Alert if LLM API cost exceeds daily/batch budget.
     - Alert if FH budget is >90% used.

3. **Deployment:**
   - `docker-compose.prod.yml` for VPS deployment.
   - `deploy/vps_setup.sh` script (Nginx, SSL, systemd, firewall).
   - Backup script for Postgres and MinIO.

4. **Documentation:**
   - `README.md`: Architecture diagram, Quickstart.
   - `docs/CONFIGURATION.md`: Explain how to tune the prompts and ranking weights.
   - `docs/API_KEYS.md`: List all required keys (Gemini, FH, Elsevier, etc.).

**Deliverables:**
- Complete `tests/` directory.
- `monitoring/` configs (Prometheus/Grafana).
- `deploy/` scripts.
- `docs/` directory.

Provide the monitoring setup and the performance test for the batch processor.
```

---
```
## **PROMPT 10: Advanced Features & Polish**

```
Add advanced features and polish the application for production use.

**Requirements:**

1. **Iterative Idea Refinement:**
   - Add a "Refine this direction" button on the Idea Detail page.
   - User clicks it -> new prompt is sent to a cheaper LLM (Gemini Pro 2.5): "The user found this idea interesting: {idea_json}. Generate 3 variations that dive deeper into {methodology} or apply it to a different dataset."
   - Store these as new `IdeaCandidate` records linked to the parent idea.

2. **Improved Ranking - Learning from Feedback:**
   - Track user actions: "Promote to Prototype" (positive signal), "Archive" (negative signal).
   - Use this data to adjust the weights in the hybrid scoring formula. (e.g., if users keep archiving ideas that require proprietary data, increase the penalty for `data_availability`).

3. **Export & Publishing Tools:**
   - **Manuscript Drafter**:
     - Button on Idea Detail: "Draft Manuscript Outline".
     - Uses an LLM to generate a standard paper outline (Intro, Related Work, Methods, etc.).
     - Populates "Related Work" using the FH Crow summary.
     - Populates "Methods" using the `IdeaCandidate.methodology` and generated code.
     - Export to Overleaf compatible LaTeX or .docx.

4. **Dashboard Improvements:**
   - **Cost Transparency**: Show estimated cost *before* running a batch synthesis or generating a prototype.
   - Bulk Operations: Select multiple ideas to Archive.

**Deliverables:**
- `services/idea_generation/refiner.py`
- Updates to `services/ranking/scorer.py` for feedback learning.
- `services/export/manuscript_drafter.py`
- Corresponding UI updates.
```
