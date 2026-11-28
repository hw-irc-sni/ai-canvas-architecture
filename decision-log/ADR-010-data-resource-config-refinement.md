# ADR-010 — DataResourceConfig Refinement: Data Resource Types, Provider and Consumer Roles, and Provisioning

Date: 2026-11-22

# Status

Proposed

Refines ADR-005 and ADR-007. Extends the draft CRD in `CRDS/dataresourceconfig-v1alpha1-crd.yaml`.

# Context

ADR-005 introduced `DataResourceConfig` as the declarative resource for the data dependencies of agentic components. ADR-007 described how Canvas integrates with external data platforms. It left four questions open: runtime access standardisation, gateway mediation, the data discovery model, and a data product abstraction.

The draft CRD in `CRDS/dataresourceconfig-v1alpha1-crd.yaml` (`dataresourceconfigs.oda.tmforum.org`, `v1alpha1`) implements one case of ADR-007:
- `spec.provider` is an enum with the single value `databricks`.
- `spec.workspace` and `spec.dependentDataResources[]` are required. Each item requires `name` and a Databricks-specific `accessType` (`sql`, `vector-search`, `genie`, `functions`).
- Its operator creates one DependentAPI per item, pointing at a Databricks Managed MCP endpoint, and reports `status.phase`, `message`, `resourceCount` and `endpoints[]`.
- The operator sets `phase: Skipped` on any DataResourceConfig whose provider is not `databricks`, "allowing future operators to handle other platforms".

ADR-009 makes TMFC060 Knowledge Management the Canvas provider of knowledge-bases. This introduces a case the draft cannot express: **a data resource served by an ODA Component inside the Canvas**. It raises three needs:

1. **A provider side.** TMFC060 must declare the stores it serves (a vector store and a knowledge graph), so that the Canvas can discover them. The draft has only a consumer side: `dependentDataResources` is required with `minItems: 1`.
2. **Provisioning.** A consuming component should be able to ask for a knowledge-base and receive it ready to use, with endpoints and access rights, without manual steps.
3. **A platform-neutral type.** `accessType` describes Databricks MCP servers. The Component v2alpha1 proposal (`dataFunction.dependentDataResources[]`) already uses a platform-neutral `type` enum, `vectorStore | knowledgeGraph | dataProduct | cache`, with a `retrieval` block for vector stores.

The same resource name and version cannot carry two incompatible schemas in one cluster. The refinement must therefore extend the draft CRD in a backward-compatible way: every Databricks DataResourceConfig valid today must stay valid.


# Decision

`DataResourceConfig` stays a **standalone, namespaced** CRD, `oda.tmforum.org/v1alpha1`. It is linked to its component by the `oda.tmforum.org/componentName` label or by an ownerRef, so the Component CRD does not change. The draft schema is extended with the changes below. Everything else is unchanged.

## 1. Provider value `odaComponent`

`spec.provider` gains the value **`odaComponent`**: the data resource is served by an ODA Component in the Canvas. The serving component is named per resource, in `providerRef`. The value is generic: TMFC060 is the first such provider, but not the only one possible.

## 2. Role

A new field `spec.role` takes `consumer` (the default, matching every existing CR) or `provider`.
- **consumer**: the CR lists the data resources the owning component depends on, in `dependentDataResources[]`, as today.
- **provider**: the CR lists the data resources the owning component serves, in a new array `providedDataResources[]`. It is normally rendered by the provider component's own Helm chart.

## 3. Loosened required fields, restored by validation rules

`spec.required` becomes `[provider]`, and the item-level `required` becomes `[name]`. The previous constraints are restored as conditional CEL rules (`x-kubernetes-validations` on `spec`), so Databricks CRs are validated exactly as before:

| Rule | CEL |
|---|---|
| Databricks needs a workspace | `self.provider != 'databricks' \|\| has(self.workspace)` |
| Databricks items need an access type | `self.provider != 'databricks' \|\| !has(self.dependentDataResources) \|\| self.dependentDataResources.all(r, has(r.accessType))` |
| A consumer lists dependencies | `(has(self.role) && self.role == 'provider') \|\| (has(self.dependentDataResources) && size(self.dependentDataResources) > 0)` |
| A provider lists provided resources | `!has(self.role) \|\| self.role != 'provider' \|\| (has(self.providedDataResources) && size(self.providedDataResources) > 0)` |
| A component-served dependency names its provider | `self.provider != 'odaComponent' \|\| !has(self.dependentDataResources) \|\| self.dependentDataResources.all(r, has(r.providerRef))` |

## 4. Data resource type

Each item gains **`type`**: `vectorStore | knowledgeGraph | dataProduct | cache`, taken from the Component v2alpha1 proposal. It is the coarse, platform-neutral class of the resource.

The existing `dataProductType` (`table | vectorDb | stream | featureStore`) is kept as a refinement, meaningful for `type: dataProduct` and for Databricks resources. Where both are given they must agree: `dataProductType: vectorDb` corresponds to `type: vectorStore`. Databricks CRs that set neither keep working. Their operator may infer `type` from `accessType` (`vector-search` → `vectorStore`, otherwise `dataProduct`).

## 5. Full spec shape

New fields are marked with ★. All other fields are those of the draft CRD.

```yaml
apiVersion: oda.tmforum.org/v1alpha1
kind: DataResourceConfig
metadata:
  name: <name>
  labels:
    oda.tmforum.org/componentName: <owning component>    # link to the component
spec:
  provider: databricks | odaComponent                     # ★ odaComponent
  role: consumer | provider                               # ★ default consumer
  workspace: {host, authSecretRef}                        # databricks only
  dependentDataResources:                                 # role consumer
    - name: <dns-label>
      id: <catalog id>
      description: <text>
      type: vectorStore | knowledgeGraph | dataProduct | cache   # ★
      dataProductType: table | vectorDb | stream | featureStore
      required: true                                      # ★ default true
      retrieval: {topK, scoreThreshold, hybrid}           # ★ vectorStore only
      access: read | readWrite                            # ★ default read
      providerRef:                                        # ★ odaComponent only
        componentId: <e.g. TMFC060>                       #   one of componentId
        componentName: <release-component name>           #   or componentName
        namespace: <default: components>
        knowledgeId: <existing knowledge-base id>         #   one of knowledgeId
        create:                                           #   or create
          name: <knowledge-base name>
          description: <text>
          storeKind: graph | vector | hybrid
          conformsTo: [<knowledgeSpecification id or IRI>]
          source: [upload | tmf639 | tmf638 | tmf642 | tmf628 | tmf921]
      accessType: sql | vector-search | genie | functions # databricks only
      catalogRef: {catalog, schema, object}
      physicalStorage: {location, format}                 # ADR-007
      index: {catalog, schema, indexName}                 # databricks only
      genieSpaceId: <id>                                  # databricks only
      functions: [{catalog, schema, functionName}]        # databricks only
  providedDataResources:                                  # ★ role provider
    - name: <dns-label>
      type: vectorStore | knowledgeGraph | dataProduct | cache
      description: <text>
      engine: <e.g. qdrant, pyoxigraph>
      exposedAPI: <ExposedAPI name serving REST, e.g. knowledgemanagement>
      mcpAPI: <ExposedAPI name serving MCP, e.g. knowledgemcp>
      retrieval: {topK, scoreThreshold, hybrid}           # defaults for consumers
```

`providedDataResources[]` declares **stores**, not their contents. The knowledge-bases inside a store change at runtime, so they are mirrored into status by the operator rather than declared in the spec.

## 6. Full status shape

The draft status fields are kept; `phase` is still set, so existing tooling and printer columns keep working. New fields are marked with ★.

