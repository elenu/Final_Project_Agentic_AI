# Summary of the project

This document summarizes the multi-agent system implemented in `project_starter_Elena.py`, the evaluation status, strengths and areas for improvement, and next steps to obtain a full evaluation run producing `test_results.csv`.

## 1. Overview

- The project implements a small multi-agent system for a fictional paper supplier (Munder Difflin).
- Agents are implemented as `ToolCallingAgent` subclasses from the `smolagents` framework and use small database-backed helper functions to perform tasks.

- Agents and responsibilities:
  a. InventoryAgent
    - Tools: `check_inventory` and `get_item_price` (decorated with `@tool`), `get_all_inventory` and `get_stock_level` (wrapped as smolagents functions).
    - Responsible for inspecting stock levels and answering inventory queries.
  b. QuotingAgent
    - Tools: `generate_quote` and `get_item_price` (decorated with `@tool`), `get_all_inventory` and `get_stock_level` (wrapped as smolagents functions).
    - Responsible for creating customer quotes based on inventory and pricing.
  c. OrderingAgent
    - Tools: `place_order` (decorated with `@tool`) and `get_cash_balance`(wrapped as smolagent function).
    - Responsible for placing orders (sales or stock replenishment) and updating transactions.
  d. OrchestrationAgent
    - Instantiates the three worker agents and coordinates end-to-end request handling: inventory check -> quote -> order.

- Tools (helper functions):
  a. `get_all_inventory(as_of_date)` — returns inventory snapshot as a dict.
  b. `get_stock_level(item_name, as_of_date)` — returns a DataFrame with current stock for a single item.
  c. `get_item_price(item_name)`, `check_inventory(request, as_of_date)`, `generate_quote(request, inventory_check)`, `place_order(quote_details, request_date)` — provide the business logic for quoting and ordering.

- Data & DB:
  a. Uses an SQLite DB `munder_difflin.db` created via SQLAlchemy's `create_engine`.
  b. On `init_database()` the script creates `transactions`, `inventory`, `quotes`, and `quote_requests` tables from CSVs and seeds inventory.

- Main evaluation flow with `run_test_scenarios()`:
  1. Initializes DB (calls `init_database`).
  2. Reads `quote_requests_sample.csv` and iterates requests chronologically.
  3. For each request, uses `OrchestrationAgent.handle_request` to process the request and appends a result entry.
  4. Saves results to `test_results.csv`.
  
## 2. Evaluation results

It is necessary to install requirements.txt but also smolagents before running the code.

What we can find in `test_results.csv` when we run the evaluation successfully
- Columns (produced by `run_test_scenarios()`): `request_id`, `request_date`, `cash_balance`, `inventory_value`, `response`.
- Each `response` is a dictionary-like object containing `inventory_check`, `quote_details`, and `order_confirmation` returned by the orchestration agent for that request.

