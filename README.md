# Token Waste Analysis in Enterprise LLM Systems

**Part of the AI Cost Intelligence Research Programme**  
*By Siva Seetharaman — AI Product Strategist & Builder*

---

## What this notebook does

This notebook measures how many tokens in enterprise LLM system prompts are wasted — meaning they cost money on every API call while contributing zero value to the output.

Across five common enterprise prompt types, the research found that **34.1% of input tokens are waste** — redundant instructions and generic boilerplate that the model already knows or has been told multiple times. At 10,000 API calls per day, that waste costs **$9,247 per year**.

The notebook is fully reproducible. Run all 15 cells in order and you will replicate the original findings.

---

## Before you start

**You need:**
- A Google account (to run in Google Colab)
- An Anthropic API key
- Approximately $0.25 in API credits to run the full experiment

**Estimated run time:** 20 to 30 minutes end to end

**Total API calls:** 26 — 5 for the waste analyser, 21 for the baseline run

---

## One-time setup — add your API key

This notebook reads your Anthropic API key from Colab Secrets. You only need to do this once.

1. Click the **key icon** in the left sidebar of Colab
2. Click **Add new secret**
3. Name: `ANTHROPIC_API_KEY`
4. Value: your Anthropic API key (starts with `sk-ant-`)
5. Toggle **Notebook access** to on

---

## Cell-by-cell guide

Run every cell in the order it appears. Do not skip cells. Each cell depends on variables and functions defined in the cells above it.

---

### Cell 1 — Setup: install libraries, imports, API key, and tokeniser

**What it does:** This single cell handles all environment setup in one go:

- Installs four Python libraries: `anthropic` (Claude API SDK), `tiktoken` (token counting), `pandas` (data analysis), `matplotlib` and `seaborn` (charting)
- Imports all libraries needed throughout the notebook
- Reads your Anthropic API key from Colab Secrets and initialises the Claude client
- Loads the tiktoken tokeniser with `cl100k_base` encoding and defines the `count_tokens` function

**Important note on token counting:** tiktoken uses OpenAI's encoding, not Anthropic's. Sentence-level token counts in the waste audit (Cell 8) come from tiktoken. Total input and output token counts in the baseline run (Cell 9) come directly from the Anthropic API response — the authoritative billing figure. The discrepancy between the two is approximately 1 to 3%.

**Expected output:**
```
All libraries loaded successfully
Anthropic SDK version: X.X.X
Client initialised
Test token count: 12
```

If you see `SecretNotFoundError` — go back and add your API key to Colab Secrets. If any library fails to install — check your internet connection.

---

### Cell 2 — Logging wrapper: run_and_log

**What it does:** Defines `run_and_log()` — the core function that makes every API call in this experiment. Every call in the notebook passes through this function so all results are captured consistently.

For each API call, `run_and_log` records:
- Prompt name and version
- Model used
- Actual input tokens billed (from `response.usage.input_tokens`)
- Actual output tokens billed (from `response.usage.output_tokens`)
- Total cost in USD calculated from actual token counts
- The full output text from Claude

The function accepts a `max_tokens` parameter so different prompt types can have different output limits. Code review and report generation prompts need `max_tokens=2048` to avoid truncated responses. All others use `max_tokens=1024`.

**Expected output:**
```
Logging wrapper ready
```

---

### Cell 3 — Export and summary functions

**What it does:** Defines three utility functions used throughout the experiment:

- `save_results(filename)` — saves the full `results_log` as a JSON file both locally in Colab and to your Google Drive folder
- `load_results(filename)` — loads a saved JSON file, tries local Colab storage first then falls back to Drive
- `quick_summary()` — prints a grouped summary table of all logged results organised by prompt name and version

**Expected output:**
```
Export functions ready
```

---

### Cell 4 — Pipeline verification (test call)

**What it does:** Makes one real API call using a simple customer support prompt to confirm the entire pipeline is working correctly before the main experiment runs.

**This cell costs approximately $0.002.**

