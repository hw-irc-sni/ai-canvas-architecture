# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Nature of the repo

Documentation-only repository: the source of truth for the **AI-Native ODA Canvas** architecture (TM Forum Open Digital Architecture Canvas extended for agentic/AI workloads). There is no code, build, lint, or test tooling. Work consists of authoring and reviewing Markdown ADRs and diagrams.

## Layout

- `decision-log/` – Architecture Decision Records (`ADR-NNN-<slug>.md`) plus `decision-log/README.md`, which holds the ADR index table and the ADR lifecycle (Proposed → Accepted → Superseded/Deprecated).
- `diagrams/` – PNG flow diagrams (agent→LLM, agent→MCP, agent→agent, component/model install) referenced by the ADRs.
- `README.md` – architecture vision and purpose.

## ADR conventions

- Structure: title, `Date`, `# Status`, `# Context`, `# Decision`, `# Consequences`. New ADRs use the next sequential number and are proposed via pull request.
- When adding or renaming an ADR, update the index table in `decision-log/README.md`.
- Known inconsistencies to be aware of (don't assume they're fixed):
  - Two files share the number **ADR-008** (`ADR-008-Agent-Skills.md` and `ADR-008-proxy-pattern-external-integration.md`); only the proxy one is in the index. The next free number is ADR-009.
  - The index's "Doc reference" filenames (e.g. `ADR-001-agentic-components.md`) don't match the actual filenames in `decision-log/` (e.g. `ADR-001-agentic-components-deployment-model.md`). Link using the real filenames.
  - `ADR-008-Agent-Skills.md` omits the `#` heading style and the `Date`/title format used by other ADRs.

## Architecture big picture (spans ADR-001 … 008)

- **Agentic components** are the primary deployable unit (ADR-001); they declare the resources they need (models, agents via A2A, tools via MCP, APIs, data platforms) declaratively as Kubernetes CRDs handled by Canvas operators.
- **Interaction interfaces** are OpenAPI, A2A, and MCP (ADR-002).
- **Resource Inventory** (TMF Resource Inventory API) is the unified discovery registry exposing Canvas resources externally, while Kubernetes remains the authoritative config layer (ADR-003).
- **Gateway mediation**: an AI Gateway (models, e.g. LiteLLM) plus an Interaction Gateway (tools/agents, e.g. MuleSoft/Istio) mediate all runtime traffic (ADR-004), configured through AI-native config resources such as `AIGatewayConfig` and `DataResourceConfig` (ADR-005).
- Model onboarding (ADR-006), data resource access (ADR-007), the proxy pattern for external agents/tools (ADR-008), and Agent Skills as declaratively bound config (ADR-008-Agent-Skills) build on those foundations.

When editing an ADR, check the others for cross-references and keep terminology (Canvas, Interaction Gateway, AI Gateway, Resource Inventory) consistent.
