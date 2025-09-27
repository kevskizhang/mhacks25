# Paper-to-Prototype Companion - Engineering Spec (Fetch.ai / uAgents / ASI:One)

> Goal: An agentic workflow that ingests a research paper (PDF/arXiv), proposes an implementable claim, scaffolds a runnable prototype (notebook or small repo), executes it, verifies results, and returns a concise "Reproduction Card". Must support Chat Protocol for ASI:One and be publishable on Agentverse.
> Revision: 2024-03-15 (feasibility-focused compression)

---
## 0) Revision highlights
- Compressed to three uAgents (Orchestrator, Research Analyst, Prototype Engineer) while keeping internal modules that can later be promoted into standalone agents.
- Staged Agentverse integration: build and validate the pipeline locally first, then enable mailbox/manifest and publish once stable.
- Adopted heuristic-first claim mining and notebook scaffolding templates with optional LLM hooks, lowering dependence on external APIs.
- Documented execution safety, job persistence, and test strategy suited to the leaner architecture.

---
## 1) User stories

* **US1 - Prototype from link:** As a user, I send a paper URL; the agents return a prototype run (metric + plot) and a Reproduction Card.
* **US2 - Choose claim:** I can ask "show claims" and select one to prototype.
* **US3 - Status & artifacts:** I can ask for job status, download artifacts, and re-run.
* **US4 - ASI:One chat:** I can do all of the above via chat (Chat Protocol) without any custom UI.
* **US5 - Marketplace:** The agent is discoverable and reachable on Agentverse.

---
## 2) System overview

### Agents (uAgents)

* **Orchestrator** - ASI:One entrypoint, chat intent router, job planner/state manager, persistence, and follow-up messaging.
* **Research Analyst** - fetches PDF/metadata, extracts structured context, proposes 2-3 implementable claims with heuristics and optional LLM assist.
* **Prototype Engineer** - generates the notebook template, optionally synthesises tiny data, executes the run, verifies metrics, and renders the Reproduction Card before returning artifacts.

### Internal services (Python modules)

* `services/pdf_ingest.py` - PyMuPDF extraction, fallback to text-only if layout parsing fails.
* `services/claim_miner.py` - rule-based heuristics for metric extraction, optional LLM prompt hook (behind feature flag).
* `services/notebook_scaffold.py` - Jinja2 rendering of `prototype.ipynb`.
* `services/data_synth.py` - helper for simple datasets (classification, regression, time-series).
* `services/notebook_runner.py` - papermill / nbclient execution with timeout controls.
* `services/verifier.py` - numeric/trend checks matched to claim evidence.
* `services/reporter.py` - Markdown + HTML reproduction card output.
* `services/storage.py` - helpers for job directories, metadata JSON, artifact paths.

### Storage

* Local filesystem under `runtime/jobs/<job_id>/...`
* Lightweight job state persisted as JSON (`meta.json`, `claims.json`, `run.json`) for crash recovery.

### Chat/Marketplace

* Chat Protocol handler implemented in Orchestrator.
* Manifest + metadata prepared early; Agentverse publish occurs after MVP pipeline is stable.
* Mailbox/hosting configured once ready for end-to-end smoke test with ASI:One.

---
## 3) Tech stack

* **Python 3.11+**
* **uagents** (Fetch.ai)
* **Pydantic** for message schemas
* **PyMuPDF** (fallback: `pdfminer.six`) for PDF parsing
* **jinja2** for templated notebook/report generation
* **papermill** (or `nbclient`) for notebook execution
* **matplotlib / numpy / scikit-learn** for CPU-only baselines
* **SQLite** (optional) or JSON files for persistence
* **FastAPI** (optional local status panel)
* Optional: **OpenAI/Fetch LLM API** for advanced claim synthesis (feature-flagged)

---
## 4) Project layout

```
paper-proto/
  agents/
    orchestrator.py
    research_analyst.py
    prototype_engineer.py
    common/messages.py
    common/state.py
    common/utils.py
  services/
    pdf_ingest.py
    claim_miner.py
    notebook_scaffold.py
    data_synth.py
    notebook_runner.py
    verifier.py
    reporter.py
    storage.py
  templates/
    prototype.ipynb.j2
    reproduction_card.md.j2
    README.md.j2
  runtime/
    jobs/<job_id>/
      input.pdf
      meta.json
      claims.json
      prototype.ipynb
      execution.log
      metrics.json
      artifacts/
        *.png *.csv
      report/
        reproduction_card.md
        index.html
  ui/
    server.py   # optional FastAPI panel
  env/
    requirements.txt
  scripts/
    dev_bootstrap.sh
    run_local.sh
  README.md
```

---
## 5) Data models (Pydantic)