```yaml
status:
  phase: Pending | Ready | Error | Skipped    # Skipped is deprecated, see section 8
  message: <text>
  resourceCount: <int>
  endpoints: [{name, accessType, mcpUrl, status, dependentApiName}]   # databricks
  observedGeneration: <int>                                           # ★
  conditions:                                                         # ★ Kubernetes conditions
    - {type: Ready | ProviderResolved | KnowledgeBaseReady | AccessGranted,
       status: "True" | "False" | Unknown, reason, message, lastTransitionTime}
  providedResources:                                                  # ★ role provider
    - {kind: knowledge | knowledgeSpecification, id, name, type,
       storeKind, layer, lifecycleStatus, href}
  bindings:                                                           # ★ role consumer
    - {name, providerComponent, kbId, secretName, dependentApiName,
       access, ready}
```

## 7. Provisioning by the Data Resource Management operator

A new Canvas operator, **TMFOP012 Data Resource Management**, handles every DataResourceConfig with `provider: odaComponent`. The Databricks operator keeps handling `provider: databricks`. TMFOP012 authenticates to provider components with Canvas system credentials (`canvasSystemRole`).

**Provider role.** TMFOP012:
1. reads `providedDataResources[]` and resolves the named ExposedAPIs to URLs;
2. lists the provider's knowledge-bases and ontology modules (TMFC060: `GET /knowledge`, `GET /knowledgeSpecification`) and subscribes to its event hub;
3. writes them to `status.providedResources[]`, refreshes them on change events, and sets `Ready`.

**Consumer role.** For each item in `dependentDataResources[]`, TMFOP012:
1. resolves the provider component from `providerRef` (`componentId` or `componentName`) and its provider DataResourceConfig (`ProviderResolved`);
2. uses `knowledgeId`, or creates the knowledge-base from `create{…}` through the provider API (`KnowledgeBaseReady`);
3. **grants access:** it assigns the provider component's client role to the consumer component's identity client in the Canvas identity platform. With TMFC060 that is `knowledge-viewer` for `access: read`, plus `knowledge-editor` for `access: readWrite` (`AccessGranted`);
4. writes a **binding Secret** and a **DependentAPI**, both owned by the DataResourceConfig;
5. records the result in `status.bindings[]` and sets `Ready` once every `required` item is bound. An unbound optional item does not block `Ready`.

**Binding Secret.** Named `<dataresourceconfig>-<item name>-binding`, in the consumer's namespace, with an ownerRef to the DataResourceConfig. Keys:

| Key | Value |
|---|---|
| `apiUrl` | The knowledge-base resource URL: `<provider TMFxxx ExposedAPI URL>/knowledge/{kbId}` |
| `queryUrl` | `<apiUrl>/query` (read-only SPARQL) |
| `mcpUrl` | The provider's MCP ExposedAPI URL |
| `kbId` | The knowledge-base id |
| `providerComponent` | The provider component name |
| `qdrantCollection` | The vector collection name, **for reference only** |

The Secret contains **no store credentials**. The consumer calls the URLs with its own component credentials, and the provider authorises it through the role granted in step 3.

**DependentAPI.** One per binding, `apiType: mcp`, marked as a data-resource dependency the same way as the Databricks operator's DependentAPIs. It names the provider's MCP ExposedAPI, so the dependency is visible and resolved by the Canvas Dependency Management operator like any other dependent API. Its resolved URL is the same as `mcpUrl`.

**Deletion.** Deleting a consumer DataResourceConfig deletes its Secrets and DependentAPIs (through ownerRefs) and revokes the granted roles. The knowledge-base itself is **kept**, because its content may be shared or costly to rebuild.

## 8. Status ownership

An operator writes `status` **only** on DataResourceConfigs whose `spec.provider` it owns. It ignores all others without writing to them. The Databricks operator's `phase: Skipped` behaviour is deprecated: with two operators watching the same resource, writing `Skipped` would overwrite the other operator's status. The `Skipped` value stays in the enum for compatibility.

