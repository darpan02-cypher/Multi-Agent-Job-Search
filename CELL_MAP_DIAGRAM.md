# Execution Flow & Cell Map

## Complete Notebook Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-AGENT JOB SEARCH WORKFLOW                          │
│                          Execution Flow Diagram                             │
└─────────────────────────────────────────────────────────────────────────────┘

SETUP PHASE (Cells 1-5)
═════════════════════════════════════════════════════════════════════════════

  ┌──────────────────┐
  │  CELL 1 (Info)   │  Setup instructions
  │ "IMPORTANT:..."  │  └─ Explains API key requirement
  └────────┬─────────┘    └─ Shows Groq website link
           │
           ▼
  ┌──────────────────┐
  │  CELL 2 (Check)  │  Validate GROQ_API_KEY
  │ Read env var     │  └─ Prints: "✓ GROQ_API_KEY is set"
  └────────┬─────────┘    └─ OR: "⚠️  GROQ_API_KEY NOT set"
           │
           ▼
  ┌──────────────────┐
  │  CELL 3          │  Install dependencies
  │ %pip install...  │  └─ langgraph, langchain-groq, pymupdf, etc
  └────────┬─────────┘    └─ Takes ~30 seconds
           │
           ▼
  ┌──────────────────┐
  │  CELL 4          │  Verify installation
  │ Check versions   │  └─ Prints package versions
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │  CELL 5          │  Initialize LLM
  │ ChatGroq()       │  └─ Creates Groq LLM instance
  └────────┬─────────┘    └─ Ready for agent use
           │

CONFIGURATION PHASE (Cells 6-9)
═════════════════════════════════════════════════════════════════════════════
           │
           ▼
  ┌──────────────────┐
  │  CELL 6          │  Define Tools
  │ @tool decorators │  ├─ YCombinatorSearch
  └────────┬─────────┘  ├─ ParseResumePDF
           │            └─ WriteTailoredResumeSection
           │
           ▼
  ┌──────────────────┐
  │  CELL 7          │  Build Agent Executors
  │ create_tool_...  │  ├─ Search Agent (YCombinatorSearch)
  └────────┬─────────┘  ├─ Screen Agent (ParseResumePDF)
           │            └─ Tailor Agent (ParseResumePDF + WriteTailedResume)
           │            └─ Wrapped with AgentExecutor
           │
           ▼
  ┌──────────────────┐
  │  CELL 8          │  Build Graph
  │ StateGraph()     │  ├─ Node: search_agent
  └────────┬─────────┘  ├─ Node: enrich_node (conditional)
           │            ├─ Node: screen_agent
           │            ├─ Node: human_select_interrupt 🟡
           │            ├─ Node: tailor_agent
           │            └─ compile(checkpointer=MemorySaver)
           │
           ▼
  ┌──────────────────┐
  │  CELL 9 (Opt)    │  Visualize Graph
  │ .draw_mermaid_.. │  └─ Creates workflow_graph.png
  └────────┬─────────┘

