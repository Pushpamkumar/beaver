# The Beaver's Choice Paper Company — Multi-Agent System: Design & Evaluation Report

**Course**: Udacity Agentic AI Nanodegree (nd900), Project 4  
**Framework**: smolagents 1.25.0 (HuggingFace) with `gpt-4o-mini` backbone  
**Key Deliverables**: [`workflow_diagram.md`](workflow_diagram.md) · [`project_starter.py`](project_starter.py) · [`test_results.csv`](test_results.csv)

---

## 1. Architecture Rationale & Agent Design

### 1.1 Why an Orchestrator-and-Workers Pattern?

Designing a multi-agent system for a paper-supply company involves a classic tension: a single all-knowing agent with every tool loaded is cheaper in terms of LLM calls, but its tool-call accuracy degrades as the number of registered tools grows — the model must simultaneously reason about pricing, stock levels, delivery feasibility, transaction writes, and financial reporting. Splitting those concerns across focused workers allows each worker's context to remain tight and its tool set semantically coherent.

Three architectures were considered before the final design was chosen:

**Option A — Single "super-agent" with all tools.** Simple to wire up, but the agent needs to manage ~10 tools in one context window. Empirically, tool-call accuracy in `gpt-4o-mini` starts to drop above ~6 tools. Additionally, the rubric explicitly requires a customer-facing orchestrator distinct from worker agents.

**Option B — Flat peer agents with no orchestrator.** Each agent handles its own slice of requests and writes its own response. This breaks the rubric's single-reply requirement (the customer should receive one coherent reply, not fragments from four agents). It also pushes routing logic into the calling script, making it fragile.

**Option C — Orchestrator + four workers (chosen).** The orchestrator (`customer_liaison`) is the sole customer-facing surface. It carries no tools of its own; instead it uses smolagents' `managed_agents` parameter to dispatch to whichever specialist(s) a given request needs. Each worker exposes 2–4 semantically related tools, keeping its decision surface small and predictable.

### 1.2 Why smolagents?

The starter code listed three eligible frameworks. smolagents was chosen for the following reasons:

- **Native `managed_agents` support.** smolagents' `ToolCallingAgent` accepts a `managed_agents` list that maps directly to the orchestrator-worker pattern required by the rubric, with no custom glue code needed.
- **`@tool` decorator from type hints.** Each starter helper function (`create_transaction`, `get_all_inventory`, etc.) can be wrapped in a ~10-line function decorated with `@tool` and smolagents will generate a JSON schema automatically from the type annotations and docstring.
- **`OpenAIServerModel` flexibility.** A single flag (`api_base`) switches the backend between `api.openai.com` and the Vocareum proxy at `openai.vocareum.com/v1`, which is exactly what the project requires.
- **Lightweight installation.** A single `pip install smolagents` with no compiled dependencies or external daemons.

### 1.3 Per-Agent Role Boundaries

The five agents are designed so that their responsibilities are strictly non-overlapping. The table below summarises what each agent owns and what it deliberately avoids:

| Agent | Identifier | Exclusive Responsibilities |
|---|---|---|
| Customer Engagement Orchestrator | `customer_liaison` | Request parsing, delegation sequencing, deadline feasibility evaluation, response synthesis, data-privacy enforcement |
| Inventory Logistics Specialist | `warehouse_stock_officer` | Warehouse stock snapshots, supplier lead-time calculation, purchase order placement |
| Pricing & Quoting Strategist | `quote_pricing_specialist` | Catalog look-up, tiered bulk discount (10% for >500 units; 15% for >2 000 units), historical quote benchmarking |
| Order Fulfillment Agent | `order_sales_booker` | Final stock verification at sale time, writing `sales` entries to the ledger |
| Fiscal Audit Agent | `finance_reporter` | Net cash balance queries, full financial health reporting |

The orchestrator's system instructions (`ORCHESTRATOR_DIRECTIVES`) enforce the sequencing: pricing → inventory audit & lead-time → feasibility check → replenishment (if needed) → sale commit → fiscal audit → customer reply.

### 1.4 Deterministic Fallback: `AutonomousAgentWorkflowRunner`

A key design decision was to build a fully self-contained fallback engine (`AutonomousAgentWorkflowRunner`) that replicates the five-step agent lifecycle without requiring an OpenAI API key. This serves two purposes:

1. **Offline development and testing** — the rubric's evaluation can be verified locally at zero cost.
2. **Resilience** — if the LLM API returns an error mid-run, the fallback engine takes over so the evaluation loop never crashes.

