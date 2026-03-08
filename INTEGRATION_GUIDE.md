# System Integration Guide: yc_search.py + Notebook

## How It All Works Together

```
┌─────────────────────────────────────────────────────────────────┐
│                  YOUR MULTI-AGENT JOB SEARCH SYSTEM              │
└─────────────────────────────────────────────────────────────────┘

INPUT: Search terms, Location, Resume PDF
   ↓
   
┌─────────────────────────────────────────┐
│  NOTEBOOK: MultiAgentJobSearchWorkflow   │
│  (LangGraph Workflow Orchestrator)       │
└─────────────────────────────────────────┘
   ↓
   
   ┌──→ [Search Agent] → YCombinatorSearch tool
   │                              ↓
   │                    ┌─────────────────────┐
   │                    │    yc_search.py     │
   │                    │   (Job Source)      │
   │                    └─────────────────────┘
   │                              ↓
   │                    Queries HN Algolia API
   │                    "Who is Hiring?" thread
   │                              ↓
   │                         JSON: Jobs
   │
   └──→ [Screen Agent] → ParseResumePDF tool
                               ↓
                           Resume Text
                               ↓
                         Rank + Filter Jobs
                               ↓
                    🟡 INTERRUPT (Human Choice)
                               ↓
   ┌──→ [Tailor Agent] → ParseResumePDF tool
   │                     WriteTailoredResumeSection tool
   │                              ↓
   │                    Generate Bullet Points
   │                              ↓
   │                    Append to tailored_resume.txt
   │
   └──→ Final Output: tailored_resume.txt

OUTPUT: Tailored resume bullets for selected jobs
```

---

## Component Details

### 1. yc_search.py (Job Source)

**What it does:**
- Queries Hacker News Algolia API for "Who is hiring?" thread
- Parses job comments
- Extracts company, title, location, description
- Filters by search terms and location

**Usage in notebook:**
```python
# Called by the YCombinatorSearch tool in the agents
from yc_search import yc_search as yc_search_real

jobs = yc_search_real(
    terms=["python", "ml"],      # Search for these keywords
    location="Remote",             # Filter by location
    limit=50,                      # Max results
    verbose=True                   # Debug output
)
# Returns: List[Dict] with id, source, title, company, location, description, url
```

**Can also run standalone:**
```bash
python yc_search.py --terms python ml --location Remote --verbose
```

---

### 2. Notebook Tools (Agent Capabilities)

**YCombinatorSearch Tool**
```python
Arguments:
  - terms: List[str]               # ["python", "ml", "backend"]
  - location: Optional[str]        # "San Francisco" or None

Returns: List[Dict]
  [{
    "id": "hn_12345",
    "source": "ycombinator",
    "title": "Senior Python Engineer",
    "company": "TechCorp",
    "location": "Remote",
    "description": "Build ML pipelines...",
    "url": "https://news.ycombinator.com/item?id=12345"
  }, ...]
```

**ParseResumePDF Tool**
```python
Arguments:
  - path: str                      # "path/to/resume.pdf"

Returns: str
  # Full extracted text from PDF, cleaned and normalized
```

**WriteTailoredResumeSection Tool**
```python
Arguments:
  - path: str                      # "tailored_resume.txt" or custom path
  - content: str                   # Tailored content to append

Returns: str
  # Status message or error
```

---

### 3. Agents (LLM-Powered Decision Makers)

**Search Agent**
- **Role:** Fetch jobs
- **Tools:** YCombinatorSearch
- **Input:** Search terms + location
- **Output:** JSON array of jobs
- **Model:** Groq Llama-3.1-8b

**Screen Agent**
- **Role:** Rank jobs by resume fit
- **Tools:** ParseResumePDF
- **Input:** Jobs list + resume path
- **Output:** Filtered/ranked jobs (top 10)
- **Model:** Groq Llama-3.1-8b

**Tailor Agent**
- **Role:** Generate tailored content
- **Tools:** ParseResumePDF + WriteTailoredResumeSection
- **Input:** Selected jobs + resume path
- **Output:** Dict[job_id → preview text]
- **Model:** Groq Llama-3.1-8b

---

## Data Flow Example

### Step 1: User Input
```python
initial_state = {
    "seed_terms": ["python", "machine learning"],
    "location": "San Francisco",
    "resume_pdf_path": "my-resume.pdf"
}
```

### Step 2: Search Agent Execution
```
Agent Input:
  {"terms": ["python", "machine learning"], "location": "San Francisco"}

Agent Calls:
  → YCombinatorSearch(terms=["python", "machine learning"], location="San Francisco")
    → yc_search.py queries HN for jobs matching keywords
    → Returns: [{"id": "hn_111", "title": "ML Engineer", ...}, ...]

Agent Output:
  [
    {"id": "hn_111", "title": "ML Engineer", "company": "StartupA", ...},
    {"id": "hn_222", "title": "Backend Dev", "company": "StartupB", ...},
    ...
  ]
```

### Step 3: Screen Agent Execution
```
Agent Input:
  jobs=[...50 jobs...], resume_path="my-resume.pdf"

Agent Calls:
  → ParseResumePDF(path="my-resume.pdf")
    → Reads PDF, returns: "Experienced Python developer with 5 years ML experience..."
  → Analyzes: Which jobs match the skills in the resume?

Agent Output:
  [
    {"id": "hn_111", "title": "ML Engineer", ...},  # 95% match
    {"id": "hn_333", "title": "Data Scientist", ...}, # 87% match
    {"id": "hn_222", "title": "Backend Dev", ...}, # 72% match - still included
    ...top 10...
  ]
```

