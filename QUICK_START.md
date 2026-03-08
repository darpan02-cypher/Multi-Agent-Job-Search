# Quick Start: 5 Minute Setup

## Prerequisites

```bash
# 1. Get API key from https://console.groq.com (free tier available)
# 2. Set it in your shell BEFORE starting Jupyter:

export GROQ_API_KEY="gsk_YOUR_ACTUAL_KEY_HERE"

# 3. Install packages
pip install -q langgraph langchain langchain-groq pymupdf requests pydantic

# 4. Verify resume file exists in workspace
ls -la "Himanshi Shrivas-CV -Software_Engineer.pdf"

# 5. Start Jupyter
jupyter notebook
```

---

## Execution Checklist

**Before opening the notebook:**
- [ ] API key exported: `echo $GROQ_API_KEY` (should print your key)
- [ ] Packages installed: `pip list | grep langchain`
- [ ] Resume file present: `ls *.pdf`

**In Jupyter (run cells 1-12 in order):**
1. ✅ **Setup info** - Cell #1
2. ✅ **API key check** - Cell #2 (should print "✓ GROQ_API_KEY is set")
3. ✅ **Install packages** - Cell #3
4. ✅ **Check versions** - Cell #4
5. ✅ **LLM Config** - Cell #5 (loads Groq)
6. ✅ **Agent Tools** - Cell #6 (defines YC search, etc)
7. ✅ **Agent Construction** - Cell #7 (builds agents)
8. ✅ **Graph Config** - Cell #8 (compiles workflow)
9. ⏭️ **Visualization** - Cell #9 (optional, shows diagram)
10. ✅ **Graph Execution** - Cell #10 (RUN THIS!)
11. 🟡 **Human Selection** - Cell #11 (PICK 2 JOBS)
12. ✅ **Final Results** - Cell #12 (see output)

---

## Expected Output Timeline

```
┌─ Run Cell #10 (Graph Execution)
│
├─ Search Agent starts
│  └─ Queries HN for "python full stack ml" jobs
│  └─ Returns 10-50 job listings
│  └─ Screen Agent ranks by resume fit
│
└─ 🟡 INTERRUPTED FOR HUMAN INPUT
   
   Payload appears:
   {
     "instruction": "Select top 2 job IDs or objects to tailor.",
     "ranked_jobs": [
       {"id": "hn_111", "title": "Python Engineer", ...},
       {"id": "hn_222", "title": "ML Engineer", ...},
       ...
     ]
   }

┌─ Run Cell #11 (Human Selection)
│
├─ You pick 2 jobs from the list
│  └─ selected_ids = ["hn_111", "hn_222"]
│
└─ Resume execution
   └─ Tailor Agent generates 4-6 bullet points per job
   └─ Writes to tailored_resume.txt

Run Cell #12 (Final Results)
   └─ Prints preview of tailored content
```

---

## What Happens Behind the Scenes

```
CELL #10: graph_app.invoke(initial_state)
│
├─ Search Node
│  └─ Agent calls: YCombinatorSearch tool
│  └─ Tool calls: yc_search.py (HN Algolia API)
│  └─ Returns: Jobs JSON
│
├─ Screen Node
│  └─ Agent calls: ParseResumePDF tool
│  └─ Parses your resume PDF
│  └─ Ranks jobs by relevance
│  └─ Returns: Top 10 jobs
│
└─ Interrupt: Wait for human selection
   
CELL #11: graph_app.invoke(Command(resume=selected_ids))
│
└─ Tailor Node
   ├─ Agent calls: ParseResumePDF (load resume again)
   ├─ Agent calls: WriteTailoredResumeSection (for each selected job)
   │  └─ Generates: "4-6 bullet points aligning resume to job"
   │  └─ Appends to: tailored_resume.txt
   └─ Returns: Preview of generated content
```

---

## File Outputs

After running the workflow, check these files:

### tailored_resume.txt (Important!)
```
===== JOB: hn_111 (Python Engineer @ TechCorp) =====
• Led development of full-stack Python applications serving 100k+ users
• Implemented async APIs using FastAPI achieving 99.9% uptime
• Mentored 3 junior developers on best practices
• Reduced infrastructure costs by 30% through optimization

===== JOB: hn_222 (ML Engineer @ StartupAI) =====
• Designed and deployed ML pipelines processing 1TB+ of daily data
• Built feature engineering pipeline improving model accuracy by 15%
• ...
```

