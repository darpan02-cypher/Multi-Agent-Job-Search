# Multi-Agent Job Search System - Execution Guide & Troubleshooting

## System Overview

This system is a **multi-agent LangGraph workflow** that:
1. **Searches** Y Combinator jobs using the HN Algolia API (via `yc_search.py`)
2. **Screens** jobs against your resume
3. **Selects** jobs via human-in-the-loop interrupts
4. **Tailors** resume bullets for selected jobs using Groq LLM

### Architecture

```
User Input (search terms, location, resume path)
          ↓
[Search Agent] → Calls YCombinatorSearch tool → yc_search.py
          ↓
      [Decision] → If >12 jobs: enrich, else skip
          ↓
[Screen Agent] → Parses resume + ranks jobs
          ↓
[Human Interrupt] → User selects 2 jobs
          ↓
[Tailor Agent] → Generates tailored resume bullets
          ↓
      Output → tailored_resume.txt
```

---

## Files in the System

| File | Purpose |
|------|---------|
| `MultiAgentJobSearchWorkflow (1).ipynb` | Main Jupyter notebook with LangGraph workflow |
| `yc_search.py` | Standalone Python script for Y Combinator job search |
| `Himanshi Shrivas-CV -Software_Engineer.pdf` | Resume file (example) |
| `tailored_resume.txt` | Output file (created by WriteTailoredResumeSection tool) |

---

## How to Run the System

### Step 1: Prerequisites

Ensure you have:
- **Python 3.9+**
- **Jupyter installed** (for notebook execution)
- **Groq API Key** - Get it from https://console.groq.com/

### Step 2: Install Dependencies

Run this in the terminal from the workspace root:

```bash
pip install -q langgraph langchain langchain-groq openai pymupdf requests pydantic
```

Or run the "Install packages" cell in the notebook.

### Step 3: Set Groq API Key (CRITICAL - Do NOT expose in code!)

**Option A: Set as environment variable (recommended)**
```bash
export GROQ_API_KEY="your-actual-key-here"
```

Then run Jupyter:
```bash
jupyter notebook
```

**Option B: Set in notebook cell (at the top)**
```python
import os
os.environ["GROQ_API_KEY"] = "your-actual-key-here"  # MOVE TO .env file later!
```

### Step 4: Run the Notebook Cells in Order

Follow the **"Quick Start"** section in the notebook:

1. ✅ **Install packages** cell
2. ✅ **Check API key** cell (should print "GROQ_API_KEY is set")
3. ✅ **OpenAI LLM Configuration** cell (loads Groq LLM)
4. ✅ **ReAct Agent Tools** cell (defines search, resume parsing, resume writing tools)
5. ✅ **Agent Construction** cell (builds the agent executors)
6. ✅ **LangGraph Configuration** cell (compiles the workflow graph)
7. ✅ **Graph Visualization** cell (optional, shows workflow diagram)
8. ✅ **Graph Execution** cell (runs search → screen)
9. ✅ **Human Interaction** cell (resumesafter interrupt)
10. ✅ **Final Result** cell (displays tailored resumes)

---

## Running `yc_search.py` Standalone

If you want to test the job search independently:

### CLI Usage

```bash
python yc_search.py --terms python ml --location Remote --limit 25 --verbose
```

**Arguments:**
- `--terms`: Space-separated search keywords (e.g., `python ml data`)
- `--location`: Location filter (e.g., `Remote`, `San Francisco`)
- `--limit`: Max results to return (default: 25)
- `--verbose`: Print debug info to stderr

**Output:** JSON array of job objects

### Python API Usage

```python
from yc_search import yc_search

jobs = yc_search(
    terms=["python", "ml"],
    location="Remote",
    limit=25,
    verbose=True
)

for job in jobs:
    print(f"{job['title']} @ {job['company']} ({job['location']})")
    print(f"  ID: {job['id']}")
    print(f"  Link: {job['url']}")
    print()
```

---

## Common Issues & Fixes

### Issue 1: GROQ_API_KEY NOT SET

**Error Message:**
```
ValueError: GROQ_API_KEY is not set
```

**Fix:**
```bash
# In your shell, BEFORE starting Jupyter
export GROQ_API_KEY="gsk_..."
jupyter notebook
```

Or set it in the notebook (but DO NOT commit to git):
```python
import os
os.environ["GROQ_API_KEY"] = "gsk_..."
```

---

### Issue 2: Graph Execution Returns Empty `__interrupt__`

**Symptom:**
```
first_result = {'__interrupt__': [], 'seed_terms': ['python', 'full stack', 'ml']}
```
The workflow doesn't proceed past search.

**Causes & Fixes:**

1. **Agent Executor Not Working** → Re-run the "Agent Construction" cell
2. **LLM Not Responding** → Check GROQ_API_KEY is set
3. **Fallback Jobs Used** → Check for error messages in the search agent output
4. **Network Issue** → Retry; HN Algolia may rate-limit briefly

