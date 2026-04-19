# Digital Twin ROI Scoper

A constraint-aware digital twin scoping agent. You give it your industry, use case, and resource constraints — it queries the LPI tools, reasons about what fits your situation, and tells you exactly what's realistic, what to prioritize, and what to defer.

**The key difference from a generic SMILE explainer:** this agent reasons *against* your constraints. A hospital with 2 developers and no cloud gets a completely different plan than a manufacturer with a dedicated ML team. Every recommendation cites the source tool that informed it, so the reasoning is fully traceable.

---

## How It Works

The agent calls four LPI tools in sequence:

| Step | Tool | Why |
|------|------|-----|
| 1 | `smile_overview` | Grounds the LLM on the full methodology before it reasons about phases |
| 2 | `get_insights` | Scenario-specific advice for the exact use case + industry combination |
| 3 | `get_case_studies` | Real-world analogues — outcome claims are cited from here, not hallucinated |
| 4 | `query_knowledge` | Technical depth on what the implementation actually involves |

The LLM synthesizes these four sources into a structured report. It doesn't just summarize — it filters recommendations through the constraint lens. If the constraint is "no cloud," the agent flags which SMILE activities require cloud infrastructure and explains the risk of attempting them anyway, citing which case study showed that failure mode.

### Design Decisions

**I decided to use four tools** instead of three because `smile_overview` gives the LLM a complete methodology map before it reasons about phase prioritization. Without it, the model would conflate phase descriptions when recommending what to prioritize.

**My approach to explainability** was to require a "Decision Trace" section in the output: for each major recommendation, the report lists the decision, the source that informed it, and the constraint it addresses. This means you can audit every call the agent made.

**Trade-off on LLM choice:** local Ollama vs. cloud API. I chose Ollama because it has no API key requirement and works fully offline — the trade-off is output quality on the smallest models. I mitigated this by trimming context slices and adding a fallback path that constructs a minimal valid plan from the raw tool data if JSON parsing fails.

---

## Requirements

- Node.js 18+ (for the LPI MCP server)
- Python 3.10+
- Ollama running locally (`ollama serve`)

```bash
pip install requests
ollama pull qwen2.5:1.5b
```

## Setup

```bash
# From lpi-developer-kit root
npm install
npm run build
```

## Run

**Interactive mode:**
```bash
python agent.py
```

**CLI mode:**
```bash
python agent.py --industry healthcare --usecase "patient flow optimization" --constraints "2 developers, 2 months, no cloud"
python agent.py --industry manufacturing --usecase "predictive maintenance" --constraints "small team, 3 months, limited budget"
python agent.py --industry "smart buildings" --usecase "energy optimization" --constraints "no cloud, 1 engineer, 6 weeks"
```

**Different model:**
```bash
python agent.py --model qwen2.5:3b --industry agriculture --usecase "crop monitoring" --constraints "solo developer, 1 month"
```

---

## Output Structure

```
== ROI SCOPE REPORT ==

## 1. Realistic Scope
[what's achievable + why, citing source tools]

## 2. Expected Outcomes
[case study references from get_case_studies — named, not generic]

## 3. SMILE Phases to Prioritize
[the agent chose these phases because they fit the constraint envelope]
[each phase: name, reason (citing source), timeline]

## 4. What to Skip or Defer
[specific risks of attempting too early, with evidence from case studies]

## 5. First 3 Actions
[concrete steps, each citing the tool that informed it]

## 6. Decision Trace
[decision → source → constraint addressed — full reasoning chain]

## Sources
[Tool: get_insights(...)] — provided scenario-specific advice (used in sections 1, 3)
[Tool: get_case_studies(...)] — provided real-world analogues (used in sections 2, 4)
[Tool: query_knowledge(...)] — provided technical implementation context (used in section 3)
[Tool: smile_overview()] — provided methodology baseline (used in section 3)

== PROVENANCE ==
[1] smile_overview()
[2] get_insights(...)
[3] get_case_studies(...)
[4] query_knowledge(...)
```

---

## A2A Agent Card

`agent.json` at the root describes this agent's capabilities, expected inputs, and output format following the A2A protocol. An orchestrator can discover and invoke this agent programmatically using that card.

## Files

```
lpi-roi-scoper/
├── agent.py          Main agent — MCP connection, tool calls, LLM synthesis, report output
├── agent.json        A2A Agent Card — discovery schema for orchestrators
├── requirements.txt  Python dependencies
└── README.md         This file
```
