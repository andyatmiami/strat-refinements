## Strategy
**Components**: odh-dashboard, kubeflow (notebook controller), notebooks, model-metadata-collection, Jupyter extension (new)

### Technical Approach

This strategy delivers a "Customize" entry point in the RHOAI Model Catalog that guides users from model selection directly into a preconfigured workbench with curated, runnable customization notebooks. The core user journey is: **Model Catalog -> Customize button -> preconfigured Workbench -> Jupyter extension surfaces notebook bundles -> user starts customizing**. This addresses the current gap where customization workflows are invisible from the Model Catalog and require users to independently discover notebooks, repositories, and workbench configurations.

The implementation spans four areas:

**1. Dashboard: "Customize" Action in Model Catalog UI.** The odh-dashboard's Model Catalog page (delivered via the gen-ai-ui federated plugin or the core dashboard, depending on where model catalog lives in the modular architecture) gains a "Customize" button alongside existing actions like "Deploy." When clicked, the dashboard creates a Notebook CR with specific annotations indicating: (a) the selected model identifier from the model-metadata-collection catalog, (b) customization intent, and (c) the workbench image to use — a training-ready image or Jupyter Minimal Data Science image. The dashboard frontend reads the model-metadata-collection YAML catalogs (already mounted as a volume from the model-metadata-collection data container) to determine which models support customization and what notebook bundles are available. No intermediate capability-selection pages are shown — the user goes directly to a workbench.

**2. Workbench Provisioning with Model-Aware Configuration.** When the dashboard creates the Notebook CR, it includes: a PVC mount for model weights, datasets, and outputs at a known path (so notebooks can reference model artifacts directly without path discovery); a reference to the appropriate workbench image (Jupyter Minimal Data Science for lightweight exploration or a training-ready image for GPU workloads); and an annotation carrying the selected model's OCI URI from the catalog. Per Paul McCarthy's architectural guidance from the RFE comments, the workbench itself does NOT need GPU attached — GPUs are consumed by training jobs (TrainJob, PyTorchJob) launched from within the notebook, not by the workbench pod. This means the Customize flow can provision a lightweight CPU workbench quickly and defer GPU allocation to the actual training execution.

**3. Jupyter Extension for Notebook Discovery and Materialization.** A new Jupyter extension (JupyterLab server extension + frontend extension) runs inside the workbench. On activation, it reads the model identifier from the Notebook CR's annotations (injected as environment variables by the notebook controller webhook) and queries a notebook bundle registry to discover curated notebooks relevant to the selected model and customization intent. The extension pulls versioned notebook bundles from a Git repository and clones them into the user's workspace. Notebook bundles cover: instruction fine-tuning (OSFT with Kubeflow Trainer), PEFT/LoRA fine-tuning, embedding fine-tuning for RAG, prompt tuning, evaluation workflows (including LLM-as-judge), and synthetic data generation. The extension handles version management so users always get the latest product-endorsed notebooks.

**4. Model Metadata Extension for Customization Support.** The model-metadata-collection component's catalog schema needs extension to include customization metadata per model: which notebook bundles are compatible, which customization workflows are supported, and minimum resource requirements. This metadata is consumed by both the dashboard (to show/hide the Customize button per model) and the Jupyter extension (to filter available notebook bundles).

The approach deliberately avoids building a step-by-step wizard or configuration UI — the notebook itself serves as the guided happy path, matching the RFE's UX principles of "example-first guidance over abstract configuration."

### Affected Components

| Component | Change | Owner Team |
|-----------|--------|------------|
| odh-dashboard (gen-ai-ui or core) | Add "Customize" button to Model Catalog, implement Notebook CR creation with customization annotations, read customization metadata from model catalogs | Dashboard Team |
| kubeflow (notebook controller) | Extend webhook to inject model-related environment variables from Notebook CR annotations into workbench containers; mount PVC for model weights at standardized path | Notebooks Team |
| notebooks (workbench images) | Include the new Jupyter extension in Jupyter-based workbench images (Minimal Data Science, PyTorch, potentially others) | Notebooks Team |
| model-metadata-collection | Extend catalog schema with customization metadata (supported workflows, compatible notebook bundles, resource requirements per model) | Model Catalog Team |
| Jupyter extension (new) | New JupyterLab extension for notebook bundle discovery, Git-based fetching, and materialization into user workspace | Notebooks Team / New Team |
| Notebook bundle repository (new) | Curated repository of versioned notebook bundles for various customization workflows, organized by model family and task | Training / AI Engineering Team |
| trainer (ClusterTrainingRuntimes) | Ensure ClusterTrainingRuntime definitions are compatible with notebooks launched from the Customize flow; may need additional runtimes | Training Team |

