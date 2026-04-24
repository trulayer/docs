# Core Concepts

This page defines the first-class concepts in TruLayer AI. Understanding these will help you instrument your app correctly and interpret what you see in the dashboard.

---

## Trace

**A trace is the top-level unit of work — one user request, one pipeline run, or one batch job.**

A trace captures everything that happened from input to output: which model was called, how long it took, how much it cost, and whether it errored. All spans, events, and evaluations belong to a trace. Traces are stored as a TimescaleDB hypertable, meaning they are append-only and queried efficiently by time range.

**Key fields:** `id`, `project_id`, `session_id`, `name`, `input`, `output`, `model`, `latency_ms`, `cost`, `error`, `metadata`

**Example:** A user sends a message to your chatbot. Your backend starts a trace, captures the full prompt as `input` and the final reply as `output`, then closes the trace with `latency_ms` and `cost` set.

**SDK:** Pass a `trace_id` when calling `POST /v1/ingest`. **Dashboard:** Traces list at **Observability > Traces**.

---

## Span

**A span is a timed operation within a trace — one LLM call, one tool call, one retrieval step.**

Spans have a `type` (e.g., `llm`, `tool`, `retrieval`, `chain`) and can be nested via `parent_span_id`, forming a tree that represents the full execution path. Each span records its own `input`, `output`, `model`, `latency_ms`, `cost`, and timing via `start_time` / `end_time`. Spans map to OpenTelemetry spans via `otel_trace_id` and `otel_span_id` if you use the OTel bridge.

**Key fields:** `id`, `trace_id`, `parent_span_id`, `name`, `type`, `input`, `output`, `model`, `latency_ms`, `cost`, `error`, `start_time`, `end_time`, `otel_trace_id`, `otel_span_id`

**Example:** Within a single chatbot trace, you have three spans: a `retrieval` span that hits your vector store, an `llm` span for the GPT-4o call, and a `tool` span for a function call the model made.

**SDK:** Include `span_id` and optionally `parent_span_id` in each ingest payload. **Dashboard:** Span waterfall at **Observability > Traces > [trace] > Spans**.

---

## Session

**A session groups multiple traces that belong to one user conversation or workflow execution.**

When your app maintains a multi-turn conversation, each user turn produces a separate trace. Attaching the same `session_id` to all of them lets TruLayer correlate them into a single conversation view. Sessions are not a separate database table — they are a `session_id` UUID on the `Trace` model.

**Key field on Trace:** `session_id` (nullable UUID)

**Example:** A user has a five-turn conversation with your assistant. Each turn is a separate trace, all sharing the same `session_id`. The dashboard shows the full conversation timeline in one view.

**SDK:** Generate a stable UUID for the conversation on your server and pass it as `session_id` on every `POST /v1/ingest` call for that conversation. **Dashboard:** Session timeline at **Observability > Sessions**.

---

## Event

**An event is a point-in-time observation attached to a trace or span — a log line, a token count, a tool output, a guardrail trigger.**

Unlike spans, events have no duration. They carry a `level` (`info`, `warn`, `error`) and a free-text `message`, with optional structured `metadata` for anything machine-readable. Events belong to a trace and optionally to a specific span via `span_id`. Like traces and spans, they are stored as a TimescaleDB hypertable.

**Key fields:** `id`, `trace_id`, `span_id`, `level`, `message`, `metadata`

**Example:** During an LLM call, your app logs `{"token_count": 1240, "model": "gpt-4o"}` as an `info` event attached to the `llm` span. A content-filter trip logs as an `error` event on the same trace.

**SDK:** Include `events` in the ingest payload or call `POST /v1/ingest/events` separately. **Dashboard:** Events panel at **Observability > Traces > [trace] > Events**.

---

## Eval

**An eval is a programmatic quality check run against a trace — it produces a score, a pass/fail label, and optional reasoning.**

Evals are created by eval rules that trigger automatically on new traces (via the Kafka consumer pipeline) or run on-demand against a dataset. Each eval records the `metric_name` being measured, the `evaluator_type` (`llm`, `heuristic`, `code`), a numeric `score`, a `label`, and its `status` (`pending`, `running`, `completed`, `failed`). The `reasoning` field captures the judge's chain-of-thought when an LLM evaluator is used.

**Key fields:** `id`, `trace_id`, `metric_name`, `score`, `label`, `evaluator_type`, `status`, `reasoning`, `eval_run_id`

**Example:** You define an eval rule for `answer_relevance`. Every new trace triggers the rule; the LLM judge scores it 0–1 and writes the result back as an `Evaluation` with `metric_name = "answer_relevance"` and `score = 0.87`.

**SDK:** Create eval rules via `POST /v1/eval-rules`; query results via `GET /v1/eval`. **Dashboard:** Eval results at **Evals > Results**; rules at **Evals > Rules**.

---

## Feedback

**Feedback is a human or programmatic label attached to a trace — thumbs up/down, a rating, or a free-text comment.**

Feedback is explicit signal from end users or reviewers, as opposed to automated eval scores. The `label` field holds the signal (`thumbs_up`, `thumbs_down`, a numeric rating string, or a custom label). `comment` captures free text. `user_id` is optional and lets you tie feedback back to the end user who gave it (your app's user ID, not a TruLayer user).

**Key fields:** `id`, `trace_id`, `user_id`, `label`, `comment`, `metadata`

**Example:** You render a thumbs-up/down widget below each AI response. When the user clicks thumbs-down, your backend calls `POST /v1/feedback` with `{"trace_id": "...", "label": "thumbs_down", "comment": "Wrong answer"}`.

**SDK:** `POST /v1/feedback` to submit; `GET /v1/feedback` to query. **Dashboard:** Feedback at **Observability > Feedback**.

---

## Metric

**A metric is an aggregated analytic computed over traces or evals — average latency, token cost, eval pass rate, error rate.**

Metrics are not a stored model like the concepts above — they are computed by the analytics layer on query. The backend uses TimescaleDB's `time_bucket()` function to downsample hypertable data into configurable time windows (hourly, daily). You query metrics via `GET /v1/metrics` with filters for `project_id`, `model`, time range, and bucketing interval.

**Common metrics:** `avg_latency_ms`, `p95_latency_ms`, `total_cost`, `error_rate`, `eval_pass_rate`, `span_count`

**Example:** Your weekly review shows p95 latency spiked on Tuesday — you drill into the **Metrics** chart, filter by `model=gpt-4o`, and find a cluster of retrieval spans that took 4 seconds each.

**SDK:** `GET /v1/metrics` with query parameters. **Dashboard:** Time-series charts at **Analytics > Metrics**.