```python
# agents/common/messages.py
from pydantic import BaseModel, HttpUrl
from typing import List, Dict, Any, Optional

class StartJob(BaseModel):
    job_id: str
    pdf_url: HttpUrl
    requester: Optional[str] = None  # chat user id

class PaperSection(BaseModel):
    heading: str
    text: str

class PaperMeta(BaseModel):
    title: str
    authors: List[str]
    abstract: str
    sections: List[PaperSection]
    keywords: List[str]
    source_url: HttpUrl

class Claim(BaseModel):
    id: str
    description: str
    evidence: Dict[str, Any]           # {"metric": "accuracy", "target": 0.90, "tolerance": 0.8}
    prerequisites: List[str]           # ["needs: classification data", "lib: sklearn"]
    confidence: str                    # "low" | "medium" | "high"

class AnalystResult(BaseModel):
    job_id: str
    paper: PaperMeta
    claims: List[Claim]
    warnings: List[str] = []

class ClaimChoice(BaseModel):
    job_id: str
    claim_id: str
    auto_selected: bool = False

class PrototypePlan(BaseModel):
    job_id: str
    claim: Claim
    notebook_config: Dict[str, Any]    # {"metric_name": "accuracy", "dataset": "synthetic:classification", ...}
    runtime_hints: Dict[str, Any] = {} # {"timeout": 90}

class ExecutionUpdate(BaseModel):
    job_id: str
    phase: str                         # "scaffolding" | "running" | "verifying" | "reporting"
    message: str

class ExecutionResult(BaseModel):
    job_id: str
    metrics: Dict[str, float]
    artifacts: List[str]               # relative paths
    log_tail: str

class ReportReady(BaseModel):
    job_id: str
    report_path: str                   # runtime/jobs/<id>/report/index.html
    passed: bool
    summary: str
```

**Job state**

```python
# agents/common/state.py
from enum import Enum
class Status(str, Enum):
    QUEUED = "QUEUED"
    ANALYZING = "ANALYZING"
    CLAIM_READY = "CLAIM_READY"
    EXECUTING = "EXECUTING"
    VERIFYING = "VERIFYING"
    REPORTING = "REPORTING"
    DONE = "DONE"
    ERROR = "ERROR"

# meta.json snapshot example
{
  "job_id": "...",
  "status": "EXECUTING",
  "paper": {"title": "...", "authors": [...], "abstract": "..."},
  "claims": [...],           # cached Claim list
  "chosen_claim": {...},     # Claim
  "metrics": {"accuracy": 0.82},
  "artifacts": ["artifacts/confusion.png"],
  "report_path": "report/index.html",
  "warnings": [],
  "errors": []
}
```

---
## 6) Core flows

### F1: Prototype from URL (happy path)

1. User: `prototype <url>` via ASI:One chat.
2. Orchestrator generates `job_id`, persists `meta.json`, sends `StartJob` to Research Analyst, and acknowledges the chat.
3. Research Analyst downloads the PDF, extracts metadata, mines claims, stores `runtime/jobs/<job_id>/claims.json`, and returns `AnalystResult`.
4. Orchestrator updates status to `CLAIM_READY`, surfaces top claims to user (auto-select claim[0] unless the user chooses).
5. When a claim is chosen (`choose <n>` or auto), Orchestrator builds a `PrototypePlan` with notebook/runtime hints and sends it to Prototype Engineer.
6. Prototype Engineer emits `ExecutionUpdate` messages during scaffolding and notebook execution; it writes artifacts under the job directory.
7. On completion, Prototype Engineer returns `ExecutionResult`, runs verifier/report generation internally, and emits a final `ReportReady`.
8. Orchestrator updates status to `DONE` (or `VERIFYING`/`REPORTING` midpoint), saves metrics/report path, and replies in chat with summary + artifact links.

### F2: Status / artifacts

* `status [job_id]` returns current `Status`, phase message, last log lines, and whether claims or metrics are ready.
* `results [job_id]` returns current metrics, artifact filenames, report link, and pass/fail.

### F3: Error recovery

* Any agent can send an error payload; Orchestrator sets `ERROR`, stores reasons, and suggests actions (try alternate claim, run minimal template, switch to text-only ingestion). Jobs remain resumable by re-triggering `run`.

---
## 7) Chat Protocol interface (ASI:One)

**Intents**

* `prototype <url>`
* `show claims [<job_id>]`
* `choose <n> [<job_id>]`
* `run [<job_id>]` (replay last plan)
* `status [<job_id>]`
* `results [<job_id>]`
* `help`

**Chat responses**