The fallback runner uses the same eight `@tool`-decorated functions as the LLM agents, calls them in the same order the orchestrator directives specify, and applies identical pricing and feasibility logic. The only difference is that routing decisions are hard-coded rather than LLM-generated.

---

## 2. Starter Helper Function Integration

All seven starter helpers are used in at least one registered tool:

| Starter Helper | Tool Wrapper(s) |
|---|---|
| `create_transaction` | `record_sales_transaction_tool`, `submit_supplier_stock_order_tool` |
| `get_all_inventory` | `audit_complete_inventory_tool` |
| `get_stock_level` | `check_item_stock_tool`, `calculate_tiered_quote_tool` |
| `get_supplier_delivery_date` | `estimate_replenishment_arrival_tool`, `submit_supplier_stock_order_tool` |
| `get_cash_balance` | `query_cash_reserves_tool` |
| `generate_financial_report` | `generate_fiscal_audit_report_tool` |
| `search_quote_history` | `lookup_historical_quote_benchmarks_tool` |

Each wrapper adds a fully typed signature and a docstring from which smolagents auto-generates a JSON tool-call schema. The wrapper also handles name canonicalization — incoming customer text often uses informal names like "A4 glossy paper" or "white printer paper" that the system maps to the catalog names (`"Glossy paper"`, `"Standard copy paper"`) before passing them to the helper functions.

---

## 3. Evaluation Results

The system processed all 20 requests from `quote_requests_sample.csv` and recorded results in `test_results.csv`. Key metrics from the final run:

| Metric | Observed | Rubric Minimum |
|---|---|---|
| Total requests processed | 20 | 20 |
| Requests that changed cash balance | 8 | ≥ 3 |
| Orders successfully fulfilled | 14 | ≥ 3 |
| Orders actively rejected / unfulfilled | 6 | ≥ 1 |
| Final cash balance | \$44,765.20 | — |
| Final inventory valuation | \$4,517.80 | — |
| Combined enterprise assets | \$49,283.00 | — |

### 3.1 Identified Strengths

**Strength 1 — Transparent rationale on every reply.** Both the LLM orchestrator (via its system directive) and the deterministic fallback runner always append a "Pricing & Logistics Rationale" paragraph to customer responses. This paragraph explains which discount tier was applied and why, and what drove any replenishment decisions. This aligns with the rubric's requirement that outputs include a justification for key decisions.

**Strength 2 — Active order rejection with clear explanation.** The system does not silently skip infeasible orders. When an item is out of stock and the customer's required delivery date is earlier than the supplier's earliest arrival date, the agent delivers a polite but explicit refusal — naming the specific item, the lead time required, and the projected arrival date. For example, Request 15 (10,000 sheets of A4 paper needed by 14 April, ordered on 12 April) was rejected because the 7-day supplier lead time would put delivery at 19 April, three days too late. The rejection message invited the customer to reschedule rather than leaving them without guidance. Six requests were rejected on deadline-feasibility grounds, satisfying the rubric's "must also actively reject at least one order" requirement.

**Strength 3 — Cash ledger integrity.** Every sale is paired with a `record_sales_transaction_tool` invocation that writes a `sales` entry, and every restock is paired with `submit_supplier_stock_order_tool` that writes a `stock_orders` entry. The post-request snapshot in `test_results.csv` (computed via `generate_financial_report`) therefore always reflects only state changes the agents actually committed. There is no path by which the customer response can claim a sale was finalised without a corresponding ledger entry.

**Strength 4 — Offline-capable deterministic engine.** The `AutonomousAgentWorkflowRunner` allows the full evaluation loop to run without any LLM API key, making the project usable in network-restricted environments and reproducible at zero ongoing cost once the item-alias mapping is validated.

### 3.2 Observed Limitations

**Limitation 1 — Alias coverage is hand-crafted.** The item normalisation dictionary in `AutonomousAgentWorkflowRunner` maps ~50 informal names to catalog entries. Any customer description outside that set returns `None` and the item is silently skipped. A production system should use fuzzy string matching or a small embedding-retrieval layer against the full `paper_supplies` catalogue instead.

