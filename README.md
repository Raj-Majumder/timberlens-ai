# TimberLens AI — Multi-Agent Business Intelligence System

> A production multi-agent AI system built for TimberLens Creations LLC — a bespoke hardwood guitar stand workshop in Richardson, TX. Designed, built, and validated end-to-end by the business owner as a live testbed for agentic AI architecture.


Portfolio note: This repository contains the full architecture and source code. The system runs against private business data (Excel financials) and a local reference CLI (Chub) that are not included in this repo. See the Setup section for how to adapt it to your own data. This project was built to demonstrate that the same agentic AI architecture patterns used in enterprise AI strategy can be designed, validated, and operated at small-business scale — with real data, real outputs, and real business decisions as the test cases.
---

## What it does

TimberLens AI turns a single natural-language prompt into a fully synthesised executive report — drawing on live business data, consulting five specialised AI agents, and optionally delivering the output as a formatted PDF via email.

**Example prompt:**
```
"Do a competitive analysis of guitar stands in my area, send me email with pdf"
```

**What happens:**
1. Live Excel business plan data is synced into shared agent memory
2. Five specialist agents are consulted in parallel (Finance, Supply Chain, Marketing, Master Artisan, Web/Dev)
3. A Manager agent synthesises all inputs into a structured executive report
4. Output is rendered as formatted Markdown — and if requested, saved as a PDF and dispatched by email

---

## Architecture

```
User Prompt
    │
    ▼
┌─────────────────────────────────────┐
│         sync_excel_to_state()        │  ← Reads live FY2026 Excel plan
│         System_State.md             │  ← Shared memory for all agents
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                  Orchestrator                        │
│                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │Master Artisan│  │SupplyChain   │  │ Finance   │  │
│  └─────────────┘  └──────────────┘  └───────────┘  │
│  ┌─────────────┐  ┌──────────────┐                  │
│  │ Marketing   │  │  Web/Dev     │                  │
│  └─────────────┘  └──────────────┘                  │
│          │  any agent can call        │              │
│          ▼                           │              │
│  ┌──────────────────────────┐        │              │
│  │  fetch_hub_docs(doc_id)  │        │              │
│  │  Context Hub (Chub)      │ ← Long-term reference │
│  │  Local verified docs     │        library        │
│  └──────────────────────────┘                       │
│                  ▼                                  │
│           Manager Agent (synthesis)                 │
└─────────────────────────────────────────────────────┘
    │
    ▼
Output: Markdown report → optional PDF + Email dispatch
```

### Key components

| File | Role |
|------|------|
| `TimberLens_Master.ipynb` | Main orchestrator — runs the full agent pipeline |
| `Skills_Factory.ipynb` | CI layer — syncs and audits all skill files before a run |
| `Skills/Manager.md` | Manager agent persona + synthesis instructions |
| `Skills/Master Artisan.md` | Craft, production, and CAD design expertise |
| `Skills/Finance.md` | COGS, margin, pricing, and financial projection logic |
| `Skills/SupplyChain.md` | Wood sourcing, supplier relationships, logistics |
| `Skills/Marketing.md` | Competitive analysis, positioning, channel strategy |
| `Skills/WebDev.md` | E-commerce, SEO, platform recommendations |
| `Skills/System_State.md` | Auto-generated shared memory — synced from Excel on each run |
| `Context Hub (Chub)` | Local CLI tool — serves verified reference docs to agents on demand via `fetch_hub_docs(doc_id)` |

---

## Data architecture

The system uses a **structured state-sync pattern** as its retrieval layer:

1. On each orchestrator run, `sync_excel_to_state()` reads all 10 sheets from the FY2026 business plan Excel file
2. The full structured data is written to `System_State.md`
3. Every agent call injects both its role-specific persona (from `.md` skill file) and the live business state into the system prompt
4. Agents respond with domain-specific analysis grounded in the actual business numbers

**Live data available to all agents:**

| Excel Sheet | Contents |
|-------------|----------|
| `DASHBOARD` | KPI summary — revenue, margins, targets |
| `PL_Key_Metrics` | Profit & loss projections |
| `COGS_Per_Item` | Per-unit cost breakdown (Walnut: $132.27 / Cherry: $123.40) |
| `Wood_Sourcing` | Supplier details and lead times |
| `Wood_Cost` | Board-foot pricing and consumption per unit |
| `Labor` | Labor rate ($26.67/hr), time per unit (1.83 hrs) |
| `Consumables_Others` | Finish, hardware, consumable costs |
| `Business_Details` | Company info, pricing ($350 Walnut / $325 Cherry) |
| `Equipment_List` | Tools and capital equipment inventory |
| `Log` | Production and sales activity log |

---

## Context Hub (Chub) — long-term reference library

The Context Hub solves a specific problem that arises in multi-agent systems: **agents hallucinating technical details or business rules** when that knowledge isn't in their immediate context.

### The problem it solves

`System_State.md` provides live *operational* data (financials, costs, inventory). But agents also need access to stable *reference* knowledge — wood species specifications, finishing technique documentation, joinery standards, supplier terms, business rules — information that doesn't change run-to-run but is too large to inject into every prompt.