* Immediate acknowledgement: "Got it, starting job <job_id>."
* Progress nudges: "Analyzing paper...", "Running notebook (45s timeout)...".
* Final message includes:
  * Short summary of the prototype and claim
  * Metrics comparison (e.g., `accuracy=0.82 vs paper 0.90 (target 0.72)`)
  * Artifact links (`artifacts/confusion.png`)
  * Report link (`report/index.html`)

**Manifest metadata**

* name: `paper-to-prototype-companion`
* description: `Turns research papers into runnable prototypes; returns metrics and plots.`
* inputs: `arXiv/PDF URL`
* actions: prototype, show claims, choose, run, status, results
* tags: education, research, reproducibility
* contact: mailbox + endpoint once published

---
## 8) Research Analyst details

* **PDF pipeline:** download, cache as `input.pdf`, extract metadata via PyMuPDF; fallback to plain text if structured parsing fails; store `paper_meta` JSON.
* **Claim mining:** heuristics scan sections (methods/results) for metric verbs (`accuracy`, `BLEU`, `ROUGE`, etc.) and table captions; optional LLM prompt (flag `USE_LLM_FOR_CLAIMS`) to refine descriptions.
* **Quality gates:** limit to 3 claims, each with a metric rule or qualitative trend; attach warnings when evidence is incomplete (e.g., `["missing target value, defaulting tolerance 0.8"]`).
* **Output:** `AnalystResult` ensures downstream agent always receives at least one fallback "baseline" claim (generic reproduction of experiment setup).

---
## 9) Prototype Engineer pipeline

1. **Plan translation:** merge claim evidence + defaults into `PrototypePlan`.
2. **Scaffold:** render `prototype.ipynb` via Jinja2 with cells for setup, data (real or synthetic), model, evaluation, and artifact saving.
3. **Data synth:** when no dataset is provided, call `data_synth.py` (classification blobs, regression sine wave, etc.) and persist seeds for determinism.
4. **Execute:** run papermill/nbclient with `EXECUTION_TIMEOUT_SEC` (default 120). Stream `ExecutionUpdate` messages and tail logs.
5. **Verify:** compare metrics using `verifier.py` numeric rules; if missing metric, mark as inconclusive but continue.
6. **Report:** render Markdown -> HTML reproduction card in `runtime/jobs/<job_id>/report/`.
7. **Return:** send `ExecutionResult` followed by `ReportReady`.

---
## 10) Job state, persistence, and recovery

* Each job has a directory seeded at creation; all intermediate outputs are idempotent.
* `meta.json` holds the canonical status and is updated atomically after each phase.
* Orchestrator resumes in-flight jobs on startup by reading the job folder and re-subscribing to outstanding work if necessary.
* `execution.log` captures notebook stdout/stderr; last 50 lines are surfaced to the chat on `status`.

---
## 11) Execution safety & resource limits

* Notebook execution runs in a subprocess with:
  * timeout (`EXECUTION_TIMEOUT_SEC`, default 120)
  * max memory clamp (ulimit/psutil where available)
  * artifacts written under the job folder only
* Deny network access inside the notebook by default; allowlist can be added later.
* Sanitize user-provided URLs (only `http(s)`; size limit 25 MB).
* Record environment info (Python version, library versions) in the report for transparency.

---
## 12) Non-functional requirements

* **Performance:** MVP target <= 120 s end-to-end for template claims on CPU.
* **Reliability:** Agents handle retries; messages include `job_id` for idempotency.
* **Observability:** Structured logs with job and phase tags; optional FastAPI endpoint exposes `/healthz` and `/jobs/<id>`.
* **Configurability:** `.env` for mailbox URL, agent keys, feature flags (LLM on/off, claim heuristics thresholds).

---
## 13) Environment & config

`.env`

```
UA_MAILBOX_URL=...
UA_AGENT_KEY=...
HOST_PUBLIC_BASEURL=https://<agentverse-host>/    # used for artifact links
ARTIFACTS_CDN_BASE=...
EXECUTION_TIMEOUT_SEC=120
USE_LLM_FOR_CLAIMS=0
```

`env/requirements.txt`

```
uagents
pydantic
fastapi
jinja2
papermill
nbformat
nbclient
pymupdf
pdfminer.six
matplotlib
numpy
scikit-learn
```

---
## 14) Makefile / scripts

```
make setup        # create venv, pip install -r env/requirements.txt
make dev          # run orchestrator + analysts locally (uvicorn or simple loop)
make analyst      # run Research Analyst agent standalone (debug)
make engineer     # run Prototype Engineer agent standalone (debug)
make test         # unit tests + smoke e2e on canned paper
make run-job URL=<arxiv_or_pdf_url>
```

---
## 15) Testing plan