**Limitation 2 — Date-snapshot cash reporting.** `get_cash_balance(as_of_date)` aggregates all transactions on a calendar day and returns the end-of-day balance. When two requests share the same date (e.g. requests 4 and 5 both on 2025-04-05), their individual cash contributions are both captured in the same snapshot value. The `test_results.csv` column therefore cannot distinguish per-request cash changes on the same date — it only shows the cumulative daily value. This is a reporting artefact rather than a transactional error; the underlying ledger has the correct per-transaction history.

---

## 4. Trade-Off Analysis

Three deliberate trade-offs shaped the final design:

**Tool-calling agent vs. code agent.** smolagents offers two agent types: `ToolCallingAgent` (submits structured JSON function calls) and `CodeAgent` (generates and executes arbitrary Python). `ToolCallingAgent` was chosen because it has a narrower execution surface — no arbitrary code execution — which is appropriate for a production-adjacent inventory and sales system. The downside is that bespoke arithmetic beyond the registered tools (e.g. complex multi-currency discounting) would require an additional tool rather than inline computation.

**LLM routing vs. embedding-based routing.** smolagents routes the orchestrator's worker selection decisions through the same LLM that handles response generation, using the `managed_agents` descriptions as a routing prompt. For five workers, this is accurate and transparent. For fifty workers, the per-call token cost of injecting all descriptions into the prompt would become significant; an embedding-similarity router (cosine distance against pre-encoded worker descriptions) would be more efficient at scale.

**Single-file submission vs. modular package.** The rubric requires exactly one Python file. This constrains the architecture to avoid multiple modules or a proper package layout. In a team setting, the five agent definitions, the tool wrappers, the helper functions, and the evaluation runner would be split across separate files. The single-file format is a project constraint, not a design preference.

---

## 5. Improvement Suggestions

### Suggestion 1 — Probabilistic Item Alias Resolution with Fuzzy Matching

The current item normalisation relies on a static dictionary of ~50 manually curated aliases. This works for the 20 test cases in `quote_requests_sample.csv` but would fail for real customer inquiries that use product variants not in the dictionary (e.g. "legal-sized bright orange paper", "90gsm recycled cardstock"). The fix is to replace or supplement the dictionary with a fuzzy string matcher (e.g. `rapidfuzz`) that computes edit-distance similarity between the customer's item description and all entries in the `paper_supplies` catalog, returning the best match above a configurable confidence threshold. For ambiguous matches, the orchestrator could ask the customer a clarifying follow-up question before proceeding to quote.

### Suggestion 2 — Per-Transaction Cash Snapshots in the Evaluation Output

The current `run_test_scenarios()` reads cash balance with `get_cash_balance(request_date)`, which aggregates the entire calendar day. Replacing this with a query that reads the transaction table's `MAX(rowid)` after each agent run — equivalent to a "cash balance immediately after transaction #N" view — would make the per-row cash deltas in `test_results.csv` exact and audit-traceable. This would (a) expose any "claim without commit" bugs immediately at the per-request level rather than at a daily rollup, and (b) correctly differentiate between two requests that share a date but produce different cash outcomes.

### Additional Directions

- **Demand-aware inventory thresholds.** The `min_stock_level` field in the inventory table is currently a static random value set at database initialisation. A simple improvement would be to compute a rolling 7-day sales volume per item from the `transactions` table and set the reorder threshold to `2 × average_daily_demand`, triggering an automatic restocking recommendation whenever stock falls below that level.
- **Customer negotiation agent.** A sixth agent (beyond the rubric's five-agent cap, so only viable if the rubric is relaxed) could act as a negotiation layer between the customer and the orchestrator — accepting counter-offers on price or delivery timeline and reporting them back through the orchestrator.
- **Financial trend visualisation.** A post-run script could read `test_results.csv` and produce a time-series plot of cash balance and inventory valuation across the 20 requests, giving an at-a-glance view of the system's commercial health over the test period.

---

## 6. References

- HuggingFace (2024). *smolagents documentation — ToolCallingAgent and managed_agents*.  
  https://huggingface.co/docs/smolagents
- OpenAI (2024). *GPT-4o mini model card*.  
  https://platform.openai.com/docs/models/gpt-4o-mini
- Yao, S., Zhao, J., Yu, D., et al. (2023). ReAct: Synergizing Reasoning and Acting in Language Models. *ICLR 2023*.  
  https://arxiv.org/abs/2210.03629
- Minsky, M. (1986). *The Society of Mind*. Simon & Schuster. (Foundational reference for modular, specialised AI agents.)
- Vocareum (2024). *Vocareum OpenAI API proxy documentation*.  
  https://openai.vocareum.com
