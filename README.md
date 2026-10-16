Wealth Advisor Assistant — Multi-Agent Demo
A small, dependency-free Python project demonstrating a custom multi-agent workflow for financial data ingestion, mock CRM enrichment, explainable anomaly detection, structured reporting, logging, local memory, and a human-review checkpoint.
Safety and scope: All bundled client/CRM records are synthetic. This is an educational engineering demo, not investment, tax, legal, or individualized financial advice. Flags are review signals—not proof of fraud. Do not use this demo with real client data without security, privacy, compliance, and financial-domain review.

Features
- Orchestrator Agent: coordinates the workflow and review status.
- Data Fetcher Agent: validates required input fields and calls a CRM tool through an interface; falls back gracefully if CRM is unavailable.
- Analyzer Agent: computes cash-flow metrics and detects large transactions, repeated transaction IDs, possible duplicate transaction signatures, monthly spending spikes, invalid amounts/dates, zero income, low savings rate, and expenses above income.
- Tool abstraction: CRMTool and MemoryTool protocols separate agents from implementations.
- Error handling: clear validation errors, CRM fallback warnings, and resilient memory reads.
- Observability: console and file logs (logs/wealth_advisor.log).
- Memory: local JSON history of previous report summaries.
- Human-in-the-loop: --review pending|approve|override simulates a reviewer checkpoint. override intentionally requests review rather than silently changing financial findings.
- Tests: standard-library unittest coverage for analysis, workflow, memory, and CRM failure.
Architecture
Client JSON
    |
OrchestratorAgent
    |
    +--> DataFetcherAgent ----> CRMTool protocol ----> MockCRMClient / data/crm.json
    |            |
    |            +-- CRM failure => warning + continue without CRM context
    |
    +--> MemoryTool protocol --> JsonMemoryStore / data/memory.json (read history)
    |
    +--> AnalyzerAgent -------> metrics + anomaly flags + recommendations
    |
    +--> Human review status (pending / approved / override requested)
    |
    +--> MemoryTool (persist compact report summary)
    v
Structured JSON report
Agent interactions
1. The CLI loads and validates the top-level JSON shape.
2. The orchestrator calls the Data Fetcher Agent.
3. The fetcher checks required fields and attempts CRM enrichment. CRM errors become warnings; they do not block analysis.
4. The orchestrator loads prior client summaries from memory.
5. The Analyzer Agent produces deterministic metrics, anomaly signals, and conservative follow-up suggestions.
6. The workflow adds a human-review status and saves a compact insight for later runs.
7. The CLI writes the report to output/report.json and prints it to stdout.
Project structure
wealth_advisor_assistant/
├── data/
│   ├── client.json
│   └── crm.json
├── tests/
│   ├── test_analyzer.py
│   └── test_workflow.py
├── wealth_advisor/
│   ├── agents/
│   │   ├── analyzer.py
│   │   ├── data_fetcher.py
│   │   └── orchestrator.py
│   ├── tools/
│   │   ├── interfaces.py
│   │   ├── json_memory.py
│   │   └── mock_crm.py
│   └── app.py
├── requirements.txt
└── README.md
Requirements
- Python 3.10 or newer (uses modern type annotations).
- No third-party packages required.
Run it
From the project root:
python -m wealth_advisor.app --input data/client.json --crm data/crm.json --output output/report.json
Choose a review outcome:
python -m wealth_advisor.app --review approve
python -m wealth_advisor.app --review override
Simulate a CRM outage (analysis should still finish with a warning):
python -m wealth_advisor.app --crm-fail
Change logging verbosity:
python -m wealth_advisor.app --log-level DEBUG
The report is printed to the terminal and written to output/report.json. Logs are written to logs/wealth_advisor.log. The memory file is created automatically at data/memory.json.
Run tests
python -m unittest discover -s tests -v
Input format
See data/client.json. Required fields:
- client_id: non-empty string
- currency: currency label, e.g. USD
- transactions: list of transaction objects
- monthly_income: non-negative number
- monthly_expenses: non-negative number
Transactions should ideally contain transaction_id, ISO date (YYYY-MM-DD), merchant, category, and numeric amount. Some optional transaction fields may be omitted; malformed records are flagged and skipped where safe.
Sample output shape
The generated report contains:
- analysis: income, reported expenses, estimated surplus, savings rate, and monthly transaction totals
- crm: enrichment status, risk profile, and goals when available
- anomalies: severity, type, evidence, and explanation
- recommendations: suggested review steps
- memory_context: count of prior summaries
- review: human-review status and note
- warnings: data/API caveats
- disclaimer: limits of automated analysis
To generate a fresh sample output, run the command above. sample_output.json is included as a representative report from the bundled input.
Key decisions and trade-offs
- Custom agents instead of an agent framework: makes routing explicit, reduces dependencies, and is easy to test and explain. The agents are modular Python components, not autonomous LLMs.
- Deterministic rules instead of an LLM for anomaly detection: improves reproducibility and explainability. Thresholds are illustrative and should be calibrated for the client, currency, transaction type, and institution.
- Mock CRM JSON instead of a live API: works offline and avoids credentials. MockCRMClient is the replaceable adapter boundary.
- JSON memory instead of a database/vector store: minimal setup and transparent state for a small demo. It is not intended for concurrent production use or sensitive financial data.
- Human review is represented as a CLI decision: a practical demonstration of the checkpoint without requiring a UI. The override option does not rewrite analysis silently.
- Monthly transaction totals use positive transaction amounts only: refunds and debits/credits can require institution-specific sign conventions; real integrations should normalize these before trend analysis.
Known limitations / next steps
- Rules are heuristic; no model training or calibrated fraud detection is included.
- Monthly income/expenses are supplied aggregates and are not reconciled against transaction data.
- Memory locking is process-local only; use a database with access controls for multi-user use.
- For production: add schema validation (e.g. Pydantic), authenticated API clients, secrets management, encrypted storage, audit trails, role-based review, rate limiting, monitoring, consent and retention controls, and domain/compliance approval.
