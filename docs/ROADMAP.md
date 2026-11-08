# Roadmap

Plan **v3 (2026-11-08)**. Status: `[ ]` not started · `[~]` in progress · `[x]` done.
Bump the version and add a dated history line whenever the plan changes.

## Plan history
- **v1 (2026-11-08):** five separate projects (observability, caching, guardrails, RAG, multi-agent capstone).
- **v2 (2026-11-08):** one agent on Foundry, extended layer by layer.
- **v3 (2026-11-08):** one *platform* with a single entry point and pluggable modules. Each module is built separately so the work stays varied. Cross-cutting concerns are platform layers. This gives full-stack and iterative development experience.

## Architecture
```
Front end (Streamlit → React) ──► Gateway / BFF (FastAPI, Entra auth, Container Apps)
                                    │  baseline guardrails · caching · OTel tracing
                                    ▼
                              Module registry (config)
                           ┌────────┼──────────┐
                        Agent A   Agent B    Function C
                        (+ MCP)   (RAG)      (MCP tool)
                  Foundry Agent Service  /  MCP servers on Container Apps
```

**Request flow:**
1. The gateway authenticates the request, runs the baseline checks and starts a trace.
2. It looks up the module in the registry to get the Foundry agent id.
3. Foundry runs the agent loop and calls our MCP server over HTTPS when the model wants a tool.
4. The final answer comes back to the gateway.
5. The gateway runs the output checks, ends the trace and returns the response.

**Module contract (proposed):**
- Each module is a Foundry agent, with its own MCP server if it needs custom tools.
- Each module has one registry entry: `id, name, description, type, endpoint/agent_id, guardrail_policy`.
- Adding a module means building it and adding a registry entry. The gateway doesn't change.

**Routing:** explicit (the user picks a module) in the gateway first. In P5, a Foundry router agent picks the module, using connected agents or workflows.

**Where things run:**

| Foundry | Our code |
|---|---|
| Model deployments | FastAPI gateway (Container Apps) |
| Prompt agents (Foundry runs the loop and threads) | FastMCP tool servers (Container Apps) |
| Hosted agents (containerized code agents, for complex cases) | Front end |
| Built-in tools: AI Search, code interpreter, grounding, OpenAPI | Deploy scripts, CI |
| Connected agents / workflows (router) | Module-level checks (tool permissions, confirmation before actions) |
| Content filters, evals, red-teaming agent, tracing view | OTel instrumentation of our own code |

## Guardrails: layered
1. **Gateway baseline** (same for every module): Prompt Shields on user input, content safety, PII on the way in, auth, rate limits.
2. **Per-module policy** (declared in the registry entry, enforced by the gateway): for example groundedness on for RAG, the PII mode, an output schema.
3. **Inside the module** (where it uses its tools): indirect-injection checks on tool and document output, tool allowlists, confirmation before side-effectful actions, output validation.

The gateway can't see inside the agent's run, which is why layer 3 is needed. Measure false-positive and false-negative rates for each module.

## Builds
Each build is timeboxed to 3–4h.

### Platform
- [~] **P1 Skeleton** (started 2026-11-08)
  - Foundry project, model deployment and App Insights via Bicep/azd.
  - Gateway with the registry.
  - The first module (M1), with one MCP tool.
  - End-to-end tracing (`azure-monitor-opentelemetry`, OpenAI instrumentor, `gen_ai.*` spans).
  - KQL queries (p50/p95 latency, tokens/request, error rate, cost/day), one alert.
  - Baseline evals (15–20 items).
  - Streamlit UI; README.
  - *Scope warning:* P1 + M1 is more than 4h. Aim for one thin working path end to end first.
- [ ] **P2 Front end:** React chat UI, module picker, feedback linked to trace ids, Entra sign-in.
- [ ] **P3 Caching:**
  - APIM AI gateway in front of the model: semantic cache, token limits, token metrics.
  - Caching of tool results in the MCP servers.
  - Measure hit rate, savings and false hits; scope the cache per user.
- [ ] **P4 Guardrails:** the three layers above, plus a red-team suite (PyRIT / Foundry AI Red Teaming Agent).
- [ ] **P5 Router / multi-agent:** router agent over the modules, per-agent tracing, evals as a CI regression gate.

### Modules (pick and order as we go)
- [ ] **M1 First module:** our own FastMCP server + Foundry agent. Use case TBD; suggested is a *career copilot* (track job applications, compare job descriptions against skills, later mock interviews and take-home review).
- [ ] **M2 Knowledge / RAG module** (weak area):
  - Azure AI Search hybrid search + semantic ranker.
  - Compare two or three chunking strategies.
  - Retrieval eval (recall@k, MRR) kept separate from the generation eval.
- [ ] More ideas: ticket/inbox triage with confidence scores, learning copilot, doc Q&A with citations.

## Cross-cutting
- Run evals on every build so regressions show up as layers are added.
- Cost: use the cheapest tiers and run `azd down` when idle. APIM, AI Search and Container Apps cost money while running.
- Take-home practice stays separate:
  - [ ] Build a reusable starter template.
  - Timed practice briefs from scratch:
    - doc Q&A API with citations
    - ticket-triage agent with confidence scores
    - add caching + rate limiting to an LLM service and show the savings
