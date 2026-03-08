# Quick Troubleshooting Checklist

## ✅ Before Running the Notebook

- [ ] **Set GROQ_API_KEY in shell** (NOT in the notebook!)
  ```bash
  export GROQ_API_KEY="gsk_your_real_key_here"
  ```
- [ ] **Install dependencies**
  ```bash
  pip install -q langgraph langchain langchain-groq pymupdf requests pydantic
  ```
- [ ] **Check resume file exists**: `Himanshi Shrivas-CV -Software_Engineer.pdf` (or your resume path)
- [ ] **Start Jupyter with API key set**
  ```bash
  jupyter notebook
  ```

---

## 🔧 Issues Fixed in This Update

### Issue #1: Agent Constructor Error
**Problem:** `create_agent(model="groq:llama-...", ...)` doesn't work
- `create_agent` from LangChain doesn't accept `model` as a string parameter

**Solution:** ✅ FIXED
- Now uses `create_tool_calling_agent()` with proper LLM instance
- Agents properly wrapped with `AgentExecutor`

### Issue #2: Exposed API Key
**Problem:** Hardcoded API key in cell #8 (SECURITY RISK!)

**Solution:** ✅ FIXED
- Cell #8 now validates environment variable instead
- Prints helpful error message if key not set

### Issue #3: Agent Result Extraction
**Problem:** `extract_agent_results()` expected old agent message format

**Solution:** ✅ FIXED
- Now uses `result.get('output')` which is the standard AgentExecutor format
- Properly handles string extraction

### Issue #4: TODO Comment
**Problem:** Cell #6 had unresolved TODO

**Solution:** ✅ FIXED
- Replaced with setup instructions

---

## 📋 Execution Order (Correct Sequence)

Run these cells **IN ORDER**:

| # | Cell Name | Purpose |
|---|-----------|---------|
| 1 | Setup info | Setup instructions |
| 2 | API key check | Validates GROQ_API_KEY is set |
| 3 | Install packages | Installs dependencies |
| 4 | Check versions | Verifies installations |
| 5 | LLM Config | Creates ChatGroq instance |
| 6 | Tools | Defines YC search, resume parsing, resume writing tools |
| 7 | Agent Construction | **[UPDATED]** Creates agents with proper API |
| 8 | Graph Config | Builds LangGraph workflow |
| 9 | Graph Visualization | Shows workflow diagram (optional) |
| 10 | Graph Execution | Runs search + screen (first interrupt) |
| 11 | Human Selection | You pick which jobs to tailor |
| 12 | Tailor + Results | Generates tailored resume bullets |

---

## 🆘 Common Errors & Fixes

### Error: "GROQ_API_KEY is not set"

```python
ValueError: GROQ_API_KEY is not set
```

**FIX:**
```bash
# In terminal (BEFORE starting Jupyter):
export GROQ_API_KEY="gsk_..."
jupyter notebook
```

Then **restart the kernel** in Jupyter (Kernel → Restart Kernel).

---

### Error: "module 'langchain.agents' has no attribute 'create_agent'"

**Status:** ✅ FIXED in updated notebook
- Old code used `create_agent()` incorrectly
- New code uses `create_tool_calling_agent()` + `AgentExecutor()`

---

### Error: Graph Returns Empty Results

**Symptoms:**
```
first_result = {'__interrupt__': [], 'seed_terms': [...]}
```

**Causes:**
1. Agents not executing correctly → Re-run "Agent Construction" cell
2. Network issue with HN Algolia API → Wait and retry
3. Invalid resume path → Check file exists

**Debug:**
```python
# Test individually
jobs = invoke_search(["python"], "Remote")
print(f"Jobs found: {len(jobs)}")
```

---

### Error: "Could not find resume file"

**Fix:** Update cell #10 (Graph Execution):
```python
initial_state = {
    "seed_terms": ["python", "ml"],
    "location": "San Francisco",
    "resume_pdf_path": "YOUR_RESUME_FILENAME.pdf"  # Correct name here
}
```