Without a solution, agents either guess (hallucinate) or give generic answers. Chub gives them a way to fetch the exact document they need, exactly when they need it.

### How it works

A local CLI tool (`chub`) manages a curated library of verified reference documents. Any agent skill file (e.g. `WebDev.md`, `Master Artisan.md`) can call the bridge function at inference time:

```python
def fetch_hub_docs(doc_id):
    """Fetches clean docs from the Hub without cluttering the notebook."""
    result = subprocess.run(['chub', 'get', doc_id, '--lang', 'py'],
                            capture_output=True, text=True)
    return result.stdout if result.returncode == 0 else "Doc not found."
```

The agent receives the retrieved document as additional context and grounds its response in verified content rather than parametric memory.

### Two-layer memory architecture

| Layer | Source | Type | Updates |
|-------|--------|------|---------|
| `System_State.md` | Excel business plan | Live operational data | Every orchestrator run |
| Context Hub (Chub) | Curated doc library | Stable reference knowledge | Manually, when docs change |

This separation keeps the system prompt lean — agents pull reference docs only when their task requires them — while ensuring responses are always grounded in verified business and technical knowledge.

### Prerequisites for Chub

The Context Hub Bridge requires the `chub` CLI to be installed locally. If not present, `fetch_hub_docs()` returns a graceful fallback:
```
"Chub CLI not installed. Run API_Setup.ipynb first."
```

---

## Sample output

The system has successfully generated:

- **Competitive market analysis** — 11-section executive report with market segmentation, competitor threat matrix, SWOT, pricing map, and prioritised recommendations with ROI projections
- **Financial scenario modelling** — multi-initiative impact tables with projected revenue lift ($61,500 → $98,500) and per-initiative ROI
- **Supply chain assessments** — wood sourcing optimisation against live cost and supplier data
- **Strategic roadmaps** — quarterly action plans with owner assignment and expected impact

---

## Tech stack

- **Language:** Python 3.12
- **AI:** Anthropic API (`claude-opus-4-6`) via `anthropic` Python SDK
- **Data:** `pandas` for Excel ingestion; structured Markdown for agent memory
- **Reference library:** Context Hub (`chub`) CLI — on-demand verified doc retrieval for agents
- **Output:** `fpdf` for PDF generation; `smtplib` for email dispatch; `markdown` for HTML rendering
- **Environment:** JupyterLab (Anaconda) — desktop app migration in progress

---

## Known limitations and upgrade roadmap

This system is a validated proof of concept. The following limitations are understood and inform the next development phase:

**Current limitations:**

| Limitation | Impact | Planned fix |
|------------|--------|-------------|
| Brute-force context injection | All 10 Excel sheets injected into every agent call regardless of relevance — inefficient token usage | Replace with ChromaDB vector retrieval; each agent retrieves only relevant sheets/rows |
| No unstructured document support | System can only reason over structured Excel data | Add LangChain document ingestion pipeline for PDFs, emails, supplier quotes |
| Hardcoded local file paths | Not portable across machines | Refactor to relative paths + `.env` config file |
| Jupyter-only rendering | `display(Markdown())` is Jupyter-specific | Abstract output layer for desktop UI migration |
| No agent output evaluation | No automated quality scoring on generated reports | Integrate RAGAS-style faithfulness and relevancy metrics |

**Next phase — desktop app + true RAG:**
- Refactor orchestrator into standalone Python module (`timberlens_core.py`)
- Replace `System_State.md` inject with per-agent ChromaDB semantic retrieval
- Add LangChain document pipeline for unstructured business documents
- Package as desktop app (Tkinter or Electron wrapper)
- Add conversation history so the system supports follow-up queries

---

## Setup

### Prerequisites
```
Python 3.12+
Anthropic API key
```

### Installation
```bash
git clone https://github.com/Raj-Majumder/timberlens-ai.git
cd timberlens-ai
pip install anthropic pandas fpdf markdown python-dotenv
```

### Configuration
Create a `.env` file in the project root:
```
ANTHROPIC_API_KEY=sk-ant-your-key-here
EXCEL_FILE_PATH=/path/to/your/TimberLens_Business_Plan_FY2026.xlsx
SKILLS_PATH=/path/to/your/Skills/
```

> **Security note:** Never commit your `.env` file or API key to version control. The `.gitignore` in this repo excludes `.env` and all `.rtf` credential files.

### Running
1. Run `Skills_Factory.ipynb` first — confirms all 8 skill files are present and synced
2. Open `TimberLens_Master.ipynb`
3. Run all cells in order (System Configurator → Email/PDF Generator → Orchestrator Engine)
4. Edit the `user_request` string in the Command Console cell and run

---

## About TimberLens Creations

TimberLens Creations LLC is a bespoke woodworking business based in Richardson, TX (DFW), specialising in handcrafted solid hardwood guitar stands in Walnut and Cherry. 

This AI system was built by the owner: Rajarshi (Raj) Majumder to demonstrate that the same agentic architecture patterns used in enterprise AI strategy can be designed, validated, and operated at small-business scale — with real data, real outputs, and real business decisions as the test cases.

---