### Impacted Teams

| Team | Components Owned | Involvement |
|------|-----------------|-------------|
| Dashboard Team | odh-dashboard, gen-ai-ui plugin | Implement Customize button, Notebook CR creation flow, UI/UX for model catalog customization entry point |
| Notebooks Team | kubeflow (notebook controller), notebooks (workbench images) | Webhook annotation injection, workbench image updates to include Jupyter extension, PVC provisioning |
| Model Catalog Team | model-metadata-collection | Extend catalog schema, curate customization metadata per model family |
| Training Team | trainer, distributed-workloads | Ensure ClusterTrainingRuntimes work with Customize-launched notebooks, contribute notebook bundle content |
| AI Engineering / Content Team | Notebook bundles (new) | Author and maintain curated notebook bundles for fine-tuning, evaluation, SDG workflows |
| UXD | N/A | Design review for Customize button placement, workbench launch flow, Jupyter extension UI |
| QE | N/A | End-to-end testing of Customize flow from model selection through notebook execution |

### High Level Requirements

- [P0] Add a "Customize" button to the Model Catalog UI that is visible for models with customization support metadata
- [P0] Clicking "Customize" creates a preconfigured Notebook CR with model context (model ID, OCI URI, workbench image) and launches the workbench without intermediate pages
- [P0] Workbench is provisioned with a PVC at a known path for model weights, datasets, and outputs
- [P0] Jupyter extension discovers and lists notebook bundles compatible with the selected model
- [P0] Jupyter extension can clone/materialize a selected notebook bundle from Git into the user workspace
- [P1] Notebook bundles are versioned and the extension pulls the latest compatible version
- [P1] Support notebook bundles for at minimum: instruction fine-tuning (OSFT), PEFT/LoRA, and evaluation workflows
- [P1] Model metadata collection catalogs include customization support metadata per model
- [P1] Model weights are made available at a known standardized path inside the workbench so notebooks can reference them directly
- [P2] Support notebook bundles for embedding fine-tuning (RAG), prompt tuning, and synthetic data generation (SDG)
- [P2] Jupyter extension shows resource availability on cluster (GPU types, counts) so users can make informed decisions about training job configuration
- [P2] Support for re-entering a previously customized workbench with the same model context

### Dependencies

| Dependency | Type | Status | Impact if Blocked |
|------------|------|--------|-------------------|
| model-metadata-collection catalog schema | Internal | Needs extension | Cannot determine which models support customization or which notebook bundles to offer |
| Kubeflow Notebook Controller webhook | Internal | Exists | New annotation injection logic needed; without it, Jupyter extension cannot discover model context |
| ClusterTrainingRuntimes | Internal | Exists (15 runtimes for PyTorch) | Fine-tuning notebooks need compatible runtimes; existing runtimes likely sufficient for initial scope |
| Notebook bundle Git repository | Internal | Does not exist | No curated notebooks to materialize; must be created and maintained |
| Jupyter extension framework | External | Exists (JupyterLab 4.x extension API) | Extension development depends on stable JupyterLab extension APIs |
| Gateway API ingress for workbenches | Internal | Exists | Workbench access requires functioning HTTPRoute creation by notebook controller |
| PVC provisioning for model weights | Internal | Partially exists | Dashboard already supports PVC creation for workbenches; model weight path standardization is new |
| RHAIRFE-749 (Improving In-Workbench Access to Example Notebooks) | Internal | Unknown | Related work that may overlap or complement this feature; status needs clarification |

### Non-Functional Requirements

