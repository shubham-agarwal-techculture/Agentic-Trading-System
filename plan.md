# Agentic Trading System — Implementation Plan

## 1. Goal and scope

Build a chat-first application that turns a user's natural-language strategy idea into an explicit, reviewable backtest specification, runs that specification through a deterministic quantitative engine, and presents the results in chat and a downloadable report.

The first release is **agentic backtesting only**. It does not place live trades, connect to brokerage execution, or let the LLM calculate or fabricate performance results. Gemini is the LLM provider for the initial release.

## 2. Product workflow

1. The user starts a conversation and describes a strategy, market, time period, and other preferences in natural language.
2. The backend sends the conversation and relevant system instructions to the Gemini API.
3. Gemini extracts a typed draft specification. If required values are missing or ambiguous, the assistant asks targeted follow-up questions instead of silently guessing.
4. The UI displays the interpreted strategy, including parameters, indicators, assumptions, and any unresolved questions.
5. The user edits or confirms the specification. No backtest starts until the user explicitly confirms it.
6. The backend validates the confirmed specification and submits it to the quantitative engine as a separate backtest job.
7. The engine loads historical data, evaluates the strategy, and returns results and diagnostics.
8. The UI displays metrics and charts, offers report/CSV downloads, and allows further discussion in the same conversation. Follow-up chat must distinguish verified results from hypotheses and must not change a completed run.

## 3. Suggested architecture

Keep the LLM, quantitative engine, and presentation layer as separate components with explicit contracts.

- **Chat UI:** Chat-style conversation, strategy draft/confirmation card, clarification flow, job status, result dashboard, and downloads.
- **Application API:** Authentication and conversation state; request validation; Gemini orchestration; strategy confirmation; job creation and result/report access.
- **Gemini adapter:** The only initial integration with Gemini. Encapsulate model selection, prompts, structured response parsing, token/error handling, and timeouts so provider-specific behavior does not leak into the backtest engine.
- **Strategy specification:** A versioned, typed schema shared by the UI and backend. Include market/instrument, timeframe, date range, entry/exit rules, indicators and their parameters, position sizing, fees/slippage assumptions, and data source. Represent unspecified values explicitly; do not interpret missing values as approval to use hidden defaults.
- **Backtest worker/quantitative engine:** Deterministic execution of validated specifications against historical data. It must not call the LLM to decide trades or compute metrics.
- **Persistence:** Store conversations, draft and confirmed specifications, backtest job state, result metadata, and report references. Keep credentials and sensitive configuration out of stored chat content.
- **Historical data adapter:** A replaceable interface for data providers, with source, coverage, timestamp/timezone, adjustment, and missing-data metadata recorded for each run.
- **Artifact/report service:** Produce reproducible result files and charts, with downloads authorized against the owning user/conversation.

For an initial deployment, the API and worker may live in one codebase while remaining separate modules/process responsibilities. Add a queue and independently scaled workers when execution time or concurrency warrants it; do not introduce distributed infrastructure before the MVP requires it.

## 4. Core contracts and behavior

### Strategy specification

Define and version a schema before wiring the chat flow. Validate types, allowed operators/indicators, parameter bounds, date ordering, supported timeframe/instrument combinations, and required assumptions on the server. The browser is not a trust boundary.

Maintain a clear distinction between:

- A **draft** generated or revised during conversation.
- A **confirmed specification** explicitly approved by the user.
- An immutable **run snapshot** submitted to the engine.

Each run should record the schema version, exact confirmed parameters, data source/version, execution settings, and engine version so the result can be reproduced.

### LLM responsibilities

Gemini may interpret intent, propose schema-conforming values, explain the interpretation, identify ambiguity, and answer questions about returned results. It must not execute arbitrary code, access secrets, fetch unapproved data, or produce authoritative backtest metrics.

Use structured output constrained to the strategy schema where supported, then validate it independently in application code. Treat model output as untrusted input. Handle malformed output, refusals, rate limits, timeouts, and unavailable service explicitly; preserve the user's conversation and let them retry. Do not claim a backtest succeeded if the quantitative engine did not return a successful result.

### Quantitative engine

Start with a deliberately small, documented set of supported instruments, timeframes, indicators, and rule operators. Reject unsupported strategies with an actionable explanation rather than approximating them. Define fee/slippage and position-sizing behavior, signal timing, order fills, and missing-data policy before reporting performance. Prevent look-ahead bias and use consistent timestamp handling.

### Results and reports

Show a concise run summary, selected performance/risk metrics, equity curve, drawdown, and trades or trade samples. Clearly display the tested interval, instrument, timeframe, data source, assumptions, and any limitations. Generate downloadable report data from engine results, not from LLM prose. Include CSV exports and a human-readable report; keep chart data available for the dashboard.

## 5. Delivery milestones

### Milestone 1 — Foundations and contracts

