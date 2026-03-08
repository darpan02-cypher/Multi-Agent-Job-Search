# Quick Reference Card (Printable)

## 1-Minute Setup

```bash
# A. Get API key
# Visit: https://console.groq.com/keys (FREE)

# B. Export in terminal
export GROQ_API_KEY="gsk_YOUR_KEY_HERE"

# C. Install packages
pip install -q langgraph langchain langchain-groq pymupdf requests

# D. Start Jupyter
jupyter notebook
```

---

## 5-Minute Execution

| Step | Action | Expected Output |
|------|--------|-----------------|
| 1 | Open notebook | See 12 cells |
| 2 | Run Cell #2 | `✓ GROQ_API_KEY is set` |
| 3 | Run Cells #3-9 | Tools & agents initialize |
| 4 | **Run Cell #10** | **🟡 Shows ranked_jobs list** |
| 5 | **Pick 2 jobs** | `selected_ids = ["hn_111", "hn_222"]` |
| 6 | **Run Cell #11** | Resume execution, tailor jobs |
| 7 | Run Cell #12 | Print tailored content |
| 8 | **Check output** | `tailored_resume.txt` created |

---

## Common Errors & Instant Fixes

| Error | Fix |
|-------|-----|
| `ValueError: GROQ_API_KEY is not set` | `export GROQ_API_KEY="gsk_..."` then restart Jupyter |
| `FileNotFoundError: resume.pdf` | Update path in Cell #10: `"resume_pdf_path": "correct_name.pdf"` |
| Cells fail to import | Restart kernel: Kernel → Restart Kernel |
| Empty results from search | Retry Cell #10; HN Algolia may rate-limit |
| No interrupt received | Re-run Cell #7 (Agent Construction) |

---

## File Locations

| File | Purpose |
|------|---------|
| `MultiAgentJobSearchWorkflow (1).ipynb` | Main notebook |
| `yc_search.py` | Job search (HN API) |
| `tailored_resume.txt` | **OUTPUT FILE** ← Check here! |
| `QUICK_START.md` | Full setup guide |
| `TROUBLESHOOTING.md` | All error fixes |

---

## Cell Purpose Cheat Sheet

```
Cells 1-5    → Setup (install, config, validate)
Cells 6-9    → Agents & Graph (build workflow)
Cell 10      → MAIN: Search → Screen (pauses for selection)
Cell 11      → You select jobs + Resume execution
Cell 12      → View results + Check tailored_resume.txt
```

---

## Workflow Summary

```
Your Resume + Search Terms
          ↓
Search Agent (HN + yc_search.py)
          ↓
50 Jobs
          ↓
Screen Agent (vs resume)
          ↓
Top 10 Jobs (ranked)
          ↓
🟡 YOU SELECT 2 JOBS
          ↓
Tailor Agent (generate bullets)
          ↓
tailored_resume.txt ✅
```

---

## Success Checklist

- [ ] Cell #2: `✓ GROQ_API_KEY is set`
- [ ] Cell #10: `ranked_jobs count: 10`
- [ ] Cell #10: Shows 10 job options
- [ ] Cell #11: Resume execution succeeds
- [ ] Cell #12: Prints tailored content
- [ ] File exists: `cat tailored_resume.txt`

---

## Debug Commands (Run in new cell)

```python
# Test 1: Jobs found?
jobs = yc_search(["python"], "Remote")
print(f"Found {len(jobs)} jobs")

# Test 2: Resume loaded?
resume = parse_resume_pdf("YOUR_RESUME.pdf")
print(f"Resume: {len(resume)} chars")

# Test 3: Agent working?
result = search_agent_executor.invoke({
    "input": '{"terms": ["python"], "location": "Remote"}',
    "chat_history": []
})
print(f"Agent output: {result['output'][:100]}")
```

---

## Documentation Quick Links

- **Getting Started:** [QUICK_START.md](QUICK_START.md)
- **Full Guide:** [EXECUTION_GUIDE.md](EXECUTION_GUIDE.md)
- **Errors & Fixes:** [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- **Visual Diagram:** [CELL_MAP_DIAGRAM.md](CELL_MAP_DIAGRAM.md)
- **All Docs:** [README_DOCS.md](README_DOCS.md)

---

## Running yc_search.py Standalone

```bash
# CLI
python yc_search.py --terms python ml --location Remote --limit 25

# Python
from yc_search import yc_search
jobs = yc_search(["python", "ml"], "Remote")
```

---

## State During Execution

**Cell #10 Return:**
```python
first_result = {
    "seed_terms": [...],
    "location": "San Francisco",
    "jobs": [...50 jobs...],
    "ranked_jobs": [...10 jobs...],     # ← You see this
    "__interrupt__": [payload...]       # ← System pauses here
}
```

**Cell #11 Input:**
```python
selected_ids = ["hn_111", "hn_222"]     # ← You pick these
Command(resume=selected_ids)
```

**Cell #12 Return:**
```python
final_result = {
    "tailored_resumes": {
        "hn_111": "bulletspreview...",  # ← Output here
        "hn_222": "bulletspreview..."
    }
}
```

---

## Customization Quick Tips

```python
# CELL #10 - Change search terms
initial_state = {
    "seed_terms": ["python", "ml"],      # ← Edit here
    "location": "San Francisco",          # ← Edit here
    "resume_pdf_path": "my-resume.pdf"   # ← Edit here
}

# CELL #5 - Change LLM model
llm = ChatGroq(
    model="mixtral-8x7b-32768",  # ← Bigger model available
    api_key=groq_api_key,
    temperature=0.2
)
```

---

## Performance Expectations

- **Search:** 5-10 seconds (HN API)
- **Screen:** 10-20 seconds (LLM ranking)
- **Tailor:** 10-30 seconds (LLM generation)
- **Total:** 25-60 seconds

---

## API Key Sources

- **Groq:** https://console.groq.com/keys (FREE, instant)
- **OpenAI:** https://platform.openai.com/keys (Requires payment)

---

## Important: Security

❌ **DON'T:**
```python
# Don't hardcode in notebook
os.environ["GROQ_API_KEY"] = "gsk_..."
```

✅ **DO:**
```bash
# Set in terminal BEFORE starting Jupyter
export GROQ_API_KEY="gsk_..."
```

---

## When Something Fails

1. **Check the error message** → Search [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
2. **Restart kernel** → Kernel → Restart Kernel
3. **Run from top** → Start from Cell #2 again
4. **Test components** → See "Debug Commands" above
5. **Check files exist** → Resume PDF, yc_search.py

---

## Final Output Location

**MOST IMPORTANT:** Check this file for results!

```bash
cat tailored_resume.txt
```

Should show:
```
===== JOB: hn_111 (Title @ Company) =====
• Bullet point 1
• Bullet point 2
• Bullet point 3
...

===== JOB: hn_222 (Different Job) =====
• Different bullet 1
...
```

---

## 30-Second Summary

1. `export GROQ_API_KEY="gsk_..."` ← Get from console.groq.com
2. `pip install langgraph langchain langchain-groq pymupdf requests`
3. Open notebook, run cells 1-12
4. Select 2 jobs when prompted
5. Check `tailored_resume.txt`

**Done!** ✅

---

**Bookmark this → [README_DOCS.md](README_DOCS.md)**

Print this card for quick reference while running! 🚀