**Expected output:**
```
[test_prompt] [original] [Input #1] Input: 54 | Output: 97 | Total: 151 | Cost: $0.001617
Output: [Claude's classification response]
Input tokens: 54
Cost: 0.001617
```

If this cell throws any error — stop here and do not proceed. The pipeline must work cleanly before running the main experiment.

---

### Cell 5 — Manual block analysis (historical context, optional)

**What it does:** Defines the `count_and_show` function and demonstrates early manual token counting. The customer support prompt is split into six logical blocks and each block is counted individually using tiktoken.

**You do not need to run this cell to reproduce the main findings.** It is included for transparency — this was the early methodology that was tried and then replaced by the hybrid LLM-as-analyser approach. The research paper documents why this approach was insufficient.

If you do run it, it prints token counts per block and a basic waste calculation using the original manual tagging method.

---

### Cell 6 — Original prompts

**What it does:** Defines the five enterprise system prompts as Python variables — `P1_ORIGINAL` through `P5_ORIGINAL`. These are the prompts analysed for waste and sent through the baseline API run.

The five prompt types:

| Variable | Prompt type | Task category |
|---|---|---|
| P1_ORIGINAL | Customer support ticket triage | Classification |
| P2_ORIGINAL | Document summarisation | Generative |
| P3_ORIGINAL | Structured data extraction | Classification |
| P4_ORIGINAL | Code review and analysis | Generative |
| P5_ORIGINAL | Weekly performance report generation | Generative |

**Note:** All five prompts were AI-generated to represent realistic enterprise use cases. Original client prompts were unavailable. This is acknowledged as a limitation — real enterprise prompts may show different waste profiles.

**Expected output:**
```
Original prompts loaded:
  P1 — customer_support_triage
  P2 — document_summarisation
  P3 — data_extraction
  P4 — code_review
  P5 — report_generation
```

---

### Cell 7 — Test inputs

**What it does:** Defines realistic user messages for each prompt type — the inputs that will be sent to Claude during the baseline run. Also defines `ALL_PROMPTS`, the master list used by the baseline run loop.

Test input counts per prompt:
- P1 customer support triage: 10 inputs
- P2 document summarisation: 3 inputs
- P3 data extraction: 3 inputs
- P4 code review: 3 inputs
- P5 report generation: 2 inputs
- **Total: 21 baseline API calls**

**Expected output:**
```
Test inputs loaded
Total baseline calls to run: 21
```

---

### Cell 8 — Waste analyser: core experiment

**What it does:** This is the most important cell in the notebook. It defines the waste taxonomy prompt and the `analyse_prompt_waste` function, then immediately runs the waste analyser on all five original prompts.

**The hybrid analysis architecture:**

This experiment uses a three-layer approach that separates judgment, classification, and measurement:

1. **Human-defined taxonomy** — four waste types with precise detection tests, defined before any analysis runs
2. **LLM classification** — Claude Sonnet 4.6 reads each system prompt sentence by sentence, applies the taxonomy, and returns structured JSON with a classification and one-sentence reasoning per sentence
3. **tiktoken measurement** — counts actual tokens in each classified sentence and aggregates waste totals by type

**The four waste types:**

| Code | Name | Description |
|---|---|---|
| W1 | Redundant instruction | Same instruction stated multiple times in different words |
| W2 | Boilerplate padding | Generic behaviour descriptions the model already exhibits by default |
| W3 | Verbose few-shot | Examples longer than needed to demonstrate the output pattern |
| W4 | Dead context | Background injected on every call but relevant to fewer than 20% of queries |

**This cell makes 5 API calls. Cost: approximately $0.05.**

**Expected output:** For each prompt, a sentence-by-sentence classification printed as either `✓ ESSENTIAL` or `⚠ W1/W2/W3/W4`. The final aggregate should show:

```
OVERALL WASTE          34.1%  (627/1838 tokens)
HYPOTHESIS TARGET    : 30-40%
ACTUAL FINDING       : 34.1%
RESULT               : HYPOTHESIS CONFIRMED
```

Results are saved to `waste_analysis_results.json`.