How to reproduce evaluation locally (bash):
1. From the workspace root (where `project_starter_Elena.py` is located), I created a virtual environment with Udacity's OPENAI_API_KEY, and installed dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install pandas numpy python-dotenv sqlalchemy smolagents
```

2. Ensured the CSV files expected by the script were present in the same folder:
- `quote_requests.csv`
- `quotes.csv`
- `quote_requests_sample.csv`

3. Ran the evaluation:

```bash
python3 project_starter_Elena.py
```

4. After a successful run we find `test_results.csv` in the same folder.

Quick verification steps
- Confirmed `munder_difflin.db` appeared (SQLite DB).
- Inspected `test_results.csv` with `head` to confirm rows exist.

## 3. Strengths observed

- Clear actor separation: agents are narrowly scoped (inventory, quoting, ordering) which aids testability and maintainability.
- Used an explicit `OrchestrationAgent` to keep the end-to-end flow readable and deterministic.
- Database-backed transactions allowed persistent, auditable state changes.
- Discount & fulfillment logic (discount applied only when available stock is sufficient) prevented overpromising.

## 4. Areas for improvement

- Dependency management: the execution environment must have `smolagents`, `sqlalchemy`, and other packages installed; this should be declared in a `requirements.txt` and verified in CI.
- Tool typing: agents assert tools are `BaseTool` objects; ensuring all tool functions are decorated with `@tool` or wrapped is required (I fixed two functions, but confirm all tool functions are decorated consistently). I added a runtime validator and automatic wrapper for tools in each agent's constructor so the AssertionError ("All elements must be instance of BaseTool") won't occur when plain callables are passed.
- Concurrency: the DB operations are naive; if this system were to run concurrently, we'd need transaction handling and locking strategies.

## 5. Concrete suggested improvements

Low-effort (quick wins)
- Add `requirements.txt` with pinned versions: pandas, numpy, python-dotenv, sqlalchemy, smolagents

Medium-effort
- Add logging (structured JSON logs) instead of prints.

Higher-effort
- Add concurrency-safe database access (SQLAlchemy session management) with tests simulating concurrent order placement.
- Add an API layer and a lightweight front-end to simulate customer interactions.

## 6. Agent flow diagram (textual)

The repository includes a Mermaid flowchart that documents the full data and agent interactions:

- Diagram file: `Mermaid Chart - Create complex, visual diagrams with text.-2026-03-14-180536.mmd`

Summary of the flow (mapped directly from the Mermaid diagram):

- Data & DB initialization
  - `quotes.csv` -> `quotes_df` (DataFrame)
  - `quote_requests.csv` -> `quote_request_df` (DataFrame)
  - `paper_supplies` (list) -> `generate_sample_inventory()` -> `inventory` (DataFrame)
  - `init_database()` composes `transactions_schema`, `quotes_df`, `quote_request_df` and `inventory`, and returns an initialized `Engine`/DB (`munder_difflin.db`)

- High-level orchestrator (OrchestrationAgent)
  - The Orchestrator receives a free-text request + date and performs three main steps in sequence:
    1) Inventory check (InventoryAgent)
    2) Quote generation (QuotingAgent) using the inventory_check
    3) Order confirmation (OrderingAgent) using the quote_details
  - It also calls `generate_financial_report()` as a helper to fetch company status for reporting.

- InventoryAgent (responsibilities & helpers)
  - `check_inventory(request, as_of_date)` — parses free-text requests and returns `inventory_check` (Dict of requested item -> available stock)
  - `get_all_inventory(as_of_date)` — helper that returns a Dict of all available items and stock for a date
  - `get_stock_level(item_name, as_of_date)` — helper that returns a DataFrame with `current_stock` for the item
  - `get_item_price(item_name)` — helper returns unit price from `inventory`

- QuotingAgent (responsibilities & helpers)
  - `generate_quote(request, inventory_check)` — calculates `quote_details` (items, price totals, discounts, notes) and uses `get_item_price`
  - `search_quote_history(search_terms)` — helper to retrieve historical quotes matching given terms

- OrderingAgent (responsibilities & helpers)
  - `place_order(quote_details, request_date)` — uses `create_transaction()` to record `sales` or `stock_orders`, checks `get_cash_balance()` before committing, and returns `order_confirmation`
  - `get_supplier_delivery_date(start_date, quantity)` — helper to estimate lead times

- Reporting & Outputs
  - `generate_financial_report(as_of_date)` — aggregates cash balance, inventory valuation, and top-selling products
  - `run_test_scenarios()` — drives the evaluation over `quote_requests_sample.csv`, invoking the orchestration for each request and writing `test_results.csv`

Diagram notes (explicit links from Mermaid):
  - The DB engine is represented as the central cylinder feeding the Orchestrator.
  - InventoryAgent connects to `get_all_inventory`, `get_stock_level` and `get_item_price`.
  - QuotingAgent connects to `generate_quote` and `search_quote_history` and consumes `inventory_check`.
  - OrderingAgent connects to `place_order`, `create_transaction`, and `get_cash_balance` and produces `order_confirmation`.
  - `test_results.csv` is the final sink capturing orchestration outputs for each processed request.

I enclose same diagram in PNG and SVG formats.

## 7. Completed implementation script
- Path: `project_starter_Elena.py`
- Status: updated — key inventory helpers (`get_all_inventory`, `get_stock_level`) are decorated with `@tool`; duplicate-definition/indent issues were fixed; file compiles.

## 8. How I verified the changes
- Static/syntax check: `python -m py_compile project_starter_Elena.py` — PASS.

## 9. Next steps and recommended deliverables
These steps allowed to produce `test_results.csv` and improve robustness and observability.

Required (to run evaluation locally or in CI):
- Add `requirements.txt` with pinned versions (example below).
- Ensure CSV inputs are present: `quote_requests.csv`, `quotes.csv`, `quote_requests_sample.csv`.

Suggested short-term changes:
1) Add `requirements.txt` and a small `Makefile` or run script so reproducing the evaluation is one command.
2) Add structured logging (JSON) and replace critical print statements with logger calls.

Example `requirements.txt` (suggested):
```
pandas==2.2.2
numpy==1.26.2
python-dotenv==1.0.0
SQLAlchemy==2.0.22
smolagents==0.1.0
pytest==8.2.5
rapidfuzz==2.15.1   # optional, for better fuzzy matching
```