EXECUTION PHASE (Cells 10-12)
═════════════════════════════════════════════════════════════════════════════
           │
           ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  CELL 10: Graph Execution                │
  │            graph_app.invoke(initial_state)               │
  │                                                          │
  │  input = {                                               │
  │    "seed_terms": ["python", "full stack", "ml"],         │
  │    "location": "San Francisco",                          │
  │    "resume_pdf_path": "Himanshi Shrivas..."             │
  │  }                                                       │
  └──────────────┬───────────────────────────────────────────┘
                 │
        ┌────────▼────────┐
        │   SEARCH NODE    │
        ├──────────────────┤
        │ invoke_search()  │
        │    Agent:        │
        │ "Given terms...  │
        │  call YC tool"   │
        └────────┬────────┘
                 │
        ┌────────▼────────────────────────┐
        │    YCombinatorSearch Tool        │
        ├──────────────────────────────────┤
        │ from yc_search import yc_search  │
        │ yc_search(terms, location, ...)  │
        └────────┬───────────────────────┬─┘
                 │                       │
          ┌──────▼──────┐         ┌──────▼──────┐
          │ HN Algolia  │   OR    │ Fallback    │
          │   API Call  │         │ Stub Jobs   │
          │ ✅ Success  │         │ (if error)  │
          └──────┬──────┘         └──────┬──────┘
                 │                       │
                 └───────────┬───────────┘
                             │
                    Returns: List[Jobs]
                    Example output:
                    [{
                      "id": "hn_111",
                      "title": "ML Engineer",
                      "company": "StartupA",
                      "location": "San Francisco",
                      "description": "...",
                      "url": "..."
                    }, ...]
                             │
                             ▼
        ┌────────────────────────────────┐
        │    CONDITIONAL: needs_enrich?  │
        │  (if len(jobs) > 12)            │
        ├────────────────────────────────┤
        │  YES → enrich_node              │
        │  NO  → screen_agent            │
        └────────┬───────────┬────────────┘
                 │           │
        ┌────────▼──┐  ┌─────▼──────────┐
        │ Enrich    │  │ Screen (skip)  │
        │ Add tags  │  └─────┬──────────┘
        └────────┬──┘        │
                 │           │
                 └────┬──────┘
                      │
                      ▼
        ┌────────────────────────────────┐
        │    SCREEN NODE                 │
        ├────────────────────────────────┤
        │ invoke_screen(jobs, resume)    │
        │    Agent:                      │
        │ "Parse resume, rank jobs by    │
        │  relevance"                    │
        └────────┬────────────────────────┘
                 │
        ┌────────▼────────────────────────┐
        │    ParseResumePDF Tool           │
        ├──────────────────────────────────┤
        │ import pymupdf                   │
        │ doc.get_text() for each page    │
        └────────┬─────────────────────────┘
                 │
              Returns: Text
              (Full resume transcribed)
                 │
                 ▼
        ┌─────────────────────────────┐
        │ Agent Ranks Jobs vs Resume  │
        │ Match python to job descr?  │
        │ Match ML to job descr?      │
        │ Match experience?           │
        │ Score & sort top 10 jobs    │
        └────────┬────────────────────┘
                 │
              Returns: List[TopJobs]
                 │
                 ▼
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │          🟡 GRAPH INTERRUPTED FOR HUMAN INPUT           │
  │                                                          │
  │  Payload contains:                                       │
  │  {                                                       │
  │    "instruction": "Select top 2 job IDs...",            │
  │    "ranked_jobs": [                                      │
  │      {"id": "hn_111", "title": "ML Engineer", ...},     │
  │      {"id": "hn_222", "title": "Backend Dev", ...},     │
  │      ...9 more...                                       │
  │    ]                                                    │
  │  }                                                      │
  │                                                          │
  │  >>> YOU PICK 2 JOBS HERE <<<                           │
  │  selected_ids = ["hn_111", "hn_222"]                    │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────┐
  │              CELL 11: Resume Graph Execution             │
  │        graph_app.invoke(Command(resume=selected_ids))    │
  │                                                          │
  │  (Continue from interrupt with your 2 selections)       │
  └──────────────┬───────────────────────────────────────────┘
                 │
        ┌────────▼────────┐
        │   TAILOR NODE    │
        ├──────────────────┤
        │ invoke_tailor()  │
        │    Agent:        │
        │ "Parse resume,   │
        │  create 4-6      │
        │  bullets per job"│
        └────────┬────────┘
                 │
        ┌────────▼─────────────────────────────┐
        │   ParseResumePDF Tool (again)         │
        ├──────────────────────────────────────┤
        │ Reload resume text for tailoring     │
        └────────┬───────────────────────────┬─┘
                 │ Returns: Resume Text      │
                 │                           │
        ┌────────▼──────────────────────┐   │
        │  For each selected job:        │   │
        │  1. Match skills from resume  │   │
        │  2. Generate 4-6 bullets     │   │
        │  3. Keep bullets < 300 chars │   │
        └────────┬──────────────────────┘   │
                 │                           │
        ┌────────▼────────────────────────────────────────┐
        │  WriteTailoredResumeSection Tool (for each job) │
        ├─────────────────────────────────────────────────┤
        │ write_tailored_resume_section(                  │
        │   path="tailored_resume.txt",                   │
        │   content="===== JOB: hn_111 =====             │
        │            • Bullet 1...                       │
        │            • Bullet 2..."                      │
        │ )                                               │
        │                                                  │
        │ Appends to file (doesn't overwrite)            │
        └────────┬─────────────────────────────────────┬──┘
                 │ For job 1...                       │
                 │                                    │
        ┌────────▼──────────┐         ┌───────────────▼──┐
        │ Appends to file   │         │ For job 2...     │
        │ tailored_resume.  │         │ Appends more     │
        │ txt (job 1)       │         │ tailored_resume │
        │                   │         │ .txt (job 2)     │
        └────────┬──────────┘         └───────────┬──────┘
                 │                               │
                 └──────────┬────────────────────┘
                            │
                   Returns: Dict[job_id → preview]
                   {
                     "hn_111": "• Led ML pipeline...",
                     "hn_222": "• Built backend..."
                   }
                            │
                            ▼
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │                  END OF WORKFLOW                        │
  │          Graph returns final_result to user             │
  │                                                          │
  │  final_result contains:                                 │
  │  {                                                       │
  │    "seed_terms": [...],                                 │
  │    "location": "...",                                   │
  │    "jobs": [...50 original jobs...],                    │
  │    "ranked_jobs": [...10 screened jobs...],             │
  │    "selected_jobs": [...2 user selected...],            │
  │    "tailored_resumes": {                                │
  │      "hn_111": "bulletspreview...",                     │
  │      "hn_222": "bulletspreview..."                      │
  │    }                                                    │
  │  }                                                      │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────┐
  │                  CELL 12: Display Results               │
  │                                                          │
  │  Print final_result to user:                            │
  │  ✓ Tailored resumes keys: ['hn_111', 'hn_222']         │
  │  ✓ Sample preview of content                           │
  │  ✓ Check tailored_resume.txt for full output           │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## Cell-by-Cell Reference

| Cell # | Name | Input | Processing | Output | Status |
|--------|------|-------|-----------|--------|--------|
| 1 | Setup Info | - | Print instructions | Help message | ℹ️ Reference |
| 2 | API Check | env var | Read GROQ_API_KEY | "✓ set" or "⚠️ not set" | ✅ Validate |
| 3 | Install | pip | Install packages | "Successfully installed" | ✅ Setup |
| 4 | Versions | imports | Check versions | Package version list | ✅ Verify |
| 5 | LLM Config | API key | Create ChatGroq | llm instance | ✅ Init |
| 6 | Tools | imports | Define @tool functions | 3 tool objects | ✅ Define |
| 7 | Agent Construction | tools + llm | Create agents + executors | 3 executor objects | ✅ Build |
| 8 | Graph Config | nodes + edges | Build StateGraph | graph_app (compiled) | ✅ Compile |
| 9 | Visualization | graph | Draw Mermaid | workflow_graph.png | ⏭️ Optional |
| 10 | Graph Exec | initial_state | Run workflow (search+screen) | first_result | ⚙️ **MAIN** |
| 11 | Human Select | interrupt payload | User picks 2 jobs | Command + resume | 🟡 **INTERRUPT** |
| 12 | Tailor + Results | selected_jobs | Generate tailored content | final_result | 📊 **OUTPUT** |

---

## Key Decision Points

### After Cell #10: Check Interrupt

```python
interrupts = first_result.get("__interrupt__", [])

if interrupts:
    # ✅ SUCCESS - workflow paused at human_select node
    payload = interrupts[0].value
    ranked_jobs = payload.get("ranked_jobs", [])
    print(f"Found {len(ranked_jobs)} relevant jobs")
    
    # Print preview
    for i, job in enumerate(ranked_jobs[:5]):
        print(f"{i+1}. {job['id']}: {job['title']} @ {job['company']}")
else:
    # ❌ FAIL - workflow didn't interrupt (something went wrong)
    print("ERROR: No interrupt received")
    print("Debug:")
    print(f"  - Jobs found: {len(first_result.get('jobs', []))}")
    print(f"  - Ranked jobs: {len(first_result.get('ranked_jobs', []))}")
```

### After Cell #11: Check Results

```python
final_result = graph_app.invoke(Command(resume=selected_ids), config=config)

tailored = final_result.get("tailored_resumes", {})

if tailored:
    # ✅ SUCCESS - tailored content generated
    print(f"✓ Generated content for {len(tailored)} jobs")
    for job_id, preview in tailored.items():
        print(f"\n{job_id}:")
        print(preview[:300])
else:
    # ❌ FAIL - no tailored content
    print("⚠️ No tailored content generated")
    if "error" in tailored:
        print(f"Error: {tailored['error']}")
```

---

## State Progression During Execution

```
CELL 10 START
initial_state = {
  "seed_terms": ["python", "full stack", "ml"],
  "location": "San Francisco",
  "resume_pdf_path": "Himanshi Shrivas-CV..."
}
              ↓
         [search_agent]
              ↓
first_result.get("jobs") = [job1, job2, ..., job50]
first_result.get("needs_enrichment") = False (since < 12 jobs)
              ↓
         [screen_agent]
              ↓
first_result.get("ranked_jobs") = [top_job1, ..., top_job10]
first_result.get("shortlisted_jobs") = [top_job1, ..., top_job10]
              ↓
      [human_select_interrupt] ← 🟡 PAUSES HERE
              ↓
User selects 2 jobs: ["hn_111", "hn_222"]


CELL 11 RESUME
selected_ids = ["hn_111", "hn_222"]
first_result.get("selected_jobs") = [job_111, job_222]
              ↓
         [tailor_agent]
              ↓
final_result.get("tailored_resumes") = {
  "hn_111": "• Bullet 1...",
  "hn_222": "• Bullet 1..."
}
              ↓
         [END] ✅
```

---

## Troubleshooting Using This Diagram

**Problem: "No interrupt received (Cell 10 fails)"**
1. Check if agents created correctly → See Cell 7 output
2. Check if tools defined → See Cell 6 output
3. Check if graph compiled → See Cell 8 output
4. Debug: Print `first_result.keys()` to see what was returned

**Problem: "Interrupt received but no ranked_jobs"**
1. Screen agent didn't execute → Check resume file exists
2. Check resume path in initial_state (Cell 10)
3. Debug: Print `len(first_result.get("ranked_jobs", []))`

**Problem: "Tailor agent fails (Cell 11 timeout)"**
1. LLM may be slow → Wait 30+ seconds
2. Selected_jobs incorrect → Verify job structure
3. Resume path missing → Check resume file exists
4. Debug: Print selected jobs and resume text

---

## Architecture Summary

```
┌─────────────────────────────────────────┐
│     Notebook (Orchestrator)             │
│  - Defines agents                       │
│  - Manages graph state                  │
│  - Handles human input                  │
└────────┬──────────────────┬─────────────┘
         │                  │
    ┌────▼───────┐  ┌──────▼─────────┐
    │ yc_search  │  │   Your Resume  │
    │ (HN API)   │  │   PDF File     │
    └────────────┘  └────────────────┘

Agents → Tools → External Resources
```

The diagram shows you exactly where each cell fits and what happens at each step. Use it for reference when troubleshooting! 🎯