- Inspect the existing project and select implementation details that fit its current stack.
- Define the strategy specification, validation rules, conversation/run state model, and API contracts.
- Decide and document the first supported market/data source, strategy primitives, and backtest assumptions.
- Configure Gemini access through a server-side environment variable or secret manager; never expose the API key in client code or source control.

**Exit criteria:** A valid example strategy can be represented, validated, versioned, and rejected with useful errors when invalid or unsupported.

### Milestone 2 — Chat and strategy clarification

- Build the chat interface and conversation persistence.
- Add a Gemini-backed service that returns either a schema-validated draft or targeted clarification questions.
- Render the draft as an editable, human-readable strategy summary.
- Implement explicit confirm/edit/revise behavior and retain prior draft versions for context.

**Exit criteria:** Missing or ambiguous fields prompt for clarification; no chat response can start a run without an explicit user confirmation.

### Milestone 3 — Backtesting vertical slice

- Implement historical data loading for the selected source and a small initial strategy vocabulary.
- Implement deterministic simulation, trade records, fees/slippage, and key metrics.
- Add validated job submission, progress/error states, and persisted immutable run snapshots.
- Connect confirmed specifications to the engine without passing model-generated executable code.

**Exit criteria:** A confirmed example strategy can run end to end, and repeating it with the same data and engine settings gives reproducible results.

### Milestone 4 — Results, reports, and follow-up chat

- Build the results dashboard and chart views.
- Add downloadable report and CSV artifacts.
- Support follow-up questions grounded in persisted run results; identify when an answer is explanatory rather than a new calculation.
- Add access checks and artifact lifecycle handling.

**Exit criteria:** The user can inspect, download, and discuss a completed run without the LLM inventing metrics or mutating the run.

### Milestone 5 — Hardening and release

- Add operational logging, error monitoring, rate limits, request/job limits, and Gemini usage/cost visibility.
- Review authentication/authorization, input validation, prompt-injection boundaries, secret handling, and data retention.
- Test data gaps, unsupported strategies, provider failures, worker failures, repeat submissions, and large/long-running jobs.
- Document supported strategies, assumptions, known limitations, setup, and deployment.

**Exit criteria:** Core user flows and failure cases are tested; users can understand the limits and assumptions of every reported result.

## 6. Initial API surface (illustrative)

Adapt naming and transport to the existing project conventions.

- `POST /conversations` — create a conversation.
- `POST /conversations/{id}/messages` — append a user message and return assistant text plus a draft specification or clarification request.
- `PUT /conversations/{id}/strategy-draft` — update editable draft fields.
- `POST /conversations/{id}/strategy-confirmations` — validate and explicitly confirm a draft.
- `POST /backtests` — submit a confirmed specification and return a job identifier.
- `GET /backtests/{id}` — retrieve job state and completed results.
- `GET /backtests/{id}/report` and `GET /backtests/{id}/trades.csv` — authorized artifact downloads.

All endpoints that access conversations, runs, or files must enforce ownership. Use idempotency or duplicate-submission protection for backtest creation.

## 7. Validation and quality strategy

- **Unit tests:** Schema validation, strategy parsing, indicator calculations, fees/slippage, fills, metrics, and report serialization.
- **Engine tests:** Small fixed datasets with hand-checkable expected signals, trades, and results; regression fixtures for known cases.
- **LLM integration tests:** Mock Gemini for deterministic application tests; separately test structured-output parsing and explicit provider error handling. Avoid depending on live model output for core CI.
- **API/UI integration tests:** Clarification → draft → edit → confirmation → job → results → download, including unauthorized access and failure states.
- **Financial correctness checks:** No future data in signal decisions, no accidental duplicate fills, coherent position/cash accounting, and metrics computed from engine output.
- **Reproducibility checks:** Persisted run snapshots and data/engine metadata are sufficient to explain and reproduce a run.

## 8. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| The model misunderstands or invents parameters | Show an editable specification, ask about ambiguity, validate on the server, and require explicit confirmation. |
| Backtest appears profitable due to simulation errors | Define execution assumptions, test against hand-calculated fixtures, guard against look-ahead bias, and disclose limitations. |
| Historical data is incomplete or inconsistent | Record provenance and coverage; surface gaps and reject runs when data quality is insufficient. |
| LLM/API outage blocks chat | Return a clear retryable error and retain conversation/draft state; keep engine operation independent of Gemini. |
| API key or user data is exposed | Keep Gemini calls server-side, use managed secrets/environment configuration, enforce authorization, and minimize logged sensitive content. |
| Scope expands into live trading | Keep execution out of Phase 1 and use a separate, explicitly reviewed product/security phase if it is ever considered. |

## 9. Decisions to confirm during implementation

The description does not specify the application framework, market/data provider, initial strategy set, authentication needs, or deployment environment. Resolve these against the existing repository before implementation. A practical MVP should start with one market/data source and a small set of transparent, testable indicators and rule operators rather than trying to support arbitrary strategies.