- **Performance**: Workbench creation from "Customize" click to JupyterLab ready must complete within the same timeframe as standard workbench creation (current baseline needs measurement — flagged in Open Questions). Jupyter extension notebook bundle listing must render within 3 seconds of JupyterLab becoming interactive.
- **Security**: Jupyter extension must authenticate to the notebook bundle Git repository using the platform's existing credential patterns (ServiceAccount token or user-provided Git credentials). Extension must not expose cluster-level information beyond what the user's RBAC permits. Notebook bundle content must be sourced from product-endorsed repositories only.
- **Backwards Compatibility**: Models without customization metadata in the catalog must continue to display normally without the "Customize" button. Existing workbench creation flows must be unaffected. The Jupyter extension must be non-disruptive in workbenches launched outside the Customize flow.

### Out-of-Scope

- Introducing new Workbench onboarding flows or setup wizards beyond the Customize entry point
- Replacing advanced, fully manual customization workflows — users who prefer direct notebook management retain their existing paths
- Abstracting notebooks away entirely — notebooks remain the primary guidance surface
- Building a step-by-step configuration wizard in the dashboard UI
- Publishing customized/fine-tuned models back to the Red Hat Model Catalog (per Brian Gallagher's comment, push-to-catalog capability does not yet exist on the platform)
- GPU attachment to the workbench pod itself — GPUs are consumed by training jobs, not workbenches
- Model deployment from within the Customize flow — deployment remains a separate action
- Integration with pipelines (Data Science Pipelines / Kubeflow Pipelines) for automated customization workflows — this is a future phase

### Acceptance Criteria (Proposed -- requires PM/Engineering validation)

- Given a model with customization metadata in the catalog, when a user views the model in the Model Catalog, then a "Customize" button is visible alongside existing actions (Deploy), measured by UI element presence in the model detail view.

- Given a model without customization metadata, when a user views the model in the Model Catalog, then no "Customize" button is displayed, measured by absence of the button element.

- Given a user clicks "Customize" on a supported model, when the action completes, then a Notebook CR is created with annotations containing the model ID, OCI URI, and customization intent, measured by Notebook CR inspection showing expected annotation keys and values.

- Given a workbench launched via "Customize," when JupyterLab becomes interactive, then the Jupyter extension is active and displays a list of compatible notebook bundles for the selected model, measured by extension panel rendering with at least one notebook bundle listed.

- Given a user selects a notebook bundle in the Jupyter extension, when the clone/materialize action completes, then the notebook files are present in the user's workspace directory and are immediately openable, measured by file presence in the JupyterLab file browser.

- Given a workbench launched via "Customize," when inspecting the pod spec, then a PVC is mounted at the standardized model weights path, measured by volume mount inspection showing the expected mount path.

- The Jupyter extension does not cause errors or UI disruption in workbenches launched outside the Customize flow, measured by successful launch and operation of a standard workbench with the extension-enabled image.

### Effort Estimate

**L (Large, 6-9 sprints)** — This feature touches 5+ components across at minimum 4 teams (Dashboard, Notebooks, Model Catalog, Training/Content). The major effort drivers are:
- **New Jupyter extension** (3-4 sprints): Server extension + frontend extension for bundle discovery, Git fetching, and materialization. This is net-new code with its own build, test, and packaging pipeline.
- **Dashboard changes** (2-3 sprints): Model Catalog UI modifications, Notebook CR creation with new annotation schema, reading customization metadata from catalogs.
- **Notebook controller webhook changes** (1-2 sprints): Annotation injection into workbench containers, PVC mount standardization.
- **Model metadata schema extension** (1 sprint): Catalog schema changes, metadata curation for initial set of models.
- **Notebook bundle repository** (2-3 sprints): Content authoring and curation for at least 3-5 customization workflows across supported model families. Ongoing maintenance cost.
- **Cross-team integration and E2E testing** (1-2 sprints): End-to-end flow validation across all components.

The RFE assessment noted architecture review is still pending, and the automationbot flagged that cross-team coordination spans 5+ teams. This is consistent with an L estimate. If the Jupyter extension scope grows or if the notebook bundle content requirement expands significantly, this could approach XL.

### Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Architecture review is still pending (per assessment comments) | Fundamental design decisions may change, invalidating implementation work | Prioritize architecture review before sprint planning; block development start on architecture sign-off |
| Jupyter extension development complexity exceeds estimate | Extension delivery delays the entire Customize flow since it is on the critical path | Prototype the extension early (sprint 1) to validate JupyterLab extension API feasibility; define an MVP extension that covers basic bundle listing and cloning |
| Notebook bundle content maintenance burden | Bundles become stale or incompatible with new model versions, undermining the "curated" value proposition | Establish a CI pipeline that validates notebook bundles against current workbench images and model catalog entries weekly; assign clear content ownership |
| Model metadata collection schema changes break existing dashboard consumers | Dashboard model catalog display regresses | Schema extension must be additive (new optional fields only); validate backward compatibility with existing dashboard catalog rendering before merging |
| Cross-team coordination across 5+ teams creates scheduling conflicts | Feature delivery slips due to team availability misalignment | Identify a single feature lead to coordinate across teams; establish weekly sync during active development; define clear interface contracts early |
| RHAIRFE-749 (In-Workbench Notebook Access) overlaps with this feature | Duplicate or conflicting implementations for notebook discovery within workbenches | Investigate RHAIRFE-749 status and scope; align designs if both are active; if 749 delivers a general notebook discovery mechanism, the Customize flow's Jupyter extension should build on it rather than duplicate |

### Assumptions

- The model-metadata-collection catalog schema can be extended with optional fields without breaking existing consumers — **needs validation**
- JupyterLab 4.x extension APIs are stable enough for a production-grade extension with Git integration — **needs validation**
- A single curated notebook bundle Git repository is sufficient for the initial release (vs. multiple per-workflow-type repositories) — **needs validation**
- Existing ClusterTrainingRuntimes (15 PyTorch variants) are sufficient for fine-tuning notebooks launched from the Customize flow — **needs validation**
- The ODH Notebook Controller webhook can be extended to inject custom environment variables from Notebook CR annotations without upstream Kubeflow changes — **confirmed** (webhook already injects sidecars, CA bundles, and Elyra config)
- GPU resources are not needed on the workbench pod itself for the Customize flow — **confirmed** (per Paul McCarthy's architectural guidance, training GPUs are consumed by TrainJob/PyTorchJob, not the workbench)
- The dashboard already has access to model catalog data via the model-metadata-collection volume mount — **confirmed**

### Open Questions

| Question | Why It Matters | Owner |
|----------|---------------|-------|
| What is the status and scope of RHAIRFE-749 (Improving In-Workbench Access to Example Notebooks)? | If 749 delivers a general notebook discovery mechanism, this feature should build on it rather than creating a parallel system | Product Management |
| What is the current baseline for workbench creation time (click to JupyterLab ready)? | Needed to set a meaningful performance NFR for the Customize flow | Notebooks Team |
| Which models in the current catalog should have customization support in the initial release? | Determines scope of metadata curation and notebook bundle authoring work | Model Catalog Team / Product Management |
| Where should the notebook bundle Git repository be hosted and who owns content maintenance? | Critical for content freshness and the extension's Git fetching mechanism | AI Engineering / Content Team |
| What is the preferred image for Customize-launched workbenches: Jupyter Minimal Data Science, a new Customize-specific image, or model-dependent? | Affects workbench image build scope and user experience consistency | Notebooks Team / Architecture |
| How should model weights be provisioned at the known PVC path — pre-downloaded, lazy-fetched from OCI, or symlinked? | Impacts workbench startup time and storage requirements significantly | Architecture / Notebooks Team |
| Has architecture review been scheduled, and what are the key questions for the review? | Assessment explicitly flagged "Requires architecture review: Yes" with significant work still needed | Architecture Team |
| Should the Jupyter extension UI be a JupyterLab sidebar panel, a launcher tile, or a modal dialog? | UX decision affects extension development approach and user discoverability | UXD |

### Supporting Documentation

- Source RFE: [RHAIRFE-1115](https://redhat.atlassian.net/browse/RHAIRFE-1115)
- Related RFE: [RHAIRFE-749 — Improving In-Workbench Access to Example Notebooks](https://redhat.atlassian.net/browse/RHAIRFE-749)
- Architecture context: PLATFORM.md, odh-dashboard.md, notebooks.md, kubeflow.md, model-metadata-collection.md, trainer.md (rhoai-3.4-ea.2)
- High level architecture plan: [Google Doc referenced in assessment](https://docs.google.com/document/d/1uH6UQUbvchS9X1lhJ__uR5toRRHKSQ1AAWtLTJB0-Wg/edit?tab=t.2k8shl52b2yu)
- Design doc: _To be created_
- ADR: _To be created if needed_
