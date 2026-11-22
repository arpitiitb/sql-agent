# Text-to-SQL Agent: from 65% to 95% execution accuracy

An LLM system that answers business questions about insurance data by writing SQL, built step by step and **measured at every step** on a hand-checked test set. Six versions are compared, from a plain prompt to a tool-using agent to a trusted "semantic view", with accuracy, token cost and latency for each.

All data is **synthetic** (generated in the notebook with a fixed seed). No real or employer data is used.

## Results

Development set: 20 business questions with hand-written reference SQL. Model: `gpt-4o-mini`, temperature 0.

| Version | What it adds | Correct (of 20) | Tokens / question | Seconds / question |
| --- | --- | --- | --- | --- |
| v0 Baseline | One call, table definitions only | 13 (65%) | 226 | 1.3 |
| v1 Context | Column descriptions, business glossary, 3 worked examples | 16 (80%) | 558 | 1.1 |
| v2 Agent | Tools to run SQL, read errors and results, fix, then submit | 18 (90%) | 1,461 | 2.7 |
| v3 Context + rules | SQL rules learned from error analysis | 17 (85%) | 651 | 1.0 |
| v4 Agent + rules | v2 with the v3 rules | 18 (90%) | 1,594 | 2.5 |
| **v5 Context + semantic view** | A trusted `policy_summary` view for policy-level metrics | **19 (95%)** | **727** | **1.0** |

Scores are after manual review of failures: answers that were correct but in a different shape (for example labels instead of 0/1 codes) are counted as correct and recorded in `text2sql_results.csv`.

**Held-out check:** 5 new questions that none of the fixes were designed from.

| Version | Correct (of 5) |
| --- | --- |
| v1 Context | 2 |
| v3 Context + rules | 3 |
| v4 Agent + rules | 3 |
| **v5 Context + semantic view** | **4** |

The gap between 95% (development) and 80% (held-out) shows some tuning to the development set. Both numbers are reported on purpose.

## What I learned

1. **Missing business knowledge was the biggest problem, and the cheapest to fix.** The baseline did not know that "open" includes reopened claims, that "litigated" means an attorney is involved, or that states are stored as `FL`, not `Florida`. A glossary and column descriptions (about 300 extra tokens) fixed all of these: 65% to 80%.
2. **The hardest errors were SQL that runs and looks plausible but is wrong.** Loss ratio and claim frequency need every policy, including those with no claims, and must not repeat a policy's premium once per claim. Every prompt-based version kept making this join mistake, even with an explicit rule, and repeated it on unseen questions.
3. **An agent loop did not fix those errors.** Self-checking catches SQL errors and empty results, not logic that looks right. The agents used about 2.5x the tokens and time of a single call for little or no accuracy gain.
4. **A trusted view fixed them in one cheap call.** Moving the error-prone logic into a `policy_summary` view (one row per policy, already aggregated) took accuracy to 95% at single-call cost, and it was the only version to get both policy-level metrics right on the held-out questions. This is the "semantic layer" approach: define trusted metrics once, and let the model choose them instead of rebuilding them.
5. **One instruction can break another.** A rule to return percentages as 0-100 made the model multiply ratios by 100. Every prompt change was re-tested on the full set to catch this kind of regression.
6. **Forcing a structured finish improved reliability.** Requiring the agent to end with a `submit_sql` tool call (`tool_choice="required"`) stopped it from answering in prose.

## Evaluation method

- **Execution accuracy:** run the generated SQL and the reference SQL and compare result sets, not SQL text, since many queries are correct.
- Numbers match within 0.5%; extra columns in the prediction are allowed; row order is ignored.
- Every failure was reviewed by hand and put in one bucket: missing knowledge, real SQL mistake, ambiguous question, or scorer too strict.
- A scorer bug was found during error analysis (rounding twice made 0.74997 and 0.750 look different) and fixed, and all saved queries were re-scored without new API calls.

## Known limitations

- 20 + 5 questions is small: one question is 5 percentage points (20 points on the held-out set), so differences of one or two questions between versions are within run-to-run noise.
- One question ("average cost of litigated versus non-litigated injury claims") still fails in most versions with a missing GROUP BY.
- SQLite and a 3-table synthetic schema; real warehouses have many more tables, so table selection would become a retrieval problem.

## Safety

The SQL tool allows a single SELECT only, previews at most 20 rows, and the agent has a step cap. In production I would also use a read-only database user, query timeouts, allow-listed tables, masking of sensitive columns, and logging of every generated query.

## Repository

| File | What it is |
| --- | --- |
| `notebooks/01_agent_basics.ipynb` | A tool-calling agent built by hand: SQL and calculator tools, the loop, guardrails, step cap |
| `notebooks/02_text2sql_agent.ipynb` | The text-to-SQL project: data, test set, six versions, scoring, error analysis |
| `text2sql_results.csv` | Every development-set question, version, generated SQL, result and review bucket |
| `text2sql_heldout_results.csv` | The held-out results |

## How to run

Open a notebook in Google Colab, add `OPENAI_API_KEY` in Colab Secrets, and run all cells. No other setup is needed. A full run of notebook 2 costs well under a dollar with `gpt-4o-mini`.

## Next steps

- Grow the test set to 50+ questions and run each version several times to measure run-to-run variance.
- Add tracing (Langfuse or LangSmith) to inspect each agent step.
- Rebuild the agent in LangGraph and add table retrieval for larger schemas.
