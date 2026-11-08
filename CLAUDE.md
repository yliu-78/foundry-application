# foundry-application — context for Claude

## Who and why
- The owner is an AI engineer who is upskilling.
  - Strong: agentic and multi-agent systems, coding.
  - Weaker: RAG, Azure/Foundry SDKs, deployment.
- Goals:
  - Hands-on skills with Microsoft Foundry, monitoring, caching and guardrails.
  - Full-stack experience.
  - Readiness for 3–4h timed take-home tests.
- Explain the Azure, SDK and deployment steps; don't over-explain the Python itself.

## What this repo is
One **platform hosted on Microsoft Foundry**. It has a single entry point (gateway + front end) and loosely coupled **modules** (agents or functions). Each module is a separate 3–4h build that plugs in. Platform builds add cross-cutting layers: monitoring, caching, guardrails, routing.

Full plan, build list and status: @docs/ROADMAP.md

## Core principles
- **Foundry-first, kept as code.** Foundry is the runtime; this repo is the source of truth.
  - Agents are defined in `agents/<module>/` (agent.yaml + instructions.md) and pushed to Foundry by a script using `azure-ai-projects`, later run by GitHub Actions.
  - The portal is only for experiments and inspection. Anything built there must be copied back into the repo.
- **Keep Python thin.** Foundry runs:
  - the agent loop
  - built-in tools (AI Search, code interpreter, grounding)
  - evals and tracing
  - content filters

  Our code covers only:
  - the FastAPI gateway
  - custom tools (FastMCP servers)
  - the front end (Streamlit first, React later)
  - deploy scripts and CI
- **No VMs.** Gateway and MCP servers run on Azure Container Apps. Infra is in Bicep and deployed with `azd up` / `azd down` (tear down when idle to save cost).
- **Auth:** `DefaultAzureCredential` everywhere. Managed identity gets the "Azure AI User" role on the Foundry project. No API keys in code.
- A Foundry agent **can't reach localhost**. For dev, use a dev tunnel or deploy the MCP server early.
- The front end talks only to the gateway, never to Foundry directly.

## Intended layout
```
infra/          Bicep + azure.yaml (azd)
agents/<mod>/   agent.yaml, instructions.md  → deployed to Foundry
mcp_servers/    FastMCP tool servers → Container Apps
gateway/        FastAPI: module registry, routing, baseline guardrails, OTel
frontend/       Streamlit (P1/P2), React later
evals/          datasets + eval scripts (azure-ai-evaluation), results logged to Foundry
.github/workflows/
```

## Caveat
Foundry SDKs and feature names change quickly. This covers hosted agents, connected agents and workflows, and the major version of `azure-ai-projects`. Check the current Microsoft docs and pin package versions before writing code against them.
