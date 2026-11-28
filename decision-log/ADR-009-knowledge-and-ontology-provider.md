# ADR-009 — TMFC060 Knowledge Management as the Canvas Knowledge and Ontology Provider

Date: 2026-11-22

# Status

Proposed

# Context

Agentic components need shared knowledge: documents they can search semantically, a graph of facts they can reason over, and a common vocabulary that gives both a meaning. In the AI-Native Canvas this capability does not exist yet:
- no Canvas repository contains retrieval-augmented generation (RAG), vector or embedding code
- `DataResourceConfig` (ADR-005, ADR-007) declares data dependencies on external platforms, but nothing in the Canvas provides knowledge itself
- Resource Inventory (ADR-003) projects only the `ODAComponent` and `API` categories, so knowledge stores cannot be discovered
- the Canvas Portal has a Knowledge page with two empty sections, "Knowledge-Base" and "Federated Ontology"

Two prototypes address parts of the problem:
- **FTO** (Federated Telecom Ontology) organises ontology modules in three layers (Foundational, Telecom, Implementation), resolves `owl:imports` through a catalog, and links modules with reified bridge mappings that carry an `agreed`/`proposed` status. It is a read-only MCP server over stdio.
- **c-kg** is a writable instance graph with MCP-only access. It has useful tool designs (entity, subgraph and path lookups) but no REST API, no governance and no vector search.

The ODA Component Directory already reserves **TMFC060 Knowledge Management**. Its inception form defines six functions, KM-FX-01 to KM-FX-06: data processing, information extraction, knowledge generation, knowledge fusion, knowledge storage and knowledge recommendation. Its draft specification has a placeholder `TMFxxx` v5 API with two resources: `knowledge` and `knowledgeSpecification`.

An architectural decision is needed on where knowledge and ontologies live in the Canvas, who governs them, and how agents reach them.


# Decision

**TMFC060 Knowledge Management is the Canvas provider of knowledge-bases and ontology modules.** It is deployed as an ordinary ODA Component (ADR-001), not as a Canvas platform service.

## Resources

TMFC060 exposes the TMFxxx Knowledge Management v5 API, following TMF630 v5 conventions, with two resources:
- `knowledge`: a knowledge-base. It conforms to one or more ontology modules and has graph, query, search, recommend, document and ingest sub-resources.
- `knowledgeSpecification`: an ontology module. It has content, mapping, review, validation and term sub-resources.

This resolves the specification's open decision D-01: `knowledgeSpecification` governs ontology modules.

## Hybrid knowledge-bases

A knowledge-base is **hybrid**: an RDF instance graph and a document and vector store, linked so that search hits carry the ontology IRIs of the entities they mention (graph-RAG). Each knowledge-base has its own named graph and its own vector collection. The ingestion pipeline implements KM-FX-01 to KM-FX-06 and runs as TMF701 processes.

## Federated ontology governance

Ontology modules and their mappings are governed by a **propose → review → rule** workflow with a shared lifecycle:

`draft → proposed → inReview → agreed | rejected → published → deprecated`, where `rejected` returns to `draft` for revision.

Each transition belongs to one component role: `ontology-editor` proposes, `ontology-reviewer` reviews and comments, `ontology-approver` rules, publishes and deprecates. An item can be ruled `agreed` only if validation passes. Machine-generated changes, whether from the ingestion pipeline or from MCP clients, always enter the workflow as proposals and are never applied directly. Rulings are recorded with approver and date, and written back to the module's RDF as FTO's `bridge:status`, so modules stay compatible with FTO.

## Federation

Federation works at two levels:
- **Module federation.** Modules keep FTO's layers and catalog-based `owl:imports`. They are loaded from an upload, a git repository or a mounted volume. Other ODA Components can contribute modules, which enter the same governance workflow.
- **Query federation.** A remote SPARQL endpoint or a peer TMFC060 instance is modelled as a `knowledgeSpecification` with `specKind: remote`. Queries reach it through SPARQL `SERVICE`, to allowlisted endpoints only.

## Interaction interfaces

Following ADR-002, the Component exposes:
- `knowledgemanagement` (`openapi`, TMFxxx)
- `processflow` (`openapi`, TMF701)
- `knowledgemcp` (`mcp`)

The MCP server wraps the REST API with the caller's token, so an agent has exactly the rights of its role. Every call is authorised in the component against its declared component roles: `knowledge-viewer`, `knowledge-editor`, `ontology-editor`, `ontology-reviewer`, `ontology-approver`, plus `canvasSystemRole`.

## Content

**The component image and chart ship no ontology content.** Ontologies are deployment data, supplied by the operator of the Canvas. This keeps the reference implementation free of licensing constraints on any particular ontology.


# Alternatives Considered

**Fork the FTO prototype.** FTO has the module, layer and bridge model this ADR adopts. But it is a read-only, stdio-only server with no API, no persistence model for knowledge-bases and no governance workflow. Its model is reused; its code is not.

**Fork the c-kg prototype.** c-kg is writable but MCP-only, with no REST API, no vector search and no governance. Its tool designs are reused by the TMFC060 MCP server; its code is not.

**Vector-only RAG.** A document and vector store alone is simpler, but it cannot answer structural questions, cannot be validated against an ontology, and gives no shared vocabulary across components. The hybrid model keeps vector search and adds these.

**A Canvas platform service instead of a component.** Knowledge could be a Canvas service, like Resource Inventory. But TMFC060 is already defined as a business component in the ODA Component Directory. Deploying it as a component lets several independently governed instances coexist, and lets it use the standard component machinery: exposed APIs, roles, events and dependent APIs. Canvas support is limited to discovery and binding, see ADR-010 and ADR-011.


# Consequences

## Benefits

Provides a reference implementation for a component that exists in the ODA Component Directory only as a specification.

Gives agentic components a governed, shared vocabulary and searchable knowledge through standard interaction interfaces (ADR-002).

Keeps ontology governance auditable: every change is a proposal with a recorded ruling.

Remains compatible with existing FTO module repositories.

## Trade-offs

The TMFxxx API is a draft. Its resource model may change when TM Forum assigns a real API number (specification open decision D-03).

The component introduces stateful stores (graph, vector, relational) that must be operated, backed up and sized.

The ingestion pipeline depends on an LLM and an embedding model, which the component resolves through the Canvas model access path (ADR-006) when available.

## Follow-up Decisions

- **ADR-010** refines `DataResourceConfig` so that TMFC060 can declare the stores it provides, and so that consuming components can bind to its knowledge-bases.
- **ADR-011** projects data resources into Resource Inventory, so that TMFC060 instances and their knowledge-bases are discoverable.

## Open Items

These standards-side questions are parked and do not block the reference implementation: the SID Knowledge ABE (D-02), the real API number (D-03), functional framework ownership (D-04, D-05), the Capability Catalog boundary (D-07) and the inventory index (D-09).