* **Unit:** pdf_ingest (given fixture PDF), claim_miner heuristics (returns <=3 claims), verifier numeric rules.
* **Service:** prototype_engineer executes notebook template on synthetic dataset and produces metrics/artifact.
* **Integration:** orchestrator -> analyst -> engineer loop on a canned arXiv PDF stored locally.
* **Chat:** simulate Chat Protocol intents (`prototype`, `show claims`, `choose`, `status`, `results`) using uAgents test harness.
* **Artifacts:** assert `metrics.json` exists, artifact count >= 1, report HTML contains summary and metric table.

---
## 16) Agentverse integration plan

1. **Local MVP:** run all three agents locally, validate end-to-end job success, and capture test transcripts.
2. **Staging mailbox:** enable mailbox in env, run smoke tests via CLI to ensure intents still resolve.
3. **Manifest prep:** finalize metadata, icons, and descriptive text; verify health endpoint returns 200.
4. **Publish:** register the orchestrator agent on Agentverse with Chat Protocol declared; list Research Analyst and Prototype Engineer as cooperative agents in the description.
5. **ASI:One smoke test:** execute `prototype <url>` from ASI:One chat and capture the Reproduction Card link as evidence for submission.
6. **Monitoring:** track job metrics and errors post-publish; provide fallback command (`help`) with troubleshooting tips.

---
## 17) Stretch goals (post-MVP)

* Fetch real datasets via HuggingFace with license filtering.
* Support multiple claims per job and comparison dashboard.
* Introduce a reproducibility score rubric.
* Shared jobs: allow job IDs to be reopened by collaborators.

---
## 18) Implementation tasks

1. Bootstrap repo, virtualenv, and project layout.
2. Implement shared utilities (`storage`, `state`, `messages`, logging helpers).
3. Build Research Analyst agent: PDF ingest + claim miner + message handlers.
4. Build Prototype Engineer agent: plan ingestion, scaffold, run, verify, report.
5. Wire Orchestrator agent: chat intent parser, job planner, persistence.
6. Create Jinja templates (`prototype.ipynb.j2`, `reproduction_card.md.j2`, `README.md.j2`).
7. Implement data synthesis helpers and verifier numeric checks.
8. Implement execution safety (timeouts, sandbox) and artifact handling.
9. Write tests (unit + service + integration) plus fixtures for canned PDFs.
10. Add CLI scripts / Make targets and optional FastAPI UI.
11. Prepare Agentverse manifest, mailbox config, and deployment docs.
12. Record demo script (chat transcript + artifact screenshots).

---
## 19) Example Chat UX (copy text for tests)

* `prototype https://arxiv.org/pdf/XXXX.XXXXX.pdf`
* `show claims`
* `choose 2`
* `run`
* `status`
* `results`

---
## 20) Sample intent router (drop-in)

```python
# orchestrator_chat.py
INTENTS = {
    "prototype": r"^prototype\s+(?P<url>\S+)",
    "show_claims": r"^show\s+claims(\s+(?P<job>\S+))?$",
    "choose": r"^choose\s+(?P<idx>\d+)(\s+(?P<job>\S+))?$",
    "run": r"^run(\s+(?P<job>\S+))?$",
    "status": r"^status(\s+(?P<job>\S+))?$",
    "results": r"^results(\s+(?P<job>\S+))?$",
    "help": r"^(help|\?)$"
}
```

---
## 21) Notebook code block (model/eval minimal snippet)

```python
# in prototype.ipynb template (code cell)
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, ConfusionMatrixDisplay
rng = np.random.default_rng(42)

# data (or loaded externally)
X, y = make_classification(
    n_samples=800,
    n_features=8,
    n_informative=5,
    random_state=42
)
Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.25, random_state=42)

clf = LogisticRegression(max_iter=500)
clf.fit(Xtr, ytr)
pred = clf.predict(Xte)
acc = accuracy_score(yte, pred)

# save metric
import json, os
os.makedirs("artifacts", exist_ok=True)
with open("metrics.json", "w") as f:
    json.dump({"accuracy": float(acc)}, f)

# plot artifact
disp = ConfusionMatrixDisplay.from_predictions(yte, pred)
plt.tight_layout()
plt.savefig("artifacts/confusion.png")
plt.close()
print("accuracy", acc)
```

---
## 22) Acceptance criteria (MVP)

* Given a valid PDF URL, within 120 seconds the orchestrator+analyst+engineer trio return:
  * At least one viable claim (auto-selected or user-chosen).
  * A run metric (JSON) and at least one artifact `.png`.
  * A generated Reproduction Card (HTML) stored under `report/index.html`.
  * A clear Chat response with summary, metric comparison, and artifact link(s).
* Orchestrator agent registered on Agentverse with Chat Protocol enabled and reachable via ASI:One chat.