**Debug:** Add `verbose=True` to `invoke_search()` calls:
```python
jobs = invoke_search(["python"], verbose=True)
print(f"Found {len(jobs)} jobs")
```

---

### Issue 3: Resume File Not Found

**Error Message:**
```
FileNotFoundError: [Errno 2] No such file or directory: 'MyResume.pdf'
```

**Fix:**
1. Ensure resume is in the same directory as the notebook
2. Update `resume_pdf_path` in the initial_state:
   ```python
   initial_state = {
       "seed_terms": ["python", "ml"],
       "location": "San Francisco",
       "resume_pdf_path": "Himanshi Shrivas-CV -Software_Engineer.pdf"  # CORRECT NAME
   }
   ```

---

### Issue 4: Cells Fail Due to Import Errors

**Error:** `ModuleNotFoundError`

**Fix:**
```bash
# Restart the kernel first
pip install --upgrade langgraph langchain langchain-groq pymupdf
# Then in notebook: Kernel → Restart Kernel
```

Then re-run cells from the top in order.

---

### Issue 5: Agent Returns Non-JSON Output

**Symptom:**
```
extract_json_from_markdown([]) returns empty list
```

**Causes:**
- Model hallucination or timeout
- Prompt not clear enough
- LLM chose to explain instead of output JSON

**Fixes:**
1. Re-run the cell (models are non-deterministic)
2. Check if `tailor_agent_executor.invoke()` is actually using the LLM
3. Enable `verbose=True` in the invoke functions to see full agent output

---

### Issue 6: `yc_search.py` Returns No Results

**Cause:** No matching jobs found in the latest "Who is hiring?" thread.

**Debug:**
```bash
python yc_search.py --terms python --verbose
```

Check stderr output. If thread_id is None:
- The latest thread may have expired or been deleted
- Algolia API may have rate-limited
- Network connection issues

**Workaround:** Test with well-known terms:
```bash
python yc_search.py --terms engineer --limit 10 --verbose
```

---

## Workflow State Transitions

```
INITIAL STATE:
{
  "seed_terms": ["python", "ml"],
  "location": "San Francisco",
  "resume_pdf_path": "path/to/resume.pdf"
}

↓ [n_search]

AFTER SEARCH:
{
  "jobs": [
    {"id": "hn_123", "title": "ML Engineer", "company": "Startup", ...},
    {"id": "hn_124", "title": "Backend Dev", ...},
    ...
  ],
  "needs_enrichment": False  # or True if >12 jobs
}

↓ [n_enrich] ← OPTIONAL (only if needs_enrichment=True)

AFTER ENRICHMENT:
{
  "enriched_jobs": [...],  # same jobs with "enriched_tag" field
}

↓ [n_screen]

AFTER SCREENING:
{
  "ranked_jobs": [...top 10...],
  "shortlisted_jobs": [...top 10...]
}

↓ [n_select] ← INTERRUPTS HERE FOR HUMAN INPUT

INTERRUPT PAYLOAD:
{
  "instruction": "Select top 2 job IDs or objects to tailor.",
  "ranked_jobs": [...]
}

← USER SELECTION: [job_id1, job_id2] or [job_obj1, job_obj2]

↓ [n_tailor]

FINAL STATE:
{
  "tailored_resumes": {
    "hn_123": "4 bullet points...",
    "hn_124": "4 bullet points...",
  }
}

↓

OUTPUT:
tailored_resume.txt ← appended with tailored content
```

---

## Customization Guide

### 1. Change the LLM Model

In the **"OpenAI LLM Configuration"** cell:

```python
# Current (Groq)
llm = ChatGroq(model="llama-3.1-8b-instant", api_key=groq_api_key, temperature=0.2)

# Alternative Groq models:
llm = ChatGroq(model="mixtral-8x7b-32768", api_key=groq_api_key, temperature=0.2)  # Larger
```

### 2. Change Search Terms or Location

In the **"Graph Execution"** cell:

```python
initial_state = {
    "seed_terms": ["python", "ml"],      # ← Change search terms
    "location": "Remote",                  # ← Change location
    "resume_pdf_path": "path/to/resume.pdf"
}
```

### 3. Change Resume Output File

In the **"ReAct Agent Tools"** cell, update `WriteTailoredResumeSection`:

```python
def write_tailored_resume_section(path: str, content: str) -> str:
    path = path or "my_tailored_resume.txt"  # ← Change default output file
    # ...
```

### 4. Adjust Screening Criteria

In the **"Agent Construction"** cell, modify `SCREEN_SYSTEM` prompt:

```python
SCREEN_SYSTEM = (
    "You are a screening agent. Filter jobs STRICTLY relevant to the resume. "
    "Return ONLY jobs that match 80%+ of skills in the resume."  # ← Add strictness rules
)
```

---

## End-to-End Example

### Scenario: Search for Python ML jobs in San Francisco, review results, tailor for 2 selected jobs

