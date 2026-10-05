# Agentic Commerce Assistant

A multi-agent e-commerce shopping assistant built on **LangGraph** and **Groq**.
Eighteen agents, orchestrated by a LangGraph state machine, reach the world only
through a validated tool layer over a swappable repository.

It is not a chatbot wrapper. A turn is planned, decomposed, routed across
parallel workers, audited against tool evidence, and only then written into prose.

A real transcript, captured from the seeded dataset with **no Groq key configured**
— `USR-1001` starts with a Lenovo Legion and a charger already in their cart:

```
"Find gaming laptops under ₹90,000"
  → Found 3 product(s) matching "Find gaming laptops" under 90,000. Results:
    (1) ASUS TUF Gaming F15 FX507ZC4 at ₹74,990, 4.3★; (2) Acer Nitro V 15 at
    ₹58,990, 4.0★; (3) HP Victus 15 at ₹67,490, 4.1★.

"Add the first one to my cart"
  → Added ASUS TUF Gaming F15 FX507ZC4 Gaming Laptop to your cart — ₹74,990
    each, cart subtotal now ₹2,06,880. Total ₹2,44,118.40.

"Remove the charger"
  → Removed Apple 20W USB-C Power Adapter from your cart. Subtotal is now ₹2,04,980.

"Apply SAVE20"
  → SAVE20 applied — you save ₹5,000. Total ₹2,35,976.40 (you save ₹5,000 with SAVE20).

"Checkout"
  → Ready to place your order. Subtotal ₹2,04,980; Discount -₹5,000;
    Tax (18% GST) ₹35,996.40; Shipping Free; Total ₹2,35,976.40. Expected
    delivery by Monday (17 Aug). Nothing has been charged yet. BIGSPEND10 would
    save you more on this cart. Place this order for ₹2,35,976.40?

"yes"
  → Order ORD-20260810-6AG1 placed for ₹2,35,976.40, expected Monday (17 Aug).
    Payment via UPI. Tracking XPR99447572600 with XpressBees.
```

Three things in that exchange are worth noticing. *"The first one"* resolves against
the numbered list from the previous turn. *"The charger"* finds a product named
"Power Adapter" — the word appears only in its tags. And *"Checkout"* places
nothing: it quotes the total, notes a better coupon exists, and waits for the
explicit "yes".

---

## Quick start

```bash
make install                       # venv + dependencies
cp .env.example .env               # then set GROQ_API_KEY
make run                           # http://localhost:8000/docs
```

In another terminal:

```bash
make smoke                         # drives 15 turns through the live API
```