### Step 4: Human Interrupt
```
Payload sent to user:
  {
    "instruction": "Select top 2 job IDs or objects to tailor.",
    "ranked_jobs": [top 10 from step 3]
  }

User selects:
  [job_id_111, job_id_333]  # The ML Engineer and Data Scientist roles
```

### Step 5: Tailor Agent Execution
```
Agent Input:
  selected_jobs=[
    {"id": "hn_111", "title": "ML Engineer", ...},
    {"id": "hn_333", "title": "Data Scientist", ...}
  ],
  resume_path="my-resume.pdf"

Agent Calls:
  → ParseResumePDF(path="my-resume.pdf")
    → Returns: "Experienced Python developer..."
  → For job hn_111, generates 4-6 tailored bullets about ML projects
    → WriteTailoredResumeSection(path="tailored_resume.txt", content="... bullets ...")
  → For job hn_333, generates 4-6 tailored bullets about data science background
    → WriteTailoredResumeSection(path="tailored_resume.txt", content="... bullets ...")

Agent Output:
  {
    "hn_111": "• Led development of real-time ML pipeline processing 100k events/day...",
    "hn_333": "• Analyzed 5+ TB datasets to identify growth opportunities..."
  }

File Output (tailored_resume.txt):
  ===== JOB: hn_111 (ML Engineer @ StartupA) =====
  • Led development of real-time ML pipeline processing 100k events/day...
  • Implemented feature engineering using scikit-learn and PyTorch...
  • Reduced model inference latency by 40% through optimization...
  • Collaborated with data scientists to define success metrics...
  ...
  
  ===== JOB: hn_333 (Data Scientist @ StartupC) =====
  • Analyzed 5+ TB datasets to identify growth opportunities...
  • Built predictive models achieving 94% accuracy...
  ...
```

---

## Integration Points

### yc_search.py → Notebook

**Where:** Tool definition in "ReAct Agent Tools" cell

```python
@tool("YCombinatorSearch")
def yc_search(terms: List[str], location: Optional[str] = None) -> List[Dict]:
    """Search Y Combinator (HN 'Who is hiring?') posts via helper script."""
    try:
        from yc_search import yc_search as yc_search_real  # Import the standalone script
        results = yc_search_real(terms or [], location, limit=50)
        # Normalize results to match expected format
        cleaned: List[Dict] = [...]
        return cleaned
    except Exception as e:
        # Fallback if import fails (e.g., network issue)
        return [fallback_stub_jobs]
```

**Key requirement:** `yc_search.py` must be in the same directory as the notebook (workspace root)

---

## Running Without the Notebook

If you want to use yc_search.py independently:

### As a CLI tool:
```bash
python yc_search.py --terms "machine learning" "python" --location "San Francisco" --limit 50 --verbose
```

Output:
```json
[
  {
    "id": "hn_12345",
    "source": "ycombinator",
    "title": "Senior ML Engineer",
    "company": "TechCorp",
    "location": "San Francisco",
    "description": "We're building ML infrastructure...",
    "url": "https://news.ycombinator.com/item?id=12345"
  },
  ...
]
```

### As a Python library:
```python
from yc_search import yc_search

results = yc_search(
    terms=["python", "ml"],
    location="Remote",
    limit=25,
    verbose=True
)

for job in results:
    print(f"{job['title']} @ {job['company']}")
    print(f"Apply: {job['url']}")
```

---

## Architecture Advantages

| Feature | Benefit |
|---------|---------|
| **Modular Design** | Each tool can be tested independently |
| **Fallback Jobs** | If yc_search.py fails, stubs keep workflow running |
| **Human-in-the-Loop** | You control which jobs to tailor (not automated) |
| **Persistent Graph State** | Can resume workflowafter interrupts |
| **Multi-Agent** | Each agent specialized for one task |
| **LLM-Powered Ranking** | Resume matching is intelligent, not keyword-based |

---

## Troubleshooting Integration Issues

### Issue: yc_search.py returns 0 results
```
Debug:
  python yc_search.py --terms python --verbose
  
Expected:
  [yc_search] Using thread id: 12345
  Found 5+ Python-related jobs
  
Fix:
  - Check HN Algolia API is working (try in browser)
  - Verify search terms match job postings
  - No recent "Who is hiring?" thread may have caused fault
```

### Issue: Tool not found error in agent
```
Error: "Tool 'YCombinatorSearch' not found"

Cause:
  - Tools defined in "ReAct Agent Tools" cell not imported
  - Agents built before tools defined
  
Fix:
  - Run "ReAct Agent Tools" cell first
  - Then run "Agent Construction" cell
  - Then run "Graph Configuration" cell
```

### Issue: Resume not parsed correctly
```
Debug:
  resume_text = parse_resume_pdf("my-resume.pdf")
  print(f"Loaded {len(resume_text)} characters")
  print(resume_text[:500])
  
Should show:
  Loaded 3500+ characters
  Resume text preview...
  
Fix:
  - Verify PDF is not encrypted
  - Try opening PDF manually
  - Check file permissions
```

---

## Performance Notes

| Component | Typical Time |
|-----------|--------------|
| yc_search.py (HN API call) | 3-10 seconds |
| Screen Agent (LLM ranking) | 5-15 seconds |
| Tailor Agent (LLM generation) | 10-30 seconds per job |
| **Total workflow** | 30-60 seconds |

---

## Next Steps

1. **Start the notebook:** `jupyter notebook`
2. **Follow Quick Start guide** in notebook markdown
3. **Run cells in order** (search → screen → tailor)
4. **Review tailored_resume.txt** for output
5. **Customize** search terms, resume path, or LLM model as needed

Ready to run? Let's go! 🚀