**Caveat:** The LLM analyser uses Claude Sonnet 4.6 to classify prompts — the same model family being studied. Model bias may exist. Full methodology limitations are documented in the research paper.

---

### Cell 9 — Baseline API run

**What it does:** Runs all 21 API calls — each original prompt against all its test inputs — and saves the complete results. This generates the real token count data used by the H4 cost projection.

**Different max_tokens per prompt type to prevent truncation:**

| Prompt | max_tokens | Reason |
|---|---|---|
| Customer support triage | 512 | Short classification output |
| Document summarisation | 1024 | Medium structured output |
| Data extraction | 512 | JSON output only |
| Code review | 2048 | Verbose analytical output — 1024 caused truncation |
| Report generation | 2048 | Seven required sections — 1024 caused truncation |

**This cell makes 21 API calls. Cost: approximately $0.15.**

After all calls complete you should see:
```
All outputs within limits. No truncation detected.
BASELINE RUN COMPLETE — 21 calls logged
```

Results are saved to `baseline_results.json` locally and to Google Drive.

---

### Cell 10 — Mount Google Drive

**What it does:** Mounts your Google Drive and creates a `TokenWasteResearch` folder. All result files saved during the experiment are also stored here so they survive between Colab sessions.

You will be prompted to authorise Drive access the first time you run this.

**Expected output:**
```
Mounted at /content/drive
Save directory ready: /content/drive/MyDrive/TokenWasteResearch
```

**Note:** This cell defines `SAVE_DIR` which is used by the data cleaning cell (Cell 13). Run this before Cell 13.

---

### Cell 11 — H4: Annual cost of waste projection

**What it does:** Loads the baseline results and calculates the annual dollar cost of the token waste identified in Cell 8. Projects cost at four enterprise call volumes.

**How it calculates:** Takes actual average input tokens per prompt from the baseline API calls, applies the waste percentage from the LLM analyser audit, and calculates the dollar cost of those waste tokens at $3.00 per million input tokens.

**Key output:**

| Daily call volume | Annual waste cost |
|---|---|
| 1,000 calls/day | $925 |
| 10,000 calls/day | $9,247 |
| 50,000 calls/day | $46,233 |
| 100,000 calls/day | $92,467 |

**Important nuance:** The 34.1% figure is waste as a proportion of input tokens only. Overall waste as a percentage of total API spend — including output tokens — is 5.6%. For verbose generative prompts like code review and report generation, output tokens dominate the total bill and dilute the input waste percentage when measured against total cost. Both figures are accurate and both are reported.

Results saved to `h4_cost_projection.json`.

---

### Cell 12 — Data inspection utility

**What it does:** Reads `baseline_results.json` and reports how many records exist per prompt type. Use this to verify your baseline data before running H1 and H3.

Correct record counts are:
- customer_support_triage: 10 records
- document_summarisation: 3 records
- data_extraction: 3 records
- code_review: 3 records
- report_generation: 2 records
- **Total: 21 records**

If you see a `test_prompt` record listed — run Cell 13 to clean it out.

---

### Cell 13 — Data cleaning utility

**What it does:** Removes any test call records from `baseline_results.json`. If the pipeline test call from Cell 4 ended up in the results file, this cell removes it cleanly.

Safe to run even if no test records are present — it will simply report zero records removed.

**Run Cell 10 before this cell** — Cell 13 saves the cleaned file to Google Drive and needs `SAVE_DIR` to be defined.

---

### Cell 14 — H1 validation: waste by prompt category

**What it does:** Tests whether generative prompts waste more tokens than classification prompts. Compares the two classification prompts against the three generative prompts using the waste percentages from the LLM analyser audit.

**Post-hoc hypothesis disclosure:** H1 was formed after observing the waste audit results, not before. This is disclosed in both the cell comments and the research paper. The finding is valid but carries a different epistemic status than H0 which was defined before the experiment.

**Expected output:**
```
Classification    30.8% average waste
Generative        36.0% average waste
Difference: +5.2 percentage points
HYPOTHESIS CONFIRMED
```