**It runs without a Groq key.** With `GROQ_API_KEY` unset the assistant serves in
*deterministic mode*: rule-based planning and routing, templated replies, and all
32 tools fully functional. Every example above works in that mode — the transcript
was captured with no key configured. `/health/ready` reports `degraded`, not
unhealthy, because the system still answers every question. See
[Degraded mode](#degraded-mode).

Docker:

```bash
GROQ_API_KEY=gsk_… docker compose up -d
```

---

## What it does

| | |
|---|---|
| **Discovery** | product search with structured filters, concept ("semantic") search, comparison, similarity, recommendations, specifications, reviews, stock |
| **Purchase** | cart add/remove/update, coupon validation and application, price quoting, checkout |
| **Post-purchase** | order tracking, cancellation, return eligibility and creation, refund estimates |
| **Support** | policy knowledge base, shipping options, wishlist |
| **Personalisation** | preference learning ("remember I like black shoes"), conversation memory, follow-up resolution |

The dataset is 94 products across 10 departments, 43 brands, 42 categories, 12
orders, 8 coupons, 185 reviews and 20 policy entries — all realistic, all in
`data/*.json`.

---

## Architecture

Dependencies point downward only. Nothing in a lower layer knows about a higher one.

```
                        ┌──────────────┐
   HTTP ───────────────►│     api      │  FastAPI: validate, delegate, serialise
                        └──────┬───────┘
                               ▼
                        ┌──────────────┐
                        │    graph     │  LangGraph: the ONLY orchestrator
                        └──────┬───────┘
                               ▼
                        ┌──────────────┐      ┌──────────┐
                        │    agents    │◄────►│  memory  │
                        └──────┬───────┘      └──────────┘
                               ▼
                        ┌──────────────┐
                        │    tools     │  allow-list, validation, timeout, audit
                        └──────┬───────┘
                               ▼
                        ┌──────────────┐
                        │   services   │  commerce rules
                        └──────┬───────┘
                               ▼
                        ┌──────────────┐
                        │ repositories │  protocols + JSON implementations
                        └──────┬───────┘
                               ▼
                            data/*.json

  config · schemas · models · utils · observability  — cross-cutting
```

Four rules hold throughout, and most of the design follows from them:

1. **Agents never call agents.** Control flow lives in the graph's edges, so the
   system's behaviour is a property of the topology rather than an emergent
   consequence of agents invoking each other.
2. **Agents reach the world only through tools.** No agent holds a repository or a
   service. Every fact in a reply traces to a recorded tool call.
3. **Only structured objects cross boundaries.** Pydantic contracts everywhere, so a
   mis-shaped hand-off fails at the seam that produced it.
4. **Nothing irreversible happens without explicit confirmation** of that specific
   action.

Full detail: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## The workflow

```
      user message
           │
           ▼
      ingest → memory_load → planner
                                │
             ┌──────────────────┼──────────────────┐
             │ needs            │ has tasks        │ nothing to do
             │ clarification    ▼                  │
             │              router                 │
             │                 │ fan out           │
             │        ┌────────┴────────┐          │
             │        ▼   ▼   ▼   ▼   ▼ ▼          │
             │      12 worker nodes, in parallel   │
             │        └────────┬────────┘          │
             │                 ▼                   │
             │               join                  │
             │                 │                   │
             │                 ▼                   │
             └───────────► validation ◄────────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼ clarify             ▼ approve / revise / reject
              clarification            response
                    │                     │
                    └──────────┬──────────┘
                               ▼
                         memory_write → reply
```

Parallelism comes from a conditional edge returning a *list* of node names; all
twelve workers converge on `join`, which runs once. The graph is acyclic —
response regeneration on a grounding violation happens inside the response agent
under a retry budget, not as a graph cycle.

Full detail, including why `memory_write` follows response generation:
[`docs/WORKFLOW.md`](docs/WORKFLOW.md).

---

## The agents

| Agent | Responsibility |
|---|---|
| **planner** | Decomposes a message into tasks; extracts price bounds, coupons, ordinals |
| **router** | Assigns tasks to nodes; chooses parallel vs sequential |
| **memory** | Loads the turn's context; commits its outcome |
| **product_search** | Structured and concept search, details, specifications, reviews |
| **comparison** | Side-by-side comparison; names the trade-off per axis |
| **recommendation** | Similarity, taste-vector personalisation, complements |
| **inventory** | Available-to-promise stock, restock estimates |
| **pricing** | Totals, GST, shipping, coupon adjudication with specific reasons |
| **cart** | Add, remove, update; resolves "the charger" and "the first one" |
| **checkout** | Places orders. Previews by default |
| **order** | Tracking and cancellation |
| **returns** | Eligibility, windows, refund estimates, creation |
| **knowledge** | Policy answers with calibrated confidence |
| **wishlist** | Saved-for-later management |
| **preference** | Learns and forgets stated preferences |
| **validation** | Decides what the reply may assert |
| **clarification** | Asks the one question that unblocks the request |
| **response** | Writes the reply, from grounded facts only |

Every agent declares a full prompt contract — role, responsibilities, guardrails,
inputs, tools, output schema, examples, failure handling — as a typed
`PromptSpec`. Construction *fails* if any mandatory part is missing, and
`GET /agents` serves the manifest. See [`docs/AGENTS.md`](docs/AGENTS.md).

---

## How hallucination is prevented

Structurally, not by asking a model to behave:

1. **Tools write the facts.** Each tool returns a natural-language `summary`
   composed by the component that holds the data.
2. **Validation decides what is assertable.** Deterministic checks — money
   reconciled to the paisa, confirmation present for irreversible actions, every
   claim traceable to a recorded tool call, contradictions between workers —
   produce a `grounded_facts` list. A model pass can only make the verdict
   *stricter*, never looser.
3. **The response agent may use only those facts**, and its output is verified
   afterwards: any money amount or identifier not present in the facts causes the
   reply to be regenerated, then discarded in favour of a template.

```python
# tests/unit/test_agents_and_memory.py
assert detect("It costs ₹74,990.", facts) == []       # supported
assert detect("It costs ₹64,990.", facts)             # invented → rejected
assert detect("Order ORD-99999999-9999 shipped.", facts)  # invented → rejected
```

---

## Degraded mode

Without a Groq key — or while the provider's circuit breaker is open — the system
keeps working:

| | With a key | Without |
|---|---|---|
| Planning | Groq (reasoning tier) | Rule-based intent detection |
| Routing | Deterministic, model-assisted for complex plans | Deterministic |
| Workers | Deterministic | Deterministic (unchanged) |
| Validation | Deterministic + model pass | Deterministic |
| Reply | Generated, then grounding-verified | Templated from grounded facts |
| Tools | All 32 | All 32 |

The graph shape is identical. Degraded mode is a supported operating state, which
is also what makes the whole system testable without a network — 423 tests run
with no API key.

---

## Observability

Every turn produces a trace: `decision_path`, per-node and per-agent durations,
tool latency, model latency, token counts, retries, failures and warnings.

```bash
curl -s localhost:8000/metrics | jq          # aggregates, latency percentiles
curl -s localhost:8000/metrics/turns | jq    # recent turns with decision paths
curl -s localhost:8000/agents | jq           # prompt contracts + granted tools
curl -s localhost:8000/tools | jq            # 32 tools, schemas, mutation flags
curl -s localhost:8000/graph | jq            # topology + Mermaid diagram
```

Request a trace inline with `{"include_trace": true}`:

```json
{
  "decision_path": ["planner","router","product_search","join","validation","response"],
  "tools_called": ["search_products"],
  "validation_verdict": "approve",
  "total_duration_ms": 52.1,
  "tokens_total": 0,
  "degraded": true
}
```

Logs are structured JSON with a `trace_id` on every line, so one grep returns the
HTTP request, each node, each tool call and each model call for a turn.

---

## API

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/chat` | Conversational turn (runs the full graph) |
| `GET` | `/chat/sessions/{id}` | Inspect a conversation |
| `DELETE` | `/chat/sessions/{id}` | Forget a conversation |
| `GET` | `/products/search` | Filtered catalogue search |
| `GET` | `/products/{id}` | Product detail, stock, reviews, similar |
| `GET` | `/cart/{user_id}` | Cart + live price breakdown |
| `POST` | `/cart/add` · `/cart/remove` · `/cart/update` · `/cart/coupon` | Cart mutations |
| `POST` | `/checkout` | Preview (default) or place an order |
| `GET` | `/orders` · `/orders/{id}` | Order list and detail |
| `POST` | `/orders/{id}/cancel` | Cancel (requires `confirm`) |
| `GET` | `/returns` · `/returns/eligibility` | Returns and eligibility |
| `POST` | `/returns` | Raise a return |
| `GET` | `/health` · `/health/ready` · `/metrics` · `/agents` · `/tools` · `/graph` | Operations |

`/chat` runs the agent graph. The rest act directly through services — a client
posting a product id has already done the interpreting.

Interactive docs at `/docs`.

---

## Testing

```bash
make test              # 423 tests
make test-unit         # fast, isolated
make test-integration  # graph + API
make coverage
make check             # lint + typecheck + test
```

No test touches the network. Each test gets a freshly *generated* dataset in a
temp directory, so the suite is immune to whatever state `data/` is in — which
matters, because running the server mutates carts, orders, stock and coupon
counters.

The model paths are covered by driving agents with a scripted stub, including the
cases that matter most: a drifting reply being rejected, an incomplete routing
being discarded, and a compliant model being unable to loosen a validation verdict.

---

## Configuration

Every setting is typed and validated in `assistant.config.settings`; nothing reads
`os.environ` directly. `GROQ_API_KEY` is the only required variable, and even that
is optional. See [`.env.example`](.env.example) for all of them and
[`docs/SETUP.md`](docs/SETUP.md) for the rationale.

Production configurations are checked at startup: `debug`, prompt capture and
wildcard CORS are all rejected when `ASSISTANT_ENVIRONMENT=production`.

---

## Documentation

| | |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Layers, boundaries, and the reasoning behind them |
| [`docs/WORKFLOW.md`](docs/WORKFLOW.md) | Graph topology, state, reducers, parallelism, diagrams |
| [`docs/AGENTS.md`](docs/AGENTS.md) | Every agent, its contract, and how they communicate |
| [`docs/SETUP.md`](docs/SETUP.md) | Installation, configuration, running, troubleshooting |
| [`docs/EXTENDING.md`](docs/EXTENDING.md) | Adding an agent, a tool, or a capability |
| [`docs/MIGRATION_POSTGRES.md`](docs/MIGRATION_POSTGRES.md) | Replacing JSON with PostgreSQL |

---

## Stack

Python 3.12 · LangGraph 1.2 · langchain-groq · FastAPI · Pydantic v2 · uv · pytest ·
Docker

No OpenAI, Anthropic, Gemini or Ollama. No PostgreSQL, MongoDB or any external
database — the catalogue is static JSON behind a repository layer designed to be
swapped.


Built an agentic code review platform where an 8-stage multi-agent LangGraph pipeline profiles a repository (from a Git URL, local folder or ZIP), plans the review, audits files and cross-cutting issues, and produces a scored report that streams live to a React dashboard over SSE. To cut AI false positives, a critic agent and a proof engine check every high-severity finding, either by tracing it in code or by running it in a locked-down Docker sandbox; a rotating API-key pool, a per-key rate limiter and two-tier model routing let it run within free-tier LLM limits, and the system ships with 160+ automated tests.