## 9. Examples

Provider, rendered by the TMFC060 chart:

```yaml
apiVersion: oda.tmforum.org/v1alpha1
kind: DataResourceConfig
metadata:
  name: km-knowledgemanagement
  labels:
    oda.tmforum.org/componentName: km-knowledgemanagement
spec:
  provider: odaComponent
  role: provider
  providedDataResources:
    - name: knowledge-vectors
      type: vectorStore
      engine: qdrant
      exposedAPI: knowledgemanagement
      mcpAPI: knowledgemcp
      retrieval: {topK: 8, scoreThreshold: 0.3, hybrid: true}
    - name: knowledge-graph
      type: knowledgeGraph
      engine: pyoxigraph
      exposedAPI: knowledgemanagement
      mcpAPI: knowledgemcp
```

Consumer, rendered by an agentic component's chart:

```yaml
apiVersion: oda.tmforum.org/v1alpha1
kind: DataResourceConfig
metadata:
  name: assurance-agent-knowledge
  labels:
    oda.tmforum.org/componentName: assurance-agent
spec:
  provider: odaComponent
  role: consumer
  dependentDataResources:
    - name: incident-kb
      type: vectorStore
      access: read
      retrieval: {topK: 5, hybrid: true}
      providerRef:
        componentId: TMFC060
        create:
          name: incident-knowledge
          storeKind: hybrid
          source: [tmf642, tmf628]
```


# Answers to the ADR-007 Open Questions

## Runtime Access Standardization

**Pattern A, mediated access, is the standard for data resources served by ODA Components.** ADR-007 defines Pattern A as MCP tool access. This ADR widens it to any *mediated* interface of the provider component: its MCP ExposedAPI **or** its REST ExposedAPI. Consumers never receive direct store credentials (for example to Qdrant or the graph store), and no SDK path to the stores is offered (Pattern B). Databricks resources keep their current Managed MCP access, which is already Pattern A.

## Gateway Mediation

Binding URLs are the provider's **ExposedAPI URLs** as published by the API Management operator. Traffic therefore follows the Canvas API gateway, and the Interaction Gateway for MCP (ADR-004), like any other component-to-component call. Whether an in-cluster consumer should instead use the Service URL of the ExposedAPI implementation is to be confirmed by measurement in the reference implementation; it does not change this decision.

## Data Discovery Model

Data resources are discoverable through **Resource Inventory**, projected from DataResourceConfig. See ADR-011.

## Data Product Abstraction

**No separate Data Product CRD is introduced.** The `type` enum, with `dataProduct` as one value and `dataProductType` as its refinement, is the abstraction. A dedicated resource can be revisited if data products gain a lifecycle of their own.


# Consequences

## Benefits

One CRD covers external data platforms and component-served data, with platform-neutral types aligned to the Component v2alpha1 proposal.

Existing Databricks DataResourceConfigs stay valid without change.

A consuming component receives a ready-to-use knowledge-base, with endpoints and access rights, from a single declarative resource.

Store credentials never leave the provider component; access is governed by the provider's own roles.

Data dependencies appear as DependentAPIs, so existing Canvas tooling shows them.

## Trade-offs

Conditional CEL rules replace simple `required` lists, which makes the schema harder to read. CEL validation needs Kubernetes 1.25 or later.

Two operators watch one resource type and must respect status ownership.

TMFOP012 needs rights in the identity platform to assign provider client roles to consumer clients. How a Canvas operator obtains a token for this, and for calling the provider API, is to be confirmed against the existing operators (Identity Config, Dependency Management) during implementation.

Kept knowledge-bases may outlive their consumers and need housekeeping.

## Follow-up

- `CRDS/dataresourceconfig-v1alpha1-crd.yaml` is updated to this schema once this ADR is accepted. The ODA Canvas `oda-crds` chart carries the same CRD.
- ADR-011 projects DataResourceConfig into Resource Inventory.