**Qualification:** Code review is a generative prompt but has the lowest waste of all five prompts at 25.3% — lower than both classification prompts. This outlier is explained by H3: code review has the highest technical specificity of all five prompts, and specificity is a stronger predictor of waste than category alone.

---

### Cell 15 — H3 validation: waste vs technical specificity

**What it does:** Tests whether more technically specific prompts accumulate less waste. Scores each prompt on a 1 to 3 specificity scale and verifies that waste percentage decreases as specificity increases.

Specificity scores used:
- Code review: High (3) — OWASP references, SOLID principles, six precisely defined output sections
- Customer support triage: Medium (2) — named categories, priority levels, company context
- Data extraction: Medium (2) — specific JSON schema, field definitions, null handling rules
- Report generation: Medium (2) — seven specific sections, British English, formal tone requirement
- Document summarisation: Low (1) — generic qualitative instructions, no domain knowledge required

**Post-hoc hypothesis disclosure:** H3 was formed after observing the waste audit data. Specificity scores were assigned with knowledge of the waste percentages — creating potential circularity. Independent scoring by domain experts who did not see the waste data would strengthen this finding. Both limitations are disclosed in the cell comments and the research paper.

**Expected output:**
```
High specificity    25.3% waste
Medium specificity  33.7% average waste
Low specificity     43.5% waste
Pattern: as specificity increases, waste decreases
HYPOTHESIS CONFIRMED
```

Each step down in specificity adds approximately 9 percentage points of waste. The practical implication: write prompts with structural precision rather than qualitative descriptions. Instead of "be thorough" define the exact sections you need. Instead of "be professional" specify the register, language, and constraints.

---

## Files produced by this notebook

| File | Contents | Produced by |
|---|---|---|
| `waste_analysis_results.json` | Sentence-level waste classification for all 5 prompts | Cell 8 |
| `baseline_results.json` | All 21 API call results with token counts and costs | Cell 9 |
| `h4_cost_projection.json` | Annual cost projections at 4 call volume tiers | Cell 11 |

All files are also saved to `Google Drive/MyDrive/TokenWasteResearch/` for persistence across Colab sessions.

---

## Confirmed findings

| Hypothesis | Result | Status |
|---|---|---|
| H0 — 30–40% average waste exists in enterprise prompts | 34.1% actual | ✅ Confirmed |
| H1 — Generative prompts waste more than classification prompts | 36.0% vs 30.8% | ✅ Confirmed with qualification |
| H2 — W1 and W2 are universal waste types across all prompt categories | 100% of waste is W1 or W2. W3 and W4 zero. | ✅ Confirmed |
| H3 — Waste inversely correlates with technical specificity | High 25.3% → Medium 33.7% → Low 43.5% | ✅ Confirmed directionally |
| H4 — Waste represents a calculable annual cost at enterprise scale | $9,247/year at 10k calls/day | ✅ Confirmed |

---

## Adapting this notebook to your own prompts

To audit your own system prompts for token waste:

1. Replace `P1_ORIGINAL` through `P5_ORIGINAL` in Cell 6 with your actual system prompts
2. Replace the test inputs in Cell 7 with realistic queries from your real workload
3. Run Cell 8 — the waste analyser will classify every sentence in your prompts
4. Run Cell 9 — the baseline run will measure actual token costs from the API
5. Run Cell 11 — H4 will calculate your annual waste cost from your real data

The waste taxonomy and hybrid analyser architecture work on any enterprise prompt type regardless of domain or task type.

---

## Research paper and further reading

Full methodology, hypothesis documentation, the complete story of how the methodology evolved through three failures, and all limitations are documented in the research paper.

📄 Full research paper — [your Google Drive PDF link]  
✍️ Substack essay — [your Substack URL]  
📊 LinkedIn series — [your LinkedIn profile]

---

## About

**Siva Seetharaman** is an AI Product Strategist and Builder with 12 years of experience shipping AI platforms across the US, UK, and India. This notebook is part of the AI Cost Intelligence research programme — original experiments into enterprise LLM economics, token waste, and cost governance.

*AI Cost Intelligence Research Programme — May 2026*