```python
# Step 1: Define inputs
config = {"configurable": {"thread_id": "my-session"}}
initial_state = {
    "seed_terms": ["python", "machine learning", "data"],
    "location": "San Francisco",
    "resume_pdf_path": "Himanshi Shrivas-CV -Software_Engineer.pdf"
}

# Step 2: Run graph (search + screen)
first_result = graph_app.invoke(initial_state, config=config)

# Step 3: Inspect interrupt payload
interrupts = first_result.get("__interrupt__", [])
if interrupts:
    payload = interrupts[0].value
    ranked_jobs = payload.get("ranked_jobs", [])
    
    # Print shortlist for human review
    print(f"Found {len(ranked_jobs)} relevant jobs:")
    for i, job in enumerate(ranked_jobs[:5]):
        print(f"{i+1}. {job['title']} @ {job['company']} ({job['location']})")
        print(f"   Link: {job['url']}\n")
    
    # Step 4: Simulate human selection (pick first 2)
    selected_ids = [ranked_jobs[0]['id'], ranked_jobs[1]['id']]

# Step 5: Resume execution and tailor
from langgraph.types import Command
final_result = graph_app.invoke(Command(resume=selected_ids), config=config)

# Step 6: View tailored output
print("Tailored resume sections:")
for job_id, content in final_result.get("tailored_resumes", {}).items():
    print(f"\n{job_id}:")
    print(content[:500])  # Preview first 500 chars
```

---

## Security Best Practices

### ⚠️ CRITICAL: Do NOT expose API keys in notebooks!

**BAD (DON'T DO):**
```python
# ❌ Hardcoded in notebook
os.environ["GROQ_API_KEY"] = "gsk_abc123def456..."
```

**GOOD (DO THIS):**
```bash
# 1. Set in shell before running Jupyter
export GROQ_API_KEY="gsk_abc123def456..."

# 2. Or use a .env file (git-ignored)
# In .env:
# GROQ_API_KEY=gsk_abc123def456...

# 3. Then load in notebook:
import os
from dotenv import load_dotenv
load_dotenv()
groq_api_key = os.getenv("GROQ_API_KEY")
```

### Files to .gitignore

Add to `.gitignore`:
```
.env
*.api-key
tailored_resume.txt
__pycache__/
*.pyc
workflow_graph.png
```

---

## Debugging Tips

### View Full Agent Output

Modify any `invoke_*` function to print intermediate messages:

```python
def invoke_search(seed_terms: List[str], location: Optional[str] = None) -> List[Dict]:
    user_input = json.dumps({"terms": seed_terms, "location": location})
    result = search_agent_executor.invoke({
        "messages": [
            {"role": "user", "content": user_input}
        ]
    })
    
    # Print all agent messages
    for msg in result.get("messages", []):
        print(f"[{type(msg).__name__}] {msg}")
    
    return extract_agent_results(result)
```

### Test Individual Tools

```python
# Test YC search
jobs = yc_search(["python"], "Remote")
print(f"YC Search returned {len(jobs)} jobs")

# Test resume parsing
resume_text = parse_resume_pdf("Himanshi Shrivas-CV -Software_Engineer.pdf")
print(f"Resume text ({len(resume_text)} chars):\n{resume_text[:500]}")

# Test resume writing
write_tailored_resume_section("test_output.txt", "Test content")
```

### Check Graph State

```python
# After graph execution, inspect state
print("Final state keys:", list(final_result.keys()))
print("Jobs found:", len(final_result.get("jobs", [])))
print("Jobs ranked:", len(final_result.get("ranked_jobs", [])))
print("Jobs tailored:", list(final_result.get("tailored_resumes", {}).keys()))
```

---

## FAQ

**Q: Can I use OpenAI instead of Groq?**
A: Yes! Replace the ChatGroq import:
```python
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4", api_key=os.getenv("OPENAI_API_KEY"))
```

**Q: How long does a full execution take?**
A: ~30-60 seconds depending on network and job count:
- Search: 5-10 sec (HN Algolia API)
- Screen: 10-20 sec (LLM ranking)
- Tailor: 10-30 sec (LLM bullet generation)

**Q: Can I run this as a scheduled job?**
A: Yes, convert the notebook to a Python script and run with cron/APScheduler.

**Q: What if yc_search.py returns 0 results?**
A: The fallback in `invoke_search()` returns stub jobs. Check:
```bash
python yc_search.py --terms python --verbose
```

**Q: How do I persist the graph state across sessions?**
A: Use `MemorySaver()` (current) or replace with:
```python
from langgraph.checkpoint.sqlite import SqliteSaver
checkpointer = SqliteSaver.from_conn_string(":memory:")
```

---

## Next Steps

1. **Set GROQ_API_KEY** in your shell
2. **Run cells 1-12** to set up dependencies and configuration
3. **Execute cell 13** (Graph Execution) to start the workflow
4. **Select jobs** in the interrupt cell
5. **View results** in the final cell
6. **Check `tailored_resume.txt`** for output

Good luck! 🚀