This file is **appended to** on each run (doesn't overwrite).

---

## Common Mistakes & Fixes

### ❌ Mistake #1: GROQ_API_KEY not set
```python
ValueError: GROQ_API_KEY is not set
```
**Fix:** Export before Jupyter
```bash
export GROQ_API_KEY="gsk_..."
jupyter notebook  # Start fresh Jupyter in same terminal
```

### ❌ Mistake #2: Running cells out of order
**Fix:** Always run from top to bottom. If something breaks:
1. Kernel → Restart Kernel
2. Run from Cell #2 onwards

### ❌ Mistake #3: Skipping the agent construction
**Symptom:** Graph returns no results
**Fix:** Run Cell #7 (Agent Construction) explicitly

### ❌ Mistake #4: Wrong resume file path
**Error:** FileNotFoundError
**Fix:** Update Cell #10:
```python
initial_state = {
    "resume_pdf_path": "CORRECT_FILENAME.pdf"  # Match actual file name
}
```

---

## Testing Each Component

### Test 1: Is YC search working?
```python
# Run in a new cell
from yc_search import yc_search
jobs = yc_search(["python"], "Remote", limit=3)
print(f"✓ Found {len(jobs)} jobs")
```

### Test 2: Is resume parsing working?
```python
# Run in a new cell
resume = parse_resume_pdf("Himanshi Shrivas-CV -Software_Engineer.pdf")
print(f"✓ Resume loaded: {len(resume)} chars")
```

### Test 3: Are agents working?
```python
# Run in a new cell (after Agent Construction)
result = search_agent_executor.invoke({
    "input": '{"terms": ["python"], "location": "Remote"}',
    "chat_history": []
})
print("✓ Agent executed")
```

---

## Quick Reference: Cell Numbers

| Cell | Name | Status |
|------|------|--------|
| 1 | Setup info | ℹ️ Read |
| 2 | API check | ✅ Must run |
| 3 | Install | ✅ Must run |
| 4 | Versions | ✅ Must run |
| 5 | LLM config | ✅ Must run |
| 6 | Tools | ✅ Must run |
| 7 | Agents | ✅ Must run |
| 8 | Graph | ✅ Must run |
| 9 | Viz | ⏭️ Optional |
| 10 | Execute | ⚙️ Main run |
| 11 | Select | 🟡 Interrupt |
| 12 | Results | 📊 Output |

---

## Success Indicators

✅ **Cell #5 Check:**
```
✓ GROQ_API_KEY is set (length: 50)
```

✅ **Cell #10 Output (First interrupt):**
```
[n_search] jobs count: 15
[n_screen] ranked_jobs count: 10
```

✅ **Cell #11 Inspection:**
```
Instruction: Select top 2 job IDs or objects to tailor.
Payload keys: ['instruction', 'ranked_jobs']
Shortlisted count: 10
Preview IDs: ['hn_111', 'hn_222', 'hn_333', ...]
```

✅ **Cell #12 Final Output:**
```
Final result keys: [..., 'tailored_resumes']
Tailored resumes keys: ['hn_111', 'hn_222']

Sample tailored content preview:
hn_111:
• Led development of ML pipeline serving 100k+ users...
• Implemented async APIs using FastAPI...
```

---

## If Something Goes Wrong

### Option 1: Restart & Rerun
```
Kernel → Restart Kernel
Then run cells 2-12 again
```

### Option 2: Check Documentation
```
EXECUTION_GUIDE.md    - Comprehensive guide
TROUBLESHOOTING.md    - Common issues & fixes
INTEGRATION_GUIDE.md  - How yc_search.py fits in
```

### Option 3: Debug Individual Components
```python
# In a new cell, test each tool:
jobs = yc_search(["python"], verbose=True)  # CLI test
resume = parse_resume_pdf("path/to/resume.pdf")  # Resume loading
write_tailored_resume_section("test.txt", "test")  # File writing
```

---

## Next Steps After Success

1. **Run with your own resume:** Update `resume_pdf_path` in Cell #10
2. **Try different search terms:** Change `seed_terms` in Cell #10
3. **Customize model:** Change Groq model in Cell #5
4. **Run standalone YC search:** `python yc_search.py --terms python --verbose`

---

## Support Quick Links

- **Groq API:** https://console.groq.com
- **HN Algolia API:** https://hn.algolia.com/ (no auth needed)
- **LangChain Docs:** https://python.langchain.com
- **LangGraph Docs:** https://langchain-ai.github.io/langgraph/

---

**Ready? Export API key, start Jupyter, and run Cell #2!** 🚀
