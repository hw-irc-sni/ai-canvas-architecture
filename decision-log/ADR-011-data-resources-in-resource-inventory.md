# ADR-011 — Data Resources in Resource Inventory

Date: 2026-11-22

# Status

Proposed

Extends ADR-003. Answers the data discovery question of ADR-007.

# Context

ADR-003 adopts TMF Resource Inventory as the unified discovery registry of the AI-Native Canvas. Resource Inventory is a stateless projection of Canvas CRDs. Its supported categories are `ODAComponent`, `API`, `Agent`, `Tool` and `Model`. Its relationship types are `exposes`, `exposedBy` and `dependsOn`. Under Future Considerations it lists "representation of data resources within the inventory model" as open.

ADR-007 asks the same question from the data side: should data resources be discoverable through Resource Inventory, external catalogs, or federated discovery? It notes that "data resources are not yet represented as Resource Inventory resources".

ADR-010 refines `DataResourceConfig`. A provider component declares the stores it serves in `spec.providedDataResources[]`. The Data Resource Management operator mirrors the knowledge-bases and ontology modules inside those stores into `status.providedResources[]`. Consumers declare their dependencies in `spec.dependentDataResources[]`. All of this is Kubernetes state, so it can be projected the same way as the other Canvas resources.

The ODA Canvas reference Resource Inventory currently projects `ODAComponent` and `API` (from Component and ExposedAPI). It copies every `componentMetadata` property of a Component as a characteristic. The component's catalog identifier (for example `TMFC060`) therefore already appears, under the name `id`, which is easily confused with the resource's own id.

Clients need to discover data resources: the Canvas Portal (which TMFC060 instances exist, and where), agents (which knowledge-bases exist about a topic), and operators.


# Decision

**Data resources are projected into Resource Inventory as a new resource category, `DataResource`, derived from `DataResourceConfig`.** The projection follows the ADR-003 model: Kubernetes stays the source of truth, Resource Inventory stores nothing, and an entry disappears when its source disappears.

## Two levels of data resource

A provider DataResourceConfig produces entries at two levels:

| Level | Source | One entry per | Characteristics |
|---|---|---|---|
| **Store** | `spec.providedDataResources[]` | Declared store, for example a vector store or a knowledge graph | `type` (`vectorStore`, `knowledgeGraph`, `dataProduct`, `cache`), `engine`, `namespace`, `dataResourceConfig` |
| **Content** | `status.providedResources[]` | Knowledge-base or ontology module served by the provider | `kind` (`knowledge` or `knowledgeSpecification`), `storeKind` (knowledge-bases), `layer` (modules), `lifecycleStatus`, `namespace`, `href` (the item's URL in the provider API) |

Store entries describe *what infrastructure* a component provides. Content entries describe *what knowledge* it holds. Agents and the portal can then discover a specific knowledge-base without calling every provider.

A Databricks DataResourceConfig (ADR-010, `provider: databricks`) produces one store-level entry per `dependentDataResources[]` item. Its `type` is taken from the item, or inferred from `accessType` as described in ADR-010.

## Identifiers

Entry ids are deterministic and derived from the source, so they are stable across queries:
- store: `<dataresourceconfig name>.<item name>`
- knowledge-base: `<dataresourceconfig name>.knowledge.<id>`
- ontology module: `<dataresourceconfig name>.knowledgeSpecification.<id>`

`resourceSpecification.id` is `DataResource`. `resourceStatus` and `operationalState` follow the DataResourceConfig `Ready` condition for store entries, and the item's `lifecycleStatus` for content entries.

## Relationship types

The relationship types of ADR-003 are extended:

| Relationship | From → To | Inverse |
|---|---|---|
| `providedBy` ★ | DataResource → ODAComponent that serves it | `provides` ★ on the ODAComponent |
| `isPartOf` ★ | Knowledge-base or module → store entry that holds it | `contains` ★ on the store |
| `dependsOn` | ODAComponent → DataResource it is bound to | none (as in ADR-003) |

★ marks the new types. A knowledge-base with `storeKind: hybrid` is `isPartOf` both the vector store and the knowledge graph. An ontology module is `isPartOf` the knowledge graph. `dependsOn` comes from the consumer DataResourceConfig's `status.bindings[]`. It points at the content entry of the bound knowledge-base, or at the store entry when no knowledge-base is bound.

## Component identifier

ODAComponent entries gain an explicit **`componentId`** characteristic, taken from `componentMetadata.id`, for example `TMFC060`. The existing `id` characteristic is kept for compatibility. Clients find all instances of a component type with a single filter, for example "ODAComponents with `componentId` TMFC060", and then follow `provides` to their data resources.

## Registry views

A **Data Registry** becomes a logical view of Resource Inventory, filtered by `category = DataResource`, alongside the Agent, Tool and Model Registry views of ADR-003. No separate registry service is introduced.

The same projection is applied wherever the Canvas offers inventory views: the Resource Inventory API, the Canvas Component Registry, and the MCP interface over Resource Inventory (category filter and tool descriptions), so that agents can discover data resources as tools.

## Example

A knowledge-base entry served by a TMFC060 instance:

```json
{
  "id": "km-knowledgemanagement.knowledge.incident-knowledge",
  "href": "/tmf-api/resourceInventoryManagement/v5/resource/km-knowledgemanagement.knowledge.incident-knowledge",
  "@type": "LogicalResource",
  "@baseType": "Resource",
  "name": "incident-knowledge",
  "category": "DataResource",
  "resourceSpecification": {"@type": "ResourceSpecificationRef", "id": "DataResource", "name": "knowledge"},
  "resourceCharacteristic": [
    {"@type": "Characteristic", "name": "kind", "value": "knowledge"},
    {"@type": "Characteristic", "name": "storeKind", "value": "hybrid"},
    {"@type": "Characteristic", "name": "lifecycleStatus", "value": "active"},
    {"@type": "Characteristic", "name": "namespace", "value": "components"},
    {"@type": "Characteristic", "name": "href", "value": "https://<gateway>/km-knowledgemanagement/tmf-api/knowledgeManagement/v5/knowledge/incident-knowledge"}
  ],
  "resourceRelationship": [
    {"@type": "ResourceRelationship", "relationshipType": "providedBy",
     "resource": {"@type": "ResourceRef", "id": "km-knowledgemanagement"}},
    {"@type": "ResourceRelationship", "relationshipType": "isPartOf",
     "resource": {"@type": "ResourceRef", "id": "km-knowledgemanagement.knowledge-vectors"}},
    {"@type": "ResourceRelationship", "relationshipType": "isPartOf",
     "resource": {"@type": "ResourceRef", "id": "km-knowledgemanagement.knowledge-graph"}}
  ]
}
```

Example discovery queries:
- all data resources: `GET /resource?category=DataResource`
- all TMFC060 instances: `GET /resource?category=ODAComponent`, then filter on the `componentId` characteristic


# Consequences

## Benefits

Answers the ADR-003 open question and the ADR-007 discovery question with the same projection model as the other Canvas resources.

Data resources, knowledge-bases and ontology modules become discoverable through a standard TM Forum API and through MCP, without a separate registry.

Consumers, providers and their dependencies form one navigable graph: component → data resource → content.

## Trade-offs

Content entries can be numerous. Clients should filter by category and characteristic, and Resource Inventory may need paging for large deployments.

Content entries are only as fresh as the operator's mirror in `status.providedResources[]`. Resource Inventory does not call provider components.

Resource Inventory and Component Registry need read access to `dataresourceconfigs`.

The new relationship types and characteristics are Canvas conventions. They are not yet part of a TM Forum resource specification.

## Future Considerations

- a TM Forum `ResourceSpecification` for `DataResource`, aligned with the outcome of the SID Knowledge ABE discussion
- federation with external data catalogs (for example Unity Catalog), so that data product entries link to their catalog record