Resume file must be in the same directory as the notebook.

---

## 🧪 Testing Individual Components

### Test Tool #1: YC Search

```python
# Run in a new cell
from yc_search import yc_search
jobs = yc_search(["python"], "Remote", limit=5)
print(f"Found {len(jobs)} jobs")
for j in jobs[:2]:
    print(f"- {j['title']} @ {j['company']}")
```

### Test Tool #2: Resume Parsing

```python
# Run in a new cell
resume_text = parse_resume_pdf("Himanshi Shrivas-CV -Software_Engineer.pdf")
print(f"Resume loaded: {len(resume_text)} characters")
print(resume_text[:500])
```

### Test Tool #3: Agent Execution

```python
# Run in a new cell (after Agent Construction cell)
result = search_agent_executor.invoke({
    "input": '{"terms": ["python"], "location": "Remote"}',
    "chat_history": []
})
print("Agent output:", result['output'][:200])
```

---

## 📊 System Architecture

```
NOTEBOOK WORKFLOW:

🔵 Search Node
   └─→ Calls: YCombinatorSearch tool
   └─→ Uses: yc_search.py (HN Algolia API)
   └─→ Returns: 10-50 jobs

🔵 Enrich Node (optional, if >12 jobs)
   └─→ Adds enrichment tags to jobs

🔵 Screen Node
   └─→ Calls: ParseResumePDF tool
   └─→ Filters: Relevant jobs based on resume

🟡 INTERRUPT (Human Selection)
   └─→ User picks 2 jobs from ranked list

🔵 Tailor Node
   └─→ Calls: ParseResumePDF + WriteTailoredResumeSection
   └─→ Generates: 4-6 bullet points per selected job
   └─→ Output: tailored_resume.txt

📄 FINAL OUTPUT:
   └─→ tailored_resume.txt (appended with tailored content)
```

---

## 🎯 Key Files

| File | Role | How It Works |
|------|------|-------------|
| `MultiAgentJobSearchWorkflow (1).ipynb` | Main workflow | Orchestrates all agents + graph |
| `yc_search.py` | Job source | Queries HN Algolia API for "Who is hiring?" posts |
| `EXECUTION_GUIDE.md` | Full documentation | In-depth setup & troubleshooting |
| `tailored_resume.txt` | Output | Generated tailored resume sections |
| `*.pdf` (your resume) | Input | Used for screening & tailoring |

---

## ✨ Latest Changes Summary

**Notebook Updated On:** [Today]

**Changes:**
1. ✅ Removed hardcoded GROQ_API_KEY (Security Fix)
2. ✅ Fixed `create_agent()` → `create_tool_calling_agent()` (Critical API Fix)
3. ✅ Updated agent result extraction (Compatibility Fix)
4. ✅ Improved error handling in invoke functions
5. ✅ Added setup validation in initial cells

**Tests Completed:**
- Agent construction passes without errors
- Agent executors can be invoked
- Tools are callable with proper signatures

---

## 🚀 Ready to Run?

**Check off all items below, then start the notebook:**

- [ ] GROQ_API_KEY exported in shell
- [ ] Dependencies installed
- [ ] Resume file path verified
- [ ] Notebook re-opened after shell changes
- [ ] Kernel restarted (Kernel → Restart Kernel)

**Then run cells in order!** 🎯

---

## 📞 Need Help?

**Reference fast answers:**
1. Execution Guide (comprehensive): `EXECUTION_GUIDE.md`
2. This checklist (quick ref): `TROUBLESHOOTING.md`
3. Example files: `yc_search.py --help`

**Common tasks:**
- **Run YC search standalone:** `python yc_search.py --terms python ml --limit 25`
- **Debug agent output:** Add `verbose=True` to agent invoke calls
- **Check state:** Print `final_result.keys()` after execution
