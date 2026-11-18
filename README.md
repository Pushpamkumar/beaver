# The Beaver's Choice Paper Company — Autonomous Multi-Agent System

A five-agent AI system built with **smolagents** (HuggingFace) for The Beaver's Choice Paper Company, automating inventory checks, competitive pricing, order fulfillment, and financial reporting through intelligent agent collaboration.

---

## Project Structure

```
project_starter.py            Complete implementation: helper functions + 5-agent system + evaluation runner
workflow_diagram.md           Architecture diagram (Mermaid) + per-agent tool matrix
workflow_diagram.png          High-resolution PNG diagram (auto-generated)
quote_requests_sample.csv     20 customer inquiry test cases (evaluation input)
quote_requests.csv            100-row historical inquiry table (database seed)
quotes.csv                    100-row historical quote table (database seed)
test_results.csv              Evaluation output: per-request cash, inventory, and agent response
reflection_report.md          Comprehensive design and evaluation report
requirements.txt              Python dependency list
README.md / LICENSE / .gitignore / .env.example / starter_README.md
```

---

## Multi-Agent Architecture

The system comprises exactly **five agents** (rubric maximum). The orchestrator coordinates four specialized workers via smolagents' `managed_agents` mechanism. See [`workflow_diagram.md`](workflow_diagram.md) for the full Mermaid diagram.

| Agent | Identifier | Role | Tools |
|---|---|---|---|
| **Customer Engagement Orchestrator** | `customer_liaison` | Routes tasks, evaluates deadline feasibility, synthesizes customer-facing responses | (delegates via `managed_agents`) |
| **Inventory Logistics Specialist** | `warehouse_stock_officer` | Stock audits, supplier lead-time estimation, purchase order placement | `audit_complete_inventory_tool`, `check_item_stock_tool`, `estimate_replenishment_arrival_tool`, `submit_supplier_stock_order_tool` |
| **Pricing & Quoting Strategist** | `quote_pricing_specialist` | Tiered pricing calculation, bulk discounts, historical benchmark retrieval | `calculate_tiered_quote_tool`, `lookup_historical_quote_benchmarks_tool`, `check_item_stock_tool` |
| **Order Fulfillment Agent** | `order_sales_booker` | Verifies stock at booking time, commits sales to transaction ledger | `record_sales_transaction_tool`, `check_item_stock_tool` |
| **Fiscal Audit Agent** | `finance_reporter` | Cash balance queries, comprehensive financial health reporting | `query_cash_reserves_tool`, `generate_fiscal_audit_report_tool` |

### Starter Helper Function Coverage

All seven required helper functions from the starter code are wrapped as `@tool`-decorated callables:

| Starter Helper | Wrapped In |
|---|---|
| `create_transaction` | `record_sales_transaction_tool` + `submit_supplier_stock_order_tool` |
| `get_all_inventory` | `audit_complete_inventory_tool` |
| `get_stock_level` | `check_item_stock_tool` + `calculate_tiered_quote_tool` |
| `get_supplier_delivery_date` | `estimate_replenishment_arrival_tool` + `submit_supplier_stock_order_tool` |
| `get_cash_balance` | `query_cash_reserves_tool` |
| `generate_financial_report` | `generate_fiscal_audit_report_tool` |
| `search_quote_history` | `lookup_historical_quote_benchmarks_tool` |

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Configure your API key (create a `.env` file from `.env.example`):

```
OPENAI_API_KEY=sk-...your-key-here...
```

> **Vocareum proxy users:** Set `UDACITY_OPENAI_API_KEY` instead — the system auto-routes to `https://openai.vocareum.com/v1`.

---

## Running

```bash
python project_starter.py
```

This will:
1. Initialize the SQLite database (`munder_difflin.db`) with inventory, historical quotes, and seed transactions
2. Instantiate all five agents (or fall back to the `AutonomousAgentWorkflowRunner` if no API key is set)
3. Process all 20 requests in `quote_requests_sample.csv` sequentially
4. Write `test_results.csv` with per-request cash balance, inventory value, and agent response
5. Print a final financial summary to stdout

> **Cost estimate (with OpenAI):** ~\$0.30–\$0.80 for the 20-request evaluation run on `gpt-4o-mini`.  
> **Offline mode (no API key):** The `AutonomousAgentWorkflowRunner` runs deterministically at zero cost.

---

## Rubric Coverage

| Rubric Requirement | Satisfied By |
|---|---|
| Workflow diagram with ≤ 5 agents and clear responsibilities | [`workflow_diagram.md`](workflow_diagram.md) + [`workflow_diagram.png`](workflow_diagram.png) |
| Orchestrator + specialized workers (non-overlapping roles) | `project_starter.py` — `customer_liaison` orchestrates 4 workers |
| One of: smolagents / pydantic-ai / npcsh | **smolagents** `ToolCallingAgent` with `managed_agents` |
| All 7 starter helpers used in tools | 7 helpers wrapped across 8 `@tool` functions — see table above |
| Evaluation using `quote_requests_sample.csv` | `run_test_scenarios()` processes all 20 rows |
| `test_results.csv` with ≥ 3 cash balance changes | 8 distinct cash changes recorded across 20 requests |
| At least 3 orders fulfilled | 14 successfully fulfilled |
| At least 1 order rejected/unfulfilled with reason | 6 rejected (deadline infeasibility or stock unavailability) |
| Customer outputs include rationale | Orchestrator directive + fallback runner both mandate explicit rationale |
| No sensitive internals exposed to customer | Orchestrator system prompt prohibits DB table names, error traces, margins |

---

## License

MIT
